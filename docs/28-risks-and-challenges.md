# 28 — Risks & Challenges

**Purpose:** Analyse the 30 major challenges from the brief. For each: problem · why · options · recommendation · MVP approach · future · tradeoffs.
**Related:** [30-architecture-review](./30-architecture-review.md), [27-roadmap](./27-roadmap.md)

## Risk heat map (top risks)

| Rank | Challenge | Likelihood | Impact | Why it's top |
|---|---|---|---|---|
| 1 | #21 Cloud application startup | High | Critical | If the app doesn't start, nothing else matters; onboarding killer |
| 2 | #10 False positives | High | Critical | Trust is the product |
| 3 | #1/#2 Intent / oracle | High | High | Determines whether verdicts are meaningful |
| 4 | #18/#29 Security & tenant isolation | Medium | Critical | Running untrusted code for many tenants |
| 5 | #9 Flaky tests | High | High | Largest source of noise |
| 6 | #13 Backend understanding | Medium | High | Impact accuracy depends on it |
| 7 | #16 AI cost | Medium | High | Unit economics |

---

### 1. Understanding intended behavior
- **Problem:** The platform must know what "correct" means for behavior it observes.
- **Why:** Intent lives in people's heads, tickets, and PR text; written specs are rare and stale.
- **Options:** require formal specs; infer from code; infer from existing tests; extract from PR/tickets; ask developers.
- **Recommended:** multi-source Intent Brain with explicit source + confidence, and developer confirmation as the highest-trust source ([09](./09-intent-brain.md)).
- **MVP:** PR/commit extraction, existing tests, OpenAPI, implicit platform claims, manual claims, Decisions.
- **Future:** tickets, requirement docs, DB constraints, security configs, design.
- **Tradeoffs:** more `UNKNOWN`s early (asks) vs. risk of wrong confident verdicts. We choose asking.

### 2. Oracle problem
- **Problem:** No automatic oracle can tell intended change from regression.
- **Why:** Code shows *what* changed, not *whether it should*.
- **Options:** differential testing vs. baseline; property/implicit oracles (no 5xx); spec-based; human-in-the-loop.
- **Recommended:** three-input comparator (Observation, Baseline, Intent) + implicit oracles for hard failures + human resolution for the rest.
- **MVP:** as recommended; low-confidence regressions displayed as `UNKNOWN`.
- **Future:** calibrated per-project models from resolution history.
- **Tradeoffs:** some real regressions will be `UNKNOWN` rather than flagged; acceptable for trust.

### 3. Dynamic applications
- **Problem:** SPAs, client-side routing, lazy loading, real-time updates, dynamic IDs.
- **Why:** URL is not state; DOM mutates continuously.
- **Options:** URL-based states; DOM hashing; a11y-structure fingerprints; visual hashing.
- **Recommended:** a11y + interactive-element fingerprints with near-duplicate merge; network/DOM quiet waits ([11](./11-runtime-exploration.md) §3).
- **MVP:** as recommended, tunable thresholds.
- **Future:** framework-aware state hooks (React DevTools protocol) for richer state identity.
- **Tradeoffs:** coarse fingerprints merge distinct states; fine ones explode state count.

### 4. State explosion
- **Problem:** Number of reachable states grows combinatorially.
- **Why:** Forms × data × roles × navigation orders.
- **Options:** exhaustive BFS; random walk; prioritized frontier; model-based.
- **Recommended:** prioritized frontier with dedupe, list-item templating, per-pattern visit caps, impact-driven targeting.
- **MVP:** heuristic priority; hard budgets.
- **Future:** AI goal-directed ranking; learned priorities from where bugs were found.
- **Tradeoffs:** incomplete coverage by design; mitigated by impact targeting.

### 5. Infinite exploration
- **Problem:** Loops (pagination, calendars, infinite scroll) never terminate.
- **Why:** Unbounded generators of "new" states.
- **Options:** depth limits; visit caps; time/action budgets; loop detection.
- **Recommended:** all of the above as hard, engine-enforced limits.
- **MVP:** hard budgets per mode; per-urlPattern cap.
- **Future:** smarter loop detection (sequence similarity).
- **Tradeoffs:** may stop before valuable deep states; nightly deep runs compensate.

