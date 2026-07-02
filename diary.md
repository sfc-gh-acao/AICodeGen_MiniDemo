# eClinical Solutions — Implementation Diary

> Every decision, every iteration, every "I tried X and switched to Y" gets recorded here. This file proves to CoCo (and to me) what has already been done and why.

---

## Session 1 — June 30, 2026

### What happened
- Read project instructions (ACE Proficiency Assessment doc) and Gong recording (call ID: 649748351206546438)
- Identified customer: eClinical Solutions, contact: Venu Mallarapu, AE: Jason Pucciarelli
- Mapped customer pains and desired outcomes from the call
- Created `plan.md` with full requirements document, technical architecture, and implementation steps
- Created `diary.md` (this file) and `documentation.md` (placeholder)
- Saved project memory for cross-session context

### Key decisions made
- **Use case chosen:** Sales & Marketing Funnel Intelligence — selected because it directly addresses the #1 pain from the call (manual funnel reporting) and also enables the "Raven-style AI assistant" that Venu was visibly excited about during the demo
- **Skipped:** Engineering AI adoption tracking — secondary use case, weaker demo story, less customer excitement in the call
- **Architecture:** Medallion (raw → curated → semantic) with Dynamic Tables and Semantic Views
- **App:** Streamlit-in-Snowflake, two tabs: Dashboard + AI Assistant (Cortex Analyst)
- **Data sources simulated:** HubSpot leads, Salesforce opportunities/activities, NetSuite invoices

### Status
Plan drafted (v1). Architecture was Cortex Analyst + Streamlit. Pending further discussion before build.

---

## Session 2 — June 30, 2026

### What happened
- Extended planning discussion: analyzed the Gong call in depth, worked through architecture decisions
- Set up GitHub MCP in CoCo (official github-mcp-server binary, macOS ARM64)
- Created GitHub repo: https://github.com/sfc-gh-acao/AICodeGen_MiniDemo
- Connected Snowflake Workspace (Snowhouse) to the repo via WORKSPACE_GIT_OAUTH (OAuth2, required `.git` suffix in URL)
- Pushed initial project files to repo
- Revised plan.md to reflect all decisions made this session

### Key decisions made

**Architecture pivot: Cortex Analyst → Cortex Agent**
- Analyst is reliable and clean but limited to one semantic view / one data domain
- Agent orchestrates multiple tools (semantic views), handles cross-functional questions, and maps to the enterprise data hub vision Venu described
- Analyst still does the actual query work — it becomes Tool 1 inside the agent
- Decision driven by: cross-functionality requirement, expandability, business user audience

**Semantic view as translation + governance (trust layer)**
- The people generating reports are sales/marketing professionals, not engineers — they ask business questions not SQL
- Semantic view bridges that gap and also enforces consistent metric definitions across teams
- This directly addresses the "every team has their own numbers" problem implicit in the call
- Governance is not a nice-to-have — it's what makes the output credible to a VP of Sales

**Data architecture simplified for demo**
- Assumption: synthetic data inserted as gold/business-ready tables (no raw ingestion layer for demo)
- Medallion structure still represented in schema naming to tell the right production story
- Dynamic Tables removed from MVP scope — production pipeline story told narratively during demo

**AI adoption tracking deprioritized**
- Venu's AI adoption use case = eClinical already uses Copilot/Cursor and wants to consolidate that usage data, NOT that they want to start using AI
- Good secondary use case but not load-bearing for this demo
- Can be added later as a second agent tool

**Agent scoping principles locked**
- Stateless (no session memory needed for MVP)
- One primary tool (sales/marketing semantic view)
- Tight system prompt, verified queries for demo happy path, rehearsed failure recovery

**Deliverable definition confirmed**
- MVP = working, demo-able, runs live, produces visible output
- Not a plan, not a mockup — something you can interact with
- Bar: working and believable, not complete or production-ready
- Presentation includes narrating 2+ CoCo interactions and explaining the path to production

### What changed from Session 1
- Cortex Analyst → Cortex Agent as the AI interface
- Raw ingestion layer removed from MVP scope
- AI adoption tracking moved from "deferred" to explicitly out of scope for primary build
- Resolved open questions: stateless chat, no conversation history needed

### Status
Plan is solid (v2). Direction confirmed: Cortex Agent + Semantic View + Streamlit. Ready to move to Phase 1 build when Amy confirms.

---

## Session 3 — July 2, 2026

