# CUTR → ELaB Gap Analysis & Migration Design

> **Purpose:** Document the changes required to convert the `cutr` codebase (Tauri desktop app for AI-assisted software development) into the `elaB` application (web-based policy-to-market intelligence platform).
>
> **Source:** `/Users/novelbamboo/Desktop/github/cutr` (existing codebase)
> **Target:** `/Users/novelbamboo/Desktop/github/elab` (PRD-driven target)
> **Date:** 2026-04-30

---

## 1. Summary

cutr is a Tauri v2 desktop application (React/TypeScript frontend + Rust backend) implementing a Synthetic Team Engine for AI-assisted software development. ELaB is a web-based policy-to-market intelligence platform that ingests government policy data, processes it through a role-constrained AI Synthetic Team, and delivers structured business opportunities as a "social feed" decision interface.

**Verdict:** Approximately 15% of cutr can be reused directly, 30% can be adapted, and 55% is net-new development. The Synthetic Team architecture and prompt management patterns are the highest-value carry-overs. The primary structural change is shifting from a desktop app to a web application.

---

Bottom line: ~15% of cutr is directly reusable (design tokens, CSS, delete confirmation), 30% adaptable (Synthetic Team engine, prompt store pattern, panel components), and 55% net-new (government API ingestion, database/auth, payment system, decision feed UI, newsletter integration).

The highest-value carryovers are the Synthetic Team sequencing/validation architecture and the prompt store resource-loading pattern. The primary structural change is Tauri desktop → Next.js web app with Supabase backend.

---

## 2. Architecture Gap: Desktop → Web

### 2.1 Runtime Platform

| Aspect | cutr (Current) | ELaB (Target) | Gap |
|---|---|---|---|
| Platform | Desktop (macOS/Windows/Linux) | Web (browser) | Complete shift |
| Frontend framework | React 19 + TypeScript + Vite | React/Next.js (recommended) | Vite → Next.js migration |
| Backend | Rust via Tauri | Node.js/TypeScript server (or serverless) | Rewrite backend |
| Storage | Local filesystem | Database (Supabase recommended) | New data layer |
| Auth | None (desktop-local) | User accounts, membership tiers | Net-new |
| Hosting | Binary distribution | Cloud deployment (Vercel/Railway) | Net-new |

**Migration Actions:**
1. Extract React/TypeScript frontend from Tauri wrapper; remove all `@tauri-apps/api` dependencies
2. Replace `invoke()` calls with REST/GraphQL API calls
3. Port prompt store logic from `src-tauri/src/prompt_store.rs` to TypeScript (Node.js module)
4. Initialize Next.js project (or keep Vite for SPA + add separate API server)
5. Set up Supabase for database, auth, and row-level security

### 2.2 Tauri Dependencies to Remove/Replace

| cutr Code | Location | Migration |
|---|---|---|
| `invoke()` API calls | `src/lib/api.ts` | Replace with `fetch()` / `axios` to web API routes |
| `tauri-plugin-shell` | `Cargo.toml`, `lib.rs` | Remove entirely |
| Tauri capability config | `src-tauri/capabilities/default.json` | Replace with CORS/middleware config |
| Tauri icon set | `src-tauri/icons/` | Replace with web favicon + PWA icons |
| Rust build config | `Cargo.toml`, `build.rs` | Replace with `package.json` server scripts |
| `prompt_store.rs` | `src-tauri/src/prompt_store.rs` | Rewrite as TypeScript `promptStore.ts` reading from `/resources/prompt-store/` |

---

## 3. Data & API Layer (Net-New)

cutr has no database, no API ingestion, and no user management. ELaB needs all three.

### 3.1 Database Schema (Supabase)

```
users
  - id, email, membership_tier, created_at, stripe_customer_id

policies
  - id, source_api, source_id, title, summary, publish_date
  - raw_text, status (new/processing/complete), created_at

policy_outputs
  - id, policy_id, role_id, created_at
  - plain_english_summary, affected_industries, agencies_involved
  - compliance_requirements, funding_pathways, potential_business_models
  - winners_losers, confidence_score, risk_surface, execution_path
  - is_validated, validator_role_id

synthetic_team_runs
  - id, policy_id, status, started_at, completed_at
  - sequence (analyst → builder → reviewer → operator → truth-teller)

role_artifacts
  - id, run_id, role, artifact_type, content, confidence
  - validated, validation_notes, created_at

opportunities
  - id, policy_id, title, description, sector
  - execution_path (JSONB: steps[]), capital_intensity, confidence_score
  - risk_level, time_horizon, funding_sources

alerts
  - id, user_id, policy_id, alert_type, read, created_at

subscriptions
  - id, user_id, tier, stripe_subscription_id, status
```