### 6. Authentication
- **Problem:** Tests need to log in as various roles.
- **Why:** Diverse auth (forms, OAuth, SSO, magic links, 2FA).
- **Options:** form login; storageState; API tokens; seeded users; mock IdP.
- **Recommended:** TestAccount abstraction with pluggable login strategies; seeded users preferred in ephemeral sandboxes ([21](./21-test-data-management.md) §2).
- **MVP:** form, seeded, storageState, API token, cookie, TOTP.
- **Future:** mock OIDC IdP; magic-link via email sink.
- **Tradeoffs:** storageState expires (maintenance); seeded users require seed scripts.

### 7. CAPTCHA / 2FA
- **Problem:** Blocks automated login.
- **Why:** Designed to stop bots.
- **Options:** bypass (unacceptable); test keys; feature flags; TOTP for test accounts.
- **Recommended:** never bypass; support legitimate test mechanisms; clear `TEST_DEFECT: auth blocked` guidance.
- **MVP:** as recommended.
- **Future:** guided setup wizard per CAPTCHA/auth provider.
- **Tradeoffs:** some apps need config changes to be testable.

### 8. Third-party integrations
- **Problem:** Payments, maps, OAuth, email, analytics, supplier APIs.
- **Why:** Real calls are unsafe/flaky/costly; outages look like regressions.
- **Options:** real sandbox endpoints; mocks; record/replay; block.
- **Recommended:** default-deny egress + per-host mode; attribution from egress logs ([21](./21-test-data-management.md) §6).
- **MVP:** sandbox/mock/block/allow + email sink.
- **Future:** record/replay; status-page correlation.
- **Tradeoffs:** mocks drift from real APIs; sandbox endpoints are flaky.

### 9. Flaky tests
- **Problem:** Non-deterministic outcomes.
- **Why:** Timing, animations, network, shared data, randomness, environment.
- **Options:** retries; quarantine; determinism controls; statistical scoring.
- **Recommended:** determinism controls (clock, motion, waits) + retry-once disagreement + history-based score + quarantine ([20](./20-test-memory.md) §7).
- **MVP:** as recommended.
- **Future:** root-cause hints, order-dependence detection.
- **Tradeoffs:** retries add time and can mask real intermittent bugs (flagged separately as FLAKY, not hidden).

### 10. False positives
- **Problem:** Wrongly reported regressions destroy trust.
- **Why:** Noise, missing intent, overconfident AI.
- **Options:** strict evidence requirements; conservative thresholds; human confirmation; calibration.
- **Recommended:** noise-first classification, deterministic verdict rules, AI cannot set verdicts, low-confidence → UNKNOWN, per-project calibration, FP rate as headline metric.
- **MVP:** all of the above.
- **Future:** learned per-project classifiers.
- **Tradeoffs:** more UNKNOWN prompts; mitigate with batching and good grouping.

### 11. Self-healing reliability
- **Problem:** Auto-repaired locators may target the wrong element and hide regressions.
- **Why:** Similar elements; renamed feature vs. removed feature.
- **Options:** silent heal; proposal only; verify-then-propose.
- **Recommended:** verify-by-rerun then propose; never silently modify ([25](./25-sequence-diagrams.md) §13).
- **MVP:** none (P4).
- **Future:** P4 proposals; auto-accept only for trivial text changes with user opt-in.
- **Tradeoffs:** manual review load.

### 12. Visual testing
- **Problem:** Pixel diffs are noisy; semantics matter.
- **Why:** Fonts, antialiasing, dynamic content, responsive layouts.
- **Options:** pixel diff; perceptual diff; DOM-layout diff; AI vision.
- **Recommended:** deterministic perceptual diff with masks → AI explains diffs, doesn't detect them.
- **MVP:** none (P4); UI structure dimension covers missing elements.
- **Future:** P4.
- **Tradeoffs:** maintenance of masks; storage cost.

### 13. Backend understanding
- **Problem:** Mapping endpoints to handlers, services, and data.
- **Why:** DI, dynamic dispatch, middleware, multiple frameworks.
- **Options:** static analysis only; runtime tracing; hybrid.
- **Recommended:** static analysis with framework plugins + network-level runtime linkage; OTel tracing later.
- **MVP:** TS/Express/NestJS/Next static + network linkage.
- **Future:** OTel auto-instrumentation in sandbox; more languages.
- **Tradeoffs:** static call graph imprecision → edge confidence and conservative impact thresholds.

