# Data Quality Risk Intelligence

An AI-powered Data Quality Risk Intelligence accelerator, built as a non-invasive intelligence layer on top of existing DQ and governance platforms (primary integration partner: **Collibra**). Designed for regulated financial institutions to predict and prevent regulatory reporting failures, automate rule generation and optimization, and provide executive-level transparency into data quality risk.

> **Current state:** Interactive POC (single-file HTML, `index.html`). Scaled build in progress — converting to a React application with a LangGraph multi-agent backend on Azure. See `ARCHITECTURE.md` for the full system architecture.

---

## Vision

Transform static DQ platforms (e.g. Collibra) into proactive regulatory data risk systems by adding an AI intelligence layer that:

- Predicts and prevents regulatory reporting failures before submission
- Automates DQ rule creation, optimization, and gap detection
- Identifies systemic data quality risk patterns across complex data supply chains
- Enables cross-report discrepancy detection (e.g. FR Y-9C vs FR Y-14Q)
- Accelerates remediation with explainable, confidence-scored insights
- Quantifies financial and regulatory exposure at the data asset level

---

## Problem Statement

Regulated financial institutions invest heavily in third-party DQ platforms (Collibra, Informatica, Atlan). These platforms provide strong governance frameworks but fall short in five key areas:

1. **Rule-based and reactive** — static thresholds, no predictive or cross-report intelligence
2. **Weak root cause intelligence** — failures identified but not explained; no lineage-driven causal inference
3. **Limited regulatory context** — no understanding of report semantics (e.g. Y-9C vs Y-14Q alignment), no cross-schedule reconciliation
4. **Low executive transparency** — dashboards show rule pass/fail, not capital impact or supervisory risk
5. **Manual remediation** — reactive, ticket-based workflows with no intelligent prioritisation

---

## Solution Architecture

The accelerator is structured around the following capability pillars. (Full technical architecture — Azure services, agent workflow, dual-ontology graph, data flow — is documented in `ARCHITECTURE.md`.)

### Pillar 1 — DQ Rule Intelligence *(current POC focus)*
- Ingest structured and unstructured inputs: BRDs, regulatory instructions, data dictionaries
- Analyse existing DQ rule libraries: identify duplicates, gaps, and ineffective thresholds
- Map rules to CDEs and regulatory reports (FR Y-9C, FR Y-14Q, CECL, Basel III/IV)
- Generate new rules: profiling-based, lineage-based reconciliation, cross-report consistency
- Natural language → SQL rule translation

### Pillar 2 — Reporting Risk Intelligence *(planned)*
- Map DQ results to reports, data domains, and CDEs
- Generate report-level risk scores and regulatory risk heatmaps
- Detect cross-report discrepancies (e.g. Y-9C HI income vs Y-14Q PPNR mortgage actuals)
- Early indicators of capital stress and revenue inconsistencies

---

## Agentic Workflow

Five specialized AI agents analyse and propose enhancements to the DQ rule inventory, in two streams that converge on a recommendation agent:

**Risk Coverage stream**
- **Risk Analyzer** — quantify criticality, assess applicable DQ dimensions
- **Effectiveness Analyzer** — identify fields/tables without rules, identify ineffective rules, quantify asset-level effectiveness
- **Health Calculator** — `Health Score = 100 − [Criticality Score × (1 − Rule Effectiveness Score)]`

**Optimization Play stream**
- **Redundancy Inspector** — exact duplicates, range superset/subset containment, superset predicate duplicates
- **Efficiency Inspector** — high failure rate (>50%), SQL vs profiling mismatch, SQL vs declared data type mismatch

**Convergence**
- **Rule Advisor** — new/enhanced rule recommendation (NL + SQL) with rationale and confidence score

These are supported by Tier 1 input-processing skill agents (unstructured, structured, semi-structured) and a FIBO mapping agent that grounds data assets in industry-standard financial concepts.

---

## Adjacent Business Use Cases

The same architecture — dual-ontology knowledge graph, vector store, multi-agent inference, Collibra integration, lineage traversal — generalises well beyond DQ rule intelligence. The following data management and operational risk & resiliency use cases are enabled by the same platform with incremental extension rather than rebuild.

### Data Management

**1. Cross-Report Reconciliation & Consistency**
Detect and explain discrepancies between regulatory reports that draw on shared data (e.g. FR Y-9C Schedule HI vs FR Y-14Q Schedule G). The graph already models which CDEs feed which report line items, so the same traversal that infers DQ impact can flag where the same underlying concept produces inconsistent values across reports — a known supervisory focus area.

**2. CDE Discovery & Criticality Classification**
Use the FIBO mapping agent and lineage graph to automatically identify which data elements are *critical* (feed regulatory reports, have deep downstream dependencies) versus peripheral. This automates a traditionally manual, expert-dependent CDE designation exercise and keeps it current as lineage changes.