### 3.2 Government API Ingestion Layer (Net-New)

| API | Purpose | Implementation |
|---|---|---|
| Congress.gov API | Bills, laws, legislative status | Cron job / scheduled function |
| Federal Register API | Executive orders, federal rules | Cron job / scheduled function |
| Grants.gov + USAspending.gov | Grant opportunities, federal spending | Cron job / scheduled function |
| OpenFEC / FEC API | Campaign finance data | Cron job / scheduled function |

**Build:** Node.js ingestion workers (could be Next.js API routes, serverless functions, or a standalone worker) that poll government APIs, normalize data into `policies` table, and queue Synthetic Team processing.

### 3.3 Web API Routes (Net-New)

```
GET    /api/policies              → Policy feed (paginated, filterable)
GET    /api/policies/:id          → Single policy detail + outputs
POST   /api/policies/:id/process  → Trigger Synthetic Team run
GET    /api/opportunities         → Opportunity listing (filterable by sector)
GET    /api/feed                  → Decision feed (role-attributed artifacts)
POST   /api/alerts/subscribe      → Subscribe to high-impact alerts
GET    /api/user/profile          → User profile + membership
POST   /api/checkout              → Stripe checkout session
POST   /api/webhooks/stripe       → Stripe webhook handler
```

---

## 4. Synthetic Team Engine

### 4.1 What Can Be Preserved from cutr

The Synthetic Team architecture is the **highest-value carryover** from cutr:

| cutr Artifact | Reuse Potential | Notes |
|---|---|---|
| Archetype sequencing pattern | **High** — Directly reusable | Enforced sequence (Analyst → Builder → Reviewer → Operator → Truth-Teller) is identical in structure |
| Role definition format (`archetype-profiles.ts`) | **High** — Adaptable | Change content from "SDLC role" to "policy role" but keep the pattern |
| Validation gate pattern (Reviewer) | **High** — Directly reusable | cutr's `ValidationPanel` concept maps to ELaB's Reviewer gate |
| `prompt_store/` architecture | **High** — Adaptable | Rewrite prompts for policy domain but keep the resource-loading pattern |
| `RunHistoryPanel` pattern | **Medium** — Adaptable | Maps to "Synthetic Team Run" history |
| `InterpretationPanel` pattern | **Medium** — Adaptable | Maps to output display panels |
| `ObservedStatePanel` | **Medium** — Adaptable | Maps to Truth-Teller outcome tracking |
| ACR `view-model.ts` | **Medium** — Adaptable | The state machine for chat room can be adapted for decision feed pipeline |

### 4.2 Role Definitions: Required Changes

cutr's 6 roles are defined for software development. ELaB's 6 roles are policy analysis roles. The **structure** stays; the **content** changes entirely.

#### Side-by-Side Role Mapping

| cutr Role | ELaB Role | Shared DNA | What Changes |
|---|---|---|---|
| Analyst | **Analyst (Iris Cole)** | Input → structured output | cutr: extracts requirements from user input → ELaB: extracts policy signals from government text |
| Builder | **Builder (Kenji Sato)** | Structured input → concrete artifacts | cutr: builds software architecture → ELaB: builds market distortion models and business models |
| Facilitator | **Facilitator (Sofia Alvarez)** | Constraint enforcement, coherence | Same function; different constraints (policy consistency vs. software consistency) |
| Operator | **Operator (Ivan Petrov)** | Pipeline orchestration, delivery | Same function; different output (policy feed vs. development artifacts) |
| Reviewer | **Reviewer (Elena Kovacs)** | Validation gate, false positive detection | Same function; different validation criteria (economic plausibility vs. code correctness) |
| Truth-Teller | **Truth-Teller (Daniel Costa)** | Outcome tracking, reality check | Same function; different metrics (market outcomes vs. software quality) |

#### Role Artifact Changes

