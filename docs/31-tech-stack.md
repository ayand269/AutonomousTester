# 31 — Technology Stack

**Purpose:** Pick a concrete technology stack for the MVP, settle the stack-related ⚑ items left open in [05](./05-system-architecture.md) §7 and [29](./29-open-questions.md), and record where this doc departs from earlier recommendations.
**Related:** [05-system-architecture](./05-system-architecture.md), [06-deployment-architecture](./06-deployment-architecture.md), [17-cloud-execution](./17-cloud-execution.md), [22-data-model](./22-data-model.md), [29-open-questions](./29-open-questions.md), [30-architecture-review](./30-architecture-review.md)

**Status:** Proposed. Approval goes through [30](./30-architecture-review.md) §6 (D7, D8, D9, D15, D16).

## 1. Selection drivers

| Driver | Stack consequence |
|---|---|
| MVP targets TypeScript web apps and runs Playwright | One language (TypeScript) across analyzers, engine, control plane, and dashboard |
| Zone 2/3 output is untrusted ([05](./05-system-architecture.md) §2) | One schema system that validates at every boundary and also gives JSON Schema to the AI Gateway |
| Run pipeline needs timeouts, retries, heartbeats, supersession ([17](./17-cloud-execution.md) §1) | A durable workflow engine instead of hand-built state on a job queue |
| Tenant isolation ([04](./04-non-functional-requirements.md) NFR-TEN, Q-SEC-1) | Postgres RLS and a data-access layer that sets tenant context on every transaction |
| Untrusted multi-container apps ([17](./17-cloud-execution.md) §3) | VM boundary per run; cloud with bare-metal instances |
| Team of ~6–8 engineers ([27](./27-roadmap.md) §4) | Managed services, a modular monolith, minimal infra to operate |

## 2. Stack by layer

### 2.1 Languages and repository

| Concern | Choice | Notes |
|---|---|---|
| Primary language | **TypeScript on Node.js 22 LTS** | Same ecosystem as Playwright, the TS compiler API, and MVP target apps |
| Exception | **Go** for the execution-host microVM manager only | Firecracker tooling (`firecracker-go-sdk`) is Go/Rust. Keep this component small |
| Monorepo | **pnpm workspaces + Turborepo** | Shared packages: `domain`, `schemas` (DSL, Action, facts, verdicts), `ai-gateway`, `analyzers/*` |
| Schemas | **Zod**, exported to JSON Schema | One source of truth for API contracts ([23](./23-api-contract.md)), Zone 2 fact validation, the DSL/Action types ([14](./14-playwright-engine.md)), and `AIGateway.run(schema)` structured output |
| Unit/integration tests | **Vitest** | |
| Platform E2E | **Playwright** | Also used by the dogfooding reference apps ([06](./06-deployment-architecture.md) §3) |

### 2.2 Zone 1 — Control Plane

| Concern | Choice | Notes |
|---|---|---|
| Service shape | **Modular monolith**: one codebase, several process roles (`api`, `webhook`, `orchestrator`, `worker`) | Module boundaries follow [05](./05-system-architecture.md) §3 so roles can split into services later without a rewrite |
| HTTP framework | **Fastify** | Schema-first, fast, simpler than NestJS for a BFF |
| Git provider | **Octokit** (`@octokit/app`, `@octokit/webhooks`) behind the `GitProvider` interface | GitLab adapter at M4 |
| Run orchestration | ⚑ **Temporal (Temporal Cloud)**. See §3.1 | Run state machine ([07](./07-domain-model.md) §4.1) as a Temporal workflow; stages as activities |
| Background jobs | **BullMQ on ElastiCache Redis** | Only for simple, non-critical jobs: comment coalescing, notifications, brain-maintenance fan-out |
| Primary DB | **PostgreSQL 17 (Amazon RDS)**, RLS enabled | Settles Q-DATA-5 and Q-SEC-1. JSONB for `attrs`; graph tables partitioned by project ([22](./22-data-model.md)) |
| DB access | **Kysely** (Drizzle acceptable), **not Prisma** | RLS needs `SET LOCAL app.org_id` per transaction; recursive CTEs and partitioned tables need SQL-level control |
| Migrations | Kysely migrations (or `node-pg-migrate`), forward-only | Matches [06](./06-deployment-architecture.md) §5 |
| Graph traversal | In-memory adjacency maps loaded per project | No graph database in MVP ([05](./05-system-architecture.md) §7) |
| Secrets | **AWS KMS** envelope encryption, per-project data keys, write-only vault API | Settles Q-SEC-3 |
| Audit log | Postgres append-only table + periodic export to S3 Object Lock (WORM) | [22](./22-data-model.md) |
| Auth | GitHub OAuth via **Better Auth** (or Auth.js); **WorkOS** when enterprise SSO is required | |

