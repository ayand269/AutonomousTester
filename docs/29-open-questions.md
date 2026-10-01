# 29 — Open Questions

**Purpose:** Questions to resolve before (or during) implementation. Each has a **recommended default** so work isn't blocked, and an owner type. Items marked ⚑ are also critical decisions in [30-architecture-review](./30-architecture-review.md) §6.

| ID | Question | Recommended default | Decide by | Owner |
|---|---|---|---|---|
| **Scope** |||||
| Q-SCOPE-1 ⚑ | Which Git providers are in MVP? | GitHub at M1, GitLab.com at M4 (before GA) | Phase 0 | Product |
| Q-SCOPE-2 | Which frontend frameworks are MVP? | Next.js, React SPA (React Router) | Phase 0 | Product + Eng |
| Q-SCOPE-3 | Which backend frameworks are MVP? | Express, NestJS, Next.js route handlers | Phase 0 | Product + Eng |
| Q-SCOPE-4 | Who is the design-partner cohort and how many? | 5–8 TS teams, 10–100 engineers, existing Playwright suite preferred | Phase 0 | Product |
| **Analysis** |||||
| Q-ANA-1 | Next backend language after TS? | Decide from design-partner/prospect demand (Python likely) | P4 | Product |
| Q-ANA-2 | Symbol-level or file-level call graph in MVP? | Symbol-level with fallback to file-level if spike S2 shows graphs too large/slow | Phase 0 (S2) | Eng |
| Q-ANA-3 | Impact parameters (τ, decay, hop limit) defaults? | Values in [12](./12-change-impact-analysis.md) §2.2, calibrated on fault-injection harness | M2 | Eng |
| Q-ANA-4 | How much source code should be parsed initially? | All TS/JS under detected Applications, excluding `node_modules`, build output, generated code, files > 1 MB | Phase 0 | Eng |
| **Execution** |||||
| Q-EXE-1 ⚑ | microVM per run vs pooled cloud VM per run for MVP? | microVM if S1 shows ops is manageable; pooled VM fallback otherwise (early stage: rented per-run microVMs, [31](./31-tech-stack.md) §3.4) | Phase 0 (S1) | Eng + Security |
| Q-EXE-2 | Browser matrix default? | Chromium only in MVP; Firefox/WebKit opt-in P2 | M1 | Product |
| Q-EXE-3 ⚑ | Which cloud provider(s)? | Single provider with nested-virt/bare-metal options and managed Postgres/Redis/KMS; early stage: Hetzner VPS + microVM provider, scale-up: AWS ([31](./31-tech-stack.md) §2, §5) | Phase 0 | Eng + Finance |
| Q-EXE-4 | Default exploration budgets? | [11](./11-runtime-exploration.md) §2 values, recalibrated on reference apps | M2 | Eng |
| Q-EXE-5 | How should long-running tests be handled? | Run budgets + per-test checkpoints; sharding in P2 | M1 | Eng |
| Q-EXE-6 | Private package registries / private base images? | Registry credentials as build-scoped secrets; egress allow-list entry | M1 | Eng |
| **Tests** |||||
| Q-TST-1 | Auto-accept policy for derived/generated tests? | Derived tests: auto-accept after 5 consecutive green main runs. AI-generated (P3): manual accept only | M2 / P3 | Product |
| Q-TST-2 | Write tests back to user repo via PR? | Not in MVP; opt-in P4 | P4 | Product |
| Q-TST-3 | Retry once for all failures or only suspect ones? | All failures in non-quarantined tests, once, after data reset | M3 | Eng |
| Q-TST-4 | When to support OAuth via mock IdP? | P3 | P2 | Product |
| Q-TST-5 | How should test data be generated? | Seeded generators + project seed scripts; fixtures format P2 | M1 | Eng |
| **Intent** |||||
| Q-INT-1 | Convention for in-repo intent docs (e.g. `/qa/intent/*.md`)? | Offer a lightweight convention in P2 (structured claims in Markdown front-matter) | P2 | Product |
| Q-INT-2 | Which ticket systems first? | GitHub Issues + Jira, linked from PR text | P3 | Product |
| Q-INT-3 | How should requirements enter the Intent Brain? | MVP: manual claims + PR text + OpenAPI; P3: docs/tickets with review queue | M3 | Product |
| Q-INT-4 | Figma/design in MVP? | No — P5 | Phase 0 | Product |
| Q-INT-5 | Verdict thresholds θe/θr/θc initial values? | Conservative (bias to UNKNOWN); set on eval suite at M3 | M3 | Eng |
| **AI** |||||
| Q-AI-1 ⚑ | Which AI models/providers initially? | One frontier-provider family for Large/Medium tiers + a small fast model; gateway supports ≥2 providers from day one; models proposed in [31](./31-tech-stack.md) §3.5 | Phase 0 | Eng + Legal |
| Q-AI-2 | Allow customer-supplied model keys / endpoints (BYO)? | P3, enterprise | P2 | Product |
| Q-AI-3 | Default AI data policy? | `code_snippets` with zero-retention provider terms; org can lower to `metadata_only`/`disabled` | Phase 0 | Legal + Security |
| **CI/CD** |||||
| Q-CI-1 | GitLab at M1 or M4? | M4 | Phase 0 | Product |
| Q-CI-2 | GitLab status for "needs confirmation"? | `success` + MR note flagged, unless project opts into `failed` for UNKNOWN | M4 | Product |
| Q-CI-3 | Test PR head SHA or merge commit? | Head SHA default; merge-commit option | M1 | Product + Eng |
| Q-CI-4 | Should the check be blocking by default? | Non-blocking (neutral) by default until FP rate is proven per project | M1 | Product |
| **Security / Data** |||||
| Q-SEC-1 ⚑ | Postgres RLS in addition to app-level scoping? | Yes | Phase 0 | Security |
| Q-SEC-2 | Ownership verification method for staging URLs (P4)? | DNS TXT or well-known file token | P3 | Security |
| Q-SEC-3 | How should secrets be stored? | Cloud KMS envelope encryption with per-project data keys; vault service; write-only API | Phase 0 | Security |
| Q-DATA-1 ⚑ | Initial retention policy? | [19](./19-security-and-safety.md) §6 defaults | Phase 0 | Product + Legal |
| Q-DATA-2 | Keep traces for passing tests? | Sample 10%, 7 days | M1 | Eng |
| Q-DATA-3 | Keep PR snapshots after close? | 30 days, then drop diffs (keep run metadata) | M2 | Eng |
| Q-DATA-4 | Platform fixture format? | JSON/YAML under `.autoqa/fixtures` with named queries | P2 | Eng |
| Q-DATA-5 ⚑ | Primary DB: PostgreSQL or MongoDB? | PostgreSQL | Phase 0 | Eng |
| **Cost / Business** |||||
| Q-BIZ-1 | How should users control execution cost? | Budgets at org/project/run; visible cost per run; mode per trigger | M1 | Product |
| Q-BIZ-2 | Pricing unit (runs, sandbox-minutes, seats)? | Sandbox-minutes + seats; AI included within fair-use | Pre-GA | Business |
| **Mocking** |||||
| Q-MOCK-1 | How should external services be mocked? | Egress proxy per-host modes; stub definitions in env profile; email sink; record/replay P3 | M1 | Eng |
| Q-MOCK-2 | How much runtime instrumentation is necessary? | MVP: browser + service logs + egress logs; OTel P4 | Phase 0 | Eng |