| cutr Artifact | ELaB Artifact |
|---|---|
| Requirements spec | Policy Object (structured policy data) |
| System architecture | Distortion Surface (market impact map) |
| Code artifacts | Opportunity Graph (business opportunities) |
| Test results | Validation Outcome (plausibility check) |
| Deployment pipeline | Execution Path Object (actionable steps) |
| Bug reports | Outcome Delta Report (prediction vs. reality) |

### 4.3 Prompt Store Migration

**Keep:** The resource-loading architecture (`index.json` → load by key → resolve templates)

**Rewrite:** Every prompt file for policy domain.

| cutr Prompt | ELaB Replacement |
|---|---|
| `system.md` (software development) | `system.md` (policy analysis system prompt) |
| `user.md` (SDLC context) | `user.md` (policy context template) |
| `output-schema.json` (code artifacts) | `output-schema.json` (10-field policy output schema) |
| `kiss.md` (software KISS) | `kiss.md` (policy KISS) |
| `role-card.md` (dev roles) | `role-card.md` (policy roles) |
| `synthetic-team.md` | `synthetic-team.md` (unchanged pattern, new team config) |

**New prompts needed:**
- `due-diligence.md` — Policy opportunity analysis prompt (from `assets/Prompt - Due Diligence.md`)
- `law-search.md` — Policy search/discovery prompt (from `assets/Prompt - Law Search.md`)
- `scoring.md` — Confidence scoring rubric (from `assets/Prompt - Scoring.md`)
- Per-role prompts for Analyst, Builder, Reviewer, Operator, Truth-Teller, Facilitator

---

## 5. Frontend: UI & Pages

### 5.1 What Can Be Reused from cutr

| cutr Component | Reuse Potential | Notes |
|---|---|---|
| Design token system (`src/theme/`) | **High** — Keep entire system | Colors, spacing, typography tokens; rebrand for ELaB |
| `css/base.css` | **High** — Keep | CSS reset, base styles |
| Component CSS files | **High** — Keep patterns | Adapt card/button/form/typography styles |
| `AppFrame.tsx` | **Medium** — Adapt | Shell layout pattern; new header/nav for ELaB |
| `AppHeader.tsx` | **Low** — Rewrite | New branding, navigation, auth controls |
| `ArchetypeCardGrid.tsx` | **Medium** — Adapt | Can become "Role Grid" or "Synthetic Team Dashboard" |
| `ArchetypePanel.tsx` | **Medium** — Adapt | Generic panel pattern; adapt content for policy artifacts |
| `ArchetypeHeader.tsx` | **Low** — Adapt | Role display pattern |
| `ArtifactPanel.tsx` | **Medium** — Adapt | Output/artifact display panel |
| `InterpretationPanel.tsx` | **Medium** — Adapt | Analysis display panel |
| `ValidationPanel.tsx` | **Medium** — Adapt | Validation results display |
| `RunHistoryPanel.tsx` | **Medium** — Adapt | Processing run history |
| `DeleteConfirmationOverlay.tsx` | **High** — Reuse directly | Generic confirmation modal |
| `LinkedOverlay.tsx` | **Low** — Rewrite | Different linking concept in ELaB |
| Design token TypeScript files | **High** — Keep | `tokens/colors.ts`, `tokens/spacing.ts`, `tokens/typography.ts`, `tokens/effects.ts` |

### 5.2 Pages/Screens: Complete Rewrite Required

All cutr screens are designed for software development workflows. ELaB needs entirely different screens.

| cutr Screen | ELaB Replacement | Why |
|---|---|---|
| `RootScreen.tsx` | **Landing/Home** — Public policy feed, CTAs, membership pitch | Different user journey |
| `ProjectsScreen.tsx` | **Policy Feed** — Chronological feed of processed policies with opportunities | Policies, not projects |
| `ProjectScreen.tsx` | **Policy Detail** — Single policy with full 10-field output, execution paths, threaded decision chain | Different content |
| `ArchetypeProfileScreen.tsx` | **Role Card** — Read-only role definition + recent outputs for that role | Similar concept, different content |
| `ArchetypeChatRoomScreen.tsx` | **Decision Feed** — Threaded role-attributed artifacts with interaction modes | "Social feed" not chat room |
| `LinkedScreen.tsx` | **Funding Trace** — Policy → funding → opportunity traceability view | Different linking concept |
| `SettingsScreen.tsx` | **Settings** — Account, membership, alert preferences | Keep pattern, rewrite content |

