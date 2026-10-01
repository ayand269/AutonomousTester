# 30 — Architecture Review

**Purpose:** An honest review of the concept and this specification: what is solid, risky, missing, what should change, what must stay out of MVP, and which decisions need human approval before implementation.
**Related:** all documents; especially [26-mvp](./26-mvp.md), [27-roadmap](./27-roadmap.md), [28-risks-and-challenges](./28-risks-and-challenges.md), [29-open-questions](./29-open-questions.md)

## 1. What is solid

1. **The core thesis is right:** separating *actual* behavior (Application Brain), *intended* behavior (Intent Brain), and *history* (Test Memory), and refusing to equate "changed" with "broken", is the differentiator versus generic AI browser agents.
2. **Deterministic execution with AI as a reasoning layer** (structured actions → validator → Playwright) is implementable, testable, and cost-controllable.
3. **Control plane / execution plane split** with ephemeral sandboxes is the standard, proven pattern for running untrusted code (CI providers do this).
4. **Change-impact over the graph** is well-founded: reverse dependency traversal plus runtime coverage from tests is implementable with today's TS tooling, and path explanations make it auditable.
5. **Evidence-first reporting** with explicit confidence directly addresses the false-positive risk.
6. **Incremental adoption path** (run existing tests first) gives real value at M1 before any AI-heavy feature.
7. **Provider abstraction** (Git, AI, analyzers) is cheap to build up front and avoids lock-in.

## 2. What is risky

| Risk | Why | Mitigation in this spec | Residual |
|---|---|---|---|
| **Getting arbitrary apps running in the cloud** | Largest real-world failure mode; env vars, seeds, services are undocumented | Phase-0 spike S1; narrowed project profile; validation run with step diagnostics | High — may limit addressable market initially |
| **False positives / trust** | Noise sources are many; one bad week loses a team | Noise-first classification, conservative thresholds, UNKNOWN over REGRESSION, non-blocking default, calibration | Medium |
| **Intent sparsity** | Most teams have little written intent | Minimal intent + implicit claims + confirmations | Medium — "UNKNOWN fatigue" possible |
| **Confirmation fatigue** | Too many questions → ignored | Max 3 per PR, grouping, fingerprint carry-over, calibration | Medium |
| **Static analysis accuracy** | Dynamic dispatch, DI, generated clients | Edge confidence, runtime coverage priority, escalation on partial graph, shadow full runs | Medium |
| **Sandbox startup latency** | microVM + build + seed could take many minutes | Caches, warm pools, snapshot restore, overlap stages | Medium — may miss time-to-feedback expectations |
| **Security of multi-tenant untrusted execution** | Any escape is catastrophic | Three zones, microVMs, separate accounts, egress deny, pen test | Low–medium |
| **Scope** | The brief describes a multi-year platform | Vertical-slice roadmap, explicit non-goals | Medium — needs discipline |
| **Baseline dependency** | Behavior-change detection requires green main runs | Explicit Baseline entity; first runs degrade gracefully | Low |
| **AI eval coverage** | Diagnosis/intent quality is hard to measure | Offline eval suite with fault injection gates changes | Medium |

## 3. What is missing (from the brief) — now addressed or flagged

