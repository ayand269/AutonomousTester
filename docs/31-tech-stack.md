# 31 — Technology Stack

**Purpose:** Pick a concrete technology stack that fits a **$50/month budget** during development and the early design-partner phase, define the funded scale-up target, and record where the early stack departs from earlier recommendations.
**Related:** [05-system-architecture](./05-system-architecture.md), [06-deployment-architecture](./06-deployment-architecture.md), [17-cloud-execution](./17-cloud-execution.md), [19-security-and-safety](./19-security-and-safety.md), [22-data-model](./22-data-model.md), [29-open-questions](./29-open-questions.md), [30-architecture-review](./30-architecture-review.md)

**Status:** Proposed. Approval goes through [30](./30-architecture-review.md) §6 (D7, D9, D15, D16).

## 1. Budget constraint and drivers

**Hard constraint:** total platform spend ≤ **$50/month** (infrastructure + AI) until there are paying users. Spend scales up only when usage or revenue justifies it (§6).

| Driver | Stack consequence |
|---|---|
| $50/month cap | No always-on sandbox hosts, no managed DB/Redis/NAT/load balancer, no paid SaaS tiers. Pay per run for sandboxes; hard caps on every variable cost |
| Untrusted customer code ([05](./05-system-architecture.md) §2) | Still never executed **or parsed** on the control-plane server; runs in a fresh isolated VM per run, rented per second |
| Product promise: isolated cloud run with zero CI setup ([26](./26-mvp.md) §1) | The platform runs the sandbox, not the customer's CI |
| MVP targets TypeScript web apps and runs Playwright | One language (TypeScript) across analyzers, engine, control plane, and dashboard |
| Zone 2/3 output is untrusted | One schema system that validates every boundary and also gives JSON Schema to the AI Gateway |
| Cheap now, scalable later | Every infrastructure choice sits behind an interface (`SandboxProvider`, `JobQueue`, `SecretVault`, `ArtifactStore`, `AIGateway`) so moving to the scale-up stack (§5) swaps implementations, not code |

## 2. Stages

| Stage | When | Where things run | Budget |
|---|---|---|---|
| **0 — Development** | Building the platform; only the team's own reference apps | Everything on the developer's machine via docker-compose; sandbox = local Docker (trusted code only, [06](./06-deployment-architecture.md) §3) | ~$0–25 (AI calls only; most tests use recorded AI responses) |
| **1 — Early phase** | First design partners connect real repos | One VPS for the control plane; per-run rented microVMs for analysis and execution (§3) | **≤ $50** |
| **2 — Scale** | Paying users; a trigger in §6 fires | Funded AWS topology from [06](./06-deployment-architecture.md) (§5) | Grows with usage |

## 3. Early-stage stack (Stages 0–1)

### 3.1 Languages and repository

| Concern | Choice | Notes |
|---|---|---|
| Primary language | **TypeScript on Node.js 22 LTS** | Same ecosystem as Playwright, the TS compiler API, and MVP target apps |
| Monorepo | **pnpm workspaces + Turborepo** | Shared packages: `domain`, `schemas` (DSL, Action, facts, verdicts), `ai-gateway`, `analyzers/*`, `sandbox-providers/*` |
| Schemas | **Zod**, exported to JSON Schema | One source of truth for API contracts ([23](./23-api-contract.md)), Zone 2 fact validation, the DSL/Action types ([14](./14-playwright-engine.md)), and `AIGateway.run(schema)` structured output |
| Tests | **Vitest**; **Playwright** for platform E2E and reference apps | AI calls in tests replay recorded responses; CI never calls the AI API |
| CI | GitHub Actions (free tier) | |

### 3.2 Zone 1 — Control Plane (one VPS)