### 2.3 Zone 2 — Analysis Tier

| Concern | Choice | Notes |
|---|---|---|
| Sandbox | ⚑ **ECS Fargate tasks in a dedicated account (B)**. See §3.2 | One task per analysis job; egress limited to the Git provider via security group + egress proxy |
| Parsing | **TypeScript compiler API** (no-emit, no plugins, no `tsconfig` plugin loading) + **`web-tree-sitter`** | Never runs `npm install`, build scripts, or config files that execute code |
| OpenAPI | `@apidevtools/swagger-parser` with remote `$ref` resolution **disabled** | |
| ORM models | Prisma schema via an AST parser (e.g. `@mrleebo/prisma-ast`); Mongoose/TypeORM via TS AST | Never invoke the Prisma CLI or import model files |
| Output | Facts validated against Zod schemas in Zone 1, size-capped | [05](./05-system-architecture.md) §2 |

### 2.4 Zone 3 — Execution Plane

| Concern | Choice | Notes |
|---|---|---|
| Sandbox provider | ⚑ **M1: one EC2 VM per run from a warm pool. M2+: Firecracker microVMs on `*.metal` hosts** | Settles Q-EXE-1 by sequencing. Both behind a `SandboxProvider` interface. The switch is gated on spike S1 and on startup-time measurements ([04](./04-non-functional-requirements.md) NFR-PERF) |
| In-VM runtime | **containerd + Docker Compose** for the app stack | |
| Sandbox Agent / QA Execution Engine | TypeScript (Node.js) | Engine uses the repo's own `@playwright/test` version for imported specs ([14](./14-playwright-engine.md) §6) |
| Browser | **Chromium only** | Q-EXE-2 |
| Egress control | **Envoy** (SNI-based allow-list, per-call access logs labeled by host) | Default deny ([19](./19-security-and-safety.md) §3) |
| Mocks | **WireMock** reached via per-sandbox DNS override; per-sandbox CA injected only when an HTTPS host is mocked | Email sink: **Mailpit** |
| Dependency cache | Per-org cache keyed by lockfile hash; never shared across orgs | NFR-TEN-2 |

### 2.5 AI

