# 03 — Functional Requirements

**Purpose:** Numbered, testable functional requirements with phase tags and design references.
**Related:** [02-requirements](./02-requirements.md), [26-mvp](./26-mvp.md), [27-roadmap](./27-roadmap.md)

Tags: `MVP` (milestones M1–M4, see [27-roadmap](./27-roadmap.md)), `P2`–`P5` later phases. "Ref" points to the design doc.

## FR-ONB — Onboarding & configuration
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-ONB-1 | User signs in with GitHub (GitLab at M4) OAuth; an organization is created or joined | MVP | 19 |
| FR-ONB-2 | User creates a project and installs the GitHub App on selected repositories | MVP | 16 |
| FR-ONB-3 | User selects repository and default branch | MVP | 16 |
| FR-ONB-4 | System detects Applications (frontend/backend/admin) and proposes an EnvironmentProfile with per-field provenance | MVP | 17 §4 |
| FR-ONB-5 | User can edit/confirm profile; a validation run shows per-step startup results and logs | MVP | 17 §5 |
| FR-ONB-6 | User configures secrets (write-only) and test accounts per role | MVP | 19, 21 |
| FR-ONB-7 | User configures triggers with defaults pre-filled | MVP | 16 §4 |
| FR-ONB-8 | System runs initial analysis, imports existing Playwright tests, and (optionally) onboarding exploration | MVP | 08, 11 |
| FR-ONB-9 | Repo-file config `.autoqa.yml` overriding non-sensitive dashboard settings | P2 | 16 §4 |

## FR-GIT — Git integration
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-GIT-1 | Verify webhook signatures and dedupe by delivery ID | MVP | 16 §3 |
| FR-GIT-2 | Normalize PR/MR/push events into ChangeSets with all fields listed in 16 §5 | MVP | 16 |
| FR-GIT-3 | Compute diffs as `base...head` for PRs; handle force-push | MVP | 12 §1 |
| FR-GIT-4 | Cancel superseded runs on new pushes to the same PR (configurable) | MVP | 17 §1 |
| FR-GIT-5 | Publish check/status lifecycle and one sticky comment per PR | MVP | 16 §6 |
| FR-GIT-6 | Accept authorized comment commands (`/qa rerun`, `/qa intentional`, `/qa false-positive`, `/qa full`) | MVP | 16 §6 |
| FR-GIT-7 | Periodic reconciliation for missed webhooks | MVP | 16 §3 |
| FR-GIT-8 | GitLab provider (MR hooks, notes, statuses) | MVP (M4) | 16 |
| FR-GIT-9 | CI step / CLI trigger with optional external preview URL | P4 | 16 §7 |

## FR-ANA — Static analysis & Application Brain
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-ANA-1 | Analysis runs in the sandboxed Analysis Tier; only schema-validated facts reach the control plane | MVP | 05 §2 |
| FR-ANA-2 | Extract TS/JS files, symbols, imports, calls (best effort) | MVP | 10 §3 |
| FR-ANA-3 | Extract UI routes (Next.js, React Router), API endpoints (Express, NestJS, Next route handlers), OpenAPI | MVP | 10 §3 |
| FR-ANA-4 | Extract DB models (Prisma, Mongoose, TypeORM) | MVP | 10 §3 |
| FR-ANA-5 | Discover existing tests (Playwright; Cypress P3) and link statically to routes/modules | MVP | 10 §4 |
| FR-ANA-6 | Detect external services from SDK imports / URLs / env names | MVP | 10 §3 |
| FR-ANA-7 | Cluster nodes into features; user can rename/merge | MVP | 10 §3 |
| FR-ANA-8 | Incremental snapshots per commit; PR overlays never alter canonical lineage | MVP | 08 §3–4 |
| FR-ANA-9 | Nightly full re-analysis with drift metric | MVP | 08 §4.2 |
| FR-ANA-10 | Record static-vs-runtime discrepancies | MVP | 10 §6 |
| FR-ANA-11 | Backend runtime tracing via injected OpenTelemetry | P4 | 10 §4 |
| FR-ANA-12 | Additional languages (Python, Go, Java) analyzers | P5 | 10 |

