# Document Upload & Processing Flow

## Overview

This document describes the complete flow from document upload to underwriting decision (Option B architecture where the agent handles all processing).

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER INTERFACE                                  │
│                         (Upload Submission PDF)                              │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         UPLOAD API LAYER                                     │
│                   (submission_intake.py)                                     │
│                                                                              │
│  Functions:                                                                  │
│  • upload_submission() - Save PDF to Volume                                  │
│  • generate_submission_id() - Create unique ID                              │
│  • get_submission_file() - Retrieve file path                               │
│  • update_submission_status() - Update after processing                     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      UNITY CATALOG VOLUME                                    │
│                                                                              │
│  /Volumes/insurance_underwriting/commercial_property/                        │
│  ├── submission_documents/                                                   │
│  │   └── SUB-20250115-A1B2/                                                 │
│  │       ├── submission.pdf          ← Uploaded document                    │
│  │       └── metadata.json           ← Upload metadata                      │
│  │                                                                           │
│  └── processed_extractions/                                                  │
│      └── SUB-20250115-A1B2.json      ← Extracted data (saved by agent)     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AGENT SERVING ENDPOINT                                 │
│                                                                              │
│  Receives:                                                                   │
│  {                                                                           │
│      "submission_id": "SUB-20250115-A1B2",                                  │
│      "file_path": "/Volumes/.../submission.pdf",                            │
│      "broker_id": "BRK-001",                                                │
│      "broker_name": "Premier Insurance Agency"                              │
│  }                                                                           │
│                                                                              │
│  Agent executes 5 tools in sequence (see below)                             │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           RESPONSE                                           │
│                                                                              │
│  {                                                                           │
│      "submission_id": "SUB-20250115-A1B2",                                  │
│      "decision": "APPROVE",                                                  │
│      "confidence": 0.92,                                                     │
│      "risk_score": 38,                                                       │
│      "risk_tier": "Standard",                                                │
│      "premium": 42500.00,                                                    │
│      "rationale": "...",                                                     │
│      "decision_factors": [...],                                              │
│      "compliance": {...},                                                    │
│      "guidelines_used": [...]                                                │
│  }                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Agent Tool Execution Flow

