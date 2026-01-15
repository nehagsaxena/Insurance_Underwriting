# Insurance Underwriting Agentic AI Solution
## Implementation Outline for Databricks Mosaic AI

---
<img width="838" height="789" alt="image" src="https://github.com/user-attachments/assets/1060f390-f796-4229-b220-18a1712f8734" />

## 1. Solution Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FRONTEND (Node.js + React)                          │
│         Upload Portal │ Submission Tracker │ Decision Dashboard             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATABRICKS MODEL SERVING ENDPOINT                        │
│                         (REST API Gateway)                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MOSAIC AI ORCHESTRATOR AGENT                           │
│                      (llama-3.3-70b-instruct)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         TOOL FUNCTIONS                              │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │   │
│  │  │  Document  │ │    Risk    │ │  Pricing   │ │ Compliance │       │   │
│  │  │ Processor  │ │  Assessor  │ │  Engine    │ │  Checker   │       │   │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │   │
│  │  ┌────────────┐ ┌────────────┐                                     │   │
│  │  │    RAG     │ │  Decision  │                                     │   │
│  │  │ Retriever  │ │   Maker    │                                     │   │
│  │  └────────────┘ └────────────┘                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
          │                                           │
          ▼                                           ▼
┌───────────────────────────┐          ┌──────────────────────────────────────┐
│   VECTOR SEARCH INDEX     │          │        UNITY CATALOG TABLES          │
│   (RAG - Guidelines)      │          │  Submissions │ Risk Assessments      │
│                           │          │  Decisions   │ Rate Tables           │
│  • Underwriting Guidelines│          │  Claims      │ Agent Logs            │
│  • Policy Documents       │          │  Guidelines  │ Location Risk         │
│  • Regulations            │          └──────────────────────────────────────┘
└───────────────────────────┘
```

---

## 2. Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Frontend | Node.js + Express + React | Upload UI, dashboards |
| API Layer | Databricks Model Serving | REST endpoint for agent |
| Agent Framework | Mosaic AI Agent Framework | Tool orchestration |
| LLM | llama-3.3-70b-instruct | Reasoning & decisions |
| RAG | Databricks Vector Search | Guidelines retrieval |
| Data Storage | Unity Catalog | Tables & volumes |
| Embeddings | databricks-gte-large-en | Document embeddings |

---

## 3. Project Structure

```
insurance-underwriting-agent/
│
├── databricks/
│   ├── setup/
│   │   ├── 01_create_catalog_schema.sql
│   │   ├── 02_create_tables.sql
│   │   ├── 03_create_vector_search.py
│   │   └── 04_deploy_model_serving.py
│   │
│   ├── data_generation/
│   │   ├── generate_synthetic_submissions.py
│   │   ├── generate_underwriting_guidelines.py
│   │   ├── generate_rate_tables.py
│   │   ├── generate_claims_history.py
│   │   └── generate_location_risk_factors.py
│   │
│   ├── agents/
│   │   ├── orchestrator_agent.py
│   │   ├── tools/
│   │   │   ├── document_processor.py
│   │   │   ├── risk_assessor.py
│   │   │   ├── pricing_engine.py
│   │   │   ├── compliance_checker.py
│   │   │   └── rag_retriever.py
│   │   └── prompts/
│   │       └── system_prompt.txt
│   │
│   └── serving/
│       └── endpoint_config.py
│
├── frontend/
│   ├── server.js
│   ├── package.json
│   └── src/
│       └── components/
│
└── README.md
```

---

## 4. Unity Catalog Schema

### 4.1 Catalog & Schema
```
Catalog: insurance_underwriting
Schema: commercial_property
```

### 4.2 Tables Required

| Table Name | Purpose | Key Columns |
|------------|---------|-------------|
| `submissions` | Store incoming submissions | submission_id, insured_name, property_address, property_type, tiv, status |
| `risk_assessments` | Risk scoring results | assessment_id, submission_id, location_score, property_score, overall_score, risk_tier |
| `underwriting_decisions` | Final decisions + explainability | decision_id, submission_id, decision_type, rationale, decision_factors (JSON), premium |
| `rate_tables` | Premium rating factors | state, property_type, construction_type, base_rate, modifiers |
| `claims_history` | Historical claims (synthetic) | claim_id, insured_ein, loss_date, claim_type, incurred_amount |
| `underwriting_guidelines` | RAG source documents | doc_id, title, category, content, property_types, states |
| `location_risk_factors` | Geo risk data | state, zip_prefix, flood_zone, hurricane_zone, iso_ppc_rating |
| `agent_execution_logs` | Audit trail | trace_id, submission_id, tools_invoked, llm_tokens, processing_time |

---

## 5. Agent Tools Specification

### 5.1 Document Processor Tool
**Purpose**: Extract structured data from uploaded PDF/images

**Input**: 
- document_base64 (string)
- document_type (string)

**Output**:
```json
{
  "insured_info": { "name", "ein", "contact" },
  "property_info": { "address", "type", "construction", "year_built", "sqft" },
  "coverage_requested": { "tiv", "deductible", "coverage_types" },
  "risk_details": { "sprinklers", "alarm", "prior_losses" },
  "extraction_confidence": { "overall", "missing_fields" }
}
```

### 5.2 Risk Assessor Tool
**Purpose**: Calculate risk scores from submission data

**Input**: submission_data (object)

**Output**:
```json
{
  "location_risk": { "score": 0-100, "factors": [...] },
  "property_risk": { "score": 0-100, "factors": [...] },
  "protection_risk": { "score": 0-100, "factors": [...] },
  "loss_history_risk": { "score": 0-100, "factors": [...] },
  "overall_risk": { "score": 0-100, "tier": "Preferred|Standard|Substandard|Decline" }
}
```

**Risk Tier Logic**:
- Preferred: score ≤ 35
- Standard: score 36-55
- Substandard: score 56-75
- Decline: score > 75

### 5.3 Pricing Engine Tool
**Purpose**: Calculate premium based on risk

**Input**: submission_data, risk_assessment

**Output**:
```json
{
  "total_insured_value": number,
  "base_rate_per_100": number,
  "base_premium": number,
  "tier_modifier": number,
  "credits_applied": [{ "name", "factor", "explanation" }],
  "surcharges_applied": [{ "name", "factor", "explanation" }],
  "final_premium": number
}
```

### 5.4 Compliance Checker Tool
**Purpose**: Validate regulatory requirements

**Input**: submission_data, risk_assessment, decision

**Output**:
```json
{
  "passed": boolean,
  "flags": [{ "flag", "severity", "description" }],
  "requirements": [{ "requirement", "status" }],
  "adverse_action_required": boolean
}
```

### 5.5 RAG Retriever Tool
**Purpose**: Retrieve relevant underwriting guidelines

**Input**: query, property_type?, state?, category?

**Output**: Array of relevant guideline documents with relevance scores

### 5.6 Decision Maker Tool
**Purpose**: Make final underwriting decision

**Decision Logic**:
- **Auto-Approve**: risk_score ≤ 55 AND compliance.passed AND confidence ≥ 0.8
- **Auto-Reject**: risk_score > 75 OR critical_compliance_flags
- **Refer-to-Underwriter**: All other cases

**Output**:
```json
{
  "decision_type": "Auto-Approve|Auto-Reject|Refer-to-Underwriter",
  "decision_rationale": "Natural language explanation",
  "confidence_level": number,
  "decision_factors": [
    { "factor_name", "factor_value", "impact", "weight", "explanation" }
  ]
}
```

---

## 6. Vector Search Configuration

### 6.1 Endpoint
- **Name**: `underwriting_vs_endpoint`
- **Index**: `insurance_underwriting.commercial_property.guidelines_index`

### 6.2 Source Table
- **Table**: `underwriting_guidelines`
- **Embedding Column**: `content`
- **Embedding Model**: `databricks-gte-large-en`
- **Primary Key**: `doc_id`

### 6.3 Sync Type
- Delta Sync with TRIGGERED pipeline

---

## 7. Orchestrator Agent Design

### 7.1 System Prompt Summary
The agent should:
1. Process uploaded documents first
2. Retrieve relevant guidelines via RAG
3. Assess risk across 4 dimensions
4. Calculate premium if risk acceptable
5. Check compliance requirements
6. Make decision with full explainability

### 7.2 Tool Execution Flow
```
User Upload → process_document → retrieve_guidelines → assess_risk 
           → calculate_premium → check_compliance → make_decision
