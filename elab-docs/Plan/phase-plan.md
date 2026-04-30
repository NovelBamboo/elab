Implement codebase for /Users/novelbamboo/Desktop/github/elab from the existing /Users/novelbamboo/Desktop/github/cutr codebase.

DO NOT make any changes to `/Users/novelbamboo/Desktop/github/cutr`

## Multi-phase implementation plan for `/elab`

Core correction: do **not** migrate CUTR as a desktop app. Use CUTR as a source library for role logic, prompt-store patterns, UI primitives, and Synthetic Team discipline. Rebuild ELaB as a hosted SaaS.

Sources: CUTR→ELaB gap design says only ~15% is directly reusable, ~30% adaptable, and ~55% net-new; the PRD defines ELaB as a 72-hour policy-to-market decision engine; AiD requires simulation before build, not discovery during implementation.

---

# Target architecture

```txt
Frontend:        Next.js / React / TypeScript
API:             NestJS / TypeScript
DB:              Neon Postgres
ORM:             Prisma or Drizzle
Queue:           BullMQ + Redis
Hosting:         Railway
Workers:         Railway services + cron jobs
LLMs:            OpenAI + DeepSeek adapter layer
Ingestion:       RSS + Congress.gov + Federal Register + Grants.gov + USAspending
Search/RAG:      pgvector first; dedicated vector DB later only if needed
Auth:            Clerk or Auth.js
Payments:        Stripe
Email:           Resend
Observability:   Sentry + OpenTelemetry + Logtail/Axiom
Testing:         Vitest + Playwright
```