| Concern | Choice | Notes |
|---|---|---|
| Hosting | ⚑ **One Hetzner Cloud VPS** (≈4 vCPU / 8 GB) running Docker Compose, **Caddy** for TLS | ~$8/month. Runs no customer code and parses no customer code |
| Service shape | **Modular monolith**: one codebase, process roles `api`, `webhook`, `worker`, `web` | Module boundaries follow [05](./05-system-architecture.md) §3 so roles can split later |
| HTTP framework | **Fastify** | Schema-first, lightweight |
| Git provider | **Octokit** (`@octokit/app`, `@octokit/webhooks`) behind `GitProvider` | GitHub App is free |
| Database | **PostgreSQL 17 in a container on the VPS**, RLS enabled | Nightly `pg_dump` (encrypted) to R2; restore tested monthly. Data access via **Kysely** (RLS needs `SET LOCAL app.org_id` per transaction) |
| Job queue + Run orchestration | ⚑ **pg-boss** (Postgres-backed queue) behind a `JobQueue` interface | Replaces both Redis/BullMQ and Temporal. Run state machine ([07](./07-domain-model.md) §4.1) lives in Postgres as the source of truth; a reaper enforces stage timeouts, heartbeats, and supersession ([17](./17-cloud-execution.md) §1) |
| Secrets | ⚑ **libsodium envelope encryption** behind a `SecretVault` interface: per-project data keys, wrapped by a master key held only in the VPS environment (not in the DB or backups) | Replaces cloud KMS until Stage 2. Write-only API and audit log unchanged ([19](./19-security-and-safety.md)) |
| Graph traversal | In-memory adjacency maps loaded per project | No graph database ([05](./05-system-architecture.md) §7) |
| Auth | **Better Auth** + GitHub OAuth | Self-hosted, free |

### 3.3 Zone 2 — Analysis Tier

| Concern | Choice | Notes |
|---|---|---|
| Sandbox | ⚑ **Short-lived rented microVM per analysis job** (same provider as §3.4), **no secrets injected** | Preserves the rule that the control plane never parses untrusted source. Costs ~cents per job. Egress limited to the Git provider where the provider allows it |
| Parsing | **TypeScript compiler API** (no-emit, no plugins) + **`web-tree-sitter`** | Never runs `npm install`, build scripts, or config files that execute code |
| OpenAPI | `@apidevtools/swagger-parser` with remote `$ref` resolution disabled | |
| ORM models | Prisma schema via an AST parser (e.g. `@mrleebo/prisma-ast`); Mongoose/TypeORM via TS AST | Never import model files or run the Prisma CLI |
| Output | Facts validated against Zod schemas on the VPS, size-capped | [05](./05-system-architecture.md) §2 |

### 3.4 Zone 3 — Execution Plane

| Concern | Choice | Notes |
|---|---|---|
| Sandbox provider | ⚑ **Per-run rented microVM, billed per second**: **Fly Machines** first candidate (Firecracker), **E2B** or **Modal** sandboxes as alternatives. Behind `SandboxProvider` | Fresh VM per run, destroyed after; never reused ([17](./17-cloud-execution.md) §3). Settles Q-EXE-1 for the early stage. Choice made by spike S1 (§9) |
| Per-run size | ~4 GB RAM, 2–4 vCPU; larger only per `resourceClass` | Estimated ~$0.02–0.05 per 15–20 min run; verify current provider pricing |
| Stage 0 provider | **Local Docker** (`LocalDockerProvider`) | Reference apps only; never customer code |
| In-VM runtime | **Docker Compose** for the app stack | Must work inside the provider's VM (spike S1) |
| Sandbox Agent / QA Execution Engine | TypeScript (Node.js) | Uses the repo's own `@playwright/test` version for imported specs ([14](./14-playwright-engine.md) §6) |
| Browser | **Chromium only** | Q-EXE-2 |
| Egress control | Provider-level outbound restrictions where available; otherwise an in-VM proxy (**Envoy** or a small Node proxy) with logging | An in-VM proxy can be bypassed by the app, so it labels traffic for third-party attribution but is **not** a security boundary. Accepted limitation for design partners; enforced default-deny returns at Stage 2 |
| Mocks / email | **WireMock** and **Mailpit** containers inside the VM | |
| Secrets delivery | Provider's per-machine secret/env injection at VM start, decrypted from `SecretVault` for that run only | Bundle scoped to the run ([17](./17-cloud-execution.md) §2) |

### 3.5 AI

