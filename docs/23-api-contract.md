# 23 — API Contract (High Level)

**Purpose:** Define the external platform API surface and webhook endpoints at a coarse level. Per the brief, this is intentionally not over-designed; field-level schemas are finalized in Phase 1 once domain boundaries are validated.
**Related:** [07-domain-model](./07-domain-model.md), [16-ci-cd-integration](./16-ci-cd-integration.md), [19-security-and-safety](./19-security-and-safety.md)

## 1. Conventions
- REST over HTTPS, JSON. Base: `/api/v1`.
- Auth: session cookie (dashboard) or `Authorization: Bearer <PAT>` (org-scoped personal/API tokens with scopes) for CLI/CI.
- Tenancy: org derived from token/session; project IDs validated against org.
- Idempotency: `Idempotency-Key` header on POSTs creating runs/resolutions.
- Pagination: cursor-based (`?cursor=&limit=`).
- Errors: RFC 9457 problem+json.
- Long operations return `202` with a resource to poll; server-sent events for run progress (`/runs/{id}/events`).

## 2. Endpoints

### Authentication & identity
| Method | Path | Purpose |
|---|---|---|
| GET | `/auth/{provider}/login` · `/auth/{provider}/callback` | OAuth login (GitHub/GitLab) |
| POST | `/auth/logout` | |
| GET | `/me` | Current user, memberships |
| POST/GET/DELETE | `/orgs/{orgId}/tokens` | API tokens (scoped) |
| GET/POST/PATCH/DELETE | `/orgs/{orgId}/members` | Membership management |

### Projects & repositories
| Method | Path | Purpose |
|---|---|---|
| POST | `/projects` | Create project |
| GET/PATCH/DELETE | `/projects/{id}` | Read/update/archive |
| GET | `/integrations/{provider}/installations` | List installations/repos available |
| POST | `/projects/{id}/repositories` | Connect repository (installation + repo id) |
| POST | `/projects/{id}/analysis` | Trigger (re)analysis; returns snapshot job |
| GET | `/projects/{id}/applications` | Detected applications |

### Configuration
| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/projects/{id}/environment-profiles` | List versions / create new version (incl. auto-detected proposal via `?proposal=true`) |
| POST | `/projects/{id}/environment-profiles/{v}/validate` | Start a startup-only sandbox run |
| GET/PUT/DELETE | `/projects/{id}/secrets/{name}` | Write-only value; GET returns metadata only |
| GET/POST/PATCH | `/projects/{id}/test-accounts` | |
| GET/POST/PATCH/DELETE | `/projects/{id}/triggers` | TriggerRules |
| GET/PUT | `/projects/{id}/action-policy` | ActionPolicy (relaxations admin-only, audited) |
| GET/PUT | `/projects/{id}/budgets` | Run/monthly budgets |

### Runs & results
| Method | Path | Purpose |
|---|---|---|
| POST | `/projects/{id}/runs` | Manual run: `{ref|sha, mode, scope?{flowIds, featureKeys, routes}, request?: "Test the booking flow", targetUrl? (P4)}` |
| GET | `/projects/{id}/runs` | List (filters: branch, pr, status, verdict) |
| GET | `/runs/{runId}` | Status, stages, plan, impact summary, counts |
| GET | `/runs/{runId}/events` | SSE progress |
| POST | `/runs/{runId}/cancel` · `/runs/{runId}/rerun` | |
| GET | `/runs/{runId}/results` | TestResults |
| GET | `/runs/{runId}/findings` | Findings |
| GET | `/runs/{runId}/report` | Rendered report (JSON / markdown / html) |

### Evidence
| Method | Path | Purpose |
|---|---|---|
| GET | `/runs/{runId}/artifacts` | Manifest |
| GET | `/artifacts/{artifactId}/url` | Short-lived signed URL (authz checked) |

### Findings & feedback
| Method | Path | Purpose |
|---|---|---|
| GET | `/projects/{id}/findings` | Inbox (open/awaiting) |
| GET | `/findings/{id}` | Detail incl. evidence, rule trail |
| POST | `/findings/{id}/resolutions` | `{resolution, reason?}` |
| POST | `/behavior-changes/{id}/resolutions` | Same, for individual changes |
| POST | `/baselines` | Accept observation as baseline `{runId, subject}` |

### Project Brain (read-mostly)
| Method | Path | Purpose |
|---|---|---|
| GET | `/projects/{id}/graph?sha=&types=&around=&depth=` | Subgraph query |
| GET | `/projects/{id}/graph/discrepancies` | Static vs runtime |
| GET/POST/PATCH | `/projects/{id}/flows` | Flows (accept proposed, rename, set criticality) |
| GET/POST/PATCH | `/projects/{id}/tests` | TestDefinitions (accept/retire, edit DSL) |
| GET | `/projects/{id}/tests/{tid}/history` | Test Memory |
| POST | `/projects/{id}/impact/preview` | Impact for `{baseSha, headSha}` without running |

### Intent Brain
| Method | Path | Purpose |
|---|---|---|
| GET/POST/PATCH | `/projects/{id}/intent/claims` | List/add manual claims, confirm/reject proposed |
| GET | `/projects/{id}/intent/decisions` | Decision log |
| POST | `/projects/{id}/intent/sources` | Upload/link requirement doc, OpenAPI (P3 for docs/tickets) |

### Git provider webhooks (public, signature-authenticated)
| Method | Path | Purpose |
|---|---|---|
| POST | `/webhooks/github` | GitHub App events |
| POST | `/webhooks/gitlab/{hookId}` | GitLab hook events |

### Worker job API (internal, mTLS, not public)
| Method | Path | Purpose |
|---|---|---|
| POST | `/internal/jobs/lease` | Worker pulls a job for its zone/capabilities |
| POST | `/internal/jobs/{id}/heartbeat` | Liveness + progress |
| POST | `/internal/jobs/{id}/results` | Stage output (schema-validated, size-capped) |
| POST | `/internal/jobs/{id}/observations` | Streamed batches |
| POST | `/internal/runs/{id}/secret-key` | Attested fetch of run secret key (execution plane only) |

## 3. Example: create manual run

```http
POST /api/v1/projects/prj_9/runs
Idempotency-Key: 5f1e...
{ "ref": "feature/participants-first", "mode": "custom", "request": "Test the booking flow" }

202 Accepted
{ "runId": "run_456", "status": "queued", "links": { "self": "/api/v1/runs/run_456", "events": "/api/v1/runs/run_456/events" } }
```

## 4. Stability
- `v1` endpoints for runs, results, findings, resolutions, webhooks are the first to stabilize (needed by CLI/CI).
- Brain and intent endpoints are marked **experimental** until Phase 4.
