# DQ Risk Intelligence — System Architecture

> **Status:** Scaled build. Azure sandbox deployment in progress.  
> **Reference UI:** `index.html` (single-file POC — source of truth for all UI components and synthetic data)  
> **Functional requirements:** See `REQUIREMENTS.md` (to be provided separately)

---

## Architecture Overview

The DQ Risk Intelligence platform is an AI-powered agentic system built as a non-invasive intelligence layer on top of **Collibra** (primary integration partner) and other enterprise DQ/governance platforms. It is structured around five layers: Data Sources & Storage, Ingestion, Agent Orchestration, Memory, and Tool/UI delivery — all deployed on Azure. The agent orchestration layer runs a **multi-agent workflow** of five specialized analysis/recommendation agents, supported by a set of input-processing skill agents.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AZURE SANDBOX                                      │
│                                                                                 │
│  ┌──────────────┐   ┌───────────────────┐   ┌──────────────────────────────┐  │
│  │ DATA SOURCES │   │ API GATEWAY &     │   │     AGENT ORCHESTRATOR       │  │
│  │              │   │ SECURITY          │   │   (multi-agent workflow)     │  │
│  │ Collibra     │◀─▶│ Azure API Mgmt    │──▶│  Risk → Effectiveness →      │  │
│  │ (REST G/P)   │   │ Azure AD &        │   │  Health │ Redundancy +       │  │
│  │ Others       │   │ Key Vault         │   │  Efficiency → Rule Advisor   │  │
│  └──────────────┘   │ Key Vault         │   │  ┌─────────────────────┐    │  │
│                     └───────────────────┘   │  │    LLM Garden       │    │  │
│  ┌──────────────┐                           │  │  (Claude / Azure    │    │  │
│  │ DATA STORAGE │   ┌───────────────────┐   │  │   OpenAI gateway)   │    │  │
│  │              │   │ INGESTION LAYER   │   │  └─────────────────────┘    │  │
│  │ Data Tables  │   │                   │   │  ┌─────────────────────┐    │  │
│  │ Structured   │──▶│ Doc Intelligence  │──▶│  │ Semantic Kernel /   │    │  │
│  │ Unstructured │   │ Python OCR        │   │  │ LangChain / CrewAI  │    │  │
│  └──────────────┘   │ LLM-Based         │   │  └─────────────────────┘    │  │
│         ▲           │                   │   │  ┌─────────────────────┐    │  │
│         │           │ Microsoft         │   │  │ Azure ML /          │    │  │
│         │           │ GraphRAG          │   │  │ Azure Databricks    │    │  │
│         │           └───────────────────┘   │  └─────────────────────┘    │  │
│         │                                   └──────────────┬───────────────┘  │
│         │                                                  │                  │
│         │           ┌──────────────────────────────────────▼──────────────┐  │
│         │           │                    MEMORY                            │  │
│         │           │                                                      │  │
│         │           │  Azure AI Search / Databricks Vector Search          │  │
│         │           │  Azure Cosmos DB / Cosmos DB (Gremlin) — Graph      │  │
│         │           │  Azure PostgreSQL — Structured operational store     │  │
│         │           └──────────┬───────────────────────────────────────────┘  │
│         │                      │                                               │
│  ┌──────┴──────┐  ┌────────────▼──────────────────────────────────────────┐  │
│  │  DATA LAYER │  │                   TOOL LAYER + UI                      │  │
│  │             │  │                                                        │  │
│  │ Azure Blob  │  │  Azure Functions   Logic Apps   React UI (Static Web   │  │
│  │ Storage     │  │                                  Apps) + Collibra      │  │
│  └─────────────┘  └────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                         │
                               ┌─────────▼──────────┐
                               │   Anthropic API     │
                               │  claude-sonnet-4-6  │
                               └─────────────────────┘
