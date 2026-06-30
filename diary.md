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

<!-- Template for future entries:

## Session N — [Date]

### What happened

### Decisions made / changed

### What I tried that didn't work

### Status at end of session

-->
