# 22 — Data Model (Physical)

**Purpose:** Map the domain model to physical storage: tables, key columns, indexes, high-volume data handling, and retention.
**Related:** [07-domain-model](./07-domain-model.md), [19-security-and-safety](./19-security-and-safety.md) §6

Assumes the ⚑ recommended **PostgreSQL** (see [05-system-architecture](./05-system-architecture.md) §7). If MongoDB is chosen instead, the same aggregates map to collections; the tenancy and indexing rules remain.

## 1. Storage allocation

| Data | Store | Why |
|---|---|---|
| Tenancy, config, runs, results, findings, resolutions, intent, test definitions | PostgreSQL | Relational integrity, transactions, RLS |
| Graph nodes/edges (per snapshot diff) | PostgreSQL (partitioned by project) | Bounded size per project; loaded into memory for traversal |
| Observations (high volume) | PostgreSQL partitioned by month **or** columnar files (Parquet) in object storage with an index row per run ⚑ | Raw observations are write-once, read rarely; digests are what's queried |
| Artifacts | Object storage | Size |
| Queue state | Redis (BullMQ) | Ephemeral |
| Secrets | Vault/KMS-backed secret store | Isolation |
| Audit log | PostgreSQL append-only table (+ periodic export to WORM object storage) | Integrity |

## 2. Tables (key columns)

All tables include `org_id uuid not null`, `created_at`, `updated_at`; project-scoped tables include `project_id`. RLS policy: `org_id = current_setting('app.org_id')::uuid`.

```sql
-- Tenancy
organizations(id pk, name, plan, data_region, settings jsonb)
users(id pk, email unique, display_name)
memberships(org_id, user_id, role, pk(org_id,user_id))
provider_installations(id pk, org_id, provider, external_installation_id, scopes text[], status, secret_ref)

-- Configuration
projects(id pk, org_id, name, default_branch, primary_repository_id, settings jsonb, status)
repositories(id pk, org_id, project_id, installation_id, provider, external_id, full_name, default_branch)
applications(id pk, org_id, project_id, repository_id, name, kind, root_path, framework, detected_by)
environment_profiles(id pk, org_id, project_id, version int, body jsonb, source, created_by, unique(project_id,version))
secret_refs(id pk, org_id, project_id, name, vault_path, scope, expose_to_ai bool default false, rotated_at, unique(project_id,name))
test_accounts(id pk, org_id, project_id, role, login_strategy, secret_ref_ids jsonb, login_flow_id)
trigger_rules(id pk, org_id, project_id, event, branch_pattern, cron, mode, blocking, enabled, config jsonb)
action_policies(id pk, org_id, project_id, version, rules jsonb)

-- Application Brain
snapshots(id pk, org_id, project_id, repository_id, commit_sha, parent_snapshot_id, branch_scope, analyzer_version, status, is_materialized bool, stats jsonb,
          unique(repository_id, commit_sha, branch_scope))
graph_nodes(project_id, key, snapshot_id, op  /* add|update|tombstone */, type, application_id, name, location jsonb, attrs jsonb, layer, risk_tags text[],
            pk(project_id, snapshot_id, key))
graph_edges(project_id, edge_key /* hash(from,to,type,layer) */, snapshot_id, op, from_key, to_key, type, layer, confidence real, evidence jsonb,
            seen_count int, last_seen_run_id, stale bool, pk(project_id, snapshot_id, edge_key))
graph_discrepancies(id pk, project_id, type, static_edge_key, runtime_edge_key, first_seen_run_id, count)
flows(id pk, org_id, project_id, name, description, origin, status, steps jsonb, criticality, feature_keys text[])
ui_states(project_id, fingerprint, url_pattern, title, landmarks jsonb, interactive jsonb, last_seen_run_id, pk(project_id, fingerprint))

-- Intent Brain
intent_sources(id pk, org_id, project_id, type, external_ref, content_hash, trust, ingested_at)
intent_claims(id pk, org_id, project_id, kind, statement, structured jsonb, scope_node_keys text[], scope_flow_ids uuid[], source_id, confidence real,
              status, supersedes_id, valid_branch, valid_from_sha, created_by)
decisions(intent_claim_id pk fk, behavior_change_id, resolution_id, decided_by, reason, old_summary, new_summary)

-- Testing
test_definitions(id pk, org_id, project_id, name, category, origin, format, flow_id, covers_node_keys text[], intent_claim_ids uuid[], account_role,
                 tags text[], status, current_version int, stable_key text /* imported: file#titlePath */, unique(project_id, stable_key))
test_definition_versions(test_definition_id, version, body jsonb, created_by, reason, pk(test_definition_id, version))

-- Execution
change_sets(id pk, org_id, project_id, repository_id, trigger, provider, base_sha, head_sha, merge_base_sha, branch, base_branch, pr_number, pr_title,
            pr_body, author, commit_messages jsonb, changed_files jsonb, changed_symbols jsonb, delivery_id unique, received_at)
runs(id pk, org_id, project_id, change_set_id, trigger_rule_id, mode, env_profile_version, action_policy_version, snapshot_id, plan jsonb, impact jsonb,
     budget jsonb, status, stage_timings jsonb, sandbox_id, summary jsonb, run_seed bigint, requested_by, superseded_by)
sandboxes(id pk, org_id, run_id, worker_host, image_digests jsonb, status, started_at, healthy_at, destroyed_at, logs_artifact_id)
test_results(id pk, org_id, run_id, test_definition_id, version, attempt, raw_outcome, verdict, confidence, duration_ms, steps jsonb, failure_signature)
observations(run_id, test_result_id, seq, type, data jsonb, ts) PARTITION BY RANGE (ts)
artifacts(id pk, org_id, project_id, run_id, kind, storage_key, size_bytes, sha256, redacted bool, retention_class, expires_at)

-- Test Memory / Analysis
baselines(id pk, org_id, project_id, subject_type, subject_id, branch, commit_sha, run_id, digest jsonb, accepted_by, superseded_by,
          unique(project_id, subject_type, subject_id, branch) where superseded_by is null)
behavior_changes(id pk, org_id, project_id, run_id, subject_type, subject_id, dimension, before jsonb, after jsonb, related_node_keys text[],
                 intent_evaluation jsonb, verdict, confidence, status)
findings(id pk, org_id, project_id, run_id, title, verdict, severity, confidence, affected_node_keys text[], affected_flow_ids uuid[], expected, actual,
         analysis jsonb, ai_generated bool, status, fingerprint)
finding_behavior_changes(finding_id, behavior_change_id, pk(finding_id, behavior_change_id))
resolutions(id pk, org_id, project_id, target_type, target_id, resolution, reason, resolved_by, resolved_at, channel)
test_stats(project_id, test_definition_id, branch, window, pass_count, fail_count, flip_count, flakiness real, duration_p50, duration_p90, last_failure_signature,
           pk(project_id, test_definition_id, branch, window))
external_service_health(project_id, host, day, requests, errors, timeouts, p50_ms, pk(project_id, host, day))
calibration(project_id, dimension, params jsonb, updated_at, pk(project_id, dimension))

-- Platform
ai_calls(id pk, org_id, project_id, run_id, task, template_version, model, input_hash, tokens_in, tokens_out, cost_usd numeric, latency_ms, cache_hit, created_at)
audit_log(id bigserial pk, org_id, actor_type, actor_id, action, target_type, target_id, details jsonb, ip, created_at)  -- insert-only role
```