```

---

## Layer-by-Layer Specification

### 1. Data Sources

| Source | Role |
|--------|------|
| **Collibra** | **Primary integration partner.** Source of rule libraries, execution results, issue logs, asset metadata, CDE classifications, and lineage. Bidirectional REST integration: GET requests ingest metadata/rules/lineage in; POST requests push AI-generated rules, risk annotations, and FIBO mappings out. REST connections established in the Azure sandbox. |
| **Others** | Additional enterprise systems as defined in functional requirements (e.g. core banking, GL, risk engine, regulatory submission platforms). |

---

### 2. Data Storage

Two storage categories feed the ingestion layer:

| Type | Examples | Ingestion Path |
|------|----------|----------------|
| **Structured data** | Rule execution tables, CDE metadata, risk scores, audit logs, regulatory mappings | Direct to Azure PostgreSQL via ingestion pipeline |
| **Unstructured data** | Regulatory instructions (FR Y-9C, FR Y-14Q manuals), BRDs, data dictionaries, policy documents | Document Intelligence / OCR → chunked → Azure AI Search vector index |

**Azure Blob Storage** serves as the landing zone for all raw unstructured documents before processing.

---

### 3. API Gateway & Security

| Component | Role |
|-----------|------|
| **Azure API Management** | Single entry point for all API traffic. Enforces rate limiting, request validation, API versioning, and logging. Routes requests to agent orchestration layer. |
| **Azure Active Directory** | Identity and access management. MSAL-based authentication for the React UI. Service principal auth for inter-service calls. Role-based access control (Data Steward, Risk Officer, Executive, Admin). |
| **Azure Key Vault** | Stores all secrets at runtime: Anthropic API key, Collibra credentials, PostgreSQL connection strings, Cosmos DB keys. Never hardcoded. |

---

### 4. Ingestion Layer

Two parallel ingestion pipelines process incoming data:

#### Pipeline A — Document Intelligence
Processes unstructured inputs (regulatory manuals, BRDs, data dictionaries):

```
Raw document (Blob Storage)
    → Azure Document Intelligence / Python OCR / LLM-based extraction
    → Semantic chunking (by section, clause, or field definition)
    → Embedding generation (Azure OpenAI text-embedding-3-large)
    → Azure AI Search vector index
```

**Chunking strategy:**
- Regulatory manuals → chunk by section/subsection heading
- BRDs → chunk by requirement statement
- Data dictionaries → chunk by field/attribute definition
- Chunk overlap: 10–15% to preserve cross-boundary context

#### Pipeline B — Microsoft GraphRAG
Processes structured metadata and lineage to build the domain knowledge graph:

```
Collibra metadata + lineage export
    → Microsoft GraphRAG pipeline
    → Entity extraction (DataAsset, DQRule, SourceSystem, Report, CDE)
    → Relationship inference (lineage, regulatory mapping, rule ownership)
    → Azure Cosmos DB Gremlin graph
```

---

### 5. Agent Orchestrator

The intelligence core of the platform. Built on **LangGraph** for multi-agent workflow orchestration (with Semantic Kernel / LangChain / CrewAI patterns as applicable) running on **Azure ML / Azure Databricks**. It runs two tiers of agents: **input-processing skill agents** that normalise raw inputs into structured signals, and a **multi-agent analysis workflow** of five specialized agents that evaluate risk, evaluate rules, and recommend rule changes.

#### LLM Garden
Routes LLM calls to the appropriate model based on task type, cost, and latency requirements. Primary model is **Claude (claude-sonnet-4-6)** via Anthropic API. Azure OpenAI serves as fallback/alternative where internal routing policy requires it.

#### Tier 1 — Input-Processing Skill Agents

These agents normalise heterogeneous inputs into structured signals consumed by the analysis workflow. Each is specialized for one input modality.

| Skill Agent | Input Modality | Output |
|-------------|----------------|--------|
| **UnstructuredDataAgent** | Regulatory instructions, BRDs, policy PDFs, data dictionaries | Semantic chunks + embeddings → Vector store; extracted regulatory requirements → Graph |
| **StructuredDataAgent** | Tabular data — rule execution results, profiling output, CDE metadata from Collibra | Normalised records → PostgreSQL; profiling statistics → analysis workflow |
| **SemiStructuredDataAgent** | JSON/XML rule exports, API payloads, lineage graphs | Parsed entities + relationships → Cosmos Gremlin graph |
| **FIBOMappingAgent** | Collibra CDE / column metadata | FIBO concept mappings (confidence-scored) → Graph + PostgreSQL; low-confidence → steward review queue |

#### Tier 2 — Analysis & Recommendation Workflow (5 Agents)

The workflow runs in two parallel streams that converge on the **Rule Advisor**. Stream colours match the source workflow diagram: **Risk Coverage** and **Optimization Play**.

```
                      ┌──────────────── Criticality Score ───────────────┐
                      │                                                   ▼
              ┌───────────────┐      ┌──────────────────┐      ┌──────────────────┐
  RISK   ───▶ │ Risk Analyzer │ ───▶ │  Effectiveness   │ ───▶ │ Health Calculator│
  COVERAGE    └───────────────┘      │    Analyzer      │      └──────────────────┘
                      │              └──────────────────┘
                      │ risk signals          │ rule effectiveness,
                      │ (context unit)        │ gaps, patterns tripped
                      ▼                        ▼
              ┌──────────────────────────────────────────┐
              │              Rule Advisor                 │
              └──────────────────────────────────────────┘
                      ▲                        ▲
                      │ rule groupings,        │ failure rate, SQL vs
                      │ redundancy patterns    │ profiling/type mismatch
  OPTIMIZATION ┌──────────────────┐   ┌──────────────────┐
  PLAY    ───▶ │   Redundancy     │   │   Efficiency     │
              │    Inspector     │   │    Inspector     │
              └──────────────────┘   └──────────────────┘