```

### 7.3 Decision Explanation Requirements
Every decision must include:
- Decision type (Approve/Reject/Refer)
- Natural language rationale
- Structured decision factors with:
  - Factor name & value
  - Impact (Positive/Negative/Neutral)
  - Weight in decision
  - Explanation

---

## 8. Model Serving Endpoint

### 8.1 Configuration
- **Endpoint Name**: `underwriting-agent-endpoint`
- **Model**: `insurance_underwriting.commercial_property.underwriting_agent`
- **Workload Size**: Small (scalable)
- **Scale to Zero**: Enabled
- **Auto Capture**: Enabled (for logging)

### 8.2 API Contract

**Request**:
```json
{
  "inputs": {
    "submission_id": "string",
    "documents": [{ "filename", "mime_type", "content_base64" }],
    "broker_info": { "name", "id" },
    "additional_context": "string"
  }
}
```

**Response**:
```json
{
  "trace_id": "string",
  "submission_id": "string",
  "decision": {
    "decision_type": "string",
    "decision_rationale": "string",
    "confidence_level": number,
    "decision_factors": [...]
  },
  "risk_assessment": {...},
  "premium_calculation": {...},
  "compliance_results": {...},
  "processing_time_ms": number
}
```

---

## 9. Frontend API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/submissions` | Upload & process new submission |
| GET | `/api/submissions/:id` | Get submission status |
| GET | `/api/decisions/:submission_id` | Get decision with explainability |
| GET | `/api/traces/:trace_id` | Get agent execution trace |

