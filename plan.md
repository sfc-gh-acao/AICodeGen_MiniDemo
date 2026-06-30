# eClinical Solutions — ACE Proficiency MVP Plan

> **Ground rule:** No code is written until this plan is marked solid. All scope and architectural decisions are logged here first.

---

## Requirements Document

### Situation Summary

**Customer:** eClinical Solutions — a clinical trial software company  
**Contact:** Venu Mallarapu (vmallarapu@eclinicalsol.com) — recently inherited IT, focused on data modernization  
**AE:** Jason Pucciarelli | **Renewal:** October 2026

eClinical has been a Snowflake customer but is underutilizing the platform. Their vision is to build an "enterprise data hub" that consolidates data from all business functions and eventually powers AI/ML. Their most immediate pain is that cross-functional reporting is entirely manual — MQL tracking, pipeline analysis, finance backlog, and commission calculations all live in spreadsheets and disconnected systems.

Data is scattered: marketing lives in HubSpot, sales lives in Salesforce, finance in NetSuite, and pipeline tracking bridges the gaps in spreadsheets. This is explicitly stated in the call, not assumed. Every team has their own numbers and their own definitions, which means no one trusts the same report.

They were notably excited during the call by a demo of Snowflake's internal "Raven" AI assistant — the idea of asking natural-language questions against their own business data and getting instant answers without needing a data team to pull a report.

---

### Key Pains (from the Gong — explicit vs. implied)

| # | Pain | Evidence | Source System |
|---|------|----------|--------------|
| 1 | Sales funnel tracking is manual in spreadsheets | Explicitly stated | HubSpot + Salesforce |
| 2 | Marketing and sales data lives in separate systems with no unified view | Explicitly stated ("disparate systems") | HubSpot + Salesforce |
| 3 | Finance/services backlog and "bid vs. did" reporting doesn't exist in one place | Explicitly stated | Salesforce + NetSuite |
| 4 | AI tool usage (Copilot, Cursor) is not consolidated or visible | Explicitly stated ("trying to collate that information") | Internal tooling |
| 5 | Commission and bonus calculations are manual | Explicitly stated | Salesforce |
| 6 | No cross-functional view connecting marketing activity to revenue | Implied — requests cross-functional dashboards | Multiple systems |

---

### Desired Outcomes

1. **Cross-functional dashboards** for leadership reporting — replace Excel exports, support board-level PowerPoint
2. **AI assistant** (Raven-style) — ad-hoc natural language queries against business data, no SQL knowledge required
3. **Enterprise data hub** — single platform consolidating all department data, foundational for AI/ML later
4. **AI adoption visibility** — consolidated view of Copilot/Cursor usage, trends, and productivity across engineering teams

---

### Why a Semantic Layer + Agent

The people who need these reports are sales and marketing professionals asking business questions — not engineers writing SQL. "What was last quarter's MQL action rate?" is a business question. The person asking it should not need to know which tables to join or how ARR is calculated.

This creates two requirements:

**Translation:** Natural language must map to the right query against the right data. That's the semantic view — it defines business terms formally (what counts as an MQL, how ARR is calculated, what the action SLA is) and provides the vocabulary for the AI to generate correct queries.

**Governance / trust:** When different teams have historically had different numbers for the same metric, the semantic view is where you fix that. One definition, centrally maintained, used by every question anyone asks. This is what makes the output credible to a VP of Sales, not just a developer.

A **Cortex Agent** sits on top of one or more semantic views and handles the orchestration. It decides which tool (semantic view) to call based on the question, synthesizes results, and responds. This architecture directly supports the cross-functional and expandability requirements — adding a new data domain (forecasting, resource planning, AI adoption) means adding a new tool, not rebuilding the interface.

---

### What We Will Build (MVP Scope)

**Core use case: Sales & Marketing Funnel Intelligence, queried through a Cortex Agent**

A Streamlit-in-Snowflake application with two tabs:

1. **Dashboard tab** — pre-built visualizations showing key funnel KPIs (charts and tables). For board-level and leadership reporting. Always reflects current data.

2. **Agent chat tab** — Cortex Agent backed by a semantic view over the sales/marketing data. Business users ask plain English questions, the agent queries the semantic view and returns answers, Streamlit renders results including visualizations where appropriate.

Synthetic data simulates the full customer journey: HubSpot leads → Salesforce opportunities/activities → NetSuite contracts/invoices.

The demo story:
> *"Right now your sales team is tracking this in spreadsheets and your marketing team is in HubSpot. Here's what it looks like when that data is consolidated, defined consistently, and your sales leader can just ask it a question — without calling anyone."*

---

### Out of Scope

- Live connectors to actual HubSpot / Salesforce / NetSuite
- Power BI integration (Streamlit-in-Snowflake is the demo UI)
- Commission calculation engine
- User authentication / row-level security per rep
- Production-grade error handling and monitoring
- Raw ingestion pipeline (demo assumes gold/business-ready synthetic data; production pipeline story told narratively)
- AI adoption tracking as a primary feature (deferred — can be added as a second agent tool later)

---

### Assumptions

