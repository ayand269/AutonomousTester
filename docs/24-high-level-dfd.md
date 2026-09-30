# 24 — Data Flow Diagrams

**Purpose:** Context, Level 1, and Level 2 DFDs.
**Related:** [05-system-architecture](./05-system-architecture.md), [25-sequence-diagrams](./25-sequence-diagrams.md)

**Notation (Mermaid):** `[Rectangle]` external entity · `([Stadium])` process · `[(Cylinder)]` data store · edge labels are data flows. Trust-zone subgraphs are shown where relevant.

---

## 1. Context DFD (Level 0)

```mermaid
flowchart LR
    DEV[Developer]
    GIT[GitHub / GitLab]
    AI[AI model providers]
    EXT[Third-party sandbox services]
    P(["Autonomous QA Platform"])
    OBJ[(Evidence storage)]

    DEV -- "push / PR / MR" --> GIT
    GIT -- "webhook events" --> P
    P -- "clone at SHA (short-lived token)" --> GIT
    P -- "check status, PR/MR comment" --> GIT
    DEV -- "config, secrets, resolutions, manual runs" --> P
    P -- "reports, questions, evidence links" --> DEV
    P -- "redacted prompts" --> AI
    AI -- "structured outputs" --> P
    P -- "sandbox-only test traffic" --> EXT
    P -- "artifacts" --> OBJ
    OBJ -- "signed evidence URLs" --> DEV
```

The "Cloud Test Runner" from the brief is internal to the platform (Execution Plane) and appears at Level 1.

---

## 2. Level 1 DFD

```mermaid
flowchart TB
    DEV[Developer]
    GIT[GitHub / GitLab]
    LLM[AI providers]

    subgraph CP["Control Plane"]
        P1(["1 Webhook ingest & trigger evaluation"])
        P2(["2 Run orchestration"])
        P5(["5 Change impact analysis"])
        P6(["6 Test planning"])
        P7(["7 Test generation / derivation"])
        P10(["10 Comparator & verdicts"])
        P11(["11 AI analysis via gateway"])
        P12(["12 Reporting"])
        P13(["13 Feedback handling"])
        D1[(Config: projects, envs, triggers, policies)]
        D2[(Project Brain: Application Brain)]
        D3[(Project Brain: Intent Brain)]
        D4[(Project Brain: Test Memory)]
        DQ[(Job queue)]
        DV[(Secret vault)]
    end
    subgraph AT["Analysis Tier"]
        P3(["3 Static analysis"])
    end
    subgraph EP["Execution Plane"]
        P8(["8 Sandbox provisioning"])
        P9(["9 Test execution & runtime exploration - Playwright"])
    end
    DE[(Evidence store)]

    GIT -- events --> P1
    P1 -- ChangeSet --> P2
    D1 -- trigger rules --> P1
    P2 -- analyze job --> DQ --> P3
    GIT -- source at SHA --> P3
    P3 -- facts: nodes, edges, tests, env hints --> D2
    D2 -- graph --> P5
    P1 -- changed files --> P5
    D4 -- coverage, history --> P5
    D3 -- scoped claims --> P5
    P5 -- ImpactSet --> P6
    D4 -- tests, durations, flakiness --> P6
    P6 -- uncovered flows --> P7
    P7 -- proposed/derived tests --> D4
    P7 -- prompts --> P11
    P6 -- test plan --> P2
    P2 -- execute job --> DQ --> P8
    D1 -- env profile, policy --> P8
    DV -- scoped secret bundle --> P8
    P8 -- healthy sandbox --> P9
    P9 -- artifacts --> DE
    P9 -- results, observations --> P10
    P9 -- runtime edges, UI states --> D2
    D4 -- baselines --> P10
    D3 -- intent claims --> P10
    P10 -- non-PASS context --> P11
    P11 -- redacted prompts --> LLM
    LLM -- structured output --> P11
    P11 -- diagnosis, judgements --> P10
    P10 -- verdicts, findings --> D4
    P10 -- findings --> P12
    DE -- evidence refs --> P12
    P12 -- status, comment --> GIT
    P12 -- report --> DEV
    DEV -- resolutions --> P13
    GIT -- comment commands --> P13
    P13 -- decisions --> D3
    P13 -- resolutions, baseline acceptance --> D4
```

---

## 3. Level 2 DFDs

### 3.1 Repository ingestion (onboarding)