**3. Metadata Completeness & Stewardship Gap Detection**
Surface assets with missing business definitions, absent ownership, unmapped lineage, or no FIBO concept — the metadata equivalent of DQ gaps. Prioritised by criticality so stewards address the highest-impact gaps first.

**4. Data Lineage Validation & Impact Analysis**
Traverse the lineage graph to answer "if this source system or field changes, what is affected downstream?" Supports change-impact assessment, migration planning, and decommissioning decisions — all grounded in the same graph used for DQ impact.

**5. Policy & Regulatory Change Management**
When a regulatory instruction changes, the vector store identifies which document sections changed and the graph identifies which CDEs, rules, and reports are governed by the affected requirement — turning a regulatory update into a targeted, prioritised work list.

**6. Reference & Master Data Consistency**
Apply the redundancy and efficiency inspection patterns to reference/master data domains to detect conflicting definitions, duplicate golden records, and taxonomy drift across systems.

### Operational Risk & Resiliency

**7. Control Inventory & Coverage Assessment**
Generalise "DQ rule coverage" to "control coverage." Map operational controls to the risks and processes they mitigate, and surface coverage gaps, redundant controls, and ineffective controls — the operational-risk analogue of the DQ rule health model.

**8. Root Cause Analysis & Issue Clustering**
Use graph-based root cause analysis (lineage + metadata traversal) to trace operational incidents and DQ failures to their upstream origin, cluster recurring issues, and identify systemic control breakdown patterns rather than treating each incident in isolation.

**9. Third-Party / Vendor Data Risk**
Model external data providers and feeds as source-system nodes in the graph. Assess the downstream regulatory and operational exposure created by a vendor feed failure or quality degradation — supporting third-party risk management and concentration-risk analysis.

**10. Operational Resilience & Critical Process Mapping**
Map critical business services to the data assets, systems, and controls they depend on. Traverse the graph to identify single points of failure and assess resilience posture — supporting operational resilience requirements (e.g. impact tolerance mapping).

**11. Audit & Regulatory Exam Readiness**
Because risk scores are deterministic (materiality weights stored on graph edges) and every insight is confidence-scored with an explainability trace, the platform produces an auditable record of how each risk conclusion was reached — directly supporting internal audit and regulatory examination evidence requests.

**12. Predictive Early-Warning Signals**
Combine historical execution patterns (PostgreSQL) with graph structure and regulatory context to surface early-warning indicators — emerging control degradation, rising failure rates on critical paths, or aggregation gaps that precede a reporting failure.

> These use cases are **adjacent opportunities**, not committed scope. They are documented to show the platform's extensibility and to inform roadmap prioritisation once the core DQ Rule Intelligence and Reporting Risk Intelligence pillars are delivered.

---

## Current POC — Feature Summary

The current POC is a single self-contained HTML file (`index.html`) built with vanilla HTML, CSS, and JavaScript using synthetic data.

### What is built

**Landing Page**
- Module entry point with two intelligence tiles (DQ Rule Intelligence — live; Reporting Risk Intelligence — preview) and a distinct Configure & Connect setup card
- Opens by default; each module accessible via tile, with a Home button to return

**Header & Navigation**
- Branded purple gradient header aligned to the Knowledge Suite visual identity
- "Connect to Knowledge Sources" modal — drag & drop file upload plus four connector types: Database, REST API, NAS/Network File Share, Cloud Storage

**Filter Bar**
- Data Domain, Regulatory Report, System, and Health Score filters
- Free-text search; all filters include tooltip descriptions

**KPI Row (6 metrics)**
- Avg. Criticality Score, Total Active DQ Rules, Avg. Rule Effectiveness Score, Avg. Health Score, High-Risk Data Assets, Rules w/ Optimization Opportunities — each with a calculation-methodology tooltip

**Charts Row (3 charts)**
- *DQ Rule Health* donut — Good / Medium / Poor health distribution; click a segment to filter the inventory
- *Optimization Opportunities* donut — Modify rule logic (high false positives) / Decommission rules (redundant) / Shift rules upstream (lineage-based, shown as coming soon); click to filter
- *Rule Coverage Heatmap* — currently a pending placeholder (regulatory report mapping, planned for a later sprint)

**Data Asset Risk Inventory**
- Table of synthetic data assets with columns: Data Asset Name, Data Asset Type (Column/Table), Domain, DQ Rules, Optimize, Health Category, Health Score
- Rows dim when filtered out (not removed), preserving table structure
- Click any row to load DQ Risk Insights for that asset
- Active filter bar with dismissible pills and "Clear all"