```

**Risk Coverage stream:**

| Agent | Objectives | Inputs | Outputs |
|-------|------------|--------|---------|
| **Risk Analyzer** | 1. Quantify criticality. 2. Assess applicable DQ dimensions. | Regulatory report applicability (report names, timing, reg report instructions, dimension applicability); operational impact (process description, timing, data requirements, dimension applicability) | **Criticality Score** (→ Health Calculator); risk signals / context unit (→ Rule Advisor) |
| **Effectiveness Analyzer** | 1. Identify fields/tables without DQRs. 2. Identify ineffective DQRs (probabilistic). 3. Quantify asset-level rule effectiveness. | Applicable DQ dimensions from Risk Analyzer; rule libraries; execution history | **Rule Effectiveness Score**; applicable DQ dimensions with gaps; patterns tripped; confidence score & explainability for gaps |
| **Health Calculator** | Calculate a health score (0–100). | Criticality Score (from Risk Analyzer); Rule Effectiveness Score (from Effectiveness Analyzer) | **Health Score** = 100 − [Criticality Score × (1 − Rule Effectiveness Score)] |

**Optimization Play stream:**

| Agent | Objectives | Inputs | Outputs |
|-------|------------|--------|---------|
| **Redundancy Inspector** | Identify redundancy (grouping DQRs by target `table.field`): exact duplicates; range superset/subset containment; superset predicate duplicates. | Rule groupings; redundant check pattern(s) tripped | Redundancy findings; confidence score & explainability (→ Rule Advisor) |
| **Efficiency Inspector** | Identify: high failure rate (>50%); SQL vs profiling mismatch; SQL vs declared data type mismatch. | Rule description, SQL; patterns tripped; profiling results for field/table | Efficiency findings; confidence score & explainability (→ Rule Advisor) |

**Convergence:**

| Agent | Objectives | Inputs | Outputs |
|-------|------------|--------|---------|
| **Rule Advisor** | Provide new/enhanced rule recommendation (natural language + SQL). 1. Provide rationale for recommendation. 2. Confidence score based on upstream signals / requesting agents. | Rule effectiveness score, DQ dimensions with gaps, patterns tripped, confidence & explainability for gaps, context unit (risk signals) from Risk Analyzer; redundancy & efficiency findings | Recommended/enhanced DQ rule (NL + SQL) with rationale and confidence score → steward review → Collibra |

#### Orchestration Pattern
```
Collibra GET sync OR UI action OR scheduled trigger
    → Azure API Management → Agent Orchestrator
    → Tier 1 skill agents normalise inputs (unstructured / structured / semi-structured)
    → Risk Coverage stream:  Risk Analyzer → Effectiveness Analyzer → Health Calculator
    → Optimization Play stream:  Redundancy Inspector + Efficiency Inspector (parallel)
    → Both streams feed Rule Advisor
    → Rule Advisor produces NL + SQL recommendation with confidence + rationale
    → Stored to PostgreSQL, surfaced in DQ Risk Insights UI
    → On steward Accept → Collibra POST pushes the approved rule
