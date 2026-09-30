# 04 — Non-Functional Requirements

**Purpose:** Quality attributes the platform must meet. Per the brief, no SLA numbers are invented: where a number appears it is a **Hypothesis** with rationale, to be validated by measurement during MVP milestone M1–M2 on reference apps and design partners.
**Related:** [00-product-vision](./00-product-vision.md) §5, [05-system-architecture](./05-system-architecture.md), [19-security-and-safety](./19-security-and-safety.md)

## 1. Reliability
| ID | Requirement |
|---|---|
| NFR-REL-1 | Every accepted webhook results in a Run record or a recorded "ignored" reason (no silent drops). Measured by reconciliation job. |
| NFR-REL-2 | Every Run reaches a terminal state; no run stays non-terminal beyond its budget + TTL margin (enforced by reaper). |
| NFR-REL-3 | Stage jobs are idempotent and retry-safe; at-least-once delivery with idempotent handlers. |
| NFR-REL-4 | Platform-caused failures are distinguishable from app-caused failures in every report (`INFRA_FAILURE` subtypes: platform vs. app-startup). |
| NFR-REL-5 | Loss of an execution host loses at most the in-flight tests of runs on that host; completed test results persist. |
| **Hypothesis** | Platform-caused run failure rate target < 1% of runs after M3. Rationale: developers ignore a check that fails for non-code reasons more than ~1 in 100 times. |

## 2. Scalability
| ID | Requirement |
|---|---|
| NFR-SCL-1 | Stateless control-plane services scale horizontally. |
| NFR-SCL-2 | Execution capacity scales on queue depth; per-org concurrency caps prevent noisy-neighbor starvation. |
| NFR-SCL-3 | Impact analysis latency grows sub-linearly with repo size via incremental snapshots and in-memory graph cache. |
| NFR-SCL-4 | Observation and artifact volume is bounded per run by caps (count/size) with graceful truncation noted in the manifest. |
| **Hypothesis** | Design for 10³ projects and 10⁴ runs/day in MVP architecture without redesign; beyond that, cellular deployment (06 §4). |

## 3. Security
Defined in [19-security-and-safety](./19-security-and-safety.md) as SEC-1…SEC-22. All `MVP`-tagged SEC items are release blockers.

## 4. Performance (developer feedback latency)
| ID | Requirement |
|---|---|
| NFR-PERF-1 | First PR status (`pending`) posted within seconds of webhook receipt (async ack). |
| NFR-PERF-2 | Stage timings (queue, analysis, provision, execution, evaluation, publish) are measured for every run and exposed. |
| NFR-PERF-3 | Analysis for a PR is incremental (changed files + dependents) rather than full. |
| NFR-PERF-4 | Evaluation/AI diagnosis runs only for non-PASS items and in parallel per finding. |
| **Hypothesis** | Smart PR run time-to-verdict P50 ≤ 10–15 min for a typical Next.js+API+DB project with warm caches. Rationale: must be comparable to existing CI so it is not the slowest check. To be validated after M1 sandbox measurements; if sandbox startup alone exceeds ~5 min, snapshot-restore warm pools become an M2 priority. |

## 5. Observability (of the platform)
| ID | Requirement |
|---|---|
| NFR-OBS-1 | Distributed tracing (OpenTelemetry) across webhook → run stages → publish, keyed by runId. |
| NFR-OBS-2 | Metrics: run duration by stage, queue wait, worker/browser/sandbox failures, env startup failures by step, AI latency/cost/tokens by task, flakiness rate, false-positive rate, run success rate. |
| NFR-OBS-3 | Structured logs with orgId/projectId/runId; no secrets (redaction middleware + tests). |
| NFR-OBS-4 | Per-tenant usage dashboards (internal) for cost and abuse detection. |
| NFR-OBS-5 | Alerts: queue backlog age, sandbox startup failure spike, AI provider error rate, webhook signature failures spike, publish failures. |

## 6. Cost control
| ID | Requirement |
|---|---|
| NFR-COST-1 | Budgets (compute minutes, AI tokens/cost) at org, project, and run levels with soft (warn) and hard (stop) limits. |
| NFR-COST-2 | Run cost (sandbox-minutes, AI cost, artifact bytes) recorded per run and shown to users. |
| NFR-COST-3 | Artifact retention classes minimize storage (pass traces sampled, videos short-lived). |
| NFR-COST-4 | AI invoked only at defined decision points; cached where deterministic. |

## 7. Tenant isolation
| ID | Requirement |
|---|---|
| NFR-TEN-1 | No cross-tenant data access through any API (automated tenancy tests on every endpoint). |
| NFR-TEN-2 | No shared sandboxes, caches, or warm-pool VMs across orgs. |
| NFR-TEN-3 | Per-org encryption keys for artifacts and secrets; crypto-shred on deletion. |

## 8. Reproducibility
| ID | Requirement |
|---|---|
| NFR-REP-1 | Each Run records SHA, env profile version, action policy version, analyzer/engine versions, image digests, browser version, run seed, clock setting. |
| NFR-REP-2 | "Rerun with same inputs" is supported and produces a new Run linked to the original. |
| NFR-REP-3 | Generated data values are recorded in step logs. |

## 9. Availability
| ID | Requirement |
|---|---|
| NFR-AVL-1 | Webhook ingest has higher availability priority than the dashboard (it can buffer and reconcile). |
| NFR-AVL-2 | Git provider or AI provider outages degrade gracefully (publish retry; deterministic-only verdicts). |
| **Hypothesis** | Initial target 99.5% monthly for API/webhook ingest in MVP, raising with maturity. Rationale: single-region managed services make this achievable without multi-region cost. |

## 10. Auditability
| ID | Requirement |
|---|---|
| NFR-AUD-1 | Append-only audit log for security-relevant actions (SEC-16). |
| NFR-AUD-2 | Every verdict stores its rule trail (baseline used, claims & weights, noise checks, AI calls with template versions). |
| NFR-AUD-3 | Every Decision links to resolution, user, reason, and evidence. |

## 11. Data retention
Defaults in [19-security-and-safety](./19-security-and-safety.md) §6 and [22-data-model](./22-data-model.md) §5; configurable per org within plan limits.

## 12. Privacy
| ID | Requirement |
|---|---|
| NFR-PRV-1 | PII minimization in evidence: redaction of known secrets and configured PII patterns; screenshots of seeded test data only (no production data). |
| NFR-PRV-2 | AI data policy honored for every AI call; sub-processors disclosed. |
| NFR-PRV-3 | Data export and deletion on request. |

## 13. Extensibility
| ID | Requirement |
|---|---|
| NFR-EXT-1 | Git providers via `GitProvider` interface; analyzers via `AnalyzerPlugin`; AI providers via adapter; environment strategies (compose, scripts, external URL) via strategy interface. |
| NFR-EXT-2 | Graph node/edge type vocabulary is additive and versioned. |
| NFR-EXT-3 | Test formats: DSL + imported Playwright; new importers (Cypress) without changing Test Memory. |

## 14. Usability (added — not in brief, but critical to trust)
| ID | Requirement |
|---|---|
| NFR-USE-1 | Max 3 confirmation prompts per PR comment; the rest on the dashboard. |
| NFR-USE-2 | Every selected test shows *why* it was selected. |
| NFR-USE-3 | Onboarding to first run requires no code changes in the user's repo. |
