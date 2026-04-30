Yes. Update the model around **publication**, not normalized graph purity.

# ELaB Publication Engine Model

```yaml
PublicationReport:
  id:
  slug:
  title:
  subtitle:
  policy_subject:
    name:
    type: bill | law | executive_order | regulation | grant | agency_action
    identifier:
    status:
  publish_status:
    draft | review | published | archived
  audience:
    primary:
    secondary:
  thesis:
  executive_summary:
  report_sections:
    policy_summary:
    timeline:
    key_provisions:
    amendments:
    exceptions:
    historical_precedents:
    stakeholder_map:
    funding_flows:
    market_impact:
    institutional_changes:
    execution_opportunities:
    constraints:
    criticisms_risks:
    quotes:
    notes_references:
  source_pack:
    primary_sources: []
    supplemental_sources: []
    citation_quality:
      verified | partial | weak
  editorial_metadata:
    author:
    synthetic_team_roles_used: []
    confidence_score:
    last_fact_check:
    version:
```

# Simplified Supporting Blocks

## Funding Block

```yaml
FundingBlock:
  title:
  source:
  destination:
  mechanism:
  amount:
  confidence:
  access_path:
  why_it_matters:
  citation_refs: []
```

## Constraint Block

```yaml
ConstraintBlock:
  title:
  type:
  affected_parties:
  severity:
  explanation:
  workaround_or_mitigation:
  citation_refs: []
```

## Time Horizon Block

```yaml
TimeHorizonBlock:
  phase: immediate | near_term | long_term
  trigger_event:
  expected_change:
  action_window:
  confidence:
  watchlist_sources: []
```

## Quote Block

```yaml
QuoteBlock:
  speaker:
  affiliation:
  stance: neutral | promoter | detractor
  quote:
  source:
  citation_ref:
```

# Final Report Template

```markdown
# [Title]

## Executive Summary

## What Changed

## Timeline
- Introduced:
- Key Amendment:
- Enacted:

## Key Provisions

## Exceptions

## Historical Precedents

## Who Backed It / Opposed It
- Legislators:
- Lobbyists:
- Interest Groups:
- Promoters:
- Detractors:

## Funding Flows
- Source:
- Destination:
- Amount:
- Mechanism:
- How to Access:

## Institutional Changes Required

## Market Impact
- Industries directly benefiting:
- Companies likely benefiting:

## Execution Opportunities
- What to do:
- Who should act:
- Dependencies:
- Timing:

## Constraints, Criticisms & Risks

## Quotes
- Neutral:
- Promoters:
- Detractors:

## Notes & References
```

# Backend Tables for Publication Engine

```text
reports
report_sections
source_documents
citation_refs
funding_blocks
constraint_blocks
time_horizon_blocks
quote_blocks
editorial_reviews
```

# Rule Change

Old rule:

> Do not store opportunity directly.

New publication-engine rule:

> Store opportunity as an editorial claim, but require citation, confidence, and reasoning.

That gives you speed without turning ELaB into a sloppy newsletter.

# Product Definition

ELaB is:

> A structured policy intelligence publication engine that turns public institutional policy changes into readable, cited, market-relevant reports.

Not a raw policy database. Not Bloomberg. Not FiscalNote. Not a graph engine.

Build the reporting spine first. Later, extract graph objects from published reports.

# Diagrams

```mermaid
flowchart TD
  A[Primary Sources] --> B[Source Pack]
  C[Supplemental Sources] --> B

  B --> D[Fact Extraction]
  D --> E[Structured Blocks]

  E --> E1[Funding Blocks]
  E --> E2[Constraint Blocks]
  E --> E3[Time Horizon Blocks]
  E --> E4[Quote Blocks]
  E --> E5[Stakeholder Blocks]

  E1 --> F[Publication Report]
  E2 --> F
  E3 --> F
  E4 --> F
  E5 --> F

  F --> G[Editorial Review]
  G --> H{Pass?}
  H -- No --> D
  H -- Yes --> I[Published ELaB Report]

  I --> J[RSS / Feed]
  I --> K[Website]
  I --> L[Client Briefing]
```

```mermaid
erDiagram
  REPORT ||--o{ REPORT_SECTION : contains
  REPORT ||--o{ SOURCE_DOCUMENT : uses
  REPORT ||--o{ CITATION_REF : cites
  REPORT ||--o{ FUNDING_BLOCK : includes
  REPORT ||--o{ CONSTRAINT_BLOCK : includes
  REPORT ||--o{ TIME_HORIZON_BLOCK : includes
  REPORT ||--o{ QUOTE_BLOCK : includes
  REPORT ||--o{ EDITORIAL_REVIEW : reviewed_by

  REPORT {
    string id
    string slug
    string title
    string policy_identifier
    string publish_status
    string thesis
    string executive_summary
    float confidence_score
    datetime published_at
  }

  REPORT_SECTION {
    string id
    string report_id
    string section_type
    int sort_order
    text content
  }

  SOURCE_DOCUMENT {
    string id
    string report_id
    string source_type
    string title
    string url
    string publisher
    datetime accessed_at
  }

  CITATION_REF {
    string id
    string report_id
    string source_document_id
    string claim
    string citation_text
  }

  FUNDING_BLOCK {
    string id
    string report_id
    string source
    string destination
    string mechanism
    string amount
    string confidence
    text access_path
  }

  CONSTRAINT_BLOCK {
    string id
    string report_id
    string type
    string severity
    text explanation
    text mitigation
  }

  TIME_HORIZON_BLOCK {
    string id
    string report_id
    string phase
    string trigger_event
    text expected_change
    string confidence
  }

  QUOTE_BLOCK {
    string id
    string report_id
    string speaker
    string affiliation
    string stance
    text quote
    string source_document_id
  }

  EDITORIAL_REVIEW {
    string id
    string report_id
    string reviewer_role
    string status
    text notes
    datetime reviewed_at
  }
```

