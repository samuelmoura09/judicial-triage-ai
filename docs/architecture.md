# 🏗️ Architecture — Judicial Triage Automation

> Detailed flow documentation for the AI-powered legal process classification and routing pipeline.

---

## Overview

The pipeline is triggered by a CRM webhook and runs fully autonomously — from data extraction through AI classification to CRM ticket creation and audit logging. No human intervention is required for standard cases.

Total average execution time: **~8–12 seconds per process**

---

## Full Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        TRIGGER LAYER                        │
│                                                             │
│   CRM Webhook ──► Header Auth Validation ──► Data Parse    │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      EXTRACTION LAYER                       │
│                                                             │
│   CNJ Number Extraction (Regex)                             │
│     ├── Masked format:   0000000-00.0000.0.00.0000          │
│     └── Unmasked format: 00000000000000000000               │
│                              │                              │
│                              ▼                              │
│   Judicial API Call (REST)                                  │
│     └── Returns: parties, subject, court, dates, status     │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      AI LAYER 1 — RELEVANCE                 │
│                                                             │
│   Input:  Process subject + parties + claim description     │
│   Model:  GPT-4.1                                           │
│   Output: E_SAUDE | NAO_E_SAUDE | NAO_PERTINENTE            │
│                                                             │
│   ├── NAO_E_SAUDE ──► Stop. No ticket created.             │
│   ├── NAO_PERTINENTE ──► Stop. No ticket created.          │
│   └── E_SAUDE ──► Continue to Layer 2                      │
└─────────────────────────────┬───────────────────────────────┘
                              │ (E_SAUDE only)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      AI LAYER 2 — CLINICAL DEPTH            │
│                                                             │
│   Input:  Full process text + Layer 1 classification        │
│   Model:  GPT-4.1                                           │
│   Output: Confirmed | Ambiguous | Not Applicable            │
│                                                             │
│   ├── Not Applicable ──► Stop. No ticket created.          │
│   ├── Ambiguous ──► Ticket with "review" flag              │
│   └── Confirmed ──► Ticket with full routing               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      OUTPUT LAYER                           │
│                                                             │
│   CRM Ticket Creation                                       │
│     ├── Title: auto-generated from process metadata         │
│     ├── Pipeline + Stage: routed by classification          │
│     ├── Custom object association: Member record linked     │
│     └── Document link: Cloud Storage URL attached           │
│                                                             │
│   Audit Log (parallel branch)                               │
│     ├── AI input snapshot                                   │
│     ├── AI output + justification                           │
│     ├── Timestamp + execution ID                            │
│     └── Written to spreadsheet via API                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Node-by-Node Breakdown

### 1. Webhook Trigger
- Receives POST from CRM on new legal process record creation
- Validated via Header Auth (shared secret)
- Payload parsed to extract process ID and metadata

### 2. CNJ Extraction (Code Node)
- Regex handles two CNJ formats in Brazilian court system
- Extracts the canonical 20-digit number for API lookup
- Falls back gracefully if number is missing or malformed

### 3. Judicial API Call (HTTP Request Node)
- REST call to external judicial data provider
- Returns structured JSON: parties, subject, claim type, court, dates
- Response mapped to standardized internal schema

### 4. AI Layer 1 — Relevance Filter (LLM Node)
- System prompt defines classification criteria in detail
- Few-shot examples cover edge cases (minors, representatives, admin vs. judicial)
- Output enforced as raw JSON — no markdown wrapping
- Branch logic routes by `classificacao` field value

### 5. AI Layer 2 — Clinical Depth (LLM Node)
- Only triggered for `E_SAUDE` cases
- Evaluates clinical specificity of the claim
- Flags ambiguous cases for human review rather than dropping them
- Output includes structured justification for audit trail

### 6. CRM Ticket Creation (HTTP Request Node)
- Calls CRM REST API with `application/json` body
- Text fields sanitized to prevent JSON serialization issues (newlines, quotes)
- Ticket associated to Member custom object via typed association
- Pipeline and stage set dynamically based on Layer 2 output

### 7. Cloud Storage Upload (HTTP Request Node)
- Process document uploaded to shared team folder
- URL returned and attached to CRM ticket as attachment

### 8. Audit Log (HTTP Request Node — parallel branch)
- Runs concurrently with ticket creation
- Logs: execution ID, timestamp, CNJ, AI input, AI output, justification
- Written to Google Sheets via API for prompt optimization analysis

---

## Error Handling

| Scenario | Behavior |
|---|---|
| CNJ not found in payload | Workflow stops, error logged |
| Judicial API returns empty | Workflow stops, no ticket created |
| AI returns malformed JSON | Retry once; if fails, flag for manual review |
| CRM ticket creation fails | Error captured, execution logged for retry |

---

## Design Principles

- **Fail safe over fail open:** when in doubt, stop rather than create a noisy ticket
- **Auditability first:** every AI decision is logged with full input/output context
- **Separation of concerns:** extraction, classification, and output layers are fully independent
- **Stateless execution:** each workflow run is self-contained — no shared state between executions

---

*Part of the [Judicial Triage Automation](../README.md) portfolio project.*