```

---

### 6. Memory Layer

Three complementary stores serve different retrieval patterns:

#### Azure AI Search / Databricks Vector Search
**Purpose:** Semantic retrieval over unstructured documents

- Stores embedded chunks of regulatory manuals, BRDs, data dictionaries, and FIBO concept definitions
- Queried at agent runtime for RAG context injection into Claude prompts
- Index updated incrementally as new documents are ingested

#### Azure Cosmos DB / Cosmos DB (Gremlin) — Graph Store
**Purpose:** Domain knowledge graph for Graph RAG and lineage traversal

**Domain Ontology (4 layers):**

*Layer 1 — DQ Operational:*
```
(:DataAsset) [:HAS_RULE] → (:DQRule)
(:DQRule) [:CONTROLS] → (:DataAsset)
(:Issue) [:CAUSED_BY] → (:DQRule)
(:StewardAction) [:RESOLVES] → (:Issue)
```

*Layer 2 — Regulatory Structure:*
```
(:RegulatoryReport) e.g. FR Y-9C, FR Y-14Q
(:ReportSchedule) [:PART_OF] → (:RegulatoryReport)
(:ReportLineItem) [:BELONGS_TO] → (:ReportSchedule)
(:DataAsset) [:POPULATES] → (:ReportLineItem)
(:DataAsset) [:SUBJECT_TO] → (:ReportingRequirement)
```

*Layer 3 — FIBO Financial Concepts (selective):*
```
(:FinancialConcept) e.g. NetInterestIncome, MortgageBackedSecurity
(:DataAsset) [:REPRESENTS] → (:FinancialConcept)
(:DataAsset) [:QUALIFIED_BY] → (:FinancialConcept)
(:FinancialConcept) [:CONTRIBUTES_TO] → (:ReportLineItem)
(:FinancialConcept) [fibo:sameAs] → FIBO URI
```

*Layer 4 — Lineage & Systems:*
```
(:SourceSystem) e.g. Collibra, GL, Core Banking, Risk Engine
(:DataAsset) [:SOURCED_FROM] → (:SourceSystem)
(:DataAsset) [:UPSTREAM_OF] → (:DataAsset)
(:DataAsset) [:TRANSFORMED_BY] → (:Pipeline)
```

**FIBO modules in scope (Phase 2):** FIBO-FND, FIBO-BE, FIBO-LOAN, FIBO-SEC

#### Azure PostgreSQL — Structured Operational Store
**Purpose:** All structured operational data — rule state, execution history, risk scores, steward actions, audit trail

**Core schema:**

```sql
DataAssets          (asset_id, name, type, domain, criticality, collibra_id)
DQRules             (rule_id, asset_id, name, logic_nl, logic_sql, status,
                     threshold, frequency, target_table, target_field,
                     collibra_rule_id)
RuleExecutions      (execution_id, rule_id, run_date, result, failure_count, pass_rate)
RiskScores          (score_id, asset_id, criticality_score, rule_effectiveness_score,
                     health_score, health_category, calculated_at)
AgentFindings       (finding_id, asset_id, rule_id, agent,  -- RiskAnalyzer / EffectivenessAnalyzer
                     finding_type, detail, patterns_tripped,  -- HealthCalculator / Redundancy / Efficiency
                     confidence, explainability, created_at)
RuleRecommendations (rec_id, asset_id, source_rule_id, recommended_nl, recommended_sql,
                     rationale, confidence, status, created_at)  -- from Rule Advisor
AIInsights          (insight_id, asset_id, rule_id, type, title, description,
                     confidence, reg_impact, status, created_at)
StewardActions      (action_id, rec_id, insight_id, action_type, modified_nl, modified_sql,
                     actioned_by, actioned_at)
ConceptMappings     (mapping_id, asset_id, fibo_uri, fibo_label, mapping_type,
                     confidence, approved_by, approved_at)
