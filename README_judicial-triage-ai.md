# ⚖️ Judicial Triage Automation with AI

> Automated pipeline for classifying and routing health-insurance-related legal processes using multi-layer AI analysis, external APIs, and CRM integration.

---

## 🧩 Problem

Health insurance operations receive a high volume of legal processes daily. Manually identifying which cases require internal investigation — and creating the appropriate support tickets — was time-consuming, error-prone, and generated noise from irrelevant or misclassified processes.

**Key challenges:**
- Distinguishing genuine health-related judicial claims from unrelated legal processes
- Identifying confirmed clinical conditions vs. ambiguous or inconclusive cases
- Routing relevant cases to the correct CRM pipeline without human intervention
- Maintaining an auditable record of AI decisions for quality control

---

## 🏗️ Architecture

```
Webhook Trigger (CRM)
        │
        ▼
┌───────────────────┐
│  Data Extraction  │  ← CNJ number extraction (masked + unmasked regex)
│  & Enrichment     │  ← External judicial data API (REST)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  AI Layer 1       │  ← Relevance classification
│  (GPT-4)          │    E_SAUDE / NAO_E_SAUDE / NAO_PERTINENTE
└────────┬──────────┘
         │
    [E_SAUDE only]
         │
         ▼
┌───────────────────┐
│  AI Layer 2       │  ← Depth analysis
│  (GPT-4)          │    Confirmed condition / Ambiguous / Not applicable
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  CRM Ticket       │  ← Automated ticket creation
│  Creation         │    Pipeline + Stage routing
│                   │    Custom object association (Member record)
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Audit Log        │  ← AI inputs/outputs logged to spreadsheet
│                   │    via workflow API for prompt optimization
└───────────────────┘
```

---

## 🤖 AI Logic

### Layer 1 — Relevance Classification

The first AI layer determines whether a legal process is health-insurance-related, using a structured prompt that evaluates:

- Nature of the legal claim (type of benefit demanded)
- Parties involved (plaintiff vs. legal representative)
- Edge cases: minors represented by adults, third-party claims, administrative vs. judicial scope

**Output schema:**
```json
{
  "classificacao": "E_SAUDE | NAO_E_SAUDE | NAO_PERTINENTE",
  "justificativa": "string"
}
```

### Layer 2 — Clinical Depth Analysis

For cases classified as `E_SAUDE`, a second AI layer evaluates clinical specificity:

- Confirmed diagnosis with CID code or explicit medical report reference
- Ambiguous language (symptoms described without confirmed condition)
- Pre-existing condition indicators

This prevents over-triggering of investigation workflows on legally ambiguous cases.

---

## 🔌 Integrations

| Service | Role |
|---|---|
| **CRM Webhook** | Trigger — fires on new legal process record |
| **Judicial Data API** | Enrichment — retrieves full process metadata by CNJ number |
| **GPT-4 (OpenAI)** | AI classification — dual-layer triage |
| **CRM API** | Output — ticket creation, pipeline routing, object association |
| **Cloud Storage** | Document storage — process attachments linked to ticket |
| **Spreadsheet API** | Audit log — AI decision tracing for prompt optimization |

---

## 📊 Results & Impact

| Metric | Before | After |
|---|---|---|
| Manual triage time per process | ~8 min | ~0 min |
| Classification accuracy (audit of 18 cases) | ~55% | 100% |
| Weekly processes handled automatically | 0 | hundreds |
| Critical misclassification errors | recurring | eliminated |

> Classification accuracy improvement measured through a structured prompt audit comparing AI outputs against expert human review across 18 real anonymized cases.

---

## 🛠️ Tech Stack

- **Workflow automation:** n8n (cloud)
- **AI:** OpenAI GPT-4.1
- **Language:** JavaScript (n8n Code nodes)
- **Integrations:** REST APIs, Webhooks, OAuth2
- **CRM:** HubSpot (tickets, custom objects, associations)
- **Storage:** Google Drive (Shared Drive)
- **Audit:** Google Sheets via API

---

## 🔐 Privacy & Compliance Notes

- No personal health data (PHI) is stored in the workflow layer
- AI inputs are anonymized before logging
- All API credentials managed via environment variables and credential vaults
- Workflow designed in compliance with LGPD (Brazilian data protection law)

---

## 💡 Key Engineering Decisions

- **Dual-layer AI** instead of single classification: reduces false positives and improves routing precision
- **Regex for CNJ extraction** handles both masked (`0000000-00.0000.0.00.0000`) and unmasked formats
- **JSON output enforcement** in prompts prevents markdown-wrapped responses that break downstream parsing
- **Audit pipeline** runs in parallel to production workflow — captures AI inputs/outputs without impacting latency
- **pairedItem chain management** in n8n: isolated data retrieval nodes to prevent context loss in multi-step flows

---

## 📁 Repository Structure

```
/
├── README.md               ← This file
├── docs/
│   ├── architecture.md     ← Detailed flow diagrams
│   └── prompt-design.md    ← Prompt engineering decisions
├── prompts/
│   ├── layer1_relevance.md ← Layer 1 prompt template
│   └── layer2_clinical.md  ← Layer 2 prompt template
└── audit/
    └── prompt_audit_methodology.md ← Audit framework (18-case study)
```

---

## 👤 Author

**Samuel** — AI Automation Engineer  
Background in Portuguese Literature & Law, specializing in AI-powered operational workflows at the intersection of legal domain knowledge and automation engineering.

---

*This project is documented for portfolio purposes. All sensitive data, credentials, and company-specific identifiers have been removed or abstracted.*