## 3. Key indexes

| Table | Index | Supports |
|---|---|---|
| runs | `(project_id, created_at desc)`, `(change_set_id)` | Dashboard lists, PR view |
| change_sets | `(project_id, pr_number, received_at desc)`, `(repository_id, head_sha)` | PR history, dedupe |
| test_results | `(run_id)`, `(test_definition_id, created_at desc)` | Run view, history |
| findings | `(project_id, fingerprint)`, `(project_id, status)` | Dedupe, inbox |
| graph_nodes/edges | `(project_id, key)`, GIN on `attrs` where needed; edges `(project_id, to_key)` for reverse traversal | Impact |
| intent_claims | GIN `(scope_node_keys)`, `(project_id, status)` | Comparator retrieval |
| baselines | partial unique active per subject/branch | Comparator |
| artifacts | `(run_id)`, `(expires_at)` | Retention sweeps |

## 4. Graph materialization
- `snapshots.is_materialized = true` every K=50 snapshots on default branch: a full copy of live nodes/edges.
- Graph-at-SHA = nearest materialized ancestor + ordered diffs. Impact workers cache the latest default-branch graph per project in memory (LRU across projects), apply PR overlay on the fly.

## 5. Retention classes

| Class | Default TTL (⚑ proposal) | Enforcement |
|---|---|---|
| `short` | 7 days | Object storage lifecycle rule + nightly DB sweep |
| `standard` | 30 days | Same |
| `baseline` | While referenced by an active baseline + 30 days | Reference-counted sweep |
| `metadata` | 13 months | Monthly partition drop (observations), row deletes for others |
| `audit` | 13 months (configurable) | Export to WORM before delete |

## 6. Migration & evolution
- `jsonb` payloads carry `schemaVersion`; readers upcast old versions.
- Graph node/edge `type` vocab is additive; analyzers declare versions and the snapshot records them.
