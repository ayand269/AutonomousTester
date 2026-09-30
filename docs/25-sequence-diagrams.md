# 25 — Sequence Diagrams

**Purpose:** Key interaction sequences across components.
**Related:** [24-high-level-dfd](./24-high-level-dfd.md), [05-system-architecture](./05-system-architecture.md)

Participant abbreviations: **Dev** developer · **UI** dashboard · **API** control-plane API · **GH/GL** GitHub/GitLab · **WH** webhook ingest · **ORC** run orchestrator · **Q** job queue · **AN** analysis worker (Zone 2) · **BR** Project Brain service · **IMP** impact analyzer · **PL** planner · **EX** execution host / sandbox agent (Zone 3) · **EN** execution engine + Playwright · **CMP** comparator/verdict engine · **AIG** AI gateway · **REP** reporter · **V** vault · **S3** object storage.

---

## 1. New project onboarding

```mermaid
sequenceDiagram
    actor Dev
    participant UI
    participant API
    participant GH
    participant ORC
    participant AN
    participant BR
    participant EX
    Dev->>UI: Sign in with GitHub
    UI->>API: OAuth callback
    API-->>UI: Session, org
    Dev->>UI: Create project, install GitHub App
    UI->>GH: App installation flow
    GH-->>API: installation webhook
    Dev->>UI: Select repo and default branch
    UI->>API: POST /projects/{id}/repositories
    API->>ORC: Start onboarding analysis
    ORC->>AN: analyze job (default branch HEAD)
    AN->>GH: Clone with short-lived token
    AN-->>ORC: Facts, env proposal, test list
    ORC->>BR: Create Snapshot 1, import tests
    API-->>UI: Env proposal with provenance
    Dev->>UI: Confirm or edit env, add secrets and test accounts
    UI->>API: Save env v1, secrets (write-only)
    Dev->>UI: Configure triggers (defaults prefilled)
    UI->>API: POST /environment-profiles/1/validate
    API->>ORC: Validation run
    ORC->>EX: Provision sandbox
    EX-->>ORC: Healthy (or failing step + logs)
    alt healthy
        ORC->>EX: Run imported tests + onboarding exploration
        EX-->>ORC: Results, runtime graph, proposed flows
        ORC->>BR: Store baselines (default branch), flows
        API-->>UI: Onboarding complete, first report
    else unhealthy
        API-->>UI: Show failing step, logs, suggested fix
        Dev->>UI: Edit env and retry
    end
```

## 2. Existing project synchronization

```mermaid
sequenceDiagram
    participant GH
    participant WH
    participant ORC
    participant AN
    participant BR
    GH->>WH: push to main (before, after)
    WH->>ORC: ChangeSet
    ORC->>BR: Get parent snapshot for 'before'
    BR-->>ORC: snapshot S(n)
    ORC->>AN: analyze job (changed files since S(n))
    AN->>GH: Fetch at 'after'
    AN-->>ORC: Incremental facts
    ORC->>BR: Merge to S(n+1) (add, update, tombstone)
    BR->>BR: Integrity check, mark ready or partial
    BR->>BR: Promote runtime knowledge seen on main, decay stale edges
    Note over BR: Nightly job re-analyzes fully and records drift
```

## 3. GitHub pull request

```mermaid
sequenceDiagram
    actor Dev
    participant GH
    participant WH
    participant ORC
    participant AN
    participant IMP
    participant PL
    participant EX
    participant CMP
    participant REP
    Dev->>GH: Open PR
    GH->>WH: pull_request.opened (signed)
    WH->>WH: Verify signature, dedupe delivery
    WH->>ORC: ChangeSet (base, head, merge-base, PR text)
    ORC->>GH: Check run: queued
    ORC->>AN: analyze PR overlay
    AN-->>ORC: Changed symbols, overlay facts
    ORC->>IMP: Compute ImpactSet
    IMP-->>ORC: Affected flows, tests, risk
    ORC->>PL: Plan (Smart, budget)
    PL-->>ORC: Test plan with reasons
    ORC->>GH: Check run: in_progress
    ORC->>EX: execute job
    EX-->>ORC: Results, observations, artifacts
    ORC->>CMP: Evaluate
    CMP-->>ORC: Verdicts, findings
    ORC->>REP: Publish
    REP->>GH: Check conclusion + sticky PR comment
    Dev->>GH: Push new commit
    GH->>WH: pull_request.synchronize
    WH->>ORC: New ChangeSet, cancel superseded run
```

## 4. GitLab merge request

