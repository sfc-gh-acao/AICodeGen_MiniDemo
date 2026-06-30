# eClinical Solutions — ACE Proficiency MVP Plan

> **Ground rule:** No code is written until this plan is marked solid. All scope and architectural decisions are logged here first.

---

## Requirements Document

### Situation Summary

**Customer:** eClinical Solutions — a clinical trial software company  
**Contact:** Venu Mallarapu (vmallarapu@eclinicalsol.com) — recently inherited IT, focused on data modernization  
**AE:** Jason Pucciarelli | **Renewal:** October 2026

eClinical has been a Snowflake customer but is underutilizing the platform. Their vision is to build an "enterprise data hub" that consolidates data from all business functions and eventually powers AI/ML. Their immediate pain is that cross-functional reporting is entirely manual — MQL tracking, pipeline analysis, finance backlog, and commission calculations all live in spreadsheets and disconnected systems.

They were notably excited during the call by a demo of Snowflake's internal "Raven" AI assistant — the idea of asking natural-language questions against their own business data and getting instant answers without needing a data team to pull a report.

---

### Key Pains (from the Gong)

| # | Pain | Severity | Source System |
|---|------|----------|--------------|
| 1 | Sales funnel tracking (MQL → SQL → Opp → Closed Won) is manual in spreadsheets | High | HubSpot + Salesforce |
| 2 | Finance/services backlog and "bid vs. did" reporting doesn't exist in one place | High | Salesforce + NetSuite |
| 3 | Commission and bonus calculations are manual | Medium | Salesforce |
| 4 | No cross-functional view connecting marketing activity to revenue | High | HubSpot + Salesforce + NetSuite |
| 5 | Engineering AI adoption (GitHub Copilot, Cursor) is tracked manually or not at all | Low-Medium | Internal tooling |

---

### Desired Outcomes

1. **Dashboards** for cross-functional leadership reporting — replace Excel exports, support board-level PowerPoint
2. **AI assistant** (Raven-style) — ad-hoc natural language queries against business data, no dependency on data team
3. **Automated ingestion pipeline** — data lands in Snowflake continuously, no manual refreshes
4. **Foundation for AI/ML** — data hub structured to support forecasting, predictive models later

---

### What We Will Build (MVP Scope)

**Core use case: Sales & Marketing Funnel Intelligence**

A Streamlit-in-Snowflake application that:
- Ingests synthetic but realistic data representing the full customer journey: **HubSpot leads → Salesforce opportunities → NetSuite contracts/invoices**
- Uses a **Medallion architecture** (raw → curated → semantic) with Dynamic Tables for incremental processing
- Exposes a **Semantic View** defining business metrics (MQL action rate, funnel conversion rates, ARR, services revenue, etc.)
- Powers a **Cortex Analyst chat interface** — the "Raven-like" NL experience Venu was excited about
- Includes a **static dashboard tab** showing funnel KPIs for the board-reporting use case

The demo story:  
> *"Here's what your sales funnel looks like — from the moment marketing generates a lead in HubSpot, all the way to closed revenue in NetSuite. And instead of asking your analyst to pull a report, your sales leader can just ask it a question."*

---

### Out of Scope

- Live connector to actual HubSpot / Salesforce / NetSuite (real integration is an AE + SI engagement, not an MVP demo)
- Power BI integration (dashboard is delivered via Streamlit-in-Snowflake for demo purposes)
- Engineering AI adoption tracking (secondary use case, deferred)
- Commission calculation engine
- User authentication / row-level security per rep
- Production-grade error handling and monitoring

---

### Assumptions

1. Synthetic test data will be generated in Snowflake — realistic field names, volumes, and distributions matching what eClinical described (B2B SaaS sales cycle)
2. "Raven-style" assistant = Cortex Analyst backed by a Semantic View (no full agent framework needed for MVP)
3. Funnel metrics will use standard B2B SaaS definitions unless something more specific was stated
4. Dashboard runs in Snowflake (Streamlit-in-Snowflake) — no external BI tool required
5. All compute runs in a single standard warehouse (X-Small or Small)

---

## Technical Architecture

### Data Layer (Medallion)

