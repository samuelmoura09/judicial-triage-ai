# 🔍 Prompt Audit Methodology

> Framework for evaluating and optimizing AI classification prompts using real anonymized cases.

---

## Overview

This document describes the structured audit methodology used to evaluate and improve the AI classification prompts in the Judicial Triage pipeline. The audit was conducted across **18 real anonymized cases**, improving classification accuracy from ~55% to 100% and eliminating all critical misclassification errors.

---

## Motivation

After the initial pipeline was deployed, qualitative feedback suggested the AI was:
- Over-classifying unrelated processes as `E_SAUDE`
- Missing edge cases involving minors and legal representatives
- Producing inconsistent outputs for clinically ambiguous cases

A structured audit was needed to identify failure patterns systematically rather than fixing cases ad hoc.

---

## Audit Setup

### Case Selection

18 cases were selected to represent:
- High-confidence true positives (clear health claims with confirmed diagnosis)
- High-confidence true negatives (clearly unrelated processes)
- Edge cases (minors, representatives, ambiguous clinical language)
- Previously misclassified cases (known errors from production)

All cases were fully anonymized before inclusion in the audit:
- Party names replaced with generic identifiers (Plaintiff A, Defendant B)
- CNJ numbers replaced with fictional equivalents
- Medical condition names preserved but unlinked from individuals

### Ground Truth Establishment

Each case was reviewed by a domain expert (legal + health insurance background) and assigned a ground truth label:
- Layer 1: `E_SAUDE / NAO_E_SAUDE / NAO_PERTINENTE`
- Layer 2 (for E_SAUDE cases): `Confirmed / Ambiguous / Not Applicable`

Ground truth was established independently of any AI output to avoid anchoring bias.

---

## Evaluation Process

### Step 1 — Baseline Run
Each case was passed through the current prompt version and the output recorded:
- Classification label
- Justification text
- Any parsing errors or format violations

### Step 2 — Error Categorization
Misclassifications were categorized by failure type:

| Category | Description |
|---|---|
| **Edge case gap** | Prompt didn't account for a specific legal scenario |
| **Ambiguity collapse** | Model defaulted to the wrong class on uncertain input |
| **Format failure** | Output not parseable (markdown wrapping, missing fields) |
| **Instruction conflict** | Prompt rules contradicted each other on edge cases |
| **Context overload** | Input too long; model lost track of classification criteria |

### Step 3 — Root Cause Analysis
For each misclassification, the prompt was analyzed to identify the missing or conflicting instruction that caused the error.

### Step 4 — Prompt Revision
Targeted changes were made to the prompt addressing each identified root cause. Changes were documented in the [prompt version history](prompt-design.md#iteration-history).

### Step 5 — Rerun & Validation
The revised prompt was rerun against all 18 cases. Results were compared against ground truth and the previous version.

---

## Results

### Before Audit (v1–v3 prompts)

| Metric | Value |
|---|---|
| Overall accuracy | ~55% |
| Critical errors (wrong class, high-confidence) | 4 cases |
| Format failures | 3 cases |
| Edge case misclassifications | 5 cases |

### After Audit (v7 prompt)

| Metric | Value |
|---|---|
| Overall accuracy | 100% |
| Critical errors | 0 |
| Format failures | 0 |
| Edge case misclassifications | 0 |

### Error Distribution (pre-audit)

```
Edge case gaps          ████████████░░░░  5 cases  (45%)
Critical misclassification ████████░░░░░░░░  4 cases  (36%)
Format failures         ████░░░░░░░░░░░░  3 cases  (27%)
Instruction conflicts   ██░░░░░░░░░░░░░░  1 case   (9%)
```

*Note: some cases had multiple error types*

---

## Key Findings

### Finding 1 — Binary output is insufficient for legal classification
The original `CERTO/ERRADO` schema forced the model to choose between two extremes, causing it to default to `CERTO` on ambiguous cases. Adding `NAO_PERTINENTE` as a deliberate "uncertain" class reduced false positives significantly.

### Finding 2 — Representative identity ≠ claim identity
Five misclassifications involved cases where the model inferred health-relatedness from the identity of the legal representative (a health insurance member) rather than the nature of the actual claim. Explicit instruction was required to separate these concepts.

### Finding 3 — JSON enforcement requires negative examples
Simply saying "respond only with JSON" was insufficient. Adding an explicit instruction ("No markdown fences. No preamble. No explanation outside the JSON object.") eliminated all format failures.

### Finding 4 — Justification fields reveal hidden reasoning errors
Cases where the final label was correct but the justification revealed flawed reasoning were caught only because the justification field was required. Without it, these would have appeared as correct classifications masking potential failure modes.

### Finding 5 — Clinical ambiguity needs its own class
Routing all `E_SAUDE` cases directly to ticket creation produced many low-quality tickets for processes with no confirmed clinical basis. Adding the `Ambiguous` tier in Layer 2 preserved these cases for human review without dropping them.

---

## Reusable Framework

This audit methodology is generalizable to any AI classification pipeline. Key steps:

1. **Build a diverse ground truth set** — include edge cases intentionally, not just easy examples
2. **Establish ground truth independently** — before running the AI, not after
3. **Categorize errors by type** — "wrong answer" is not a root cause
4. **Make one change at a time** — isolate which prompt change fixed which error
5. **Require justification in output** — it's the only way to debug at scale
6. **Rerun the full set after every revision** — fixes sometimes introduce regressions

---

## Audit Tooling

The audit was supported by a dedicated extraction workflow that:
- Called the workflow execution API with `includeData=true`
- Extracted AI inputs and outputs from each execution
- Logged results to a spreadsheet with classification labels and justifications
- Enabled side-by-side comparison of prompt versions across the same case set

This tooling transformed what would have been manual copy-paste work into a reproducible, scalable evaluation process.

---

*Part of the [Judicial Triage Automation](../README.md) portfolio project.*
