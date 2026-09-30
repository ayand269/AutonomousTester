# 02 — Product Requirements

**Purpose:** Personas, jobs-to-be-done, core use cases, and product-level requirements. Detailed functional requirements are in [03](./03-functional-requirements.md); non-functional in [04](./04-non-functional-requirements.md).
**Related:** [00-product-vision](./00-product-vision.md), [26-mvp](./26-mvp.md)

## 1. Personas

| Persona | Goals | Pain today | What they need from us |
|---|---|---|---|
| **Developer (primary)** | Merge confidently, fast | E2E suites slow/flaky; unclear what their change broke | Fast, relevant PR feedback with evidence; few, precise questions |
| **Tech lead / reviewer** | Know PR risk before approving | Reviews miss cross-module impact | Impact summary + risk level on the PR |
| **QA engineer / SDET** | Coverage of what matters; less maintenance | Writing and fixing brittle tests | Imported tests + generated proposals + flaky management; control over intent |
| **Engineering manager** | Fewer escaped regressions; cost control | No visibility of quality trends | Dashboards: catch rate, flakiness, cost |
| **Org admin / security** | Safe use of source and secrets | Vendor risk | Isolation guarantees, audit, AI data policy, retention controls |

## 2. Jobs to be done

1. *When I open a PR*, tell me which user flows my change could affect, test those, and tell me — with evidence — if something regressed.
2. *When behavior changed on purpose*, don't flag it as a bug; ask me once and remember.
3. *When a test fails*, tell me whether it's my code, the test, the environment, a third party, or flakiness.
4. *When I say "test the booking flow"*, do it without me writing a test.
5. *When I onboard a repo*, get it running in the cloud with minimal configuration.
6. *As an admin*, ensure our code and secrets don't leak and no destructive actions happen.

## 3. Core use cases

| UC | Name | Primary actor | Trigger | Outcome |
|---|---|---|---|---|
| UC-1 | Onboard project | Developer/admin | Sign up | Repo connected, env confirmed, first Brain snapshot, existing tests imported, first run green/understood |
| UC-2 | PR smart run | Developer | PR opened/updated | Check + comment with impact, verdicts, evidence |
| UC-3 | Resolve behavior change | Developer | UNKNOWN verdict | Decision stored; future runs use it |
| UC-4 | Diagnose failure | Developer | REGRESSION_CANDIDATE | Evidence-backed analysis; resolution recorded |
| UC-5 | Main-branch regression | System | Push to main | Full-critical run; baselines updated |
| UC-6 | Nightly deep run | System | Schedule | Full suite + exploration + selection-recall shadow |
| UC-7 | Manual targeted test | Developer | NL request or scope pick | Plan built & run for scope |
| UC-8 | Manage intent | QA/lead | Dashboard | Claims confirmed/added; decisions reviewed |
| UC-9 | Manage tests | QA | Dashboard | Proposals accepted/rejected; flaky quarantined |
| UC-10 | Configure safety & secrets | Admin | Dashboard | Secrets, test accounts, action policy, egress, AI data policy |
| UC-11 | Review self-heal proposal (P4) | QA/Dev | Locator broke | Proposal accepted → new test version |

## 4. Product requirements (high level)

| ID | Requirement | Phase |
|---|---|---|
| PR-1 | Connect GitHub (and GitLab) repositories with least-privilege access | MVP |
| PR-2 | Configure triggers per event/branch with a run mode | MVP |
| PR-3 | Auto-detect how to build/run the app; user confirms/edits | MVP |
| PR-4 | Run each triggered execution in an isolated ephemeral cloud sandbox at the exact commit | MVP |
| PR-5 | Run the project's existing Playwright tests with rich evidence | MVP |
| PR-6 | Build and maintain a Project Brain (Application Brain, Intent Brain, Test Memory) that persists across runs | MVP |
| PR-7 | Analyze diffs and select impacted tests (Smart mode), with explanations | MVP |
| PR-8 | Detect behavior changes against baselines and classify them with the unified verdict taxonomy | MVP |
| PR-9 | Ask the developer when intent is unknown; store decisions | MVP |
| PR-10 | Separate infra, third-party, flaky, and test-defect outcomes from regressions | MVP |
| PR-11 | Provide AI-assisted failure diagnosis that cites evidence and states confidence | MVP |
| PR-12 | Report via PR/MR checks + comments and a dashboard | MVP |
| PR-13 | Basic runtime exploration to discover flows and link UI to APIs | MVP |
| PR-14 | Derive deterministic tests (flow replay, auth matrix, API contract) for impacted, uncovered areas | MVP |
| PR-15 | Enforce safety: no production, egress control, action policy, secret protection | MVP |
| PR-16 | Budgets and cost controls for compute and AI | MVP |
| PR-17 | AI-generated tests as reviewable proposals | P3 |
| PR-18 | Rich intent sources (tickets, requirement docs) | P3 |
| PR-19 | Self-healing proposals (never silent) | P4 |
| PR-20 | Visual regression with AI explanation | P4 |
| PR-21 | CI pipeline step / external preview URL mode / CLI | P4 |
| PR-22 | IDE/local developer experience (cloud remains source of truth) | P5 |

## 5. Trigger strategy (product defaults)

See [16-ci-cd-integration](./16-ci-cd-integration.md) §4. Summary: PR → Smart; push to main → Full-critical; nightly → Full + exploration; manual → user scope. Develop/other branches off by default.

## 6. Constraints & assumptions

- **A1.** Target apps are web apps whose full stack can run in a Linux sandbox via docker-compose and/or Node package scripts. (Validate in Phase 0 with ≥5 real repos.)
- **A2.** Teams use PR/MR workflows and are willing to install a GitHub App.
- **A3.** Teams can provide test accounts or seed scripts.
- **A4.** Many teams have little written intent; the Intent Brain grows through confirmations.
- **A5.** Customers accept AI processing of code snippets under a no-training provider agreement (policy is configurable to disable).
- **C1.** No testing of production in MVP.
- **C2.** No modification of user code or tests in MVP.