```mermaid
sequenceDiagram
    actor Dev
    participant GL
    participant WH
    participant ORC
    participant REP
    Dev->>GL: Open MR
    GL->>WH: Merge Request Hook (X-Gitlab-Token)
    WH->>WH: Verify token, dedupe
    WH->>ORC: ChangeSet (target branch, source SHA, diff_refs)
    ORC->>GL: Commit status: pending
    Note over ORC: Same pipeline as GitHub PR (analyze, impact, plan, execute, evaluate)
    ORC->>REP: Publish
    REP->>GL: Commit status: success or failed
    REP->>GL: Upsert MR note (sticky marker)
    Dev->>GL: Comment /qa intentional F-12 reason
    GL->>WH: Note Hook
    WH->>ORC: Command (verify Developer+ role via API)
```

## 5. Push to main

```mermaid
sequenceDiagram
    participant GH
    participant WH
    participant ORC
    participant PL
    participant EX
    participant CMP
    participant BR
    participant REP
    GH->>WH: push main
    WH->>ORC: ChangeSet, rule mode=full_critical
    ORC->>BR: Sync snapshot (see sequence 2)
    ORC->>PL: Plan: all critical + affected + integration
    PL-->>ORC: Plan
    ORC->>EX: execute
    EX-->>ORC: Results
    ORC->>CMP: Evaluate vs main baselines
    CMP->>BR: Update baselines for PASS or EXPECTED_CHANGE subjects
    CMP->>BR: Apply pending Decisions from merged PR
    ORC->>REP: Publish commit status, notify on regression candidates
```

## 6. Developer code change (end-to-end impact view)

```mermaid
sequenceDiagram
    actor Dev
    participant GH
    participant AN
    participant IMP
    participant PL
    Dev->>GH: Change backend/booking/booking.service.ts, open PR
    GH->>AN: (via ORC) analyze base...head
    AN->>AN: Hunks to symbol BookingService.create (body_change)
    AN-->>IMP: Seeds
    IMP->>IMP: Reverse traverse: controller, POST /api/booking, ui_state checkout
    IMP->>IMP: Flows Booking, Checkout. Claims ic_12. Tests via runtime coverage
    IMP-->>PL: ImpactSet risk High with path explanations
    PL-->>Dev: (in report) 6 tests selected of 148, and why
```

## 7. Smart test run

```mermaid
sequenceDiagram
    participant ORC
    participant IMP
    participant PL
    participant EX
    participant EN
    participant CMP
    ORC->>IMP: ImpactSet request
    IMP-->>ORC: ImpactSet (candidate tests, uncovered flows)
    ORC->>PL: Build Smart plan
    PL->>PL: Add critical tags, derived auth and contract checks
    PL->>PL: Budget fit, fail-fast order
    PL->>PL: Add targeted exploration for uncovered impacted routes
    PL-->>ORC: Plan
    par Provision while planning completes
        ORC->>EX: Provision sandbox
    end
    ORC->>EN: Execute plan items in order
    EN-->>ORC: Stream results
    ORC->>CMP: Evaluate as results arrive
```

## 8. Full test run

```mermaid
sequenceDiagram
    participant SCH as Scheduler
    participant ORC
    participant PL
    participant EX
    participant CMP
    participant BR
    SCH->>ORC: Nightly trigger main
    ORC->>PL: Plan Full: all active tests + deep exploration
    PL-->>ORC: Plan, sharded if above duration threshold
    ORC->>EX: Execute shards
    EX-->>ORC: Results per shard
    ORC->>CMP: Evaluate
    CMP->>BR: Compute selection recall vs what Smart would have chosen
    BR->>BR: Auto-tune threshold if recall below target
    CMP->>BR: Update flakiness stats
```

## 9. Runtime exploration

```mermaid
sequenceDiagram
    participant XC as Exploration Controller
    participant EN
    participant AIG
    participant BR
    XC->>BR: Seeds, impacted routes, static unvisited edges
    XC->>EN: use_account customer, goto base URL
    loop until budget or frontier empty
        EN-->>XC: State capture (a11y, candidates, network)
        XC->>XC: Fingerprint, dedupe, enqueue candidates
        opt every K actions
            XC->>AIG: Rank candidate IDs by goal
            AIG-->>XC: Ranked IDs (validated)
        end
        XC->>XC: Pick next, ActionPolicy check
        XC->>EN: Structured action
    end
    XC->>BR: UI states, transitions, invokes edges, proposed flows
```

## 10. Behavior change detection