### 5.3 New Screens (Net-New)

| Screen | Purpose |
|---|---|
| **Opportunity Board** | Ranked opportunities across all policies, filterable by sector |
| **Execution Path Detail** | Step-by-step action plan for a single opportunity |
| **Alert Settings** | Configure high-impact event notifications |
| **Membership / Pricing** | Tier comparison, upgrade flow, Stripe checkout |
| **Admin Dashboard** | Synthetic Team run management, error queue, ingestion status |

### 5.4 Interaction Model Migration

cutr uses a **chat room** interaction model (multi-agent conversation). ELaB uses a **social feed** interaction model.

**Key differences to implement:**
- **Decision Feed** (not chat): `[ROLE] + [CLAIM] + [ARTIFACT] + [CONFIDENCE] + [NEXT ACTION]`
- **Threaded Decision Chains**: Policy Signal → Distortion Model → Validation Attack → Execution Output → Reality Outcome
- **Three Interaction Modes**: Feed Post, @Role Mention, @Team Mention
- **Critical UX Features**: Disagreement Visibility, Confidence Decay, Kill Events, Outcome Scorecard

These are **net-new** — cutr's chat UI pattern does not directly translate.

---

## 6. Feature Gap: Missing ELaB Features

| ELaB Feature | Present in cutr? | Gap |
|---|---|---|
| Policy Feed (API ingestion) | No | Net-new ingestion pipeline |
| Opportunity Engine | No | Net-new LLM prompt chain |
| Execution Path Object (3+ steps) | No | Net-new LLM prompt chain |
| Policy-to-Funding Trace | No | Net-new data linking |
| Confidence + Risk Layer | No | Net-new scoring logic |
| Alerts (high-impact events) | No | Net-new notification system |
| Membership Tiers (Free/Operator/Allocator/Enterprise) | No | Net-new auth + Stripe |
| Newsletter Integration (Beehiiv/Substack) | No | Net-new integration |
| Basic Filtering (sector, timeline) | No | Net-new UI filters |
| Source Linking | No | Net-new citation system |
| Disagreement Visibility | No | Net-new UX pattern |
| Confidence Decay | No | Net-new logic |
| Kill Events | No | Net-new state |
| Outcome Scorecard | No | Net-new dashboard |

---

## 7. Build & Deployment Gap

| Aspect | cutr | ELaB | Action |
|---|---|---|---|
| Build system | Vite + Cargo (Rust) | Next.js / Vite (web only) | Remove Rust build, keep Vite or migrate to Next.js |
| Package manager | npm | npm/pnpm | Keep npm |
| TypeScript config | `tsconfig.json` (React JSX) | `tsconfig.json` (Next.js or Vite SPA + Node server) | Adjust target/libs |
| CI/CD | None evident | GitHub Actions → Vercel/Railway deploy | Set up CI/CD |
| Environment variables | `tauri.conf.json` | `.env.local` + Supabase env vars | New config pattern |
| Testing | Vitest | Vitest (keep) + Playwright for E2E | Add E2E tests |
| Monitoring | None | Error tracking (Sentry), analytics | Add observability |
| Makefile | GNU Make tasks | Replace with `package.json` scripts | Convert Make → npm scripts |

---

## 8. Migration Roadmap

### Phase 1: Foundation (Strip & Scaffold) — Week 1-2

1. **Initialize new web project** alongside existing cutr codebase
2. **Extract reusable assets:**
   - Copy `src/theme/` (design tokens + CSS) → new project
   - Copy `src/components/DeleteConfirmationOverlay.tsx`
   - Copy `src-tauri/resources/prompt-store/` pattern → port to `resources/prompt-store/`
3. **Port `src/lib/types.ts`** — Remove Tauri-specific types, add ELaB types
4. **Set up Supabase** — Create database schema (Section 3.1), configure auth
5. **Set up Stripe** — Products/prices for membership tiers, webhook endpoint
6. **Configure deployment** — Vercel/Railway with environment variables

### Phase 2: Backend Core — Week 3-4