| Concern | Choice | Notes |
|---|---|---|
| Gateway | **In-house thin gateway** over provider SDKs (`@anthropic-ai/sdk` + a second provider's SDK) | Budgets, redaction, caching, task-tier routing, and schema validation are product logic and should be owned in-house. No LangChain or agent frameworks |
| Models | ⚑ Large/Medium tier: **Claude Opus 5.5** (diagnosis, intent judgement). Small tier: **Claude Haiku 4.5** (classification, flow naming, finding grouping). A second provider is integrated from day one for failover | Settles Q-AI-1 pending Legal (D9) |
| Evaluation | Offline eval suite runs in CI against fixtures from the reference apps and fault-injection harness | [15](./15-ai-agent-architecture.md), D13 |

### 2.6 Dashboard

| Concern | Choice | Notes |
|---|---|---|
| Framework | **Next.js (App Router)** as BFF + UI | [05](./05-system-architecture.md) §7 |
| UI | **Tailwind CSS + shadcn/ui**, **TanStack Query** | |
| Evidence viewer | Self-hosted **Playwright Trace Viewer** embedded in the dashboard, loading traces through presigned URLs | [18](./18-evidence-and-reporting.md) |

### 2.7 Infrastructure and operations

| Concern | Choice | Notes |
|---|---|---|
| Cloud | ⚑ **AWS**, single region, three accounts under AWS Organizations (A: control, B: analysis, C: execution) | Settles Q-EXE-3 / D7. Has metal instances for Firecracker plus managed Postgres/Redis/KMS/S3. GCP is the alternative (nested virtualization on standard VMs) |
| Control-plane hosting | **ECS Fargate** + ALB + AWS WAF | |
| Object storage | **S3**, per-org prefixes, presigned single-object URLs | Revisit Cloudflare R2 if evidence-download egress cost becomes significant |
| Infrastructure as code | **Terraform**; host images built with **Packer** | [06](./06-deployment-architecture.md) §1 |
| Observability | **OpenTelemetry** → Grafana Cloud (or Honeycomb); **Sentry** for errors | NFR-OBS |
| Local dev | docker-compose (Postgres, Redis, Temporal dev server, MinIO, WireMock); execution via local Docker for trusted code only | [06](./06-deployment-architecture.md) §3 |
| CI | GitHub Actions | |

## 3. Departures from earlier recommendations

### 3.1 Temporal instead of BullMQ for Run orchestration (D15)

[05](./05-system-architecture.md) §7 recommends BullMQ for the job pipeline. The Run pipeline in [17](./17-cloud-execution.md) §1 needs per-stage timeouts, retries, idempotency per `(runId, stage, attempt)`, 2-minute heartbeats with forced teardown, supersession that cancels in-flight work, and priority/fair scheduling. Temporal provides most of these directly (activity timeouts, retry policies, heartbeats, cancellation scopes, durable state). With BullMQ, the team would have to build and maintain them.

**Trust boundary is unchanged:** only Zone 1 runs Temporal workers. Zone 2/3 workers still pull work through the narrow mTLS job API ([06](./06-deployment-architecture.md) §2). A Zone 1 activity hands the job to that API and completes asynchronously when the worker reports back, so the job API's heartbeats map onto activity heartbeats.

**Fallback if not approved:** BullMQ as in [05](./05-system-architecture.md) §7, with the Run state machine stored in Postgres as the source of truth and a reaper process enforcing timeouts and heartbeats.

### 3.2 Fargate instead of self-managed gVisor for the Analysis Tier (D16)

[05](./05-system-architecture.md) §7 recommends gVisor-sandboxed containers. Fargate runs each task inside its own VM-level isolation boundary, which is at least as strong as gVisor and needs no sandbox runtime to operate. The Zone 2 rules (no code execution, Git-only egress, no secrets) stay the same.

**Fallback if not approved:** gVisor (`runsc`) on self-managed ECS/EC2 hosts in account B.

## 4. Explicitly not used in MVP

| Technology | Reason |
|---|---|
| Neo4j / Neptune | Per-project graph fits in memory ([10](./10-application-graph.md) §8) |
| Kafka | Temporal + BullMQ cover the pipeline; no streaming requirement |
| Kubernetes | Orchestration, not isolation; high ops cost for team size ([17](./17-cloud-execution.md) §3) |
| OpenSearch | Postgres full-text is enough until search demand is proven |
| MongoDB | Relational core, transactions, and RLS favour Postgres (D8) |
| Plain containers for Zone 3 | Insufficient isolation for untrusted multi-container apps |
| Prisma | Poor fit for per-transaction RLS context |
| LangChain / agent frameworks | Conflict with deterministic-first, schema-constrained AI design ([15](./15-ai-agent-architecture.md)) |

## 5. Validation (Phase 0 spikes)

| Choice | Validated by |
|---|---|
| EC2 warm pool vs Firecracker, startup time | Spike S1 |
| In-memory graph size for symbol-level graphs | Spike S2 (Q-ANA-2) |
| Temporal ↔ job API async-completion pattern | Prototype during S1: one Run end-to-end with a forced heartbeat loss and a supersession |
| Kysely + RLS | Cross-tenant access test suite (MVP acceptance A10) run in CI from M1 |
