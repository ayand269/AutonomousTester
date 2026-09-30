# 26 — MVP Definition

**Purpose:** Define exactly what ships in the MVP, what does not, the target customer profile, and acceptance criteria.
**Related:** [27-roadmap](./27-roadmap.md), [03-functional-requirements](./03-functional-requirements.md) (all `MVP`-tagged FRs), [30-architecture-review](./30-architecture-review.md)

## 1. MVP thesis

> For TypeScript web teams on GitHub, every PR gets an **impact-aware, evidence-backed** E2E verdict from an isolated cloud run of their app — and behavior changes are classified honestly (expected / regression candidate / unknown) instead of just "failed".

The MVP is judged on **trust** (low false-positive rate) and **relevance** (changed flows are tested), not on autonomy.

## 2. Supported project profile (MVP)

| Dimension | Supported | Not supported (MVP) |
|---|---|---|
| Git provider | GitHub (M1), GitLab.com (M4) | Bitbucket, Azure DevOps, self-managed GitLab |
| Language | TypeScript / JavaScript | Others (graph degrades to file-level + runtime-only) |
| Frontend | Next.js (app & pages router), React SPA (Vite/CRA) with React Router | Angular, Vue, Svelte (runtime-only exploration still works) |
| Backend | Express, NestJS, Next.js route handlers | Others |
| Datastores | PostgreSQL, MongoDB, MySQL, Redis (container images) | Managed-only cloud DBs, proprietary services |
| Startup | docker-compose and/or package scripts in a monorepo or single repo | Kubernetes manifests, multi-repo stacks, non-Linux requirements |
| Existing tests | Playwright | Cypress (P3), Selenium |
| Auth | Form login, seeded users, storageState, API token, TOTP test accounts | OAuth via external IdP without test IdP (P3) |
| External services | Sandbox endpoints, mocks, block | record/replay (P3) |
| Target | Ephemeral sandbox only | Staging/preview URLs (P4), production (never in MVP) |

## 3. In scope (capabilities)

1. **Connect & onboard:** GitHub App, repo selection, env auto-detect + confirm, secrets, test accounts, triggers.
2. **Isolated cloud runner:** microVM (or pooled VM fallback) per run; build, seed, health; egress control; evidence upload; destroy.
3. **Run existing Playwright tests** with platform evidence overlay.
4. **Static analysis → Application Graph** (TS/JS: routes, endpoints, components, services, models, tests, external services).
5. **Runtime layer:** runtime coverage from test runs + **basic heuristic exploration** (onboarding and targeted), UI→API linkage via network.
6. **Change impact → Smart mode** with explanations, risk level, safety valves, weekly shadow full run.
7. **Derived deterministic tests:** flow replay from discovered/accepted flows, auth matrix (unauth + role), OpenAPI contract checks.
8. **Baselines + behavior-change detection** (flow sequence, HTTP status, API shape, UI structure, console errors, auth behavior, timing-info).
9. **Noise classification:** infra, third-party (egress-evidence), test defect, flaky (retry + history), quarantine.
10. **Minimal Intent Brain:** PR/commit extraction, existing-test assertions, OpenAPI, implicit platform claims, manual claims, developer Decisions.
11. **Verdict engine** with unified taxonomy, confidence, rule trail, calibration.
12. **AI:** gateway, intent extraction, failure diagnosis, free-text intent judgement (capped), finding grouping, flow naming, NL manual request mapping, env-detection assist. Offline eval suite.
13. **Reporting:** check run + sticky PR comment, dashboard report, evidence viewer, resolution UX, comment commands.
14. **Admin & safety:** roles, budgets, AI data policy, audit log, retention, deletion.

## 4. Explicitly NOT in MVP (scope protection)

| Excluded | Why | When |
|---|---|---|
| AI-generated tests (beyond deterministic derivation) | Highest false-positive risk; needs mature intent + evals | P3 |
| Self-healing | Needs stable test history; brief also defers | P4 |
| Visual regression | Noisy; needs masking UX | P4 |
| Backend tracing (OpenTelemetry injection) | Instrumentation per framework; network linkage suffices initially | P4 |
| Testing external URLs / staging / production | Safety + ownership verification | P4 (staging), never prod by default |
| CI pipeline step / CLI | Webhook path covers MVP | P4 |
| Tickets, requirement docs, Figma ingestion | Integration breadth; confirm value of minimal intent first | P3 / P5 |
| Negative/boundary/UI generated suites | Depend on generation | P3 |
| Multi-repo projects | Cross-repo linkage | P3 |
| Non-TS analyzers | Breadth | P5 |
| IDE/local CLI | Cloud first | P5 |
| Kubernetes-based app stacks | Complexity | P5 |
| Automatic code or test modification in repo | Trust | Never automatic; opt-in PRs P4 |
| Unlimited exploration | State explosion | Never; budgets always |
| CAPTCHA bypass | Safety/ethics | Never |

## 5. MVP acceptance criteria

Measured on (a) **reference apps** maintained by the team with a fault-injection harness and (b) **≥5 design-partner repos**.

| # | Criterion | Target (hypothesis → confirm at M2) |
|---|---|---|
| A1 | Env auto-detect + ≤ 15 min of user edits gets the app healthy | ≥ 4 of 5 design-partner repos |
| A2 | Existing Playwright suites run with results matching the team's own CI (same pass/fail) | ≥ 95% of tests agree |
| A3 | Smart mode selection recall on fault-injected changes (fault in code covered by at least one test) | ≥ 90% |
| A4 | Smart mode runs fewer tests than Full on typical PRs | median ≤ 40% of suite |
| A5 | False-positive rate of `REGRESSION_CANDIDATE` (resolved) | ≤ 20% during design-partner period, trending down |
| A6 | Intentional-change fixtures (with PR description) classified `EXPECTED_CHANGE` or `UNKNOWN`, never `REGRESSION_CANDIDATE` at high confidence | 100% of fixtures |
| A7 | Third-party outage fixtures (egress-simulated) classified `THIRD_PARTY_FAILURE` | ≥ 95% |
| A8 | Every non-PASS verdict has a complete evidence bundle | ≥ 99% |
| A9 | No secret value appears in any artifact, log, or AI input (automated canary-secret scan) | 0 occurrences |
| A10 | Cross-tenant access tests pass; sandbox escape pen-test finds no critical issues | Pass |
| A11 | Time-to-verdict for Smart PR run on reference app | Measured and published; target set at M2 |
| A12 | Developers can resolve a finding from the PR in ≤ 2 clicks | UX test |

## 6. MVP success signal (business)
Design partners keep the check enabled for ≥ 4 weeks and resolve ≥ 50% of `UNKNOWN` prompts (evidence that prompts are worth answering).