---

## 10. Synthetic Data Requirements

### 10.1 Submissions (50-100 records)
- Mix of property types: Office, Retail, Industrial, Warehouse
- Various states: TX, CA, FL, NY, IL
- TIV range: $1M - $50M
- Mix of construction types
- Various protection features

### 10.2 Underwriting Guidelines (30-50 documents)
Categories:
- Eligibility criteria by property type
- Pricing guidelines
- Coverage requirements
- Exclusions and limitations
- State-specific requirements
- Compliance requirements

### 10.3 Rate Tables
- Base rates by state, property type, construction
- Age factors, occupancy factors
- Deductible factors
- Minimum premiums

### 10.4 Claims History
- 200-500 synthetic claims
- Various claim types: Fire, Water, Theft, Weather
- Range of severities

### 10.5 Location Risk Factors
- By state and zip prefix
- Flood zones, hurricane zones
- ISO PPC ratings
- Crime indices

---

## 11. Implementation Sequence

### Phase 1: Data Foundation
1. Create Unity Catalog and schema
2. Create all required tables
3. Generate synthetic data for all tables
4. Validate data relationships

### Phase 2: Vector Search Setup
1. Populate underwriting_guidelines table
2. Create vector search endpoint
3. Create delta sync index
4. Test retrieval queries

### Phase 3: Agent Tools Development
1. Implement document_processor tool
2. Implement risk_assessor tool
3. Implement pricing_engine tool
4. Implement compliance_checker tool
5. Implement rag_retriever tool
6. Unit test each tool

### Phase 4: Orchestrator Agent
1. Define tool specifications
2. Create system prompt
3. Implement orchestrator agent class
4. Test agent workflow end-to-end

### Phase 5: Model Serving
1. Register agent model with MLflow
2. Deploy model serving endpoint
3. Test endpoint invocation
4. Configure auto-capture logging

### Phase 6: Frontend
1. Setup Node.js server
2. Implement API routes
3. Build React UI components
4. Integrate with Databricks endpoint

### Phase 7: Testing & Demo
1. End-to-end testing
2. Explainability validation
3. Demo preparation

---

## 12. Key Configuration Values

```python
# Databricks Configuration
CATALOG = "insurance_underwriting"
SCHEMA = "commercial_property"
LLM_ENDPOINT = "databricks-llama-3-3-70b-instruct"
EMBEDDING_MODEL = "databricks-gte-large-en"
VECTOR_SEARCH_ENDPOINT = "underwriting_vs_endpoint"
AGENT_SERVING_ENDPOINT = "underwriting-agent-endpoint"

# Business Rules
RISK_TIER_THRESHOLDS = {
    "Preferred": (0, 35),
    "Standard": (36, 55),
    "Substandard": (56, 75),
    "Decline": (76, 100)
}

AUTO_APPROVE_MAX_SCORE = 55
AUTO_REJECT_MIN_SCORE = 76
MIN_CONFIDENCE_FOR_AUTO = 0.80
```

---

## 13. Files to Create (In Order)

1. `databricks/setup/01_create_catalog_schema.sql`
2. `databricks/setup/02_create_tables.sql`
3. `databricks/data_generation/generate_underwriting_guidelines.py`
4. `databricks/data_generation/generate_rate_tables.py`
5. `databricks/data_generation/generate_location_risk_factors.py`
6. `databricks/data_generation/generate_claims_history.py`
7. `databricks/data_generation/generate_synthetic_submissions.py`
8. `databricks/setup/03_create_vector_search.py`
9. `databricks/agents/tools/rag_retriever.py`
10. `databricks/agents/tools/document_processor.py`
11. `databricks/agents/tools/risk_assessor.py`
12. `databricks/agents/tools/pricing_engine.py`
13. `databricks/agents/tools/compliance_checker.py`
14. `databricks/agents/prompts/system_prompt.txt`
15. `databricks/agents/orchestrator_agent.py`
16. `databricks/setup/04_deploy_model_serving.py`
17. `frontend/package.json`
18. `frontend/server.js`
19. `frontend/src/App.jsx`
20. `frontend/src/components/UploadPortal.jsx`
21. `frontend/src/components/DecisionView.jsx`
22. `frontend/src/components/ExplainabilityPanel.jsx`