```mermaid
sequenceDiagram
    participant EN
    participant CMP
    participant BR
    participant AIG
    EN-->>CMP: Observation digest for test td_booking
    CMP->>BR: Get baseline (td_booking, main)
    BR-->>CMP: Baseline digest
    CMP->>CMP: Diff per dimension: flow_sequence changed
    CMP->>CMP: Noise checks (none)
    CMP->>BR: Claims scoped to flow_booking
    BR-->>CMP: Decision none, PR claim ic_8f2 (flow_order, conf 0.6)
    CMP->>CMP: Structured check: after matches ic_8f2 order
    CMP->>CMP: S=0.6, C=0.5 (existing test assertion), margin low
    CMP-->>CMP: Verdict UNKNOWN (conflicting weak sources)
```

## 11. Unknown behavior requiring developer confirmation

```mermaid
sequenceDiagram
    actor Dev
    participant REP
    participant GH
    participant API
    participant BR
    participant ORC
    REP->>GH: PR comment with before/after and 4 options
    Dev->>GH: Click Intentional, reason: participant count affects availability
    GH->>API: Link to dashboard resolution (authenticated)
    API->>API: Authorize (member of org, write on repo)
    API->>BR: Resolution INTENTIONAL
    BR->>BR: Create branch-scoped Decision, supersede old claims
    BR->>BR: Mark baseline pending replacement on merge
    API->>ORC: Recompute run verdict
    ORC->>REP: Update check and comment
    Note over BR: On merge to main and green run, Decision becomes canonical and baseline replaced
```

## 12. Test failure analysis

```mermaid
sequenceDiagram
    participant EN
    participant CMP
    participant BR
    participant AIG
    participant REP
    EN-->>CMP: td_booking failed: POST /api/booking 500, console TypeError
    CMP->>CMP: Signature, noise checks (not infra, no external call failed)
    CMP->>EN: Retry once after reset
    EN-->>CMP: Same failure
    CMP->>BR: Baseline 201 (30 passes), impact path to PaymentService.charge
    CMP->>CMP: Rule: hard-failure 5xx contradicts baseline gives REGRESSION_CANDIDATE high
    CMP->>AIG: failure.diagnose(minimal context, evidence IDs)
    AIG-->>CMP: Summary, suspected files, cited evidence
    CMP->>CMP: Validate citations
    CMP->>REP: Finding with rule trail and labelled AI analysis
```

## 13. Self-healing proposal (P4)

```mermaid
sequenceDiagram
    participant EN
    participant CMP
    participant AIG
    participant BR
    actor Dev
    EN-->>CMP: Locator not found: button 'Add to Cart'
    CMP->>CMP: Classify TEST_DEFECT candidate (target missing, page otherwise healthy)
    CMP->>CMP: Heuristic match in new a11y tree: button 'Add to Basket' same position and handler
    CMP->>AIG: heal.propose(old and new a11y excerpts)
    AIG-->>CMP: Candidate with rationale
    CMP->>EN: Verify: rerun test with candidate locator in same sandbox
    EN-->>CMP: Passes
    CMP->>BR: HealProposal (confidence high), test NOT modified
    CMP-->>Dev: Proposal in report: Add to Cart to Add to Basket
    Dev->>BR: Accept
    BR->>BR: New TestDefinition version, reason recorded
```

## 14. CI/CD execution (pipeline step, P4)

```mermaid
sequenceDiagram
    participant CI as Customer CI pipeline
    participant CLI as autoqa CLI
    participant API
    participant ORC
    participant EX
    CI->>CI: Deploy preview environment
    CI->>CLI: autoqa run --sha X --target-url preview --wait
    CLI->>API: POST /runs (API token, targetUrl)
    API->>API: Verify target ownership and staging class
    API->>ORC: Run (browser-only sandbox)
    ORC->>EX: Execute against target URL
    EX-->>ORC: Results
    ORC-->>API: Verdicts
    CLI->>API: Poll run status
    API-->>CLI: Summary and exit code
    CLI-->>CI: Exit 0 or 1 by blocking policy
```

## 15. Test environment creation and destruction

```mermaid
sequenceDiagram
    participant ORC
    participant H as Exec Host
    participant A as Sandbox Agent
    participant V
    participant S3
    ORC->>H: Lease execute job
    H->>A: Restore clean microVM from golden snapshot
    A->>V: Attested request for run secret key
    V-->>A: Key (expires with run)
    A->>A: Clone at SHA, restore dependency cache by lockfile hash
    A->>A: Install and build (registry egress only)
    A->>A: Start datastores, migrate, seed, snapshot data
    A->>A: Start services, health checks
    A-->>ORC: Healthy, stage timings
    loop tests
        A->>A: Run, reset data per strategy
        A->>S3: Upload redacted artifacts (presigned)
        A-->>ORC: Results and heartbeat
    end
    A->>A: Teardown hooks
    H->>H: Destroy VM, wipe disk, release network
    H-->>ORC: Sandbox destroyed
    Note over H: Hard TTL kills VM even if control plane unreachable
```
