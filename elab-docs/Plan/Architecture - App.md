Good call. Mixing operator controls with user experience will rot fast.

Here’s the **clean separation**—no shared surface, no blurred responsibility.

---

# ELaB Architecture (Final — UX Split Applied)

## Application Split

```txt
1. elab.app.com        (User product)
2. elab.admin.com      (Operator console)

3. ELaB API
4. ELaB Data Service
5. ELaB RSS Service
```

---

## Updated System Flow

```txt
Primary Sources
        ↓
ELaB Data Service
        ↓
ELaB Datastore (Neon + R2)
        ↓
RSS Service
        ↓
ELaB API
      ↓     ↓
elab.app   elab.admin
```

---

## Repo Structure

```txt
github/
  elab-app           → elab.app.com
  elab-admin         → elab.admin.com
  elab-api
  elab-data-service
  elab-rss-service
```

---

# UX Layer Separation (Critical)

## 1. `elab.app.com` (User Product)

**Purpose:** decision consumption + interaction

### Responsibilities

```txt
- Decision Feed (primary surface)
- @role / @team interactions
- policy pages
- opportunity pages
- execution paths
- saved feeds
- alerts
- billing + account
```

### Behavior Model

```txt
feed-first
threaded responses
AI roles visible as actors
artifact-driven UI (not raw text blobs)
```

### Constraints

```txt
- no ingestion visibility
- no system internals
- no manual overrides
- no job control
```

> This is a product. Not a control panel.

---

## 2. `elab.admin.com` (Operator Console)

**Purpose:** system observability + control

### Responsibilities

```txt
INGESTION
- source health (congress.gov, etc.)
- ingestion logs
- failed fetch retries

DATA
- inspect raw records
- inspect normalized records
- version history

INTELLIGENCE
- view enrichment outputs
- inspect confidence scores
- diff interpretations

SYSTEM
- job queue status
- cron triggers
- re-run enrichment
- manual override (audited)

GOVERNANCE
- schema inspection
- migration visibility
- audit logs
```

---

### Key Interfaces

```txt
/ingestion
/policies/:id/raw
/policies/:id/normalized
/policies/:id/enriched
/jobs
/feeds
/system
```

---

### Constraints

```txt
- no user-facing UX concerns
- no feed presentation logic
- no marketing or narrative layer
```

> This is an operations surface. Not a product.

---

# API Interaction Model

## API serves BOTH apps, but differently

---

### elab.app.com usage

```txt
GET /feed
GET /policies
GET /policies/:id
GET /opportunities
POST /interactions
POST /alerts
```

---

### elab.admin.com usage

```txt
GET /admin/ingestion/logs
GET /admin/policies/:id/raw
GET /admin/policies/:id/normalized
GET /admin/policies/:id/enriched
POST /admin/recompute/:id
GET /admin/jobs
GET /admin/system/health
```

---

## Hard Separation Rule

```txt
/admin/* endpoints are NEVER exposed to elab.app
/public endpoints are NEVER used for admin control
```

Break this → security + architectural failure.

---

# Data Service (Unchanged but Re-contextualized)

Now supports **two consumers**:

```txt
1. Product (via API → elab.app)
2. Operators (via API → elab.admin)
```

---

## Internal State Must Be Visible (Admin Only)

Every record should expose:

```txt
raw → normalized → enriched → feed-ready
```

If you can’t inspect all four layers:

> You’ve built an opaque system → guaranteed SITD later

---

# RSS Service (No Change)

Still:

```txt
datastore → RSS → API → apps
```

Admin may:

```txt
- inspect feeds
- invalidate cache
```

But not generate logic.

---

# Deployment Model (Updated)

```txt
Railway Project: elab-prod

Services:
  api-service
  data-service
  rss-service
  redis

Frontend:
  elab-app (elab.app.com)
  elab-admin (elab.admin.com)
```

---

# Access Control Model

## Roles

```txt
USER
- access elab.app only

ADMIN / OPERATOR
- access elab.admin
- may access elab.app optionally

SYSTEM
- internal service auth only
```

---

# What This Fixes

### 1. Prevents product contamination

Users never see system complexity.

### 2. Prevents operator blindness

Admins can inspect full pipeline.

### 3. Enables real debugging

You can trace:

```txt
source → raw → normalized → enriched → feed
```

### 4. Keeps UX clean

No dashboards leaking into product.

---

# What You Must Decide Next

This split forces clarity on:

1. **What is editable in admin?**
    
    - can admins override enrichment?
        
    - or only re-run?
        
2. **What is immutable?**
    
    - raw sources? (should be yes)
        
3. **Audit requirements**
    
    - every override must be logged?
        
4. **Failure handling**
    
    - what happens when enrichment fails?
        

---

# Blunt Assessment

Before:

> You had a system diagram.

Now:

> You have an **operable system with control surfaces**.

If admin isn’t built early, you’ll end up blind and debugging via DB queries.

That’s how systems quietly fail.