### 14. Runtime instrumentation
- **Problem:** Seeing inside the app during tests.
- **Why:** Browser sees only network; backend is opaque.
- **Options:** none; logs parsing; OTel auto-instrumentation; custom agents.
- **Recommended:** browser-level capture + service logs in MVP; OTel injection (Node `--require` auto-instrumentation) in P4.
- **MVP:** browser + service logs.
- **Future:** spans for handles/calls/reads/writes edges.
- **Tradeoffs:** instrumentation may alter behavior/perf; opt-out per service.

### 15. Browser scaling
- **Problem:** Many concurrent browsers are CPU/memory heavy.
- **Why:** Chromium per context; video/trace recording.
- **Options:** in-sandbox browsers; separate browser pools; managed browser grids.
- **Recommended:** browser inside the run's sandbox (simplest, isolated); split later.
- **MVP:** in-sandbox, resource classes.
- **Future:** browser microVM separate from app stack; sharding.
- **Tradeoffs:** larger sandboxes vs. network hop complexity.

### 16. AI cost
- **Problem:** Token spend can exceed value.
- **Why:** Large contexts, per-action calls, retries.
- **Options:** per-action agents; selective calls; caching; small models.
- **Recommended:** selective decision points, tiered models, deterministic pre-summarization, caching, budgets ([15](./15-ai-agent-architecture.md) §6).
- **MVP:** as recommended; cost per run visible.
- **Future:** fine-tuned small models for extraction/classification.
- **Tradeoffs:** less AI "magic" in exchange for predictability.

### 17. Test data management
- **Problem:** Tests interfere via shared data; non-deterministic data.
- **Why:** Writes, unique constraints, time-dependent data.
- **Options:** reset; namespace; per-shard sandboxes.
- **Recommended:** fresh datastores per run, reset between tests by default, namespace for parallel ([21](./21-test-data-management.md) §3).
- **MVP:** reset + namespace; seeded generators.
- **Future:** fixtures format; smarter snapshots.
- **Tradeoffs:** reset costs time; namespace requires app cooperation.

### 18. Security
- **Problem:** Hostile repos, secret leakage, token misuse.
- **Why:** We run arbitrary code with customer secrets.
- **Options:** containers; microVMs; separate accounts; strict egress.
- **Recommended:** three trust zones, microVM per run, separate cloud accounts, vault + run-scoped secrets, redaction ([19](./19-security-and-safety.md)).
- **MVP:** SEC-1…22 MVP items.
- **Future:** SOC 2, self-hosted runners.
- **Tradeoffs:** operational complexity and startup latency.

### 19. Production safety
- **Problem:** Accidental destructive/real-world actions.
- **Why:** Autonomous exploration clicks things.
- **Options:** trust AI; policies; environment restrictions.
- **Recommended:** no production; egress control; ActionPolicy; destructive-pattern exclusion; test accounts only.
- **MVP:** ephemeral sandbox only.
- **Future:** staging with ownership verification; never production by default.
- **Tradeoffs:** can't catch environment-specific production issues (out of scope).

### 20. Keeping Project Brain synchronized
- **Problem:** Brain drifts from code.
- **Why:** Incremental analysis errors; runtime knowledge from branches; renames.
- **Options:** full re-analysis every time; incremental; hybrid.
- **Recommended:** incremental per commit + nightly full + drift metric; PR overlays; runtime promotion only from default branch ([08](./08-project-brain.md) §4).
- **MVP:** as recommended.
- **Future:** smarter rename tracking.
- **Tradeoffs:** compute cost of nightly full runs.

### 21. Cloud application startup
- **Problem:** Getting arbitrary repos to build and run.
- **Why:** Undocumented env vars, services, seeds, OS deps, private registries.
- **Options:** require Docker Compose; auto-detect; user config; customer-provided images.
- **Recommended:** auto-detect → user confirm → validation run with step-level diagnostics; compose preferred; private registry credentials as secrets.
- **MVP:** Phase-0 spike S1 first; supported project profile limited ([26](./26-mvp.md) §2).
- **Future:** preview-URL mode (skip build), customer images, K8s manifests.
- **Tradeoffs:** narrower initial market for higher success rate.

