Architecture is **multi-app / multi-service**, not one SaaS app.

# ELaB application split

```txt
1. ELaB SaaS App + Admin Console
   User-facing interaction layer.

2. ELaB Data Service
   Source-of-truth ingestion + normalization layer.

3. ELaB RSS Service
   Distribution/subscription layer.

4. ELaB Worker Service
   Synthetic Team, scoring, enrichment, alerts.
```

# Correct system flow

```txt
Primary Sources
  congress.gov
  federalregister.gov
  grants.gov
  usaspending.gov
  FEC / OpenFEC
        ↓
ELaB Data Service
  fetch → normalize → dedupe → store raw + structured
        ↓
ELaB Datastore
  Neon Postgres = structured truth
  Cloudflare R2 = raw documents / snapshots / artifacts
        ↓
Worker Service
  Synthetic Team analysis
  confidence scoring
  validation
  opportunity extraction
        ↓
ELaB Datastore
        ↓
RSS Service
  generated feeds
  subscriber-specific feeds
  public digest feeds
        ↓
SaaS App
  feed UI
  role threads
  opportunity pages
  alerts
```

This matches the PRD’s pipeline: **Import → Format → Distribute → Integrate**, and keeps the 72-hour policy-to-action promise intact.


# 5 Repos

```txt
github/
  elab-app
  elab-api
  elab-data-service
  elab-worker-service
  elab-rss-service
```

## 1. `elab-app`

**Owns:** user + admin interface.

```txt
- SaaS web app (w/ admin console)
- login/billing/settings
- Decision Feed UI
- opportunity pages
- role threads
```

Tech:

```txt
Next.js
React
TypeScript
CSS/design tokens from CUTR
Clerk/Auth.js
Stripe client
```

Deploy:

```txt
Railway
```

---

## 2. `elab-api`

**Owns:** public/internal API gateway.

```txt
- auth enforcement
- org/user access
- feed endpoints
- policy endpoints
- opportunity endpoints
- interaction endpoints
- billing webhooks
```

Tech:

```txt
NestJS
TypeScript
Prisma/Drizzle
Neon Postgres
Redis client
OpenAPI
```

It should not scrape sources or run LLM jobs.

---

## 3. `elab-data-service`

**Owns:** source-of-truth ingestion.

```txt
congress.gov
federalregister.gov
grants.gov
usaspending.gov
OpenFEC/FEC
```

Responsibilities:

```txt
fetch
normalize
dedupe
version
store raw source snapshots in R2
store structured records in Neon
emit jobs/events
```

Tech:

```txt
NestJS or plain Node worker
TypeScript
BullMQ producer
Cloudflare R2
Neon
```

---

## 4. `elab-worker-service`

**Owns:** intelligence production.

```txt
Synthetic Team runs
OpenAI/DeepSeek calls
policy → market translation
opportunity extraction
confidence scoring
Reviewer validation
Truth-Teller follow-up
alert generation
```

Tech:

```txt
NestJS worker
BullMQ
Redis
OpenAI
DeepSeek
Neon
R2 artifact writes
```

Role outputs must be justifiable, observable, and auditable, matching AiD Synthetic Team constraints.

---

## 5. `elab-rss-service`

**Owns:** feed distribution.

```txt
public RSS
private tokenized RSS
sector feeds
agency feeds
opportunity feeds
newsletter export feeds
```

Direction is fixed:

```txt
ELaB datastore → RSS Service → SaaS app / RSS readers / newsletter
```

Not:

```txt
RSS → datastore
```

Tech:

```txt
NestJS or Fastify
TypeScript
feed package
Neon
Redis cache
```

# Shared code problem

Since these are separate repos, avoid copy-paste with one shared package repo later:

```txt
elab-contracts
```

But not immediately. Start with copied OpenAPI/JSON schemas, then extract when stable.

Shared contracts:

```txt
PolicyItem
PolicyVersion
SourceRecord
RoleArtifact
FeedPost
Opportunity
ExecutionPath
ConfidenceScore
RSSFeedItem
```

# Build sequence

```txt
1. elab-data-service
2. elab-api
3. elab-worker-service
4. elab-rss-service
5. elab-app
```

Reason: truth first, access second, intelligence third, distribution fourth, interface last.

# Hard rule

Each repo gets its own:

```txt
README
.env.example
Dockerfile
Railway config
health endpoint
OpenAPI/contract docs where relevant
test suite
deployment pipeline
```

This keeps the system from becoming a hidden monolith. AiD explicitly treats architecture, API contracts, data models, edge cases, and failure modes as pre-build simulation artifacts, not afterthoughts.

# Service responsibilities

## 1. SaaS App

Purpose: **user interaction only**.

```txt
- Decision Feed
- @Role / @Team interactions
- opportunity detail pages
- execution paths
- saved feeds
- alert preferences
- account/billing
```

It should **not** fetch Congress.gov directly. It reads from ELaB APIs/datastore only.

---

## 2. Data Service

Purpose: **source-of-truth ingestion**.

```txt
congress.gov → normalized bills/laws
federalregister.gov → rules, notices, EOs
grants.gov → grant opportunities
usaspending.gov → spending traces
FEC/OpenFEC → political finance signals
```

Stores:

```txt
Neon:
  source_records
  policy_items
  policy_versions
  agencies
  funding_programs
  source_links

Cloudflare R2:
  raw API responses
  PDFs
  XML/JSON snapshots
  generated artifacts
```

This service owns truth. Everything else is downstream.

---

## 3. Worker Service

Purpose: **interpretation and enrichment**.

```txt
- run Synthetic Team sequence
- produce Policy Object
- produce Distortion Surface
- validate opportunities
- generate Execution Path
- compute confidence/risk
- create feed posts
```

The Synthetic Team must remain role-governed: one role, one pass, auditable output.

---

## 4. RSS Service

Purpose: **distribution, not ingestion**.

Correct direction:

```txt
ELaB datastore → RSS feeds → SaaS site / subscribers / external readers
```

Feed types:

```txt
/public/rss/policies
/public/rss/opportunities
/public/rss/verticals/:vertical
/public/rss/agencies/:agency
/private/rss/users/:token
/private/rss/orgs/:token
```

RSS is a product surface. It should expose curated ELaB outputs, not raw source chaos.

---

## 5. API Service

Purpose: **bounded access layer**.

```txt
GET /feed
GET /policies
GET /policies/:id
GET /opportunities
POST /interactions
POST /team-runs
GET /rss/feeds
POST /alerts
```

NestJS fits here cleanly.

---

# Deployment model on Railway

```txt
Railway Project: elab-prod

Services:
  api-service
  data-service
  worker-service
  rss-service
  saas-web
  admin-web
  redis
  cron-ingestion
```

External:

```txt
Neon Postgres
Cloudflare R2
OpenAI
DeepSeek
Stripe
Resend
Sentry
```


RSS should be a **first-class application/service**:

```txt
Data Service creates truth.
Worker Service creates interpretation.
RSS Service distributes structured intelligence.
SaaS App provides interaction.
```

This preserves separation of concerns and avoids turning the SaaS app into a bloated god process.