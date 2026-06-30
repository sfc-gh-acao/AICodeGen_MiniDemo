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
- Set up GitHub MCP in CoCo
- Created GitHub repo: https://github.com/sfc-gh-acao/AICodeGen_MiniDemo

### Key decisions made
- **Use case chosen:** Sales & Marketing Funnel Intelligence — selected because it directly addresses the #1 pain from the call (manual funnel reporting) and also enables the "Raven-style AI assistant" that Venu was visibly excited about during the demo
- **Skipped:** Engineering AI adoption tracking — secondary use case, weaker demo story, less customer excitement in the call
- **Architecture:** Medallion (raw → curated → semantic) with Dynamic Tables and Semantic Views
- **App:** Streamlit-in-Snowflake, two tabs: Dashboard + AI Assistant (Cortex Analyst)
- **Data sources simulated:** HubSpot leads, Salesforce opportunities/activities, NetSuite invoices

### Status
Plan is drafted. Environment setup in progress (GitHub repo created, Snowflake Workspace git integration pending).

---

<!-- Template for future entries:

## Session N — [Date]

### What happened

### Decisions made / changed

### What I tried that didn't work

### Status at end of session

-->
