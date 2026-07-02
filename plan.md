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

Synthetic data simulates the full customer journey: Salesforce leads/opportunities/activities + NetSuite contracts/invoices.

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
2. Salesforce and NetSuite object models follow standard B2B SaaS conventions — eClinical-specific custom fields assumed and narrated
3. The agent is stateless — each question is independent, no session memory
4. Agent uses one primary tool: the sales/marketing semantic view
5. All compute runs on a single standard warehouse (X-Small or Small)
6. Funnel metrics use standard B2B SaaS definitions

---

## Technical Architecture (v3 — revised July 2, 2026)

### Demo Pattern
Follows Cody's end-to-end reference implementation. Two Snowflake notebooks + one Streamlit app.

- **Notebook 1:** Create database/schema, build all dim + fact tables, load synthetic data, run validation queries, run example analytical queries, FastGen to generate semantic YAML, create semantic view, CREATE AGENT
- **Notebook 2:** Data validation framework — staging tables (VARCHAR mirrors), intentional bad records, validation rules (NULL / type / domain / FK / business rules), VALIDATION_LOG, promote clean records
- **Streamlit:** 3 tabs — Data Quality validation dashboard, Funnel Dashboard (KPI cards + charts), AI Assistant (agent chat)

### Data Sources
- **Salesforce** — full customer journey: leads, accounts, opportunities, activities, sales reps
- **NetSuite** — post-sales finance: contracts/SOWs, invoices, bid vs. did
- **HubSpot** — NOT modeled. Lead source attribution travels into Salesforce when leads sync. Mentioned in demo narrative as future expansion only.

### Star Schema Data Model

```
FACT_LEADS              [Salesforce]
├── DIM_DATE
├── DIM_ACCOUNT         [Salesforce → shared with NetSuite]
├── DIM_SALES_REP       [Salesforce]
└── DIM_LEAD_SOURCE     [Salesforce]

FACT_OPPORTUNITIES      [Salesforce]
├── DIM_DATE
├── DIM_ACCOUNT         [shared]
├── DIM_SALES_REP       [Salesforce]
├── DIM_LEAD_SOURCE     [Salesforce]
└── DIM_PRODUCT         [optional — not load-bearing for demo questions]

FACT_ACTIVITIES         [Salesforce]
├── DIM_DATE
├── DIM_ACCOUNT         [shared]
└── DIM_SALES_REP       [Salesforce]

FACT_CONTRACTS          [NetSuite]
├── DIM_DATE
├── DIM_ACCOUNT         [shared]
└── DIM_PRODUCT         [shared]

FACT_INVOICES           [NetSuite]
├── DIM_DATE
└── DIM_ACCOUNT         [shared]
```

DIM_ACCOUNT is the conformed dimension bridging Salesforce and NetSuite. Populated from Salesforce but referenced by all fact tables.

### Database Structure

```
Database: ECLINICAL_DEMO
  Schema: SALES_FUNNEL  (all tables, staging, semantic view, agent — single schema for demo)
```

Production path (narrated, not built): CURATED schema fed by Dynamic Tables from OpenFlow connectors to Salesforce/NetSuite. Medallion split told narratively.

### Snowflake Features Used

| Feature | Role in demo |
|---|---|
| Snowflake Notebooks | Two notebooks: dimensional model + data validation |
| Semantic Views | Auto-generated via FastGen, defines business metrics and relationships |
| Cortex Agent | Orchestrator backed by semantic view, used through Snowflake Intelligence |
| Streamlit-in-Snowflake | 3-tab app: Data Quality + Funnel Dashboard + AI Assistant |
| SYSTEM$CORTEX_ANALYST_FAST_GENERATION | Auto-generates semantic YAML from table metadata |
| SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML | Materializes semantic view from FastGen output |

### Key Metrics (7 Demo Questions from Venu's Call)