```
RAW schema          →  CURATED schema        →  SEMANTIC layer
──────────────────     ────────────────────     ───────────────────────
raw_hubspot_leads      dim_leads               Semantic View:
raw_sf_contacts        dim_accounts              eclinical_funnel_sv
raw_sf_opportunities   dim_opportunities
raw_sf_activities      fact_funnel_events
raw_netsuite_invoices  fact_revenue
```

- **RAW:** Static tables populated by a one-time data generation script — simulates what an OpenFlow/Snowpipe ingestion would land
- **CURATED:** Dynamic Tables with a short target lag (e.g., 1 minute) — incremental refresh, no manual ETL
- **SEMANTIC:** Semantic View over curated tables — defines business metrics, dimensions, and relationships for Cortex Analyst

### Snowflake Features Used

| Feature | Role in MVP |
|---|---|
| Dynamic Tables | Incremental transformation from raw → curated |
| Semantic Views | Business metric definitions for Cortex Analyst |
| Cortex Analyst | NL query interface powering the chat tab |
| Streamlit-in-Snowflake | App UI (dashboard + chat) |
| Synthetic data generation | Python/SQL script to populate raw tables |

### Database Structure

```
Database: ECLINICAL_DEMO
  Schema: RAW
  Schema: CURATED
  Schema: APP  (Streamlit lives here)
```

### Key Metrics to Define in Semantic View

- **MQL Volume** — total leads created in period
- **MQL Action Rate** — % of leads contacted within SLA (e.g., 48 hours)
- **Lead-to-Opportunity Conversion Rate**
- **Opportunity Win Rate**
- **Average Sales Cycle Length** (days from lead created to close)
- **ARR / New ARR** — from closed-won opportunities
- **Services Backlog** — contracted services value not yet invoiced
- **Bid vs. Did** — contracted services value vs. actual invoiced amount

---

## Implementation Steps

> Steps are ordered. No step begins before the prior one is validated.

### Phase 1 — Foundation
- [ ] Create Snowflake database, schemas, warehouse, and role structure
- [ ] Write and run synthetic data generation scripts (realistic B2B SaaS volumes: ~500 leads, ~150 opps, ~50 contracts)
- [ ] Validate raw data looks realistic for the demo story

### Phase 2 — Curated Layer
- [ ] Define and create Dynamic Tables in CURATED schema
- [ ] Validate incremental refresh behavior and data quality
- [ ] Confirm funnel join logic (lead → contact → opportunity → invoice)

### Phase 3 — Semantic Layer
- [ ] Author Semantic View (`eclinical_funnel_sv`) over curated tables
- [ ] Define all metrics and dimensions listed above
- [ ] Add verified queries (VQRs) for the most important demo questions
- [ ] Validate Cortex Analyst can answer those questions correctly

### Phase 4 — Application
- [ ] Scaffold Streamlit-in-Snowflake app (two tabs: Dashboard + AI Assistant)
- [ ] Build dashboard tab: KPI cards + funnel chart + top-of-funnel trend
- [ ] Build chat tab: Cortex Analyst integration with conversation history
- [ ] Polish UI — framing eClinical's story, not generic demo feel

### Phase 5 — Demo Prep
- [ ] Write demo script (2-3 NL questions that tell the funnel story)
- [ ] Identify 2+ CoCo interactions to narrate during presentation
- [ ] Confirm app runs end-to-end, all queries return results

---

## Snowflake Best Practices Applied

- **Medallion architecture** — clear separation of raw, curated, and semantic layers; no ELT logic in the app layer
- **Dynamic Tables over traditional tasks/streams** — simpler incremental pipeline, automatic dependency tracking, built-in refresh scheduling
- **Semantic Views for Cortex Analyst** — metric definitions live in the semantic layer, not in application code; reusable across Cortex Analyst, BI tools, and agents
- **Streamlit-in-Snowflake** — no external deployment, data never leaves the platform, no connectivity setup
- **Single warehouse, right-sized** — X-Small for development; document upgrade path for production load
- **Schema separation** — raw data isolated from transformed data; prevents accidental overwrites, makes lineage clear

---

## Open Questions (to resolve before building)

1. Should the Streamlit app use a single-page layout with tabs, or a sidebar-driven multi-page structure?
2. What time range should the synthetic data cover? (Suggest: 18 months of history, current month in progress)
3. Should the AI assistant tab retain conversation history within a session, or be stateless per question?