RegulatoryMappings  (mapping_id, asset_id, report_code, schedule_code, coverage_score)
PlatformSyncLog     (sync_id, platform, direction, entity_type, entity_id,
                     status, synced_at)  -- platform = 'Collibra'
```

---

### 7. Tool Layer

| Tool | Role |
|------|------|
| **Azure Functions** | Event-driven compute for async tasks: Collibra sync triggers, embedding pipeline jobs, scheduled rule execution ingestion, steward notification dispatch. |
| **Logic Apps** | Workflow orchestration for multi-step approval flows: steward review routing, rule push confirmation, alert escalation chains. |

---

### 8. User Interface

**React.js** (TypeScript, Vite, Tailwind CSS, Zustand, TanStack Query) deployed on **Azure Static Web Apps**.

UI modules:
- **Landing page** — module entry point with intelligence tile navigation
- **DQ Rule Intelligence** — 6-KPI strip, DQ Rule Health donut, Optimization Opportunities donut, Rule Coverage Heatmap (pending), Data Asset Risk Inventory, DQ Risk Insights panel with Accept/Modify/Reject workflow
- **Reporting Risk Intelligence** — cross-report discrepancy detection and risk scoring (in development)

See `index.html` for the full reference implementation of all UI components, data structures, filter logic, tooltip system, and insight rendering.

**Integration with Collibra** is surfaced directly in the UI: approved rules and risk annotations are pushed back to Collibra post steward review.

---

### 9. Data Layer

| Component | Role |
|-----------|------|
| **Azure Blob Storage** | **Raw landing zone** for everything extracted from Collibra (via Aspire Connectors) and for raw uploaded documents (regulatory PDFs, BRDs, data dictionaries). Decouples extraction from processing — agents do **not** read Blob directly; an ingestion pipeline processes Blob contents into the queryable stores (vector index, graph, PostgreSQL). Also stores model outputs and export artefacts. Provides immutable raw-capture for audit and pipeline replay. |

---

## End-to-End Data Flow (Collibra Integration)

This reflects the development team's data-flow architecture. There are two distinct paths: a **seed/write path** that populates Collibra with realistic content, and an **extract/read path** that feeds the agents.

```
SEED / WRITE PATH
─────────────────
Synthetic data we create
  (historical prod data,
   existing DQ rules,
   DQ results, lineage)
        │
        ▼
   PostgreSQL  ───────────────────▶  Collibra (Azure VM)
   (staging /                         seeded with realistic
   source of truth                    metadata, rules,
   for seeding)                       results, lineage


EXTRACT / READ PATH
───────────────────
Collibra (Azure VM)
        │  extracted via Aspire Connectors
        ▼
   Azure Blob Storage   ◀── raw landing zone (files: JSON / XML / extracted text)
        │
        ▼
┌────────────────────────────────────────────────────────────┐
│  INGESTION PIPELINE  (Tier 1 skill agents)                   │
│  reads raw objects from Blob and processes into:             │
│    UnstructuredDataAgent   → embeddings → VECTOR STORE       │
│    SemiStructuredDataAgent → entities/edges → GRAPH (Gremlin)│
│    StructuredDataAgent     → normalised rows → PostgreSQL    │
│    FIBOMappingAgent        → REPRESENTS bridge → GRAPH       │
└────────────────────────────────────────────────────────────┘
        │
        ▼
   QUERYABLE STORES (what agents actually query)
     • Vector store (Azure AI Search)  → semantic retrieval
     • Cosmos Gremlin graph            → impact traversal
     • PostgreSQL                      → operational state
        │
        ▼
   LangGraph multi-agent workflow (Tier 2 inference agents)
        │
        ▼
   Recommendations → steward review → Collibra POST (write-back)
