# ELaB Application Architecture

Architecture is **multi-app / multi-service**.

---

## ELaB Application Split

```txt
1. ELaB SaaS & Admin Console App
   User-facing interaction layer

2. ELaB API
   Access + orchestration layer

3. ELaB Data Service
   Source-of-truth + intelligence production

4. ELaB RSS Service
   Distribution/subscription layer
```

---

## System Flow

```txt
Primary Sources
  congress.gov
  federalregister.gov
  grants.gov
  usaspending.gov
  FEC / OpenFEC
        ↓
ELaB Data Service
  fetch → normalize → dedupe → enrich → score → store
        ↓
ELaB Datastore
  Neon Postgres = structured truth
  Cloudflare R2 = raw documents / snapshots / artifacts
        ↓
RSS Service
  generated feeds
  subscriber-specific feeds
  public digest feeds
        ↓
ELaB API
        ↓
SaaS App
  feed UI
  role threads
  opportunity pages
  alerts
```

> **Interpretation happens inside Data Service. Not in a separate worker.**

---

## Repo Structure

```txt
github/
  elab-app
  elab-api
  elab-data-service
  elab-rss-service
```

---

## 1. `elab-apps

**Owns:** user & admin interface.

```txt
- Decision Feed UI (social-first, not dashboard)
- @Role / @Team interactions
- opportunity pages
- execution paths
- alerts + subscriptions
- account / billing
```

Constraints:

```txt
- never fetch external sources
- never compute policy meaning
```

---

## 2. `elab-api`

**Owns:** access layer.

```txt
- auth + org/user scoping
- feed endpoints
- policy endpoints
- opportunity endpoints
- interaction endpoints
- alert endpoints
- billing webhooks
```

Constraints:

```txt
- no ingestion
- no enrichment
```

It exposes—not creates—truth.

---

## 3. `elab-data-service` (Critical Change)

**Owns BOTH:**

- ingestion
    
- intelligence production
    
---

### Responsibilities

```txt
INGESTION
- fetch primary + supplemental sources
- normalize records
- dedupe + version
- store raw snapshots in R2
- store structured records in Neon

INTELLIGENCE
- run Synthetic Team logic inline or scheduled
- policy → market translation
- opportunity extraction
- funding flow mapping
- constraint identification
- time horizon modeling
- confidence scoring
- generate feed-ready objects
```

---

### Internal Execution Model

```txt
- async job queue (BullMQ / equivalent)
- cron triggers
- event-driven processing (DB insert/update hooks)
```

This keeps compute **internal but asynchronous**.

---

### Storage

```txt
Neon:
  source_records
  policy_items
  policy_versions
  actors
  funding_flows
  opportunities
  constraints
  time_horizons
  confidence_scores
  feed_posts

Cloudflare R2:
  raw API responses
  PDFs
  HTML snapshots
  generated artifacts
```

---

### Hard Constraint

> Data Service defines system truth. Everything else reads it.

---

## 4. `elab-rss-service`

**Owns:** distribution only.

---

### Responsibilities

```txt
- convert feed_posts → RSS items
- generate segmented feeds
- deduplicate entries
- cache feed responses
```

---

### Feed Types

```txt
/public/rss/policies
/public/rss/opportunities
/public/rss/verticals/:vertical
/public/rss/agencies/:agency
/private/rss/users/:token
/private/rss/orgs/:token
```

---

### Constraint

```txt
- no enrichment
- no scoring
- no interpretation
```

---

## Data Flow

```mermaid
sequenceDiagram
    participant S as Source
    participant D as Data Service
    participant R2 as Cloudflare R2
    participant DB as Neon DB
    participant RSS as RSS Service
    participant API as ELaB API
    participant UI as SaaS

    S->>D: Raw source data
    D->>R2: Store immutable raw artifact
    D->>DB: Normalize + store structured record

    D->>DB: Enrich (LLM + logic)
    D->>DB: Score + generate feed_post

    DB->>RSS: Read feed-ready data
    RSS->>API: Feed output

    DB->>API: Policy + opportunity data
    API->>UI: Structured experience
```

---

## Deployment Model

```txt
Railway/Netlify Project: elab-prod

Services:
  api-service
  data-service
  rss-service
  saas-web
  admin-web
  redis (for internal jobs)

NO worker-service
```

External:

```txt
Neon Postgres
Cloudflare R2
OpenAI / DeepSeek
Stripe
Resend
Sentry
```

---

## Hard Rule

```txt
Data Service:
  owns truth + interpretation

RSS Service:
  owns distribution/syndication

API:
  owns access

App:
  owns user experience
```

---

## Check (AiD Compliance)

If you cannot answer these immediately, you’re still pre-simulation:

- What triggers enrichment? (event vs cron)
    
- When is a record “feed-ready”?
    
- How is confidence scored?
    
- What invalidates a previous interpretation?
    

Unanswered = latent SITD risk

---

# Next Steps

- DB schema (tables + relationships)
    
- API contract (real endpoints + payloads)
    
- feed scoring model
