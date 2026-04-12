# AIDataQualityACN
# Data Quality Risk Intelligence

An AI-powered Data Quality Risk Intelligence accelerator, built as a non-invasive intelligence layer on top of existing DQ platforms (primarily Collibra). Designed for regulated financial institutions to predict and prevent regulatory reporting failures, automate rule generation and optimization, and provide executive-level transparency into data quality risk.

> **Current state:** Interactive POC (single-file HTML). The immediate next phase is converting this into a structured React application and wiring up live AI and platform integrations.

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

The accelerator is structured around three capability pillars:

### Pillar 1 — Control Intelligence *(current POC focus)*
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

### Pillar 3 — Remediation Intelligence *(planned)*
- Graph-based root cause analysis using lineage and metadata
- Detect upstream system failures, repeated issue clusters, control breakdown patterns
- Generate explainable root cause narratives and AI-driven remediation recommendations
- Severity-based prioritisation and workflow tool integration (ServiceNow, Collibra)

---

## Current POC — Feature Summary

The current POC is a single self-contained HTML file (`dq_risk_intelligence_v5.html`) built with vanilla HTML, CSS, and JavaScript using synthetic data.

### What is built

**Header & Navigation**
- Branded header with purple gradient design aligned to the Knowledge Suite visual identity
- "Connect to Knowledge Sources" modal — supports drag & drop file upload and four connector types: Database (SQL Server, Oracle, Snowflake, Databricks), REST API (Collibra, ServiceNow, Informatica), NAS/Network File Share, Cloud Storage

**Filter Bar**
- Data Domain filter (Customer, Transaction, Risk, Reference, Financial)
- Regulatory Report filter (FR Y-9C, FR Y-14Q, CECL ASC 326, Basel III/IV)
- Criticality filter (Low, Medium, High, Critical)
- Free-text search
- All filters include tooltip descriptions

**Top Section — three-column layout**
- *Left:* Three vertical KPI cards — DQ Rule Effectiveness Score (87.3%), High-Risk DQ Rule Gaps (23), Total Active DQ Rules (1,247) — each with tooltip explaining calculation methodology
- *Centre:* DQ Rule Health donut chart — three segments (Effective 68%, Optimize 15%, Gaps 17%). Clicking a legend item filters the inventory below. Donut centre updates to reflect filtered count.
- *Right:* Rule Coverage Heatmap — DQ risk coverage score by Data Domain × Regulatory Report. Clicking any cell filters the inventory to matching assets. Active filters shown as dismissible pills.

**Data Asset Risk Inventory (60% width)**
- Table of 8 synthetic data assets with columns: Asset/CDE, Asset Type (CDE/Table), Domain, DQ Rules, Health, Risk Score
- Rows dim when filtered out (not removed), preserving table structure
- Click any row to load DQ Risk Insights for that asset
- Active filter bar shows current filter pills with individual clear and "Clear all"

**DQ Risk Insights Panel (40% width)**
- AI-generated analysis panel loaded per selected data asset
- Three collapsible sections: Ineffective/Gap Rules, Optimization Opportunities, Effective Controls
- Each insight card shows: confidence score, title, description
- Expandable detail view per insight showing:
  - Regulatory impact tags (e.g. FR Y-14Q, BCBS 239)
  - Business impact summary
  - For gap rules: proposed rule logic in natural language, proposed SQL, and target systems
  - Accept / Modify / Reject action buttons
  - AI Reasoning section with metadata signals and model reasoning narrative
- **Modify flow:** Opens a modal showing the full proposed rule (NL + SQL + systems), with a natural language input field to describe changes. Submits for steward review.

**Footer**
- Pending review count badge
- Last sync indicator
- Export Report, Push to DQ Platform, Approve Selected Changes buttons

**Tooltip System**
- Single body-appended floating tooltip node (`#tt-float`) using event delegation
- Works on all static and dynamically rendered elements
- Never clipped by parent overflow or stacking contexts

---

## Synthetic Data Scope

The POC uses synthetic data aligned to the MVP regulatory scope:

**Reports covered:**
- FR Y-9C (Schedule HI — Income Statement)
- FR Y-14Q (Schedule G — PPNR)
- CECL (ASC 326)
- Basel III/IV

**Data assets (8 synthetic CDEs/Tables):**

| ID | Name | Type | Domain | Rules | Health | Risk Score |
|----|------|------|--------|-------|--------|------------|
| CDE-001 | Customer ID | CDE | Customer Data | 14 | Effective | 15 |
| CDE-002 | Transaction Amount Reconciliation | CDE | Transaction Data | 8 | Gaps | 78 |
| TBL-003 | Reference Data Completeness | Table | Reference Data | 11 | Effective | 25 |
| CDE-004 | Duplicate Customer Detection | CDE | Customer Data | 5 | Optimize | 22 |
| CDE-005 | Credit Risk Rating | CDE | Risk Data | 18 | Effective | 8 |
| CDE-006 | Basel III Capital Adequacy | CDE | Financial Data | 3 | Gaps | 91 |
| CDE-007 | PPNR Mortgage Actual | CDE | Financial Data | 6 | Optimize | 44 |
| TBL-008 | Investment Mgmt Revenue | Table | Financial Data | 0 | Gaps | 85 |

