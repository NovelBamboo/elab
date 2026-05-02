# ELaB Architecture 

```mermaid
flowchart LR
    A[Primary Sources] --> B[Data Service]
    A2[Supplemental Sources] --> B

    B --> C[(Neon DB)]
    B --> D[(Cloudflare R2)]

    C --> E[RSS Service]
    C --> F[ELaB API]
    D --> F

    E --> F

    F --> G[SaaS App]
    F --> H[Admin Panel]
```

## Service Boundary

### 1. Data Service

Owns ingestion, normalization, enrichment, and persistence.

Responsibilities:

- ingest primary + supplemental sources
    
- store raw artifacts in Cloudflare R2
    
- normalize records into Neon
    
- run LLM extraction/enrichment inline or scheduled
    
- produce canonical policy intelligence objects
    
- mark records as feed-ready
    

This becomes the **source-of-truth engine**, not just ingestion.

---

### 2. Neon DB

Stores:

- normalized policy records
    
- actor entities
    
- funding flows
    
- constraints
    
- time horizons
    
- enrichment status
    
- feed eligibility
    

---

### 3. Cloudflare R2

Stores:

- raw PDFs
    
- raw HTML snapshots
    
- source extracts
    
- generated report artifacts, if needed
    

---

### 4. RSS Service

Owns distribution formatting only.

Responsibilities:

- read feed-ready records from Neon
    
- generate topic feeds
    
- deduplicate feed entries
    
- cache RSS output
    

Constraint:

> RSS Service does not enrich, score, or interpret.

---

### 5. ELaB API

Owns product access.

Responsibilities:

- expose policy records
    
- expose feed entries
    
- support SaaS app interaction
    
- provide AI chat / advisory endpoints
    
- query Neon + R2
    

Constraint:

> API does not mutate canonical policy truth except user-layer artifacts.

---

### 6. SaaS App + Admin

SaaS App:

- social-feed interaction
    
- @role AI advisory
    
- threads
    
- saved reports
    

Admin:

- source monitoring
    
- ingestion status
    
- record inspection
    
- manual correction workflow
    

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
    participant UI as SaaS/Admin

    S->>D: Raw source material
    D->>R2: Store immutable raw artifact
    D->>DB: Normalize canonical record
    D->>DB: Enrich + score + mark feed-ready

    DB->>RSS: Read feed-ready records
    RSS->>API: Feed output / cached feed

    DB->>API: Policy intelligence data
    R2->>API: Raw/source artifacts
    API->>UI: Structured product experience
```

## Repo Layout

```text
elab-data-service
elab-api
elab-rss-service
elab-app
```
## Rule

> Data Service owns truth production.  
> RSS Service owns syndication.  
> API owns access.  
> App owns interaction.