**DQ Risk Insights Panel**
- AI-generated analysis per selected data asset, in three collapsible sections: Ineffective/Gap Rules, Optimization Opportunities, Effective Controls
- Each insight: confidence score, title, description, expandable detail
- Detail view shows regulatory impact tags, business impact, and for gap rules: proposed rule logic in natural language, proposed SQL, and target systems
- Accept / Modify / Reject actions; AI Reasoning section with metadata signals and model reasoning narrative
- **Modify flow:** modal showing the full proposed rule (NL + SQL + systems) with a natural language input to describe changes, submitted for steward review

**Footer**
- Pending review count, last sync indicator, and Export / Push to DQ Platform / Approve Selected Changes actions

**Tooltip System**
- Single body-appended floating tooltip node (`#tt-float`) using event delegation; never clipped by parent overflow

---

## Synthetic Data Scope

**Reports covered:** FR Y-9C (Schedule HI), FR Y-14Q (Schedule G — PPNR), CECL (ASC 326), Basel III/IV

The POC ships with a set of synthetic data assets (columns and tables) across domains (Enterprise, Loan, Treasury, Actuals, Risk, Reference, Customer), each carrying a health category, numeric health score, rule count, optimization count, and AI-generated insights. Assets with gap insights include proposed rule logic (NL + SQL) and target systems to drive the Modify flow.

**Health Score formula:** `100 − [Criticality Score × (1 − Rule Effectiveness Score)]`

---

## Tech Stack

### Current POC
- Vanilla HTML5, CSS3, JavaScript (ES6+) — single file, no build tools
- Google Fonts: Plus Jakarta Sans, JetBrains Mono; inline SVG donut charts

### Scaled application (see `ARCHITECTURE.md`)
- **Frontend:** React (TypeScript, Vite, Tailwind, Zustand) on Azure Static Web Apps
- **Orchestration:** LangGraph multi-agent workflow on Azure ML / Azure Databricks
- **AI layer:** Anthropic Claude (`claude-sonnet-4-6`) via an LLM Garden routing layer
- **Vector store:** Azure AI Search (semantic retrieval / RAG)
- **Graph:** Azure Cosmos DB (Gremlin) — dual-ontology (DQ + FIBO) knowledge graph
- **Structured store:** Azure PostgreSQL
- **Raw landing zone:** Azure Blob Storage (Collibra extracts via Aspire Connectors)
- **Integration:** Collibra REST API (GET inbound, POST outbound write-back)
- **Gateway & security:** Azure API Management, Azure AD, Azure Key Vault

---

## Key Regulatory Context

| Report | Schedule | Focus Area |
|--------|----------|------------|
| FR Y-9C | Schedule HI | Consolidated income statement — interest income, fee income |
| FR Y-14Q | Schedule G (PPNR) | Pre-provision net revenue — mortgages, retail lending, investment management |
| CECL (ASC 326) | — | Credit loss allowance methodology and model validation |
| Basel III/IV | HC-R | Risk-weighted assets, capital adequacy ratios |
| BCBS 239 | — | Risk data aggregation and reporting principles |

Cross-report reconciliation between FR Y-9C Schedule HI and FR Y-14Q Schedule G (PPNR) is a primary use case — misalignment between these two reports is a known supervisory focus area.

---

## Repository Files

| File | Purpose |
|------|---------|
| `index.html` | Interactive POC — source of truth for all UI components, synthetic data, filter logic, and insight rendering |
| `README.md` | This file — project overview, features, and use cases |
| `ARCHITECTURE.md` | Full system architecture — Azure services, agent workflow, dual-ontology graph, data flow, build phases |
| `REQUIREMENTS.md` | Functional requirements (to be provided from Excel) |

---

## Design System

Key CSS variables (defined in `index.html`):

```css
--purple: #7c3aed          /* Primary brand / action */
--purple-light: #ede9fe    /* Backgrounds, active states */
--purple-dark: #5b21b6     /* Hover states */
--green: #059669           /* Good Health / Effective */
--amber: #d97706           /* Medium Health / Optimize */
--red: #dc2626             /* Poor Health / Gaps / critical */
--blue: #2563eb            /* Regulatory tags */
--sans: 'Plus Jakarta Sans', sans-serif
--mono: 'JetBrains Mono', monospace
```

---

## Development Notes

- Synthetic data lives in the `KPIS`, `HEATMAP`, and `ASSETS` arrays in the JS — easy to extend
- Tooltip system uses event delegation — add `class="tt-trigger"` and `data-tip="..."` to any element
- Filter system (`activeFilters` state) supports health and optimization-type dimensions — extensible with additional filter axes
- Insight items with `type:'gap'` and a `ruleNL` / `ruleSQL` / `systems` payload automatically render full proposed rule logic and trigger the Modify modal