The agent executes 5 tools in this exact sequence:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: process_document(submission_id, file_path)                          │
│                                                                              │
│ Purpose: Extract structured data from uploaded PDF                          │
│                                                                              │
│ Actions:                                                                     │
│ • Read PDF from Unity Catalog Volume                                         │
│ • Extract text using PyPDF2                                                  │
│ • Call LLM (llama-3.3-70b) for structured extraction                        │
│ • Save extracted JSON to processed_extractions volume                        │
│                                                                              │
│ Output:                                                                      │
│ • insured_info: {name, ein, contact, address}                               │
│ • property_info: {address, city, state, zip, type, construction, year_built}│
│ • coverage_requested: {tiv, deductible, coverage_types}                     │
│ • risk_details: {has_sprinklers, has_alarm, has_security, prior_losses}    │
│ • extraction_confidence: 0.0 - 1.0                                          │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: retrieve_guidelines(query, property_type, state)                    │
│                                                                              │
│ Purpose: Get relevant underwriting rules via RAG                            │
│                                                                              │
│ Actions:                                                                     │
│ • Query vector search index (guidelines_index)                              │
│ • Filter by property_type and state from Step 1                             │
│ • Return top matching guidelines                                             │
│                                                                              │
│ Example Queries:                                                             │
│ • "eligibility requirements for office buildings in Texas"                  │
│ • "sprinkler credit requirements for industrial properties"                 │
│ • "construction type guidelines for warehouses"                             │
│                                                                              │
│ Output:                                                                      │
│ • List of relevant guideline documents with content                         │
│ • Relevance scores                                                           │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: assess_risk(submission_data)                                        │
│                                                                              │
│ Purpose: Calculate risk scores across 4 dimensions                          │
│                                                                              │
│ Dimensions & Weights:                                                        │
│ ┌─────────────────┬────────┬────────────────────────────────────────────┐  │
│ │ Dimension       │ Weight │ Factors                                    │  │
│ ├─────────────────┼────────┼────────────────────────────────────────────┤  │
│ │ Location Risk   │ 30%    │ Flood, hurricane, earthquake, wildfire,   │  │
│ │                 │        │ tornado, crime index, ISO PPC rating      │  │
│ ├─────────────────┼────────┼────────────────────────────────────────────┤  │
│ │ Property Risk   │ 25%    │ Age, construction type, property type,    │  │
│ │                 │        │ TIV concentration                          │  │
│ ├─────────────────┼────────┼────────────────────────────────────────────┤  │
│ │ Protection Risk │ 20%    │ Sprinklers, alarm system, security        │  │
│ ├─────────────────┼────────┼────────────────────────────────────────────┤  │
│ │ Loss History    │ 25%    │ Prior claims count, claim severity,       │  │
│ │                 │        │ loss ratio                                 │  │
│ └─────────────────┴────────┴────────────────────────────────────────────┘  │
│                                                                              │
│ Output:                                                                      │
│ • Individual dimension scores (0-100)                                        │
│ • Overall weighted score (0-100)                                             │
│ • Risk tier: Preferred / Standard / Substandard / Decline                   │
│ • Detailed factors for each dimension                                        │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: calculate_premium(submission, risk_assessment)                      │
│                                                                              │
│ Purpose: Calculate premium (ONLY if risk tier ≠ "Decline")                  │
│                                                                              │
│ Calculation Flow:                                                            │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ Base Premium = (TIV / 100) × Base Rate                                  │ │
│ │                                                                          │ │
│ │ Adjusted Premium = Base Premium                                          │ │
│ │                    × Age Factor                                          │ │
│ │                    × Deductible Factor                                   │ │
│ │                    × Credit Factor (sprinklers, alarms, security)       │ │
│ │                    × Surcharge Factor (high TIV, loss history)          │ │
│ │                    × Tier Modifier                                       │ │
│ │                                                                          │ │
│ │ Final Premium = MAX(Adjusted Premium, Minimum Premium)                  │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│ Tier Modifiers:                                                              │
│ • Preferred: 0.85 (15% discount)                                            │
│ • Standard: 1.00 (base rate)                                                │
│ • Substandard: 1.25 (25% surcharge)                                         │
│ • Decline: Not quotable                                                      │
│                                                                              │
│ Output:                                                                      │
│ • Premium breakdown with all factors                                         │
│ • Credits and surcharges applied                                             │
│ • Final premium amount                                                       │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: check_compliance(submission, risk, pricing, decision_type)          │
│                                                                              │
│ Purpose: Validate regulatory requirements before final decision             │
│                                                                              │
│ Checks Performed:                                                            │
│ • TIV within limits ($100K - $100M)                                         │
│ • State-specific disclosure requirements                                     │
│ • Regulatory compliance                                                      │
│ • Adverse action requirements (for rejections)                              │
│ • OFAC sanctions screening                                                   │
│                                                                              │
│ State Disclosures:                                                           │
│ ┌─────────┬─────────────────────────────────────────────────────────────┐  │
│ │ State   │ Required Disclosures                                        │  │
│ ├─────────┼─────────────────────────────────────────────────────────────┤  │
│ │ CA      │ Earthquake Disclosure, Wildfire Risk Disclosure             │  │
│ │ FL      │ Hurricane Disclosure, Sinkhole Coverage Disclosure          │  │
│ │ TX      │ Hail Damage Disclosure                                      │  │
│ │ NY      │ TRIA Terrorism Disclosure                                   │  │
│ └─────────┴─────────────────────────────────────────────────────────────┘  │
│                                                                              │
│ Output:                                                                      │
│ • Pass/Fail status                                                           │
│ • List of flags (Critical, Warning, Info)                                   │
│ • Required disclosures                                                       │
│ • Adverse action requirements                                                │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 6: MAKE DECISION                                                        │
│                                                                              │
│ Decision Logic:                                                              │
│ ┌─────────────────────────────────────────────────────────────────────────┐ │
│ │ AUTO-APPROVE if:                                                        │ │
│ │ • Risk score ≤ 55                                                       │ │
│ │ • Compliance passed                                                      │ │
│ │ • No critical flags                                                      │ │
│ │ • Extraction confidence ≥ 0.80                                          │ │
│ ├─────────────────────────────────────────────────────────────────────────┤ │
│ │ AUTO-REJECT if:                                                         │ │
│ │ • Risk score > 75                                                       │ │
│ │ • Critical compliance failures                                          │ │
│ │ • Ineligible property type                                              │ │
│ ├─────────────────────────────────────────────────────────────────────────┤ │
│ │ REFER-TO-UNDERWRITER if:                                                │ │
│ │ • Risk score 56-75                                                      │ │
│ │ • Compliance warnings                                                    │ │
│ │ • Missing critical information                                          │ │
│ │ • Extraction confidence < 0.80                                          │ │
│ └─────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│ Output:                                                                      │
│ • Decision: APPROVE / REJECT / REFER-TO-UNDERWRITER                         │
│ • Confidence score                                                           │
│ • Detailed rationale                                                         │
│ • Decision factors with explanations                                         │
│ • Guidelines referenced                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Risk Tier Thresholds

