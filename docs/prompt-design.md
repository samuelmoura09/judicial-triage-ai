# 🤖 Prompt Design — Judicial Triage Automation

> Engineering decisions behind the two-layer AI classification system.

---

## Philosophy

The prompts in this pipeline were designed around three core principles:

1. **Precision over recall** — it's better to miss a case than to create noise
2. **Explicit edge case handling** — real-world legal language is ambiguous; the prompt must anticipate this
3. **Structured, parseable output** — downstream nodes depend on clean JSON; markdown wrapping breaks the chain

---

## Layer 1 — Relevance Classification

### Goal
Determine whether a legal process is genuinely related to health insurance coverage disputes.

### Key Design Decisions

**Three-class output (not binary)**

Early versions used a binary `CERTO/ERRADO` classification. This was replaced with a three-class system:

| Class | Meaning |
|---|---|
| `E_SAUDE` | Process is health-insurance-related → continue pipeline |
| `NAO_E_SAUDE` | Process is unrelated to health coverage → stop |
| `NAO_PERTINENTE` | Process is in scope but not actionable (e.g., already resolved) → stop |

The third class eliminated a significant source of false positives where resolved or administrative processes were being routed as active cases.

**Edge case: minor as plaintiff**

Legal processes where a minor is the plaintiff but an adult is the legal representative were initially misclassified. The prompt now explicitly instructs the model to evaluate the *nature of the claim*, not the identity of the representative.

**Edge case: lead as legal representative**

Cases where a member appears not as plaintiff but as legal representative for a third party were causing false positives. The prompt now distinguishes between being the subject of a health claim vs. acting as a procedural representative.

**Output format enforcement**

```
Respond ONLY with a valid JSON object. No markdown. No explanation outside the JSON.

{
  "classificacao": "E_SAUDE | NAO_E_SAUDE | NAO_PERTINENTE",
  "justificativa": "string — one paragraph explaining the decision"
}
```

The `justificativa` field is required for audit purposes even when the answer is obvious. This creates an interpretable log of every AI decision.

---

### Prompt Template (Layer 1)

```
You are a legal process classifier for a health insurance operation.

Your task: determine whether the process described below is related to a health insurance coverage dispute that requires internal investigation.

Classification rules:
- E_SAUDE: the process involves a demand for health coverage, medical procedures, reimbursements, or benefits under a health insurance contract
- NAO_E_SAUDE: the process is unrelated to health insurance (labor disputes, consumer issues, property, etc.)
- NAO_PERTINENTE: the process is in scope but not actionable (already resolved, purely administrative, duplicate)

Important edge cases:
- If the plaintiff is a minor represented by an adult, classify based on the nature of the health claim, not the representative's identity
- If the member appears as legal representative (not plaintiff), classify as NAO_PERTINENTE
- If the claim is ambiguous, lean toward NAO_PERTINENTE rather than E_SAUDE

Process data:
Subject: {{subject}}
Parties: {{parties}}
Claim description: {{claim}}

Respond ONLY with valid JSON. No markdown fences. No preamble.

{
  "classificacao": "E_SAUDE | NAO_E_SAUDE | NAO_PERTINENTE",
  "justificativa": "your reasoning here"
}
```

---

## Layer 2 — Clinical Depth Analysis

### Goal
For processes already classified as `E_SAUDE`, determine whether the clinical claim is specific enough to warrant a full investigation ticket.

### Key Design Decisions

**Why a second layer?**

Single-layer classification produced too many tickets for legally ambiguous cases — processes that mentioned health coverage but lacked a confirmed clinical condition. The second layer acts as a precision filter.

**Three-tier clinical evaluation**

| Output | Meaning | Action |
|---|---|---|
| `Confirmed` | Explicit diagnosis, CID code, or documented medical report | Full ticket with all metadata |
| `Ambiguous` | Symptoms described without confirmed condition | Ticket created with review flag |
| `Not Applicable` | Health claim exists but no clinical specificity | No ticket |

The `Ambiguous` tier was a deliberate design choice — dropping these cases entirely would miss legitimate edge cases; routing them with a flag preserves human oversight without generating noise.

**Justification requirement**

Every Layer 2 decision includes a structured justification referencing specific text from the process. This was critical for the prompt audit methodology — it allowed reviewers to evaluate AI reasoning, not just the final label.

---

### Prompt Template (Layer 2)

```
You are a clinical specificity evaluator for health insurance legal processes.

The process below has already been confirmed as health-insurance-related. Your task: evaluate whether the clinical claim is specific enough to warrant a full investigation.

Evaluation criteria:
- Confirmed: the process references a specific diagnosis, CID code, named medical procedure, or attached medical report
- Ambiguous: the process describes symptoms or general health complaints without a confirmed condition or clinical documentation
- Not Applicable: the health claim exists but is too vague, generic, or procedural to classify clinically

Process text:
{{full_process_text}}

Layer 1 classification: E_SAUDE
Layer 1 justification: {{layer1_justificativa}}

Respond ONLY with valid JSON. No markdown fences. No preamble.

{
  "classificacao": "Confirmed | Ambiguous | Not Applicable",
  "justificativa": "cite specific text from the process that supports your decision",
  "clinical_indicators": ["list", "of", "identified", "clinical", "terms"]
}
```

---

## Iteration History

| Version | Change | Reason |
|---|---|---|
| v1 | Binary CERTO/ERRADO output | Initial implementation |
| v2 | Added NAO_PERTINENTE class | Reduce false positives from resolved cases |
| v3 | Added minor/representative edge cases | Real cases exposing misclassification |
| v4 | Enforced raw JSON output | Markdown wrapping was breaking downstream parsing |
| v5 | Added Layer 2 clinical depth | Single-layer produced too many low-quality tickets |
| v6 | Added `Ambiguous` tier in Layer 2 | Preserve edge cases without generating noise |
| v7 | Added `clinical_indicators` field | Improve auditability and prompt optimization signal |

---

## Lessons Learned

- **Binary classification is almost never enough** for real-world legal language — always design for at least three classes
- **Output format is as important as classification logic** — a perfect classification in broken JSON is useless
- **Edge cases compound** — each new edge case you handle may interact with existing ones; document every change
- **Justification fields are not optional** — they are the only way to debug AI decisions at scale

---

*Part of the [Judicial Triage Automation](../README.md) portfolio project.*