| # | Question | Tables |
|---|---|---|
| 1 | Marketing funnel performance — leads → MQL → pipeline by channel | FACT_LEADS, DIM_LEAD_SOURCE, DIM_DATE |
| 2 | MQL action rate — days from lead to first rep follow-up (broken today) | FACT_LEADS, FACT_ACTIVITIES, DIM_SALES_REP |
| 3 | Pipeline to revenue — win rate, ARR, time to close by rep + segment | FACT_OPPORTUNITIES, DIM_SALES_REP, DIM_ACCOUNT |
| 4 | Commission / quota tracking — closed won ARR per rep per quarter | FACT_OPPORTUNITIES, DIM_SALES_REP, DIM_DATE |
| 5 | Bid vs. did — contracted services vs. actual invoiced | FACT_CONTRACTS, FACT_INVOICES, DIM_ACCOUNT |
| 6 | Finance backlog — contracted revenue not yet invoiced | FACT_CONTRACTS, FACT_INVOICES, DIM_DATE |
| 7 | GRR / ERR — ARR retained and expanded, for board reporting | FACT_CONTRACTS, DIM_ACCOUNT, DIM_DATE |

---

## Agent Scoping Principles

- **One primary tool** — the sales funnel semantic view
- **Defined question domain** — sales funnel, marketing performance, revenue, services. Out-of-scope returns a graceful refusal.
- **Tight system prompt** — names the agent's purpose, tool, and boundaries explicitly
- **Verified queries** — 7 demo questions tested and locked before presentation day
- **Stateless** — each question independent; sufficient for this demo
- **Failure handling** — rehearsed recovery; no off-script questions during live presentation

---

## Implementation Steps (v3)

### Notebook 1 — Dimensional Model + Agent
- [ ] Create database ECLINICAL_DEMO and schema SALES_FUNNEL
- [ ] Create dimension tables: DIM_DATE, DIM_ACCOUNT, DIM_SALES_REP, DIM_LEAD_SOURCE, DIM_PRODUCT
- [ ] Create fact tables: FACT_LEADS, FACT_OPPORTUNITIES, FACT_ACTIVITIES, FACT_CONTRACTS, FACT_INVOICES
- [ ] Load synthetic data — realistic eClinical-domain records covering all 7 demo questions
- [ ] Run validation queries (row counts, FK integrity)
- [ ] Run example analytical queries demonstrating each metric
- [ ] FastGen → SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML → verify semantic view
- [ ] CREATE OR REPLACE AGENT backed by semantic view

### Notebook 2 — Data Validation Framework
- [ ] Create VALIDATION_LOG table
- [ ] Create VARCHAR-only staging tables (STG_DIM_*, STG_FACT_*)
- [ ] Load staging data with intentional bad records (NULL required fields, invalid types, orphaned FKs, business rule violations)
- [ ] Run validation rules: NULL checks, data type, domain, FK integrity, business rules
- [ ] Validation summary and per-record scorecard
- [ ] Promote clean records to production tables

### Streamlit App — 3 Tabs
- [ ] Tab 1 — Data Quality: validation run selector, issue heatmap, pass/fail scorecard, record drill-down with fix SQL
- [ ] Tab 2 — Funnel Dashboard: KPI cards (MQL volume, win rate, ARR, bid vs. did), funnel chart, trend chart
- [ ] Tab 3 — AI Assistant: agent chat interface, pre-loaded demo questions, renders results as tables/charts

### Demo Prep
- [ ] Write demo script: manual reporting before-state → validation layer → business dashboard → agent moment
- [ ] Identify 2+ CoCo interactions to narrate
- [ ] Rehearse failure recovery for off-script agent responses
- [ ] Confirm full end-to-end run

---

## Snowflake Best Practices Applied

- **Semantic Views as trust layer** — business metric definitions are centralized, consistent across all queries, not embedded in application code
- **Cortex Agent for cross-functionality** — architecture supports adding new data domains as tools without rebuilding the interface
- **Streamlit-in-Snowflake** — no external deployment, data never leaves the platform
- **Single warehouse, right-sized** — X-Small for development; upgrade path documented for production load
- **Stateless agent** — simpler, more predictable for demo; session memory can be added in production
