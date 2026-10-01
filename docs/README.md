# Autonomous Development-Time QA Platform — Specification Set

**Status:** Draft v0.1 — for architecture review. **No production implementation until approved.**

This folder is the engineering specification for a cloud-hosted, CI/CD-triggered autonomous QA platform. It was derived from the product brief (`autonomous-qa-full-plan.txt`) and deliberately resolves several contradictions and gaps in that brief (see [30-architecture-review.md](./30-architecture-review.md) §"What should change").

## Reading order

| If you are… | Read |
|---|---|
| New to the project | 00 → 01 → 05 → 07 → 26 → 30 |
| Implementing the control plane | 05, 07, 08, 09, 12, 15, 16, 20, 22, 23 |
| Implementing the execution plane | 05, 06, 11, 14, 17, 18, 19, 21 |
| Reviewing security | 05, 17, 19, 22 |
| Planning delivery | 26, 27, 28, 29, 30 |

## Index

| # | Document | Summary |
|---|---|---|
| 00 | [Product Vision](./00-product-vision.md) | Vision, north star, principles, success metrics |
| 01 | [Problem Statement](./01-problem-statement.md) | Why current QA tooling fails; the oracle problem |
| 02 | [Requirements](./02-requirements.md) | Personas, use cases, product requirements |
| 03 | [Functional Requirements](./03-functional-requirements.md) | Numbered FRs with phase tags |
| 04 | [Non-Functional Requirements](./04-non-functional-requirements.md) | NFRs; targets as hypotheses |
| 05 | [System Architecture](./05-system-architecture.md) | Planes, tiers, components, trust boundaries |
| 06 | [Deployment Architecture](./06-deployment-architecture.md) | MVP infra and scale-out path |
| 07 | [Domain Model](./07-domain-model.md) | Normalized entities, lifecycles, state machines |
| 08 | [Project Brain](./08-project-brain.md) | Knowledge store: Application Brain + Intent Brain + Test Memory |
| 09 | [Intent Brain](./09-intent-brain.md) | Intent sources, confidence, decisions |
| 10 | [Application Graph](./10-application-graph.md) | Static + runtime graphs, diffing |
| 11 | [Runtime Exploration](./11-runtime-exploration.md) | State fingerprints, budgets, safety |
| 12 | [Change Impact Analysis](./12-change-impact-analysis.md) | Diff → impact set → test selection |
| 13 | [Test Generation](./13-test-generation.md) | Test categories, generation vs. selection, test DSL |
| 14 | [Playwright Engine](./14-playwright-engine.md) | Structured action DSL, validator, evidence hooks |
| 15 | [AI Agent Architecture](./15-ai-agent-architecture.md) | AI tasks, model tiers, data boundaries, cost |
| 16 | [CI/CD Integration](./16-ci-cd-integration.md) | Git provider abstraction, webhooks, checks |
| 17 | [Cloud Execution](./17-cloud-execution.md) | Queue, workers, sandbox lifecycle |
| 18 | [Evidence & Reporting](./18-evidence-and-reporting.md) | Evidence schema, reports, PR comments |
| 19 | [Security & Safety](./19-security-and-safety.md) | Tenant isolation, secrets, action policy |
| 20 | [Test Memory](./20-test-memory.md) | History, baselines, flakiness, feedback |
| 21 | [Test Data Management](./21-test-data-management.md) | Accounts, seeds, reset, mocks |
| 22 | [Data Model](./22-data-model.md) | Physical schema, indexes, retention |
| 23 | [API Contract](./23-api-contract.md) | High-level REST + webhook endpoints |
| 24 | [High-Level DFDs](./24-high-level-dfd.md) | Context, Level 1, Level 2 DFDs |
| 25 | [Sequence Diagrams](./25-sequence-diagrams.md) | 15 key interaction sequences |
| 26 | [MVP](./26-mvp.md) | Scope, non-goals, acceptance criteria |
| 27 | [Roadmap](./27-roadmap.md) | Revised phases with exit criteria |
| 28 | [Risks & Challenges](./28-risks-and-challenges.md) | 30 challenges analysed |
| 29 | [Open Questions](./29-open-questions.md) | Questions with recommended defaults |
| 30 | [Architecture Review](./30-architecture-review.md) | Solid / risky / missing / critical decisions |
| 31 | [Tech Stack](./31-tech-stack.md) | Proposed MVP stack per layer; resolves stack ⚑ items |