Railway supports cron-style services that run and exit; Neon is serverless Postgres with branching/autoscaling; NestJS officially supports BullMQ queues; OpenAI Structured Outputs should be used for schema-bound role artifacts. ([Railway Docs](https://docs.railway.com/cron-jobs?utm_source=chatgpt.com "Cron Jobs | Railway Docs"))

---

# Phase 0 — Repository split and migration baseline

**Goal:** make `/elab` a clean SaaS repo, not a mutated CUTR desktop app.

Create:

```txt
/elab
  apps/
    web/              # Next.js
    api/              # NestJS
    worker/           # NestJS worker or shared app entry
  packages/
    ui/               # CUTR-adapted CSS/components
    core/             # domain types, role contracts, scoring
    prompts/          # prompt-store port
    ingest/           # source adapters
  infra/
    railway/
    db/
```

Actions:

1. Copy reusable CUTR theme tokens and generic components.
    
2. Delete all Tauri/Rust assumptions.
    
3. Port CUTR prompt-store concept into TypeScript.
    
4. Convert CUTR archetypes into ELaB role cards.
    
5. Freeze an initial ELaB domain model before coding features.
    

CUTR’s Synthetic Team structure is the best asset to carry forward; the PRD requires role-attributed outputs with Analyst, Builder, Reviewer, Operator, Truth-Teller, and Facilitator.

---

# Phase 1 — Data model and backend foundation

**Goal:** build the system spine.

Neon tables:

```txt
users
organizations
subscriptions

sources
source_documents
policy_items
policy_versions

synthetic_team_runs
role_artifacts
validation_results
decision_threads
feed_posts

opportunities
execution_paths
funding_traces
confidence_scores
risk_surfaces

alerts
notifications
audit_events
llm_calls
```

NestJS modules:

```txt
AuthModule
UsersModule
PoliciesModule
SourcesModule
IngestionModule
SyntheticTeamModule
FeedModule
OpportunitiesModule
ScoringModule
AlertsModule
BillingModule
AuditModule
```

Non-negotiable backend rule: **no artifact without source trace, role, confidence, and validation state.**

---

# Phase 2 — Ingestion pipeline

**Goal:** turn public policy sources into normalized policy objects.

Build adapters:

```txt
RSS adapter
Congress.gov adapter
Federal Register adapter
Grants.gov adapter
USAspending adapter
Manual upload / admin ingest
```

Pipeline:

```txt
fetch source
→ normalize
→ dedupe
→ store raw document
→ extract metadata
→ enqueue policy-analysis job
```

Use Railway cron for scheduled fetchers and BullMQ for downstream processing. Keep ingestion boring. The hard part is interpretation, not scraping.

---

# Phase 3 — Synthetic Team engine

**Goal:** port CUTR’s role-governed engine into ELaB’s policy domain.

Run sequence:

```txt
Analyst
→ Builder
→ Reviewer
→ Operator
→ Truth-Teller
→ Facilitator interrupt/lock authority
```

Each role emits one typed artifact:

```txt
Analyst:      Policy Object
Builder:      Distortion Surface / Opportunity Graph
Reviewer:     Validation Outcome
Operator:     Live Arbitrage Feed Post / Execution Path
Truth-Teller: Outcome Delta Report
Facilitator:  Decision Resolution / Kill Event
```

Use provider abstraction:

```txt
LlmProvider
  - OpenAiProvider
  - DeepSeekProvider
  - MockProvider
```

Use JSON schema / structured outputs for every role artifact. Freeform model output should be treated as invalid.

The AiD Synthetic Team rules require one role per pass, inspectable outputs, and explicit accountability.

---

# Phase 4 — Decision Feed UX

**Goal:** social feed, not dashboard.

Build these web surfaces:

```txt
/feed
/policies
/policies/[id]
/opportunities
/opportunities/[id]
/threads/[id]
/roles
/admin/runs
/settings
/billing
```

Feed item shape:

```txt
role avatar
role name
claim
artifact summary
confidence
source links
next action
thread state: validated / under review / killed
```

Interaction modes:

```txt
Post to feed        → full team responds
@Role mention       → one role responds
@Team mention       → formal sequence
```

The PRD is explicit: ELaB’s interface is a role-attributed decision feed with disagreement visibility, confidence decay, kill events, and outcome scorecards.

---

# Phase 5 — Opportunity and execution engine

**Goal:** every policy must create action or be discarded.

Implement:

```txt
Opportunity ranking
Funding trace
Risk surface
Capital intensity
Time horizon
Execution path
Actor/action/mechanism validation
```

Execution path minimum:

```txt
actor
action
mechanism
dependencies
first 3 steps
source basis
confidence
kill criteria
```

Hard rule from PRD: if there is no executable action, discard the output.

---

# Phase 6 — Trust layer

**Goal:** prevent hallucinated arbitrage.

Build:

```txt
claim-level citations
source-distance scoring
inference-distance scoring
Reviewer rejection flow
Facilitator kill events
Truth-Teller outcome tracking
audit log
```

Confidence should mean:

```txt
source strength
+ inference distance
+ historical confirmation
- policy reversal risk
- missing funding trace
```

Do **not** make confidence equal “model confidence.” That is fake precision.

---

# Phase 7 — SaaS layer

**Goal:** make it chargeable.

Implement:

```txt
Auth
Organizations
Stripe subscriptions
Tier gates
Usage metering
Email alerts
Saved sectors
Saved policies
Digest preferences
```

Suggested gates:

```txt
Free:       digest + limited feed
Operator:   execution paths + alerts
Allocator:  deeper risk / capital ranking
Enterprise: custom sectors + priority runs
```

Matches PRD pricing/tier logic.

---

# Phase 8 — Admin and operations

**Goal:** make the system operable by one small team.

Admin surfaces:

```txt
ingestion health
queue health
failed jobs
policy review queue
role artifact inspector
LLM cost ledger
manual rerun
source freshness
kill-event log
```

Add:

```txt
Sentry
OpenTelemetry
structured logs
LLM cost tracking
queue retry policy
dead-letter queue
```

---

# Phase 9 — MVP launch scope

Do **not** launch all verticals.

Pick one:

```txt
Transportation / Infrastructure
Energy
Healthcare Compliance
Defense Contracting
Agriculture
AI / Data Privacy / Cybersecurity
```

The PRD says single vertical at launch is mandatory.

MVP must ship:

```txt
1 vertical
3–5 ingestion sources
policy feed
synthetic team run
opportunity output
execution path
confidence/risk
source trace
email alert
Stripe paid tier
admin rerun
```

Defer:

```txt
community features
complex dashboards
full financial models
multi-country support
automated trading/investing
```

---

# Build order

```txt
1. repo scaffold
2. Neon schema
3. NestJS API
4. queue + worker
5. ingestion adapters
6. prompt-store port
7. synthetic team engine
8. feed API
9. web feed UI
10. opportunity pages
11. auth + billing
12. trust/validation layer
13. admin console
14. launch vertical
```

---

# Blunt implementation stance

Use CUTR as a **donor**, not a foundation.

Keep:

```txt
theme tokens
prompt-store pattern
role sequencing
validation panels
run history concepts
Synthetic Team discipline
```

Rewrite:

```txt
backend
storage
routing
auth
deployment
policy domain types
feed UX
ingestion
billing
alerts
RAG/search
```

Remove:

```txt
Tauri
Rust backend
desktop filesystem model
native shell APIs
project/chat-room assumptions
```

The build should start only after the data model, role artifacts, feed contract, and first vertical are locked. Otherwise ELaB will become exactly what AiD warns against: fast motion without system clarity.

keep track of what phases of implementation has been completed. Track using phase-tracking.md

Complete one phase at a time. Check in once phase is completed. Wait for instructions before moving to next phase. You must be explicitly told to continue to next phase.