1. **Build API ingestion workers** — Congress.gov, Federal Register, Grants.gov, USAspending, OpenFEC
2. **Port prompt store to TypeScript** — `lib/promptStore.ts` mirroring `prompt_store.rs` logic
3. **Implement Synthetic Team pipeline** — Port sequencing, validation gates, role execution
4. **Build API routes** (Section 3.3) — Policies CRUD, processing triggers, feed endpoints
5. **Implement auth** — Supabase Auth with magic link + membership tier enforcement
6. **Implement Stripe checkout + webhooks** — Tier provisioning on successful payment

### Phase 3: Frontend Shell — Week 5-6

1. **Build layout shell** — `AppFrame` with new header, navigation, auth state
2. **Build Policy Feed** — Replace ProjectsScreen concept with chronological policy listing
3. **Build Policy Detail** — Full 10-field output display with source links
4. **Build Decision Feed** — Social feed interaction model (role-attributed artifacts, threaded chains)
5. **Build Opportunity Board** — Filterable opportunity listing
6. **Build basic filtering** — Sector and timeline navigation

### Phase 4: Monetization & Delivery — Week 7-8

1. **Build Membership pages** — Tier comparison, upgrade flow, account management
2. **Implement alert system** — Email + in-app notifications for high-impact events
3. **Integrate newsletter** — Beehiiv/Substack API for digest publishing
4. **Build Outcome Scorecard** — Per-role truth-teller dashboard
5. **Add gating** — Feature restrictions by membership tier (no execution paths for Free tier)

### Phase 5: Trust & Polish — Week 9-10

1. **Implement confidence visualization** — Confidence score UI with decay indicators
2. **Implement Disagreement Visibility** — Show when Reviewer rejects Builder output
3. **Implement Kill Events** — Thread state when opportunity is invalidated
4. **Add source linking** — Every claim links to source policy document
5. **Add uncertainty flags** — Explicit "unknown" labeling where data is weak
6. **E2E testing** — Playwright tests for critical user journeys
7. **Performance optimization** — Lazy loading, caching, API response optimization

---

## 9. Risk Register

| Risk | Severity | Mitigation |
|---|---|---|
| Desktop → Web migration loses Tauri performance benefits | Medium | Accept — web is the right platform for ELaB's distribution model |
| Prompt store rewrite introduces regressions | Medium | Port with tests; keep cutr as reference implementation |
| Government API reliability / rate limiting | High | Add retry logic, caching, graceful degradation |
| LLM output quality (hallucination) | Critical | Keep Reviewer gate; implement confidence scoring; Truth-Teller feedback loop |
| Membership conversion rates | Medium | Start with newsletter funnel (free → paid) before building full membership |
| Synthetic Team latency (72h SLA) | Medium | Pre-process common policy types; cache role outputs where appropriate |
| Scope creep (adding features beyond MVP) | High | Follow PRD's strict "must help user take action" rule; defer everything else |
| Stripe/webhook complexity | Low | Use Stripe Checkout (managed); minimal webhook surface |

---

## 10. Technology Decision Matrix

| Decision | cutr Choice | ELaB Recommendation | Rationale |
|---|---|---|---|
| Framework | React + Vite | Next.js (App Router) | SSR for SEO/public pages, API routes for backend, better for content platform |
| Backend | Rust (Tauri) | Next.js API routes + serverless functions | Unified TypeScript stack, lower complexity than separate Express/Fastify server |
| Database | None (filesystem) | Supabase | Managed Postgres, built-in auth, realtime subscriptions, row-level security |
| Auth | None (desktop) | Supabase Auth + Stripe | Magic link auth, membership tier via Stripe metadata |
| Styling | Custom CSS + tokens | Keep custom CSS + tokens | Already excellent; no need for Tailwind/etc. |
| State management | React state + context | React state + context + SWR/React Query | Data fetching/caching for API calls |
| AI/LLM | LLM via ??? (API in prompt store) | OpenAI / Claude API | Documented in PRD; prompt store already set up for LLM integration |
| Hosting | Binary distribution | Vercel (frontend) + Railway/Supabase (backend) | Standard Next.js deployment |
| Payments | None | Stripe Checkout | Simple integration, handles compliance |
| Newsletter | None | Beehiiv or Substack API | Listed in PRD; API integration for auto-publishing |