| Gap in brief | Resolution |
|---|---|
| No **Baseline** concept, yet "did behavior change?" is the first question | Added Baseline entity and update rules ([20](./20-test-memory.md) §2) |
| **Noise classification** absent from the decision tree | Noise-first step before intent comparison ([01](./01-problem-statement.md) §3, [20](./20-test-memory.md) §4) |
| **Static analysis trust boundary** undefined (analyzer in control plane contradicts "no untrusted code") | Added sandboxed Analysis Tier ([05](./05-system-architecture.md) §2) |
| **Test artifact format** for generated tests undefined | Platform-owned Action DSL ([13](./13-test-generation.md) §2) |
| **PR vs main knowledge scoping** (does PR runtime data update the Brain?) | PR overlays; promotion only from default branch ([08](./08-project-brain.md) §3) |
| **Confidence model** mentioned but not defined | Weighted source-trust model + thresholds + calibration ([09](./09-intent-brain.md) §4, [20](./20-test-memory.md) §5–6) |
| **Selection recall** measurement (how to know Smart mode doesn't miss things) | Shadow full runs + auto-tuning ([12](./12-change-impact-analysis.md) §5) |
| **Test-defect** verdict (test itself broken) | Added `TEST_DEFECT` |
| **Egress control** as a mechanism for third-party attribution & safety | Egress proxy with per-host modes ([19](./19-security-and-safety.md) §3) |
| **Usability NFRs** (question budget, explanations) | NFR-USE-* ([04](./04-non-functional-requirements.md) §14) |
| **Blocking vs non-blocking checks** | Non-blocking default until proven ([16](./16-ci-cd-integration.md) §6) |
| **Supersession** of runs on new pushes | Added ([17](./17-cloud-execution.md) §1) |
| **Pricing/cost model** | Flagged (Q-BIZ-2) — not designed here |
| **Onboarding UX for env failures** | Validation run with step diagnostics; needs UX design work (not specified in detail) |
| **Evaluation harness** (reference apps + fault injection) | Required for AI and impact tuning; must be built in Phase 0/M1 — it's product infrastructure, not optional |

Still **not** defined in this set (to do before or during M1): dashboard information architecture/UX; billing; support/ops tooling; detailed DSL JSON Schema; analyzer plugin SDK spec; exact PR comment copy.

## 4. What should change (relative to the brief)

1. **Rename and restructure the brains:** Project Brain = Application Brain + Intent Brain + Test Memory.
2. **One verdict taxonomy:** `PASS, EXPECTED_CHANGE, REGRESSION_CANDIDATE, UNKNOWN, FLAKY, INFRA_FAILURE, THIRD_PARTY_FAILURE, TEST_DEFECT`. "Intentional feature" / "requirement changed" are *resolutions*, not verdicts. The platform never says "BUG"; only developers confirm regressions.
3. **Move minimal Intent Brain into MVP** (brief: Phase 6).
4. **Move AI test generation to P3**, after intent and memory exist; MVP uses deterministic derivations.
5. **Vertical-slice roadmap** (M1–M4) instead of component phases.
6. **GitHub first**, GitLab as the last MVP milestone.
7. **Add the Analysis Tier** as a third trust zone.
8. **microVM (not container) isolation** per run.
9. **PostgreSQL over MongoDB** as the primary store (either is workable; the relational core and RLS tip the balance).
10. **Non-blocking checks by default.**
11. **Runtime coverage from existing tests** is the primary test-to-code mapping — cheaper and more accurate than exploration alone.

## 5. What should NOT be built in MVP

Protect the MVP from these, even if they are tempting in demos:

- AI-generated tests (beyond deterministic derivation) and any "agent writes your test suite" feature.
- Self-healing of any kind.
- Visual regression.
- Testing staging/preview/production URLs; CI-step mode; CLI.
- Backend OpenTelemetry instrumentation.
- Tickets / docs / Figma intent ingestion.
- Multi-repo projects, microservice fleets, Kubernetes stacks.
- Languages beyond TS/JS; frameworks beyond the MVP profile (beyond runtime-only fallback).
- Sharding, multi-browser, multi-region.
- IDE extension / local runner.
- Any automatic modification of user code or tests.
- Unbounded exploration or any AI-in-the-loop per browser action.

## 6. Critical decisions requiring human approval

| # | Decision | Recommendation | Consequence if changed | Ref |
|---|---|---|---|---|
| D1 | MVP scope & supported project profile | As [26](./26-mvp.md) §2 | Broader profile → lower onboarding success, later GA | 26 |
| D2 | Roadmap restructure (vertical slices, intent in MVP, generation in P3) | Approve | Keeping brief's order delays first usable product | 27 |
| D3 | Verdict taxonomy & "never say bug" policy | Approve | Affects UX, data model, metrics | 07, 20 |
| D4 | Git providers: GitHub M1, GitLab M4 | Approve | GitLab at M1 adds ~1 integration's worth of work before first loop | 16 |
| D5 | Sandbox isolation: microVM per run (fallback pooled VM) | Approve after spike S1 | Containers-only is not acceptable for multi-tenant untrusted code | 17 |
| D6 | Three trust zones incl. separate cloud accounts | Approve | Merging zones increases blast radius | 05, 19 |
| D7 | Cloud provider | Early stage: Hetzner VPS + rented per-run microVM provider; scale-up: AWS ([31](./31-tech-stack.md) §2, §5) | Affects cost & ops | 06, 31 |
| D8 | Primary database: PostgreSQL (+RLS) | Approve | MongoDB workable; requires app-only tenancy enforcement | 22 |
| D9 | AI providers/models & default AI data policy (`code_snippets`, zero retention) | Decide with Legal | Stricter policy reduces diagnosis quality | 15 |
| D10 | Default check behavior: non-blocking | Approve | Blocking early risks trust if FP rate is high | 16 |
| D11 | Retention defaults | Approve (§6 of 19) | Storage cost vs. evidence availability | 19, 22 |
| D12 | Generated tests stored as platform DSL, not written to repos | Approve | Writing to repos conflicts with "no code modification" | 13 |
| D13 | Investment in evaluation harness (reference apps + fault injection) as Phase 0/M1 work | Approve | Without it, impact/verdict tuning is guesswork | 15, 27 |
| D14 | Design-partner cohort & success criteria | Approve [26](./26-mvp.md) §5–6 | — | 26 |
| D15 | $50/month early-stage stack: one VPS (Postgres, pg-boss, no Redis/Temporal), rented per-run microVMs, libsodium secrets, in-VM egress labeling; AWS topology deferred to Stage 2 triggers | Approve | Funded AWS topology from day one costs ~$3k+/month at MVP volume | 31 §3, §6, §7 |
| D16 | Analysis Tier in a short-lived rented microVM with no secrets (instead of gVisor) | Approve | Parsing on the control-plane VPS would break the Zone 1 rule | 31 §3.3 |

## 7. Recommendation

Approve Phase 0: sign off D1–D16, then run spikes S1–S4. Replace the hypotheses in [04](./04-non-functional-requirements.md) and [17](./17-cloud-execution.md) with measured numbers before M1 starts. **Do not begin production implementation until this review is accepted.**
