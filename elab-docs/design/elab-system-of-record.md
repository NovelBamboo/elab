For **system-of-record**, update the model like this:

# ELaB System-of-Record Model

ELaB is no longer “report-first.” It is a **policy-to-market intelligence datastore** that can generate reports, feeds, alerts, and execution plays from structured facts.

Core rule:

> Reports are outputs. Records are the product.

This fits the original ELaB premise: convert live policy shifts into actionable market intelligence within 72 hours, using policy breakdowns, capital reallocation maps, beneficiary rankings, execution playbooks, and risk surfaces.

---

## 1. Canonical Objects

```yaml
Policy:
  id:
  source_system:
  source_url:
  external_identifier:
  type: bill | law | executive_order | regulation | grant | agency_guidance
  title:
  status:
  jurisdiction:
  introduced_at:
  enacted_at:
  effective_at:
  updated_at:
  sponsors: []
  provisions: []
  amendments: []
  exceptions: []
  votes: []
  agencies: []
  evidence_refs: []
```

```yaml
Provision:
  id:
  policy_id:
  section:
  text:
  normalized_summary:
  affected_entities: []
  obligations: []
  permissions: []
  prohibitions: []
  evidence_refs: []
```

```yaml
Entity:
  id:
  type: agency | legislator | company | industry | lobbyist | interest_group | nonprofit | school | contractor | program
  name:
  aliases: []
  identifiers:
    bioguide_id:
    fec_id:
    sam_uei:
    ticker:
    congress_id:
  evidence_refs: []
```

```yaml
EntityRelationship:
  id:
  source_entity_id:
  target_entity_id:
  relationship_type:
    sponsors | votes_for | votes_against | regulates | funds | lobbies_for | lobbies_against | benefits_from | constrained_by | competes_with
  policy_id:
  confidence:
  evidence_refs: []
  valid_from:
  valid_to:
```

---

## 2. Funding Flow Object

```yaml
FundingFlow:
  id:
  policy_id:
  source_entity_id:
  destination_entity_id:
  mechanism:
    type: grant | contract | loan | tax_credit | voucher | formula_funding | discretionary_award | reallocation | procurement
    program_name:
    authority:
  amount:
    value:
    range_min:
    range_max:
    currency: USD
    confidence: confirmed | estimated | projected | unknown
  fiscal_year:
  time_window:
    start_date:
    end_date:
  eligibility:
    eligible_entities: []
    requirements: []
    exclusions: []
  access_path:
    application_url:
    procurement_portal:
    registration_required:
    deadline:
  status:
    proposed | authorized | appropriated | obligated | awarded | spent | expired
  evidence_refs: []
```

Funding must remain structured because ELaB’s own system constraint says policy interpretation must map to observable funding shifts.

---

## 3. Constraint Object

```yaml
Constraint:
  id:
  policy_id:
  constraint_type:
    legal | regulatory | constitutional | procedural | procurement | fiscal | political | operational | market
  name:
  description:
  jurisdiction:
  affected_entities: []
  severity: low | medium | high | critical
  likelihood: low | medium | high | unknown
  trigger_condition:
  expected_delay:
  mitigation_paths: []
  status: active | pending | litigated | resolved | uncertain
  evidence_refs: []
```

---

## 4. Time Horizon Object

```yaml
TimeHorizon:
  id:
  policy_id:
  horizon_type: immediate | near_term | medium_term | long_term | unknown
  event_anchor:
    introduced | amended | enacted | rulemaking | guidance | appropriation | grant_window | enforcement | litigation | market_response
  start_date:
  end_date:
  fiscal_years: []
  expected_effects:
    - type: funding_available | compliance_required | procurement_open | market_demand | legal_risk | beneficiary_gain
      affected_entities: []
      confidence:
  dependencies: []
  leading_indicators: []
  status: projected | active | delayed | completed | blocked
  evidence_refs: []
```

---

## 5. Derived Objects — not source of truth

These are computed from canonical records.

```yaml
MarketImpact:
  id:
  policy_id:
  generated_at:
  sectors_affected: []
  capital_flows: []
  beneficiary_rankings: []
  demand_shifts: []
  risks: []
  input_records: []
  confidence:
```

```yaml
ExecutionPlay:
  id:
  policy_id:
  generated_at:
  action:
  target_entities: []
  required_conditions: []
  dependencies: []
  constraints: []
  funding_flows: []
  time_horizon:
  risk_level:
  evidence_refs: []
```

```yaml
Report:
  id:
  policy_id:
  generated_at:
  version:
  report_type:
  sections: []
  input_records: []
  citations: []
```

Your original fields — summary, analysis, industries benefiting, companies benefiting, risks, quotes — become **report sections or derived intelligence**, not canonical records. The uploaded model already points this direction: fields that can change without the policy changing do not belong in the core model.

---

## 6. Evidence Layer

This is mandatory.