## FR-EXP — Runtime exploration
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-EXP-1 | Heuristic exploration per role with fingerprint-based state dedupe | MVP | 11 §3–4 |
| FR-EXP-2 | Enforce action/time/depth/state/domain limits | MVP | 11 §5 |
| FR-EXP-3 | Exclude destructive actions via ActionPolicy and patterns | MVP | 11 §4, 19 §4 |
| FR-EXP-4 | Record UI states, transitions, UI→API invokes edges | MVP | 11 §7 |
| FR-EXP-5 | Propose Flows from goal-reaching paths | MVP | 11 §7 |
| FR-EXP-6 | Targeted exploration of impacted uncovered routes in Smart runs | MVP | 11 §2 |
| FR-EXP-7 | AI-ranked candidate actions | P2 | 11 §4 |
| FR-EXP-8 | Negative form fills during exploration | P2 | 11 §4 |

## FR-IMP — Change impact
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-IMP-1 | Map hunks to symbols and non-code change seeds | MVP | 12 §2.1 |
| FR-IMP-2 | Weighted reverse traversal with decay and broad-impact guard | MVP | 12 §2.2 |
| FR-IMP-3 | Aggregate to endpoints, flows, features, intent claims | MVP | 12 §2.3 |
| FR-IMP-4 | Map to tests via runtime coverage first | MVP | 12 §2.4 |
| FR-IMP-5 | Compute risk level | MVP | 12 §2.5 |
| FR-IMP-6 | Provide path explanations for every affected item | MVP | 12 §3 |
| FR-IMP-7 | Escalate on partial graph or broad impact | MVP | 12 §5 |
| FR-IMP-8 | Scheduled shadow full runs measuring selection recall; auto-tune τ | MVP | 12 §5 |
| FR-IMP-9 | Impact preview API without running tests | MVP | 23 |

## FR-TST — Planning & tests
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-TST-1 | Modes Quick / Smart / Full / Manual(custom) | MVP | 13 §5, 12 §4 |
| FR-TST-2 | Planner orders fail-fast within budget; reports skipped items with reason | MVP | 13 §5 |
| FR-TST-3 | Always include `@critical` tests | MVP | 13 §5 |
| FR-TST-4 | Derived deterministic checks: flow replay, auth matrix (unauth + role), OpenAPI contract | MVP | 13 §3 |
| FR-TST-5 | Manual NL request mapped to flows/features with confirmation when ambiguous | MVP | 13 §6 |
| FR-TST-6 | TestDefinition lifecycle (proposed/active/quarantined/retired) | MVP | 07 §4.2 |
| FR-TST-7 | Export DSL test to Playwright `.spec.ts` | P2 | 13 §2 |
| FR-TST-8 | AI-generated tests as advisory proposals with validation + double dry-run | P3 | 13 §4 |
| FR-TST-9 | Negative, boundary, UI-state test derivation | P3 | 13 §3 |
| FR-TST-10 | Cypress test import | P3 | 14 §6 |
| FR-TST-11 | Open PR to add tests to repo (opt-in) | P4 | 13 |

## FR-EXE — Execution
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-EXE-1 | Fresh isolated sandbox per run at exact SHA with recorded env profile & policy versions | MVP | 17 |
| FR-EXE-2 | Build, datastores, migrations, seeds, hooks, health checks with per-step timing/logs | MVP | 17 §5 |
| FR-EXE-3 | Run imported Playwright specs with platform reporter overlay | MVP | 14 §6 |
| FR-EXE-4 | Execute DSL tests and exploration actions through validator + ActionPolicy | MVP | 14 §4 |
| FR-EXE-5 | Deterministic settings: viewport, locale, timezone, clock, reduced motion | MVP | 14 §5 |
| FR-EXE-6 | Test data isolation strategies (reset default, namespace) | MVP | 21 §3 |
| FR-EXE-7 | Egress default-deny with sandbox/mock/block per host; email sink | MVP | 21 §6, 19 §3 |
| FR-EXE-8 | Heartbeats, hard TTL, checkpointed per-test results | MVP | 17 §1, §7 |
| FR-EXE-9 | Sharding a plan across multiple sandboxes | P2 | 17 §6 |
| FR-EXE-10 | Multi-browser matrix | P2 | 14 |