### What happened
- Mentor conversation with Cody informed a significant demo approach revision
- Reviewed Cody's reference implementation: composite_decking_dimensional_model.ipynb, composite_decking_data_validation.ipynb, streamlit_app.py
- Deep research into eClinical's business context, Salesforce/NetSuite data formats, and what Venu explicitly asked for on the Gong call
- Designed the full star schema data model grounded in Venu's 7 stated business questions
- Connected local project folder to GitHub repo (folder was never git-initialized before)
- Confirmed push method: GitHub MCP (same OAuth as Snowflake Workspace connection) — local git push via HTTPS 403s due to fine-grained PAT scoped only to ACE_DemoProject

### Key decisions made

**Demo pattern revised: follow Cody's end-to-end pattern**
- Two Snowflake notebooks + one Streamlit app
- Notebook 1: dimensional model, data load, FastGen semantic view, Cortex Agent
- Notebook 2: data validation framework (staging → validation rules → promotion)
- Streamlit: 3 tabs — Data Quality dashboard, Funnel dashboard, AI Assistant chat

**Data sources narrowed: Salesforce + NetSuite only**
- HubSpot is NOT a modeled data source
- Reason: Venu said "I think it's HubSpot or something" — not confirmed. Lead source attribution travels from HubSpot into Salesforce when a lead syncs, so Salesforce already contains the marketing context
- HubSpot mentioned in demo narrative as a future expansion, not built
- Salesforce = full customer journey (leads, opportunities, activities, accounts, reps)
- NetSuite = post-sales finance layer (contracts/SOWs, invoices, bid vs. did)

**Dynamic Tables confirmed not built**
- Was always the plan (plan.md already said "told narratively")
- Cody's pattern confirms: load directly into production tables, no DTs in demo

**Semantic view creation: FastGen**
- Use SYSTEM$CORTEX_ANALYST_FAST_GENERATION to auto-generate YAML from table metadata
- Then SYSTEM$CREATE_SEMANTIC_VIEW_FROM_YAML to materialize
- No manual YAML authoring

**Star schema data model locked**
- 5 fact tables: FACT_LEADS, FACT_OPPORTUNITIES, FACT_ACTIVITIES, FACT_CONTRACTS, FACT_INVOICES
- 5 dimension tables: DIM_DATE, DIM_ACCOUNT, DIM_SALES_REP, DIM_LEAD_SOURCE, DIM_PRODUCT
- DIM_ACCOUNT is the conformed dimension bridging Salesforce and NetSuite
- DIM_PRODUCT is optional — not load-bearing for any of the 7 demo questions

**7 demo questions locked (from Venu's call)**
1. Marketing funnel performance → FACT_LEADS + DIM_LEAD_SOURCE + DIM_DATE
2. MQL action rate (broken today — highest impact demo moment) → FACT_LEADS + FACT_ACTIVITIES + DIM_DATE + DIM_SALES_REP
3. Pipeline to revenue → FACT_OPPORTUNITIES + DIM_SALES_REP + DIM_ACCOUNT + DIM_DATE
4. Commission and quota tracking → FACT_OPPORTUNITIES + DIM_SALES_REP + DIM_DATE
5. Bid vs. did → FACT_CONTRACTS + FACT_INVOICES + DIM_ACCOUNT + DIM_DATE
6. Finance backlog → FACT_CONTRACTS + FACT_INVOICES + DIM_DATE
7. GRR / ERR (board reporting) → FACT_CONTRACTS + DIM_ACCOUNT + DIM_DATE

**Demo narrative beats locked**
- HubSpot beat: "In production you'd connect HubSpot directly — but because Salesforce already carries lead source attribution when a lead syncs, your marketing team's data is already represented here."
- Manual reporting before-state: Export from Salesforce → export from NetSuite → VLOOKUP in Excel → copy to PowerPoint. Every cycle, every team, slightly different numbers. This is what the demo replaces.
- Custom field assumption beat: "In a real implementation the first step is mapping your Salesforce custom fields to this model. We've made reasonable assumptions about what those look like for a B2B software company."

### What changed from Session 2
- Added validation notebook + Streamlit validation tab (new scope from Cody's pattern)
- Removed HubSpot as a modeled data source (narrowed from Session 1/2 assumption)
- Added NetSuite as an explicit second source (was implicit before)
- Replaced manual YAML authoring with FastGen
- Locked the 7 demo questions grounded in Venu's exact words
- Star schema with 5 facts + 5 dims designed and validated against the 7 questions

### Status
Data model is designed. Next step: define specific columns for each table (grounded in real Salesforce/NetSuite field names + eClinical-specific custom fields), then build Notebook 1.

---

<!-- Template for future entries:

## Session N — [Date]

### What happened

### Decisions made / changed

### What I tried that didn't work

### Status at end of session

-->