```mermaid
flowchart LR
    U[User]
    GIT[GitHub / GitLab]
    I1(["Install App / connect repo"])
    I2(["Mint read-only clone token"])
    I3(["Clone at default-branch HEAD - Analysis Tier"])
    I4(["Detect applications & frameworks"])
    I5(["Run analyzer plugins"])
    I6(["Detect environment"])
    I7(["Validate & size-cap facts"])
    I8(["Create Snapshot 1"])
    I9(["Import existing tests"])
    D1[(Config)]
    D2[(Application Brain)]
    D4[(Test Memory)]

    U -- repo selection --> I1
    I1 -- installation --> D1
    GIT -- installation token --> I2
    I2 -- token --> I3
    GIT -- source --> I3
    I3 -- file tree --> I4 -- apps --> I5
    I3 -- config files --> I6
    I5 -- nodes/edges --> I7
    I6 -- env proposal --> I7
    I7 -- facts --> I8 --> D2
    I7 -- env proposal --> D1
    I7 -- test files --> I9 --> D4
    D1 -- proposal --> U
```

### 3.2 Git diff analysis

```mermaid
flowchart LR
    CS[(ChangeSet)]
    G1(["Resolve base / merge-base / head"])
    G2(["Fetch diff: base...head"])
    G3(["Classify changed files: code / config / schema / style / test / docs"])
    G4(["Map hunks to enclosing symbols"])
    G5(["Derive change kinds: signature / body / add / remove / rename"])
    G6(["Emit seeds"])
    D2[(Application Brain: snapshot at merge-base)]
    OUT[(ChangeSet.changedSymbols + seeds)]

    CS -- SHAs --> G1 -- range --> G2 -- hunks --> G3
    G3 -- code hunks --> G4
    D2 -- symbol locations --> G4
    G4 -- symbols --> G5 --> G6
    G3 -- non-code changes --> G6
    G6 --> OUT
```

### 3.3 Project Brain synchronization

```mermaid
flowchart TB
    EV[Sync trigger: push / PR / schedule / analyzer upgrade]
    S1(["Pick parent snapshot"])
    S2(["Incremental analysis of changed + dependent files"])
    S3(["Diff nodes/edges vs parent"])
    S4(["Tombstone & integrity check"])
    S5(["Write snapshot: canonical or PR overlay"])
    S6(["Promote runtime knowledge seen on default branch"])
    S7(["Decay stale runtime edges"])
    S8(["Nightly: full re-analysis & drift metric"])
    D2[(Application Brain)]
    D4[(Test Memory: runs)]

    EV --> S1
    D2 -- parent --> S1 --> S2 --> S3 --> S4 --> S5 --> D2
    D4 -- runtime observations --> S6 --> D2
    D2 -- runtime edges --> S7 --> D2
    S8 -- divergence --> D2
```

### 3.4 Runtime exploration

```mermaid
flowchart LR
    EC(["Exploration controller - Control Plane"])
    EN(["Execution engine - sandbox"])
    FP(["Fingerprint & dedupe"])
    PR(["Prioritize frontier"])
    POL(["ActionPolicy check"])
    AIR(["AI rank - optional, bounded"])
    D2[(Application Brain)]
    D1[(Action policy)]
    DE[(Evidence)]

    D2 -- seeds, impacted routes, static edges --> PR
    PR -- next candidate --> POL
    D1 -- rules --> POL
    POL -- allowed action --> EC -- structured action --> EN
    EN -- state capture: a11y, network, console --> FP
    EN -- screenshots --> DE
    FP -- new state + candidates --> PR
    PR -- candidate list --> AIR -- ranked IDs --> PR
    FP -- states, transitions, invokes edges, proposed flows --> D2
```

### 3.5 Application graph construction

```mermaid
flowchart TB
    SRC[Source files]
    OBS[(Runtime observations)]
    A1(["Parse: tree-sitter / TS compiler, no emit"])
    A2(["Extract routes, endpoints, models, tests, external deps"])
    A3(["Resolve imports & calls"])
    A4(["Cluster features"])
    R1(["Normalize URLs to endpoint patterns"])
    R2(["Build runtime edges: invokes, transitions_to, tests"])
    C1(["Compare static vs runtime"])
    D2[(Graph: static layer)]
    D2R[(Graph: runtime layer)]
    DD[(Discrepancies)]

    SRC --> A1 --> A2 --> A3 --> A4 --> D2
    OBS --> R1 --> R2 --> D2R
    D2 --> C1
    D2R --> C1 --> DD
```