---

## 11. Files Manifest: Keep, Adapt, Rewrite

### Keep (Copy As-Is)
```
src/theme/tokens/colors.ts
src/theme/tokens/spacing.ts
src/theme/tokens/typography.ts
src/theme/tokens/effects.ts
src/theme/tokens/index.ts
src/theme/components/buttons.css
src/theme/components/cards.css
src/theme/components/forms.css
src/theme/components/typography.css
src/theme/components/layout.css
src/theme/components/feedback.css
src/theme/components/responsive.css
src/theme/components/index.css
src/theme/base.css
src/theme/index.css
src/components/DeleteConfirmationOverlay.tsx
```

### Adapt (Modify Content)
```
src/lib/types.ts              → Keep type patterns, replace all types with ELaB domain
src/lib/api.ts                → Replace Tauri invoke() with fetch()
src/lib/path.ts               → Update path utilities for web
src/components/AppFrame.tsx    → New layout shell based on same pattern
src/components/ArchetypeCardGrid.tsx  → RoleCardGrid for Synthetic Team display
src/components/ArchetypePanel.tsx     → Generic panel; use for policy output panels
src/components/ArchetypeHeader.tsx    → Role header with ELaB persona info
src/components/ArtifactPanel.tsx      → Policy artifact display panel
src/components/InterpretationPanel.tsx → Analysis display panel
src/components/ValidationPanel.tsx    → Reviewer validation display
src/components/RunHistoryPanel.tsx    → Processing run history
src/routers.tsx                       → Rewrite routes for ELaB pages
src/theme/screens/workspace.css       → Adapt for policy feed/decision feed
src/archetypes/types.ts               → Policy role types
src/archetypes/archetype-profiles.ts  → Policy role definitions
src/acr/view-model.ts                 → Decision feed state machine
src/prompt_store/                     → Rewrite prompts for policy domain
```

### Rewrite (Net-New)
```
src/screens/PolicyFeedScreen.tsx        (replaces ProjectsScreen)
src/screens/PolicyDetailScreen.tsx      (replaces ProjectScreen)
src/screens/DecisionFeedScreen.tsx      (replaces ArchetypeChatRoomScreen)
src/screens/OpportunityBoardScreen.tsx  (net-new)
src/screens/ExecutionPathScreen.tsx     (net-new)
src/screens/LandingScreen.tsx           (replaces RootScreen)
src/screens/SettingsScreen.tsx          (rewrite for account/membership)
src/screens/MembershipScreen.tsx        (net-new)
src/lib/db.ts                           (Supabase client)
src/lib/auth.ts                         (Supabase auth helpers)
src/lib/stripe.ts                       (Stripe client helpers)
src/lib/ingestion/                      (Government API workers)
src/lib/pipeline/                       (Synthetic Team orchestration)
src/lib/scoring.ts                      (Confidence scoring logic)
src/components/DecisionFeed.tsx         (Social feed interaction model)
src/components/ConfidenceBadge.tsx      (Confidence visualization)
src/components/ExecutionPathSteps.tsx   (Step-by-step action display)
src/components/MembershipGate.tsx       (Feature gating by tier)
src/app/                                (Next.js App Router pages + API routes)
```

### Remove (Not Carried Forward)
```
src-tauri/                              (Tauri/Rust layer)
src/acr/                                (Chat-specific logic; replaced by Decision Feed)
src/components/LinkedOverlay.tsx        (Different linking concept)
src/screens/ProjectsScreen.tsx          (Replaced by PolicyFeedScreen)
src/screens/ProjectScreen.tsx           (Replaced by PolicyDetailScreen)
src/screens/ArchetypeChatRoomScreen.tsx (Replaced by DecisionFeedScreen)
src/screens/LinkedScreen.tsx            (Replaced by FundingTraceScreen)
src/assets/images/ (archetype icons)    (Replace with ELaB role avatars)
dist/                                   (Build output; regenerated)
```

---

## 12. Key Architectural Decisions

1. **Next.js over Vite SPA**: ELaB is a content platform that benefits from SSR (SEO for public policy pages, faster initial loads). Next.js App Router provides API routes eliminating the need for a separate backend server.

2. **Supabase over Airtable/Notion**: PRD suggests Airtable/Baserow/Supabase/Notion. Supabase is the strongest choice: relational data fits the policy → output → opportunities model, built-in auth, and scales with paid tiers.