### 22. Different project architectures
- **Problem:** SPA vs SSR, monolith vs services, REST vs GraphQL.
- **Why:** Heterogeneity.
- **Options:** support all at once; plugin architecture; runtime-only fallback.
- **Recommended:** analyzer plugins + runtime-only graceful degradation when unsupported.
- **MVP:** TS stack plugins; runtime-only elsewhere (impact degraded to "affected application").
- **Future:** more plugins, GraphQL operations as endpoint nodes.
- **Tradeoffs:** unsupported stacks get weaker Smart mode.

### 23. Monorepos
- **Problem:** Many apps/packages; shared libs cause broad impact.
- **Why:** Workspaces, path aliases, build orchestration (turbo/nx).
- **Options:** treat as one app; per-workspace Applications.
- **Recommended:** detect workspaces → Applications; resolve path aliases; broad-impact guard for shared libs.
- **MVP:** pnpm/yarn/npm workspaces, turbo, nx detection.
- **Future:** use turbo/nx project graphs directly.
- **Tradeoffs:** shared-lib changes escalate to broad runs (cost).

### 24. Microservices
- **Problem:** Many services, possibly in many repos, with async messaging.
- **Why:** Distributed call paths not visible statically.
- **Options:** compose all; stub some; trace.
- **Recommended:** support compose-able multi-service in one repo in MVP; multi-repo and async (queues) later with tracing.
- **MVP:** single-repo compose.
- **Future:** multi-repo projects, message-edge types, OTel.
- **Tradeoffs:** large stacks increase sandbox size/cost.

### 25. Database initialization
- **Problem:** Schema, migrations, seeds must be correct and fast.
- **Why:** Missing seeds, migration ordering, large dumps.
- **Options:** migrations + seed scripts; dumps; fixtures.
- **Recommended:** detect ORM migrate/seed commands; datastore snapshot after seed for fast reset.
- **MVP:** Prisma/TypeORM/Mongoose patterns + custom hooks.
- **Future:** platform fixtures; anonymized dump import (customer-provided, never prod by default).
- **Tradeoffs:** seed quality determines exploration usefulness.

### 26. External service dependencies (infrastructure)
- **Problem:** Apps depend on cloud services (S3, SQS, Firebase) not just SaaS APIs.
- **Why:** Cloud SDKs, managed services.
- **Options:** LocalStack-class emulators; mocks; block.
- **Recommended:** emulator images (LocalStack, Firebase emulator, MinIO) as datastore-like services in env profile.
- **MVP:** MinIO/LocalStack as optional compose services where the app supports endpoint override.
- **Future:** curated emulator catalog.
- **Tradeoffs:** emulator fidelity.

### 27. Long-running test jobs
- **Problem:** Nightly/full runs exceed practical limits.
- **Why:** Large suites, exploration.
- **Options:** longer timeouts; sharding; checkpointing.
- **Recommended:** budgets, sharding (P2), per-test checkpoints, TTLs ([17](./17-cloud-execution.md) §7).
- **MVP:** budgets + checkpoints; no sharding.
- **Future:** sharding, prioritization across nights.
- **Tradeoffs:** incomplete nightly coverage if budget too low.

### 28. Artifact storage costs
- **Problem:** Traces/videos are large.
- **Why:** Every run produces MBs–GBs.
- **Options:** keep all; sample; tiered retention; compress.
- **Recommended:** retention classes; sample pass traces; video opt-in; content-addressed dedupe.
- **MVP:** as recommended; per-run artifact bytes metered.
- **Future:** cold tiers; per-plan quotas.
- **Tradeoffs:** less evidence for old passing runs.

### 29. Tenant isolation
- **Problem:** Data or compute leakage between customers.
- **Why:** Shared infrastructure.
- **Options:** logical isolation; RLS; per-tenant keys; dedicated infra.
- **Recommended:** org scoping + RLS + per-org keys + no shared sandboxes/caches; dedicated pools for enterprise.
- **MVP:** as recommended + automated tenancy tests.
- **Future:** cells, self-hosted runners.
- **Tradeoffs:** cost of no cross-tenant caching.

### 30. CI/CD reliability
- **Problem:** Missed webhooks, rate limits, provider outages, status spam.
- **Why:** External systems.
- **Options:** webhooks only; polling; hybrid.
- **Recommended:** webhooks + periodic reconciliation, idempotency, rate-limited publishing, supersession.
- **MVP:** as recommended.
- **Future:** CI step mode as alternate trigger path.
- **Tradeoffs:** reconciliation adds API usage.