**Risk Score formula (synthetic):**
`(Failure Rate × 0.5) + (Regulatory Exposure × 0.3) + (Lineage Depth × 0.2)`

---

## Tech Stack

### Current POC
- Vanilla HTML5, CSS3, JavaScript (ES6+)
- No build tools or dependencies — single file, opens directly in browser
- Google Fonts: Plus Jakarta Sans, JetBrains Mono
- SVG donut chart rendered inline

### Planned stack for full application
- **Frontend:** React (component-based, state management via hooks or Zustand)
- **AI layer:** Anthropic Claude API (`claude-sonnet-4-6`) for rule generation, gap analysis, NL→SQL translation, and insight narrative generation
- **Platform integration:** Collibra REST API (bidirectional — ingest metadata/lineage, push AI-generated rules)
- **Data layer:** To be defined based on deployment environment

---

## Repository Structure (target — post-conversion)

```
dq-risk-intelligence/
├── README.md
├── public/
│   └── index.html
├── src/
│   ├── components/
│   │   ├── Header/
│   │   ├── FilterBar/
│   │   ├── KPIColumn/
│   │   ├── DQRuleHealthChart/
│   │   ├── RuleCoverageHeatmap/
│   │   ├── DataAssetInventory/
│   │   ├── DQRiskInsights/
│   │   ├── ModifyRuleModal/
│   │   └── ConnectSourcesModal/
│   ├── data/
│   │   └── syntheticAssets.js        # Current synthetic dataset
│   ├── hooks/
│   │   └── useFilters.js             # Filter state management
│   ├── services/
│   │   ├── claudeApi.js              # Anthropic Claude API client
│   │   └── collibraApi.js            # Collibra REST API client
│   ├── App.jsx
│   └── index.js
├── dq_risk_intelligence_v5.html      # Current POC (reference)
└── package.json
```

---

## Immediate Next Steps

These are the priority items for the next development phase in Claude Code:

1. **Convert to React** — decompose the single HTML file into the component structure above, preserving all existing functionality and styling exactly
2. **Add live Claude API calls** — replace hardcoded `aiNote` and insight text with real Claude API responses using the existing synthetic data as context
3. **NL→SQL rule generation** — wire the Modify modal to call Claude API with the natural language instruction and return updated SQL for steward review
4. **Reporting Risk Intelligence pillar** — build the second pillar: cross-report discrepancy detection between Y-9C and Y-14Q using synthetic data
5. **Collibra mock integration** — simulate the Collibra API push/pull with a local mock server to demonstrate the bidirectional integration flow
6. **Expand synthetic dataset** — add more CDEs, more regulatory reports, and more complex lineage scenarios to stress-test the UI

---

## Design System

The UI follows a consistent design language. Key variables:

```css
--purple: #7c3aed          /* Primary brand / action colour */
--purple-light: #ede9fe    /* Backgrounds, active states */
--purple-dark: #5b21b6     /* Hover states */
--green: #059669           /* Effective / pass */
--amber: #d97706           /* Optimize / warning */
--red: #dc2626             /* Gaps / critical */
--blue: #2563eb            /* Regulatory tags */
--text: #111827            /* Primary text */
--text3: #6b7280           /* Secondary / labels */
--sans: 'Plus Jakarta Sans', sans-serif
--mono: 'JetBrains Mono', monospace
```

Header uses a `linear-gradient(135deg, #1e0a3c → #7c3aed)` with a radial dot grid overlay. All tooltips use a single body-appended `#tt-float` node positioned via JavaScript to avoid clipping.

---

## Collibra Integration Approach

The accelerator is designed as a non-invasive layer — it enhances Collibra, not replaces it.

**Inbound (Collibra → Accelerator):**
- Metadata, lineage graphs, DQ rule libraries via Collibra REST API
- Rule execution results and issue logs
- Asset classifications and CDE mappings

**Outbound (Accelerator → Collibra):**
- AI-generated rule recommendations (after steward approval)
- Risk score annotations on lineage
- Enriched remediation workflow tickets
- Prioritisation flags on existing issues

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

## Contributing / Development Notes

- All synthetic data lives in the `ASSETS` and `HEATMAP` arrays in the JS — easy to extend
- The tooltip system uses event delegation on `document` — add `class="tt-trigger"` and `data-tip="..."` to any element to get a tooltip automatically
- The filter system (`activeFilters` state object) supports three simultaneous dimensions: health, domain, report — easy to extend with additional filter axes
- Insight items with `type:'gap'` and a `ruleNL` / `ruleSQL` / `systems` payload automatically render the full proposed rule logic and trigger the Modify modal