3. **Custom CSS over Tailwind**: cutr's design token system is already excellent. Keeping it avoids a wholesale CSS rewrite and maintains design consistency. No need to introduce a utility framework.

4. **Separate Ingestion Workers**: Government API polling should be serverless cron functions (Vercel Cron / GitHub Actions), not triggered by user requests. This decouples data freshness from user traffic.

5. **LLM Cost Management**: Each policy processing run invokes 6+ LLM calls (one per role). Costs should be tracked per policy and factored into membership pricing. Consider batching low-priority policies.

6. **Feature Gating at API Layer**: Free tier receives truncated outputs (no execution paths). The API should enforce this, not just the frontend, to prevent circumvention.

---

## Appendix A: cutr Architecture Diagram

```
┌─────────────────────────────────────────────┐
│                  cutr (Desktop App)           │
├─────────────────────────────────────────────┤
│  React Frontend (src/)                       │
│  ┌───────────────────────────────────────┐   │
│  │ Screens: Root, Projects, Project,     │   │
│  │   ArchetypeChatRoom, Settings, Linked │   │
│  ├───────────────────────────────────────┤   │
│  │ Archetypes Module: Profiles, Routes   │   │
│  │ ACR Module: ViewModel, Types          │   │
│  │ Theme: CSS Design Tokens              │   │
│  │ Lib: API (invoke), Types, Path        │   │
│  └───────────────┬───────────────────────┘   │
│                  │ invoke()                   │
│  ┌───────────────▼───────────────────────┐   │
│  │ Rust Backend (src-tauri/src/)          │   │
│  │ lib.rs: Command handlers               │   │
│  │ prompt_store.rs: Load/manage prompts   │   │
│  │ resources/prompt-store/: LLM prompts   │   │
│  └───────────────────────────────────────┘   │
│  Local Filesystem Storage                    │
└─────────────────────────────────────────────┘
```

## Appendix B: ELaB Target Architecture

```
┌──────────────────────────────────────────────────────┐
│                  ELaB (Web Application)                │
├──────────────────────────────────────────────────────┤
│  Next.js Frontend                                     │
│  ┌────────────────────────────────────────────────┐   │
│  │ Pages: Landing, PolicyFeed, PolicyDetail,      │   │
│  │   DecisionFeed, OpportunityBoard, Settings,    │   │
│  │   Membership, ExecutionPath                    │   │
│  ├────────────────────────────────────────────────┤   │
│  │ Components: RoleCardGrid, DecisionFeed,        │   │
│  │   ConfidenceBadge, ExecutionPathSteps,         │   │
│  │   MembershipGate, DeleteConfirmation           │   │
│  │ Theme: CSS Design Tokens (from cutr)           │   │
│  │ Lib: API (fetch), Types, Scoring, PromptStore  │   │
│  └───────────────┬────────────────────────────────┘   │
│                  │ REST/HTTP                           │
│  ┌───────────────▼────────────────────────────────┐   │
│  │ Next.js API Routes                               │   │
│  │ /api/policies, /api/opportunities, /api/feed    │   │
│  │ /api/checkout, /api/webhooks/stripe             │   │
│  │ /api/alerts, /api/user                          │   │
│  └───────┬───────────────────┬────────────────────┘   │
│          │                   │                         │
│  ┌───────▼───────┐   ┌──────▼──────────┐              │
│  │ Supabase       │   │ LLM API         │              │
│  │ - Auth         │   │ (OpenAI/Claude) │              │
│  │ - Database     │   │ Prompt Store     │              │
│  │ - Row Security │   │ (resources/)     │              │
│  └───────────────┘   └─────────────────┘              │
│                                                       │
│  ┌──────────────────────────────────────────────┐     │
│  │ Ingestion Workers (Cron / Serverless)         │     │
│  │ Congress.gov → Federal Register → Grants.gov  │     │
│  │ → USAspending → OpenFEC                       │     │
│  └──────────────────────────────────────────────┘     │
│                                                       │
│  ┌──────────────────────────────────────────────┐     │
│  │ External Integrations                         │     │
│  │ Stripe (payments) + Beehiiv (newsletter)      │     │
│  └──────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```
