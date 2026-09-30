# 18 — Evidence & Reporting

**Purpose:** Specify evidence capture, storage, access, and how reports and PR comments present verdicts without overstating conclusions.
**Related:** [14-playwright-engine](./14-playwright-engine.md), [20-test-memory](./20-test-memory.md), [16-ci-cd-integration](./16-ci-cd-integration.md), [19-security-and-safety](./19-security-and-safety.md), [22-data-model](./22-data-model.md)

## 1. Evidence principles
1. No verdict other than `PASS` without evidence attached.
2. Evidence is **immutable** and content-addressed (sha256) once uploaded.
3. Evidence is **redacted before upload** (in the sandbox), not after.
4. Evidence is tenant-scoped and only reachable via short-lived signed URLs after an authorization check.

## 2. Evidence bundle per test / finding

| Item | Format | Captured | Retention class |
|---|---|---|---|
| Step log | JSON (in DB as TestResult.steps) | Always | `standard` |
| Screenshots | PNG/WebP | Failure/change + configurable per step | `standard` |
| Playwright trace | `.zip` | Recorded always; kept on non-PASS, sampled (e.g. 10%) on PASS | `standard` / `short` |
| Video | WebM | Full/nightly or opt-in | `short` |
| Network log | HAR-lite JSON (bodies redacted/limited) | Always | `standard` |
| Console + page errors | JSON lines | Always | `standard` |
| DOM + a11y snapshot | HTML (scripts stripped) / JSON | Failure/change, baseline states | `standard` |
| Service logs | Text (stdout/stderr per service) | Always (tail on success, full on failure) | `short` |
| Git diff excerpt | Unified diff of suspected files | On finding | `standard` |
| Visual diff | PNG (before/after/diff) | Visual tests (P4) | `standard` |

Evidence manifest per run lists every artifact with kind, size, hash, redaction status — used for the "evidence availability" metric.

## 3. Storage and access
- Object storage key: `org/{orgId}/project/{projectId}/run/{runId}/{kind}/{sha256}.{ext}`.
- Upload: presigned PUT URLs scoped to the run prefix, issued at run start, expire with the run.
- Download: API checks membership → issues presigned GET (≤ 5 min). Trace viewer is served from the platform's origin with the trace loaded via signed URL (never public).
- Encryption at rest with per-org keys (SSE-KMS). Lifecycle rules implement retention classes (see [22-data-model](./22-data-model.md) §5).

## 4. Run report (dashboard)

```
Project: Acme Bookings          Commit: a1b2c3 (PR #412 "Participants first")
Environment: profile v7, Chromium 1xx, ephemeral sandbox     Mode: Smart
Risk: HIGH   (PaymentService.charge changed → Booking, Checkout)

Strategy
  12 tests selected of 148 (impact), 3 derived auth checks, 1 targeted exploration
  Why: [impact paths ▸]

Results
  ✅ Passed 11   ⚠ Expected changes 1   ❓ Unknown 1   🔴 Regression candidates 1
  🔁 Flaky 0     ⛔ Infra 0   🌐 Third-party 0   ⏭ Skipped (budget) 0
```

Each **Finding** card:

| Field | Content |
|---|---|
| Title | "POST /api/booking returns 500 during checkout" |
| Verdict · Confidence · Severity | `REGRESSION_CANDIDATE` · High · Critical |
| Affected module / flow | Booking / "Booking — happy path" |
| Expected | "201 Created (baseline main@9f1c2e, 30/30 passes); contract ic_12" |
| Actual | "500 Internal Server Error; console `TypeError: Cannot read properties of undefined (reading 'amount')`" |
| Evidence | screenshot · trace · network · console · service log |
| Relevant diff | `payment.service.ts` hunk (+12 −4) |
| Analysis (AI, labelled) | "Likely related to change in `PaymentService.charge` where `price` was renamed to `amount`… Evidence: [net#14], [log#3]. Confidence: medium. Not verified." |
| Why this verdict | Rule trail: baseline passed 30×; 5xx dimension (implicit intent); no supporting intent for change |
| Actions | Confirm regression · False positive · Intentional · Ignore · Rerun |

## 5. PR/MR comment (sticky)

Concise; the dashboard holds detail.

```markdown
### 🤖 AutoQA — Smart run on `a1b2c3` · Risk: High
| 🔴 1 regression candidate | ❓ 1 needs confirmation | ⚠ 1 expected change | ✅ 11 passed |

**🔴 POST /api/booking → 500 during checkout** (confidence: high)
Baseline 201 on main. Console TypeError. Changed: `payment.service.ts`. [Evidence](…) · [Trace](…)
<sub>Reply `/qa false-positive F-81 <reason>` if this is wrong.</sub>

**❓ Booking step order changed** — Date↔Participants swapped. PR description suggests intentional.
[Intentional](…) · [Requirement changed](…) · [Investigate](…) · [Ignore](…)

<details><summary>Tests run (12) and why</summary> … </details>
<!-- autoqa:run-marker -->
```

## 6. Language rules (avoid overstating)
- Use "regression candidate", "likely", "associated with"; never "bug", "root cause is", "caused by" unless a deterministic chain supports it (e.g. stack trace frame in a changed line).
- Always show the confidence and the rule trail.
- Distinguish **observed facts** (status codes, errors) from **AI interpretation** visually.
- Third-party/infra verdicts state what was observed ("api.stripe.com timed out 3/3 via egress proxy") and that the app was not blamed.

## 7. Notifications
MVP: PR comment/check + dashboard. P2: email digest, Slack webhook for `REGRESSION_CANDIDATE` on main and nightly summaries.

## 8. Open items
- Default trace retention on PASS → Q-DATA-2.