```

### Why Collibra data lands in Blob before reaching agents

1. **Aspire Connectors emit files, not query responses.** Extracted Collibra content (asset descriptions, rule definitions, lineage exports, attached documents) needs a durable sink before processing. Blob is that sink.
2. **Decoupling.** Collibra extraction and agent processing run on different schedules. Blob buffers between them so neither has to be online simultaneously, and a failed agent run never forces a re-extract from Collibra.
3. **Unstructured/semi-structured payloads are bound for vectorization and graph-building.** This is exactly the raw material the UnstructuredDataAgent and SemiStructuredDataAgent consume. That pipeline reads from Blob.
4. **Replay and audit.** Raw extracts in Blob allow the embedding/graph pipeline to be re-run without re-extracting, and provide an immutable record of what Collibra returned at each sync — important in a regulated environment.

### How agents interact with each store

Agents do **not** treat all stores equally — access patterns differ:

| Store | Agent access pattern |
|-------|----------------------|
| **Azure Blob Storage** | **Not queried by agents.** Object storage retrieved by key. An ingestion pipeline reads whole objects from Blob and processes them into the queryable stores. Blob is strictly upstream of the agents. |
| **PostgreSQL** | **Queried directly and transactionally** via a SQL client/tool. Live operational reads/writes — rule state, execution history, agent findings, recommendations, scores. This is the agents' primary structured store at runtime. |
| **Vector store (Azure AI Search)** | **Queried at inference time** for semantic retrieval — returns relevant text chunks to ground Claude prompts. |
| **Cosmos Gremlin graph** | **Queried at inference time** for relationship traversal and impact inference (DQ ↔ FIBO ↔ regulatory layers). |

> **Note on PostgreSQL dual role:** In the seed path, PostgreSQL stages synthetic data pushed *up* to Collibra. At runtime it also serves as the agents' operational store. The build should keep the seed dataset and the runtime operational schema logically separated (separate schemas or databases) so the one-time seeding path does not entangle with live agent state.

---

## Vector Store & Knowledge Graph — Dual Ontology Model

The platform uses **one knowledge graph** that incorporates **two ontologies as distinct layers**, plus a vector store for semantic retrieval. The two stores answer fundamentally different questions:

- **Vector store** answers *"what text is semantically relevant?"* — retrieval of unstructured meaning. No concept of relationships.
- **Knowledge graph** answers *"how are these entities connected, and what is downstream?"* — traversal and impact inference. Holds structured relationships, not prose.

Impact inference needs both: the graph for the *structure* of impact, the vector store for the *language* to explain it.

### The two ontologies are layers of the same graph

**DQ Ontology — operational layer.** Models the platform's own world: data assets, DQ rules, issues, executions, stewards, lineage, source systems. Updated continuously as rules run and fail.

**FIBO Ontology — semantic grounding layer.** Models what data assets *mean* in industry-standard financial terms. Relatively static. Connects the operational world to regulatory meaning.

**The bridge** is the `REPRESENTS` relationship, built by the FIBOMappingAgent:

```
DQ layer:     (:DataAsset {NET_INT_INC_MBS}) -[:HAS_RULE]-> (:DQRule)
                       │
                       │ [:REPRESENTS]        ← bridge (FIBOMappingAgent)
                       ▼