## FR-CMP — Comparison & classification
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-CMP-1 | Maintain baselines per subject/branch with update rules | MVP | 20 §2 |
| FR-CMP-2 | Detect behavior changes across dimensions (flow, status, shape, UI structure, console, auth, timing) | MVP | 20 §3 |
| FR-CMP-3 | Noise classification (infra, third-party, test defect, flaky via retry & history) before intent comparison | MVP | 20 §4 |
| FR-CMP-4 | Intent evaluation (deterministic structured checks + capped AI free-text judgement) | MVP | 09 §4 |
| FR-CMP-5 | Verdict rules and confidence per 20 §5; low-confidence regression shown as UNKNOWN by default | MVP | 20 §5 |
| FR-CMP-6 | Finding grouping and fingerprint dedupe; resolution carry-over across PR pushes | MVP | 20 §9 |
| FR-CMP-7 | Per-project calibration from resolutions | MVP | 20 §6 |
| FR-CMP-8 | Flakiness score and quarantine | MVP | 20 §7 |
| FR-CMP-9 | Visual comparison dimension | P4 | 20 §3 |

## FR-INT — Intent Brain
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-INT-1 | Extract claims from PR descriptions & commit messages (AI, schema-constrained, scoped) | MVP | 09 §6 |
| FR-INT-2 | Derive claims from existing test assertions and OpenAPI | MVP | 09 §2 |
| FR-INT-3 | Platform implicit claims (no 5xx, no page errors, auth not loosened, no new console errors) | MVP | 09 §4 |
| FR-INT-4 | Developer confirmation UX with batched, specific prompts | MVP | 09 §5 |
| FR-INT-5 | Decisions stored with provenance; branch-scoped until merge | MVP | 09 §5 |
| FR-INT-6 | Manual claim entry and review of proposed claims | MVP | 09 §2 |
| FR-INT-7 | DB constraint and security-config derived claims | P2 | 09 §2 |
| FR-INT-8 | Ticket and requirements-doc ingestion | P3 | 09 §2 |
| FR-INT-9 | Design (Figma) ingestion | P5 | 09 §2 |

## FR-AI — AI
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-AI-1 | AI Gateway with provider adapters, tier routing, redaction, budgets, caching, schema validation, reference checks | MVP | 15 §3 |
| FR-AI-2 | Failure diagnosis for non-PASS results citing evidence IDs | MVP | 15 §2 |
| FR-AI-3 | AI output never directly sets a verdict | MVP | 15 §5 |
| FR-AI-4 | Degraded mode without AI | MVP | 05 §6 |
| FR-AI-5 | Offline evaluation suite gating template/model changes | MVP | 15 §7 |

## FR-REP — Evidence & reporting
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-REP-1 | Evidence bundle per 18 §2, redacted in sandbox | MVP | 18 |
| FR-REP-2 | Run report with strategy, counts per verdict, findings with rule trail | MVP | 18 §4 |
| FR-REP-3 | Sticky PR comment and check summary | MVP | 18 §5 |
| FR-REP-4 | In-app trace viewer via signed URLs | MVP | 18 §3 |
| FR-REP-5 | Slack/email notifications | P2 | 18 §7 |
| FR-REP-6 | Quality trend dashboards (catch rate, flakiness, cost) | P2 | 00 §5 |

## FR-HEAL — Self-healing
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-HEAL-1 | Detect probable locator renames; produce HealProposal with confidence and evidence; never auto-apply | P4 | 25 §13 |

## FR-ADM — Administration
| ID | Requirement | Phase | Ref |
|---|---|---|---|
| FR-ADM-1 | Roles owner/admin/member/viewer | MVP | 19 SEC-20 |
| FR-ADM-2 | Budgets per org/project/run for compute and AI | MVP | 15 §6 |
| FR-ADM-3 | AI data policy per org | MVP | 15 §4 |
| FR-ADM-4 | Audit log viewer and export | MVP | 19 SEC-16 |
| FR-ADM-5 | Data deletion (project/org) | MVP | 19 §6 |
| FR-ADM-6 | SSO/SAML | P3 | 19 |