```yaml
EvidenceSource:
  id:
  source_type: primary | supplemental | internal_analysis
  publisher:
  title:
  url:
  retrieved_at:
  published_at:
  document_hash:
  reliability_score:
```

```yaml
EvidenceClaim:
  id:
  evidence_source_id:
  claim_text:
  normalized_claim:
  claim_type:
    fact | quote | estimate | forecast | opinion | vote | funding | legal_text
  extracted_entity_ids: []
  policy_id:
  confidence:
```

No claim without evidence. No report without traceability.

---

## 7. Minimal Database Tables

```text
policies
policy_versions
provisions
amendments
exceptions
votes
entities
entity_relationships
funding_flows
constraints
time_horizons
evidence_sources
evidence_claims
market_impacts
execution_plays
reports
report_versions
```

---

## 8. Correct System Boundary

ELaB should store:

```text
policy facts
entity relationships
funding movement
constraints
time horizons
evidence claims
versions
```

ELaB should generate:

```text
summaries
analysis
beneficiary rankings
company benefit predictions
criticisms
quotes packages
execution plays
reports
RSS feed items
alerts
```

---

## Final correction

Your report generator can still exist. But it becomes a **renderer** sitting on top of the datastore.

Architecture:

```text
Primary Sources
  → Ingestion
  → Evidence Claims
  → Canonical Records
  → Derived Intelligence
  → Reports / RSS / Feed / Alerts
```

That is the system-of-record version.

---


Below are Mermaid diagrams for the **ELaB system-of-record** version.

---

## 1. System Architecture

```mermaid
flowchart LR
  A[Primary Sources<br/>Congress.gov<br/>Federal Register<br/>Agency Sites<br/>USAspending.gov] --> B[Ingestion Service]

  B --> C[Evidence Extraction]
  C --> D[Evidence Claims]
  D --> E[Canonical Records]

  E --> F[Derived Intelligence]
  F --> G[Reports]
  F --> H[RSS Feed]
  F --> I[Alerts]
  F --> J[Execution Plays]

  E --> K[(ELaB Datastore<br/>Neon/Postgres)]
  D --> K
  F --> K

  L[Cloudflare R2<br/>Raw Documents] --> C
  B --> L
```

---

## 2. Source-of-Truth Data Model

```mermaid
erDiagram
  POLICY ||--o{ POLICY_VERSION : has
  POLICY ||--o{ PROVISION : contains
  POLICY ||--o{ AMENDMENT : has
  POLICY ||--o{ EXCEPTION : has
  POLICY ||--o{ VOTE : records
  POLICY ||--o{ FUNDING_FLOW : drives
  POLICY ||--o{ CONSTRAINT : limited_by
  POLICY ||--o{ TIME_HORIZON : unfolds_over
  POLICY ||--o{ MARKET_IMPACT : derives
  POLICY ||--o{ EXECUTION_PLAY : generates
  POLICY ||--o{ REPORT : renders

  ENTITY ||--o{ ENTITY_RELATIONSHIP : source
  ENTITY ||--o{ ENTITY_RELATIONSHIP : target
  ENTITY ||--o{ VOTE : casts
  ENTITY ||--o{ FUNDING_FLOW : source_or_destination

  EVIDENCE_SOURCE ||--o{ EVIDENCE_CLAIM : contains
  EVIDENCE_CLAIM }o--|| POLICY : supports
  EVIDENCE_CLAIM }o--o{ ENTITY : mentions

  REPORT ||--o{ REPORT_VERSION : has
```

---

## 3. Canonical vs Derived Boundary

```mermaid
flowchart TB
  subgraph Canonical["Canonical Records: Stored Truth"]
    P[Policy]
    PV[Policy Version]
    PR[Provision]
    AM[Amendment]
    EX[Exception]
    V[Votes]
    E[Entities]
    ER[Entity Relationships]
    FF[Funding Flows]
    C[Constraints]
    TH[Time Horizons]
    ES[Evidence Sources]
    EC[Evidence Claims]
  end

  subgraph Derived["Derived Intelligence: Recomputable"]
    MI[Market Impact]
    BR[Beneficiary Ranking]
    DS[Demand Shifts]
    EP[Execution Plays]
    RS[Risk Surface]
  end

  subgraph Rendered["Rendered Outputs"]
    R[Policy Intelligence Report]
    RSS[RSS Item]
    A[Alert]
    F[Social Decision Feed Post]
  end

  Canonical --> Derived
  Derived --> Rendered
```

Hard boundary: **store facts; generate interpretation.**

---

## 4. Funding Flow Object

```mermaid
flowchart LR
  P[Policy] --> FF[Funding Flow]

  FF --> S[Source Entity<br/>Agency / Program / Appropriation]
  FF --> D[Destination Entity<br/>State / Institution / Company / Industry]
  FF --> M[Mechanism<br/>Grant / Contract / Loan / Tax Credit / Voucher]
  FF --> A[Amount<br/>Confirmed / Estimated / Projected]
  FF --> T[Time Window<br/>FY / Start / End]
  FF --> EL[Eligibility]
  FF --> AP[Access Path<br/>SAM.gov / Grants.gov / Agency Portal]
  FF --> EV[Evidence Claims]

  FF --> STATUS[Status<br/>Proposed / Authorized / Appropriated / Awarded / Spent]
```