FIBO layer:   (:FinancialConcept {NetInterestIncome}) -[:CONTRIBUTES_TO]-> (:ReportLineItem)
Regulatory:                                            -[:PART_OF]-> (:ReportSchedule) -> (:RegulatoryReport)
```

Without the FIBO bridge, a DQ failure is only *"rule X failed on asset Y."* With it, the graph traverses from that failure to *"this impacts FR Y-14Q Schedule G, governed by BCBS 239."* FIBO is what turns operational DQ events into regulatory impact statements.

### Write path vs read path

**Write path (ingestion).** Vectorization and graph relationship-modeling are both **ingestion-time** operations, split across specialized skill agents (vectorizing a PDF and inferring a lineage relationship are different tasks with different failure modes). The FIBO mapping is its own agent because it is the highest-stakes inference and must be confidence-scored and steward-reviewable in isolation. Built once, reused by all inference agents.

**Read path (inference).** The five-agent workflow queries the graph to traverse relationships and infer impact, while pulling supporting text from the vector store. The clearest example is the **Risk Analyzer**: its "quantify criticality" objective depends on traversing the FIBO bridge to count and weight downstream regulatory dependencies — an asset mapping to a FIBO concept that feeds three regulatory reports is inherently more critical than one feeding none, and only the graph can establish that.

### Design decision — materiality weighting location

Criticality/impact weighting (how much more critical an asset is given its downstream regulatory dependencies) should be **stored as edge properties on the graph** (e.g. `CONTRIBUTES_TO {materiality_weight: 0.18}`) rather than inferred by the LLM at read time. This keeps Health Score and Criticality Score **deterministic, reproducible, and auditable** — essential when a regulator asks how a risk score was derived. Claude is used for the *narrative explanation* of impact, not the *calculation* of it.

---

## Collibra Integration

The platform is designed as a non-invasive intelligence layer — it enhances Collibra, not replaces it. REST connections for both GET and POST are established in the Azure sandbox.

**Inbound — Collibra REST GET (Collibra → Platform):**
- Rule libraries, execution results, issue logs
- Asset metadata, CDE classifications, lineage graphs
- Scheduled sync (Azure Functions timer trigger) + on-demand pull
- Consumed by Tier 1 skill agents (structured + semi-structured) and written to PostgreSQL / Cosmos Gremlin

**Outbound — Collibra REST POST (Platform → Collibra):**
- AI-generated rule recommendations from Rule Advisor (after steward Accept via `StewardActions`)
- Risk score and Health Score annotations on assets/lineage
- Enriched remediation workflow tickets
- FIBO concept mapping attributes on CDE records

**Connection & Auth:**
- Collibra REST base URL and API credentials (OAuth2 / API token) stored in **Azure Key Vault**, retrieved at runtime
- A dedicated `collibraApi` service wraps GET/POST calls with retry, rate-limit handling, and response validation
- All sync operations logged to the `PlatformSyncLog` table for audit

**Sandbox setup checklist:**
1. Register the platform as an OAuth2 application in Collibra (or provision an API service account/token)
2. Store base URL + credentials in Azure Key Vault
3. Implement and test GET endpoints (assets, rules, lineage, execution results) against the Collibra sandbox instance
4. Implement and test POST endpoints (rule creation, attribute annotation) with a non-production community/domain
5. Wire scheduled sync via Azure Functions; verify `PlatformSyncLog` entries

---

## Build Phases

### Phase 1 — Foundation (Weeks 1–4)
- React app scaffolded from `index.html` POC, all components preserved
- FastAPI / Azure Functions backend skeleton with all route stubs
- Azure PostgreSQL schema deployed, synthetic data migrated
- **Collibra REST connections established in sandbox** — GET endpoints for assets, rules, lineage, execution results (read-only inbound sync)
- Tier 1 skill agents stubbed (unstructured / structured / semi-structured input processing)
- Claude API wired via LLM Garden for the Rule Advisor (replaces hardcoded recommendations)
- Azure AD auth end-to-end
- Deployed to Azure Static Web Apps (sandbox)

### Phase 2 — Intelligence (Weeks 5–8)
- Azure AI Search provisioned, first document corpus ingested (FR Y-9C and FR Y-14Q manuals) via UnstructuredDataAgent
- RAG pipeline operational for rule generation grounding
- Cosmos DB Gremlin provisioned, domain ontology deployed (Layers 1 & 2), lineage graph seeded from Collibra via SemiStructuredDataAgent + Microsoft GraphRAG
- **Full five-agent analysis workflow operational:** Risk Analyzer → Effectiveness Analyzer → Health Calculator (Risk Coverage stream); Redundancy Inspector + Efficiency Inspector (Optimization Play stream); both converging on Rule Advisor
- Health Score calculation live: `100 − [Criticality Score × (1 − Rule Effectiveness Score)]`
- FIBOMappingAgent built and run against Collibra CDE catalogue
- FIBO Layer 3 concepts added to graph (modules: FND, BE, LOAN, SEC)
- Reporting Risk Intelligence pillar built

### Phase 3 — Integration & Hardening (Weeks 9–12)
- **Collibra REST POST operational** — approved Rule Advisor recommendations pushed back to Collibra after steward Accept
- Logic Apps steward approval workflow operational
- Azure Functions async pipeline for Collibra sync and embedding jobs
- Azure Monitor + Application Insights dashboards live
- Performance testing: graph query optimisation, vector search latency, multi-agent workflow latency
- Security review: Key Vault policies, network isolation, API rate limiting
- Pilot deployment with real Collibra instance and real regulatory documents

---

## Key Regulatory Context

| Report | Schedule | Focus Area |
|--------|----------|------------|
| FR Y-9C | Schedule HI | Consolidated income statement — interest income, fee income |
| FR Y-14Q | Schedule G (PPNR) | Pre-provision net revenue — mortgages, retail lending, investment management |
| CECL (ASC 326) | — | Credit loss allowance methodology and model validation |
| Basel III/IV | HC-R | Risk-weighted assets, capital adequacy ratios |
| BCBS 239 | — | Risk data aggregation and reporting principles |

Cross-report reconciliation between FR Y-9C Schedule HI and FR Y-14Q Schedule G (PPNR) is the primary ReportingRiskAgent use case.

---

## Design System

All UI components reference the following CSS variables (defined in `index.html`):

```css
--purple: #7c3aed          /* Primary brand / action */
--purple-light: #ede9fe    /* Backgrounds, active states */
--purple-dark: #5b21b6     /* Hover states */
--green: #059669           /* Good Health / Effective / pass */
--amber: #d97706           /* Medium Health / Optimize / warning */
--red: #dc2626             /* Poor Health / Gaps / critical */
--blue: #2563eb            /* Regulatory tags */
--text: #111827            /* Primary text */
--text3: #6b7280           /* Secondary / labels */
--sans: 'Plus Jakarta Sans', sans-serif
--mono: 'JetBrains Mono', monospace
```

---

## Repository Structure (target)

```
dq-risk-intelligence/
├── index.html                  # POC reference UI (source of truth for components)
├── README.md                   # Project overview and feature summary
├── ARCHITECTURE.md             # This file — system architecture
├── REQUIREMENTS.md             # Functional requirements (to be provided)
├── public/
│   └── index.html              # Production entry point
├── src/
│   ├── components/
│   │   ├── Header/
│   │   ├── FilterBar/
│   │   ├── KPIRow/
│   │   ├── DQRuleHealthChart/
│   │   ├── OptimizationOpportunitiesChart/
│   │   ├── RuleCoverageHeatmap/
│   │   ├── DataAssetInventory/
│   │   ├── DQRiskInsights/
│   │   ├── ModifyRuleModal/
│   │   └── ConnectSourcesModal/
│   ├── data/
│   │   └── syntheticAssets.ts          # Typed synthetic dataset
│   ├── hooks/
│   │   └── useFilters.ts               # Filter state (health, optType)
│   ├── services/
│   │   ├── claudeApi.ts                # Anthropic Claude API client (via LLM Garden)
│   │   ├── collibraApi.ts              # Collibra REST client (GET + POST)
│   │   └── agentOrchestrator.ts        # Agent dispatch and response handling
│   ├── agents/
│   │   ├── skill/                      # Tier 1 — input-processing skill agents
│   │   │   ├── unstructuredDataAgent.ts
│   │   │   ├── structuredDataAgent.ts
│   │   │   ├── semiStructuredDataAgent.ts
│   │   │   └── fiboMappingAgent.ts
│   │   └── workflow/                   # Tier 2 — analysis & recommendation workflow
│   │       ├── riskAnalyzer.ts
│   │       ├── effectivenessAnalyzer.ts
│   │       ├── healthCalculator.ts
│   │       ├── redundancyInspector.ts
│   │       ├── efficiencyInspector.ts
│   │       └── ruleAdvisor.ts
│   ├── App.tsx
│   └── index.ts
├── api/                                # Azure Functions / FastAPI backend
│   ├── routes/
│   │   ├── control_intelligence.py
│   │   ├── reporting_risk.py
│   │   ├── collibra.py                 # Collibra GET/POST proxy + sync
│   │   └── rules.py
│   └── services/
│       ├── claude_client.py
│       ├── vector_store.py
│       ├── graph_store.py
│       └── sql_store.py
├── infra/                              # Azure infrastructure as code (Bicep / Terraform)
│   ├── main.bicep
│   ├── api-management.bicep
│   ├── cosmos-db.bicep
│   ├── ai-search.bicep
│   ├── postgresql.bicep
│   └── static-web-app.bicep
└── package.json
```