```mermaid
flowchart LR
  A[Policy Event] --> B{Report Worthy?}

  B -- No --> C[Monitor Only]
  B -- Yes --> D[Create Report Draft]

  D --> E[Source Collection]
  E --> F[Claim Extraction]
  F --> G[Block Assembly]

  G --> H[Funding]
  G --> I[Constraints]
  G --> J[Time Horizon]
  G --> K[Stakeholders]
  G --> L[Quotes]
  G --> M[Market Impact]

  H --> N[Report Rendering]
  I --> N
  J --> N
  K --> N
  L --> N
  M --> N

  N --> O[Editorial Review]
  O --> P{Publishable?}

  P -- No --> Q[Revision Queue]
  Q --> F

  P -- Yes --> R[Publish]
  R --> S[Feed Distribution]
```

```mermaid
stateDiagram-v2
  [*] --> Candidate

  Candidate --> Draft: selected for coverage
  Draft --> Researching: source pack opened
  Researching --> Structured: claims extracted
  Structured --> Writing: blocks assembled
  Writing --> Review: draft rendered
  Review --> Revision: fails review
  Revision --> Researching: needs more evidence
  Revision --> Writing: wording only
  Review --> Published: passes review
  Published --> Updated: material change
  Updated --> Review
  Published --> Archived: obsolete or superseded
```

```mermaid
flowchart TD
  A[Report] --> B[Executive Summary]
  A --> C[What Changed]
  A --> D[Timeline]
  A --> E[Key Provisions]
  A --> F[Exceptions]
  A --> G[Historical Precedents]
  A --> H[Stakeholders]
  A --> I[Funding Flows]
  A --> J[Institutional Changes]
  A --> K[Market Impact]
  A --> L[Execution Opportunities]
  A --> M[Constraints / Risks]
  A --> N[Quotes]
  A --> O[Notes & References]

  I --> I1[Source]
  I --> I2[Destination]
  I --> I3[Amount]
  I --> I4[Mechanism]
  I --> I5[How to Access]

  L --> L1[What to Do]
  L --> L2[Who Should Act]
  L --> L3[Dependencies]
  L --> L4[Timing]
```

```mermaid
flowchart LR
  A[Primary Source] --> B[Claim]
  C[Supplemental Source] --> B

  B --> D{Claim Type}

  D --> E[Policy Fact]
  D --> F[Funding Claim]
  D --> G[Market Claim]
  D --> H[Risk Claim]
  D --> I[Quote]

  E --> J[Citation Required]
  F --> J
  G --> J
  H --> J
  I --> J

  J --> K{Evidence Strength}
  K -- Strong --> L[Publish]
  K -- Partial --> M[Publish with Caveat]
  K -- Weak --> N[Do Not Publish]
```

```mermaid
flowchart TD
  A[AI Synthetic Team] --> B[Analyst]
  A --> C[Builder]
  A --> D[Reviewer]
  A --> E[Truth-Teller]
  A --> F[Facilitator]

  B --> B1[Extract facts, sources, unknowns]
  C --> C1[Draft report blocks]
  D --> D1[Check gaps, citations, structure]
  E --> E1[Kill weak claims]
  F --> F1[Lock publish decision]

  B1 --> C1
  C1 --> D1
  D1 --> E1
  E1 --> F1
```

```mermaid
sequenceDiagram
  participant Source as Source Monitor
  participant Analyst as Analyst
  participant Builder as Builder
  participant Reviewer as Reviewer
  participant Truth as Truth-Teller
  participant CMS as Publication Engine
  participant Feed as RSS / Site

  Source->>Analyst: New policy event detected
  Analyst->>Analyst: Gather source pack
  Analyst->>Builder: Send extracted claims
  Builder->>CMS: Assemble report draft
  Reviewer->>CMS: Review citations and structure
  Truth->>CMS: Remove weak or overstated claims
  CMS->>CMS: Final render
  CMS->>Feed: Publish report
```

```mermaid
flowchart TD
  A[Publication Engine] --> B[Input Layer]
  A --> C[Processing Layer]
  A --> D[Editorial Layer]
  A --> E[Distribution Layer]

  B --> B1[Congress.gov]
  B --> B2[Federal Register]
  B --> B3[Agency Sites]
  B --> B4[Trusted Supplemental Sources]

  C --> C1[Source Pack Builder]
  C --> C2[Claim Extractor]
  C --> C3[Block Generator]
  C --> C4[Report Renderer]

  D --> D1[Citation Review]
  D --> D2[Risk Review]
  D --> D3[Editorial Approval]

  E --> E1[Website]
  E --> E2[RSS Feed]
  E --> E3[Email Brief]
  E --> E4[Client Workspace]
```