---

## 5. Constraint Object

```mermaid
flowchart TB
  P[Policy] --> C[Constraint]

  C --> CT[Constraint Type<br/>Legal / Regulatory / Fiscal / Political / Operational]
  C --> J[Jurisdiction<br/>Federal / State / Local]
  C --> AE[Affected Entities]
  C --> S[Severity]
  C --> L[Likelihood]
  C --> TR[Trigger Condition]
  C --> DELAY[Expected Delay]
  C --> M[Mitigation Paths]
  C --> EV[Evidence Claims]

  C --> STATUS[Status<br/>Active / Pending / Litigated / Resolved / Uncertain]
```

---

## 6. Time Horizon Object

```mermaid
flowchart LR
  P[Policy] --> TH[Time Horizon]

  TH --> H[Horizon Type<br/>Immediate / Near / Medium / Long]
  TH --> EA[Event Anchor<br/>Introduced / Enacted / Guidance / Rulemaking / Grant Window]
  TH --> W[Window<br/>Start / End / Fiscal Years]
  TH --> EE[Expected Effects]
  TH --> DEP[Dependencies]
  TH --> LI[Leading Indicators]
  TH --> EV[Evidence Claims]

  EE --> FA[Funding Available]
  EE --> CR[Compliance Required]
  EE --> PO[Procurement Open]
  EE --> MD[Market Demand]
  EE --> LR[Legal Risk]
```

---

## 7. Report Generation Pipeline

```mermaid
sequenceDiagram
  participant S as Source
  participant I as Ingestion Service
  participant E as Evidence Extractor
  participant DB as ELaB Datastore
  participant D as Intelligence Engine
  participant R as Report Renderer
  participant U as User Feed

  S->>I: New policy / update detected
  I->>E: Fetch + normalize document
  E->>DB: Store evidence source + claims
  E->>DB: Update canonical records
  DB->>D: Provide policy, entities, funding, constraints, horizons
  D->>DB: Store derived intelligence
  D->>R: Generate report sections
  R->>DB: Store report version
  R->>U: Publish feed item / RSS / alert
```

---

## 8. ELaB Multi-Service Architecture

```mermaid
flowchart TB
  subgraph DataService["elab-data-service"]
    DS1[Source Crawlers]
    DS2[Document Fetchers]
    DS3[Evidence Extractors]
    DS4[Canonical Record Writers]
  end

  subgraph API["elab-api"]
    API1[Policy API]
    API2[Entity API]
    API3[Report API]
    API4[Feed API]
  end

  subgraph Worker["elab-worker-service"]
    W1[Intelligence Jobs]
    W2[Report Generation]
    W3[Embeddings]
    W4[Alert Matching]
  end

  subgraph App["elab-app"]
    APP1[Decision Feed]
    APP2[Report Reader]
    APP3[Synthetic Team UI]
    APP4[Admin Review]
  end

  subgraph RSS["elab-rss-service"]
    RSS1[Feed Builder]
    RSS2[Subscriber Delivery]
  end

  DB[(Neon Postgres)]
  R2[(Cloudflare R2)]
  Q[Job Queue]

  DataService --> DB
  DataService --> R2
  DataService --> Q
  Worker --> DB
  Worker --> Q
  API --> DB
  App --> API
  RSS --> DB
  RSS --> API
```

---

## 9. Social Decision Feed Model

```mermaid
flowchart TB
  P[Policy Update] --> I[Intelligence Engine]

  I --> A[Analyst Post<br/>What changed?]
  A --> B[Builder Post<br/>What can be built?]
  B --> R[Reviewer Post<br/>What breaks?]
  R --> O[Operator Post<br/>What actions now?]
  O --> T[Truth-Teller Post<br/>What is overstated?]

  T --> F[Threaded Decision Feed]

  U[User] --> F
  U --> M[@Mention Role]
  M --> A
  M --> B
  M --> R
  M --> O
  M --> T
```

This matches the synthetic-role constraint: one role, one pass, auditable output.

---

## 10. AiD Fit for ELaB

```mermaid
flowchart LR
  I[Ideation<br/>Premise + Boundaries] --> S[Simulation<br/>Data Models + UX + APIs]
  S --> E[Education<br/>User + Builder Onboarding]
  E --> P[Promotion<br/>Public Narrative + Pricing]
  P --> B[Build<br/>Repos + Services + QA]

  S -.locks.-> DB[System-of-Record Model]
  E -.validates.-> UX[Decision Feed UX]
  P -.mirrors.-> R[Report Product]
  B -.implements.-> SYS[Production ELaB]
```

AiD requires simulation artifacts before build, with API/data models, unknowns, and behavior locked before implementation.