## Conventions

- **Requirement IDs:** `FR-<area>-<n>` (functional), `NFR-<area>-<n>` (non-functional), `SEC-<n>` (security). Each carries a phase tag: `MVP`, `P2`, `P3`, … (see [27-roadmap.md](./27-roadmap.md)).
- **Diagrams:** Mermaid only. `flowchart` for DFDs and graphs, `sequenceDiagram` for interactions, `stateDiagram-v2` for lifecycles, `erDiagram` for data.
- **Numbers:** No SLA or performance number is stated as fact. Targets are labelled **Hypothesis** and must be validated with Phase 1 measurements.
- **Decisions needing approval** are marked **⚑ DECISION** and collected in [30-architecture-review.md](./30-architecture-review.md).

## Glossary

| Term | Definition |
|---|---|
| **Project** | A tenant-owned unit of configuration: one or more repositories that together form one testable product. |
| **Project Brain** | The persistent, per-project knowledge store. It has three parts: Application Brain, Intent Brain, Test Memory. |
| **Application Brain** | Knowledge of how the application *actually* works: static graph, runtime graph, discovered flows, environment knowledge. |
| **Intent Brain** | Knowledge of how the application *should* work: requirements, acceptance criteria, contracts, decisions, PR intent — each with source and confidence. |
| **Test Memory** | Historical record: test runs, results, baselines, flakiness, developer verdicts. |
| **Application Graph** | Versioned, typed graph of application elements (routes, components, APIs, services, models) and their edges. Has a *static* and a *runtime* layer. |
| **Static Graph** | Graph layer derived from source code and configuration. "What the code says is connected." |
| **Runtime Graph** | Graph layer derived from observed execution. "What actually happened." |
| **Flow** | A named user journey: an ordered sequence of UI states and transitions (e.g. *Booking*). |
| **State Fingerprint** | A stable hash identifying a UI state from URL pattern + DOM/a11y structure + visible landmarks; used for SPA state dedupe. |
| **Snapshot** | An immutable version of the Application Graph tied to a commit SHA. |
| **Baseline** | The last *accepted* observation of a test/flow on a reference branch; what "changed" is measured against. |
| **Observation** | A recorded fact from execution (network call, DOM state, transition, assertion outcome). |
| **Behavior Change** | A detected difference between a current Observation and its Baseline. Not a bug by default. |
| **Intent Claim** | A single statement of intended behavior with source, scope, and confidence. |
| **Decision** | A developer-confirmed Intent Claim created while resolving a Behavior Change (e.g. "flow order changed intentionally"). |
| **Verdict** | The platform's classification of a test outcome or Behavior Change: `PASS`, `EXPECTED_CHANGE`, `REGRESSION_CANDIDATE`, `UNKNOWN`, `FLAKY`, `INFRA_FAILURE`, `THIRD_PARTY_FAILURE`, `TEST_DEFECT`. |
| **Resolution** | A developer's answer to a verdict: `INTENTIONAL`, `REQUIREMENT_CHANGED`, `CONFIRMED_REGRESSION`, `FALSE_POSITIVE`, `IGNORE`. |
| **Test Definition** | A platform-owned, versioned test expressed in the Action DSL. |
| **Action DSL** | JSON structured actions/assertions (e.g. `click {role, name}`) that the Execution Engine validates and compiles to Playwright calls. |
| **Run** | One execution of a Test Plan against one commit in one Sandbox. |
| **Test Plan** | The selected/generated set of Test Definitions and exploration tasks for a Run, with reasons. |
| **Sandbox** | The ephemeral isolated environment (microVM) where the application and browser run. |
| **Control Plane** | Trusted cloud services: API, orchestration, Project Brain, AI orchestration, reporting. Never executes repository code. |
| **Analysis Tier** | Sandboxed workers that clone and statically parse repositories. No secrets, no egress; emits structured facts only. |
| **Execution Plane** | Sandboxed workers that build and run the application plus Playwright. |
| **Evidence** | Artifacts supporting a conclusion: screenshots, traces, HAR, console logs, DOM snapshots, video. |