| Concern | Choice | Notes |
|---|---|---|
| Gateway | **In-house thin gateway** over provider SDKs (`@anthropic-ai/sdk` + a second provider's SDK) | Budgets, redaction, caching, task-tier routing, schema validation. No LangChain or agent frameworks |
| Models | ⚑ **Claude Haiku 4.5** for intent extraction, classification, flow naming, finding grouping. **Claude Opus 5.5** only for failure diagnosis and free-text intent judgement, at `low`/`medium` effort | Settles Q-AI-1 for the early stage, pending Legal (D9) |
| Cost controls | Hard monthly cap in the gateway; spend limit set in the Anthropic Console; prompt caching for stable system prompts and schemas; **Batch API** (50% off) for non-urgent work (flow naming, grouping, nightly brain maintenance) | When the cap is hit, the gateway returns "AI unavailable" and verdicts fall back to deterministic-only ([05](./05-system-architecture.md) §6 degraded mode) |
| Evaluation | Offline eval suite on recorded fixtures from the reference apps and fault-injection harness | [15](./15-ai-agent-architecture.md), D13 |

### 3.6 Dashboard and operations

| Concern | Choice | Notes |
|---|---|---|
| Dashboard | **Next.js (App Router)** on the same VPS; **Tailwind CSS + shadcn/ui**, **TanStack Query** | |
| Evidence viewer | Self-hosted **Playwright Trace Viewer**, loading traces via presigned URLs | [18](./18-evidence-and-reporting.md) |
| Artifacts | ⚑ **Cloudflare R2** (S3-compatible) behind `ArtifactStore`; per-org prefixes, presigned single-object URLs | Free tier (10 GB) and no egress fees for evidence viewing. Retention defaults from [19](./19-security-and-safety.md) §6 keep usage inside the free tier |
| Observability | **Sentry** free tier (errors); **Grafana Cloud** free tier or structured logs (OpenTelemetry SDK kept in code) | |
| Infrastructure as code | Docker Compose file + a provisioning script for the VPS; Terraform introduced at Stage 2 | |

## 4. Monthly budget and caps (Stage 1)

| Item | Monthly | Enforcement |
|---|---|---|
| Hetzner VPS | ~$8 | Fixed |
| Per-run sandboxes (analysis + execution) | ≤ $15 → roughly 300–600 runs | Monthly cap in the orchestrator; per-org run quotas; provider spending limit where offered |
| AI | ≤ $20 | Gateway cap + Anthropic Console spend limit |
| R2, Sentry, Grafana, Better Auth, GitHub App | $0 | Free tiers; alerts at 80% of each free-tier limit |
| Domain | ~$1 | |
| **Total** | **≤ ~$45**, ~$5 headroom | |

When the sandbox cap is reached, new runs are queued with status `budget_exhausted` and the PR check reports "skipped — platform capacity" (neutral), never a failure.

Prices are estimates; verify current Hetzner, sandbox-provider, and Claude pricing before committing.

## 5. Scale-up target (Stage 2)

Adopted piece by piece as triggers in §6 fire, not all at once.

| Layer | Stage 2 choice |
|---|---|
| Cloud | **AWS**, three accounts under AWS Organizations (control, analysis, execution) ([06](./06-deployment-architecture.md) §2) |
| Control-plane hosting | **ECS Fargate** + ALB + AWS WAF |
| Database | **Amazon RDS PostgreSQL** (or Neon as an intermediate step) |
| Orchestration | **Temporal (Temporal Cloud)** for the Run state machine; only Zone 1 runs Temporal workers, Zone 2/3 still pull via the mTLS job API |
| Analysis Tier | **ECS Fargate tasks** in the analysis account |
| Execution | **EC2 VM per run** from a stopped warm pool, then **Firecracker** on `*.metal` hosts when startup time requires it. Firecracker is a latency improvement, not a cost saving, at CI-shaped load |
| Egress | Enforced default-deny **Envoy** egress fleet outside the VM |
| Secrets | **AWS KMS** envelope encryption, per-project data keys |
| Artifacts | Stay on **R2**, or move to S3 |
| AI models | Re-evaluate tier routing (e.g. Opus 5.5 for intent extraction) once evals show quality gains justify the cost |
| Observability / IaC | Paid Grafana Cloud or Honeycomb; **Terraform** + **Packer** |
| Optional | **GitHub Action runner mode** for customers who will not allow hosted execution (matches the P4 CI-step item in [26](./26-mvp.md) §4) |

## 6. Migration path and triggers

| Interface | Stage 1 | Stage 2 | Move when |
|---|---|---|---|
| `SandboxProvider` | Rented microVMs (Fly Machines / E2B / Modal) | EC2 per run → Firecracker | Sandbox spend > ~$300/month, provider limits block a customer, or enforced egress is required by a customer |
| `JobQueue` / orchestration | pg-boss + Postgres state machine | Temporal | Orchestrator bugs around timeouts/supersession recur, or > ~10 concurrent runs |
| Database | Postgres on the VPS | Neon → RDS | Paying customers require managed backups/HA, or DB > ~20 GB |
| Hosting | One VPS | ECS Fargate | VPS CPU > 70% sustained, or an availability commitment to customers |
| `SecretVault` | libsodium + master key on VPS | AWS KMS | Before the first paying customer or any security questionnaire |
| `ArtifactStore` | R2 free tier | R2 paid / S3 | Free-tier limits exceeded |
| AI budget | ≤ $20 cap | Per-org budgets priced into plans | AI cap is hit in two consecutive months |

## 7. Departures from earlier recommendations

| # | Earlier recommendation | Early-stage choice | Reason | Restored at |
|---|---|---|---|---|
| 7.1 | Managed Redis + BullMQ ([05](./05-system-architecture.md) §7) | **pg-boss** on the existing Postgres (D15) | Removes a paid service; Postgres already holds Run state | Temporal at Stage 2 |
| 7.2 | Self-operated Firecracker hosts or pooled cloud VMs ([17](./17-cloud-execution.md) §3) | **Rented per-run microVMs**, billed per second (D15) | No idle hosts; same VM-per-run isolation | EC2/Firecracker at Stage 2 |
| 7.3 | gVisor containers for the Analysis Tier ([05](./05-system-architecture.md) §7) | **Rented microVM, no secrets** (D16) | Same isolation goal without running a sandbox runtime | Fargate at Stage 2 |
| 7.4 | Three cloud accounts ([06](./06-deployment-architecture.md) §1) | **One VPS** for Zone 1; Zones 2–3 at the sandbox provider (D15) | Cost. The VPS never runs or parses customer code, so the Zone 1 boundary still holds | Stage 2 |
| 7.5 | Cloud KMS for secrets ([19](./19-security-and-safety.md)) | **libsodium envelope encryption** with master key outside the DB (D15) | No KMS cost; same envelope pattern, weaker key custody | Before first paying customer |
| 7.6 | Default-deny egress proxy fleet ([19](./19-security-and-safety.md) §3) | **Provider-level restrictions or in-VM labeling proxy** (D15) | No proxy fleet; in-VM proxy is not a security boundary | Stage 2 |

## 8. Explicitly not used in Stages 0–1

| Technology | Reason |
|---|---|
| AWS managed services (RDS, ElastiCache, NAT, ALB, KMS) | Each one alone takes a large share of the $50 budget |
| Temporal, Redis | pg-boss covers the queue; added at Stage 2 |
| Always-on sandbox hosts / warm pools | Idle cost; per-second rental instead |
| Neo4j / Neptune, Kafka, Kubernetes, OpenSearch | Unneeded at this scale ([05](./05-system-architecture.md) §7) |
| MongoDB | Relational core, transactions, and RLS favour Postgres (D8) |
| Prisma | Poor fit for per-transaction RLS context |
| LangChain / agent frameworks | Conflict with deterministic-first, schema-constrained AI design ([15](./15-ai-agent-architecture.md)) |
| Vercel Hobby / other non-commercial free tiers | Terms disallow commercial use once design partners are onboarded |

## 9. Validation (Phase 0 spikes)

| Question | Validated by |
|---|---|
| Which sandbox provider: Docker Compose inside the VM, cold-start time, outbound restriction options, real per-run cost | Spike S1, extended: run 3 reference apps on Fly Machines, then E2B/Modal if needed. Pass = compose works, app healthy in < 5 min, cost < $0.05 per run |
| In-memory graph size for symbol-level graphs on the 8 GB VPS | Spike S2 (Q-ANA-2) |
| pg-boss + Postgres state machine handles timeouts, heartbeat loss, supersession | Prototype one Run end-to-end with a forced heartbeat loss and a supersession |
| Real AI tokens per run vs the $20 cap | Measure on reference apps with the fault-injection harness; adjust tier routing |
| Kysely + RLS tenant isolation | Cross-tenant access test suite (MVP acceptance A10) in CI from M1 |