| Tier | Score Range | Auto Decision | Premium Modifier |
|------|-------------|---------------|------------------|
| **Preferred** | 0 - 35 | Auto-Approve eligible | 0.85 (15% discount) |
| **Standard** | 36 - 55 | Auto-Approve eligible | 1.00 (base rate) |
| **Substandard** | 56 - 75 | Refer to Underwriter | 1.25 (25% surcharge) |
| **Decline** | 76 - 100 | Auto-Reject | Not quotable |

---

## File Structure

```
/Workspace/Shared/underwriting_demo/
├── tools/
│   ├── rag_retriever.py        # Vector search for guidelines
│   ├── risk_assessor.py        # Risk scoring (4 dimensions)
│   ├── pricing_engine.py       # Premium calculation
│   ├── compliance_checker.py   # Regulatory validation
│   └── document_processor.py   # PDF extraction via LLM
├── submission_intake.py        # Upload handling (pre-agent)
├── orchestrator_agent.py       # Main agent (to be created)
└── prompts/
    └── system_prompt.txt       # Agent instructions

/Volumes/insurance_underwriting/commercial_property/
├── submission_documents/       # Uploaded PDFs
│   └── {submission_id}/
│       ├── submission.pdf
│       └── metadata.json
└── processed_extractions/      # Extracted data (JSON)
    └── {submission_id}.json
```

---

## API Endpoints

### 1. Upload Submission

```python
# POST /api/submissions/upload
from submission_intake import upload_submission

result = upload_submission(
    file_content=pdf_bytes,
    filename="submission.pdf",
    broker_id="BRK-001",
    broker_name="Premier Insurance Agency"
)

# Returns:
{
    "success": True,
    "submission_id": "SUB-20250115-A1B2",
    "file_path": "/Volumes/.../submission.pdf",
    "agent_input": {
        "submission_id": "SUB-20250115-A1B2",
        "file_path": "/Volumes/.../submission.pdf",
        "broker_id": "BRK-001",
        "broker_name": "Premier Insurance Agency"
    }
}
```

### 2. Process Submission (Agent Endpoint)

```python
# POST /api/submissions/process
# Calls the agent serving endpoint

agent_input = {
    "submission_id": "SUB-20250115-A1B2",
    "file_path": "/Volumes/.../submission.pdf",
    "broker_id": "BRK-001",
    "broker_name": "Premier Insurance Agency"
}

# Agent returns full decision with explainability
```

### 3. Get Submission Status

```python
# GET /api/submissions/{submission_id}
from submission_intake import get_submission_file

result = get_submission_file("SUB-20250115-A1B2")
```

---

## Sample Response Format

```markdown
## UNDERWRITING DECISION

**Submission ID:** SUB-20250115-A1B2
**Insured:** ABC Manufacturing Corp
**Property:** 1500 Industrial Blvd, Houston, TX 77001
**Property Type:** Industrial
**TIV:** $18,500,000

---

### DOCUMENT EXTRACTION

- **Status:** Success
- **Confidence:** 0.95
- **Missing Fields:** None

---

### DECISION: APPROVE

**Confidence:** 0.92

**Rationale:** This industrial property in Houston, TX meets all eligibility criteria with a Standard risk tier (score: 42). The property benefits from fire resistive construction, full sprinkler system, and central station alarm. The single prior claim does not significantly impact the risk profile. Premium of $48,125 is competitive for this risk class.

---

### RISK ASSESSMENT SUMMARY

| Dimension | Score | Key Factors |
|-----------|-------|-------------|
| Location | 38 | Houston area, moderate flood risk, good fire protection (PPC 3) |
| Property | 35 | 2005 construction, non-combustible, industrial use |
| Protection | 25 | Full sprinklers, central alarm, no 24/7 security |
| Loss History | 30 | 1 prior claim ($25K water damage) |
| **Overall** | **42** | **Standard** |

---

### PREMIUM

| Component | Value |
|-----------|-------|
| TIV | $18,500,000 |
| Base Rate | $0.32 per $100 |
| Base Premium | $59,200 |
| Age Factor | 1.00 (2005 building) |
| Tier Modifier | 1.00 (Standard) |
| Credits | -$11,840 (Sprinkler -15%, Alarm -5%) |
| Surcharges | $0 |
| **Final Premium** | **$48,125** |

---

### COMPLIANCE STATUS

- **Overall:** PASS
- **Disclosures Required:** Texas Hail Damage Disclosure
- **Flags:** None
- **Adverse Action Required:** No
```

---

## Next Steps

1. **Create Orchestrator Agent** - Tie all tools together with Mosaic AI Agent Framework
2. **Deploy Model Serving Endpoint** - Register and deploy the agent
3. **Build Frontend UI** - Upload portal and decision dashboard