### 3.6 Change-impact analysis

```mermaid
flowchart LR
    SEEDS[(Seeds)]
    GR[(Graph @ merge-base + PR overlay)]
    COV[(Runtime coverage map)]
    INT[(Intent claims)]
    HIST[(Failure history)]
    C1(["Weighted reverse traversal"])
    C2(["Broad-impact guard"])
    C3(["Aggregate: endpoints, flows, features"])
    C4(["Attach intent"])
    C5(["Map to tests"])
    C6(["Risk score"])
    OUT[(ImpactSet with explanations)]

    SEEDS --> C1
    GR --> C1 --> C2 --> C3 --> C4 --> C5 --> C6 --> OUT
    INT --> C4
    COV --> C5
    HIST --> C6
```

### 3.7 Test execution

```mermaid
flowchart LR
    PLAN[(Test plan)]
    ENV[(Env profile)]
    SEC[(Scoped secrets)]
    E1(["Provision microVM"])
    E2(["Build, start, seed, health-check"])
    E3(["Login per role - storageState"])
    E4(["Run test: DSL or imported spec"])
    E5(["Validate actions & policy"])
    E6(["Capture evidence"])
    E7(["Redact"])
    E8(["Reset data per isolation strategy"])
    DE[(Evidence store)]
    RES[(Results & observations)]

    ENV --> E1
    SEC --> E1
    E1 --> E2 --> E3 --> E4
    PLAN --> E4
    E4 <--> E5
    E4 --> E6 --> E7 --> DE
    E7 --> RES
    E4 --> E8 --> E4
```

### 3.8 Failure analysis

```mermaid
flowchart TB
    TR[(Failed TestResult)]
    OB[(Network, console, DOM)]
    DF[(Diff hunks)]
    IM[(Impact paths)]
    BL[(Baseline)]
    HI[(History)]
    F1(["Failure signature: normalize error, status, step"])
    F2(["Noise checks: infra, third-party, test defect, retry"])
    F3(["Correlate: failing endpoint to handler to changed symbols"])
    F4(["Assemble minimal context"])
    F5(["AI diagnose - cites evidence"])
    F6(["Validate citations; strip unsupported"])
    OUT[(Finding analysis)]

    TR --> F1
    OB --> F1
    F1 --> F2
    HI --> F2
    F2 -- "not noise" --> F3
    IM --> F3
    DF --> F3
    BL --> F4
    F3 --> F4 --> F5 --> F6 --> OUT
    F2 -- "noise verdict" --> OUT
```

### 3.9 Behavior-change classification

```mermaid
flowchart TB
    OBS[(Current observation digest)]
    BL[(Baseline digest)]
    IC[(Intent claims + decisions)]
    CAL[(Calibration thresholds)]
    B1(["Per-dimension diff"])
    B2(["Noise classification"])
    B3(["Retrieve scoped claims"])
    B4(["Deterministic structured checks"])
    B5(["AI free-text judgement - capped weight"])
    B6(["Aggregate S and C"])
    B7(["Verdict rules + confidence"])
    OUT[(BehaviorChange with verdict)]

    OBS --> B1
    BL --> B1
    B1 --> B2 --> B3
    IC --> B3
    B3 --> B4 --> B6
    B3 --> B5 --> B6
    CAL --> B7
    B6 --> B7 --> OUT
```

### 3.10 Developer confirmation

```mermaid
flowchart LR
    BC[(UNKNOWN / REGRESSION_CANDIDATE findings)]
    DEV[Developer]
    GIT[PR/MR]
    K1(["Group & rank questions, max 3 in PR"])
    K2(["Render prompt with before/after & evidence"])
    K3(["Receive resolution: dashboard / comment / API"])
    K4(["Authorize actor"])
    K5(["Apply: Decision, regression record, false-positive, ignore"])
    K6(["Recompute run verdict & update check"])
    IB[(Intent Brain)]
    TM[(Test Memory)]
    AU[(Audit log)]

    BC --> K1 --> K2 --> GIT
    K2 --> DEV
    DEV --> K3
    GIT --> K3
    K3 --> K4 --> K5
    K5 --> IB
    K5 --> TM
    K5 --> AU
    K5 --> K6 --> GIT
```