1. Synthetic data is inserted as gold-quality, business-ready tables — no raw ingestion layer needed for demo purposes
2. Schema structure still represents the medallion pattern to tell the right production story
3. Salesforce and NetSuite object models follow standard B2B SaaS conventions — specific eClinical field names TBD
4. The agent is stateless — each question is independent, no session memory
5. Agent uses one primary tool: the sales/marketing semantic view
6. All compute runs on a single standard warehouse (X-Small or Small)
7. Funnel metrics use standard B2B SaaS definitions

---

## Technical Architecture

### Data Layer

```
CURATED schema              →  SEMANTIC layer
────────────────────────────   ──────────────────────────
dim_leads                      Semantic View:
dim_accounts                     eclinical_funnel_sv
dim_opportunities
fact_funnel_events            (Agent Tool 1)
fact_revenue
```

- **CURATED:** Gold-quality synthetic tables, directly populated for demo. In production these would be fed by Dynamic Tables from a raw ingestion layer (OpenFlow / Snowpipe).
- **SEMANTIC:** Semantic View over curated tables — defines business metrics, trusted definitions, and relationships for the Cortex Agent.

*Production path (narrated in demo, not built):* Raw data from HubSpot/Salesforce/NetSuite lands via OpenFlow → Dynamic Tables transform to curated → same semantic view applies.

### Snowflake Features Used

| Feature | Role in MVP |
|---|---|
| Semantic Views | Business metric definitions and trust layer for the agent |
| Cortex Agent | Orchestrator — receives questions, calls semantic view tool, returns answers |
| Streamlit-in-Snowflake | App UI — dashboard tab + agent chat tab |
| Synthetic data (SQL/Python) | Populates gold-quality curated tables |

### Database Structure

```
Database: ECLINICAL_DEMO
  Schema: CURATED   (gold tables)
  Schema: SEMANTIC  (semantic view)
  Schema: APP       (Streamlit lives here)
```

### Key Metrics to Define in Semantic View

- **MQL Volume** — total leads created in period
- **MQL Action Rate** — % of leads contacted within SLA window
- **Lead-to-Opportunity Conversion Rate**
- **Opportunity Win Rate**
- **Average Sales Cycle Length** (days from lead created to close)
- **ARR / New ARR** — from closed-won opportunities
- **Services Backlog** — contracted services value not yet invoiced
- **Bid vs. Did** — contracted services value vs. actual invoiced amount

---

## Agent Scoping Principles

These apply to keep the demo reliable and the scope tight:

- **One primary tool** — the sales/marketing semantic view. Scope every demo question to what that tool can answer.
- **Defined question domain** — the agent answers questions about the sales funnel and marketing performance. Questions outside that domain return a graceful "I can't help with that."
- **Tight system prompt** — explicitly names the agent's purpose, the tool it uses, and what it does not do.
- **Verified queries** — the happy path questions for the demo are tested and locked down before presentation day.
- **Stateless** — each question is independent. Simpler, more predictable, sufficient for this use case.
- **Failure handling** — if the agent returns an unexpected answer, the demo recovery is rehearsed. No questions outside the verified set during the live presentation.
- **Model selection** — test at least two models; prefer the one that stays on task without over-reasoning.

---

## Implementation Steps

> Steps are ordered. No step begins before the prior one is validated.

### Phase 1 — Foundation
- [ ] Create Snowflake database, schemas, and warehouse
- [ ] Write and run synthetic data generation script (gold-quality tables: leads, opportunities, activities, invoices)
- [ ] Validate data looks realistic and covers the demo story (funnel from lead to closed revenue)

### Phase 2 — Semantic Layer
- [ ] Author Semantic View (`eclinical_funnel_sv`) over curated tables
- [ ] Define all metrics and dimensions listed above
- [ ] Add verified queries (VQRs) for the demo question set
- [ ] Validate Cortex Analyst can answer those questions correctly (Analyst is used as the tool internally)

### Phase 3 — Agent
- [ ] Define and configure Cortex Agent with semantic view as Tool 1
- [ ] Write system prompt — purpose, scope, tool description, out-of-scope behavior
- [ ] Test agent against verified question set
- [ ] Confirm reliable, consistent answers on the happy path

### Phase 4 — Application
- [ ] Scaffold Streamlit-in-Snowflake app (two tabs: Dashboard + Agent Chat)
- [ ] Build dashboard tab: KPI cards + funnel chart + trend visualization
- [ ] Build agent chat tab: agent integration, render results including charts where appropriate
- [ ] Polish UI — eClinical's story, not a generic demo

### Phase 5 — Demo Prep
- [ ] Write demo script (2-3 questions that tell the funnel story end to end)
- [ ] Identify 2+ CoCo interactions to narrate during presentation
- [ ] Prepare "path to production" narrative
- [ ] Confirm app runs end-to-end, all queries return correct results

---

## Snowflake Best Practices Applied

- **Semantic Views as trust layer** — business metric definitions are centralized, consistent across all queries, not embedded in application code
- **Cortex Agent for cross-functionality** — architecture supports adding new data domains as tools without rebuilding the interface
- **Streamlit-in-Snowflake** — no external deployment, data never leaves the platform
- **Schema separation** — curated data isolated from app layer; lineage is clear
- **Single warehouse, right-sized** — X-Small for development; upgrade path documented for production load
- **Stateless agent** — simpler, more predictable for demo; session memory can be added in production
