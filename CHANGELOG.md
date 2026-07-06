# Changelog — Judicial Triage Automation

All notable changes to this project are documented here.
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.3.0] — 2026-06

### Changed
- Restricted `trecho_confirmatorio` source to RELATÓRIO PROCESSUAL CONSOLIDADO exclusively
- Expanded `cid_em_documento` extraction scope to all document sections (previously headers only)

### Fixed
- Eliminated hallucinated citations from auxiliary documents in Layer 2 output

---

## [1.2.0] — 2026-05

### Added
- Formal prompt audit across 18 real anonymized cases
- Parallel audit log branch capturing all AI inputs/outputs to Google Sheets
- `Ambiguous` tier in Layer 2 — routes borderline cases to human review instead of dropping them

### Changed
- Layer 1 output schema: added `justificativa` field (required for all classifications)
- JSON output enforcement strengthened: explicit negative instruction added ("No markdown fences. No preamble.")

### Fixed
- Edge case: lead as legal representative of minor now correctly classified as NAO_PERTINENTE
- Edge case: espólio (deceased estate) co-plaintiff signals no longer attributed to lead
- Format failures: markdown-wrapped outputs breaking downstream JSON parsing

---

## [1.1.0] — 2026-03

### Added
- `NAO_PERTINENTE` as third classification class (previously binary CERTO/ERRADO)
- Layer 2 clinical depth analysis for E_SAUDE cases
- CRM custom object association (Member record linked to ticket)

### Changed
- Pipeline routing now driven by Layer 2 output (Confirmed / Ambiguous / Not Applicable)
- Ticket creation migrated from native n8n HubSpot node to HTTP Request node (CRM v3 API) — enables custom property population

### Fixed
- Incorrect HubSpot association type IDs causing ticket-member link failures
- Broken pairedItem chains in multi-step flows — resolved with `.first()` syntax

---

## [1.0.0] — 2025-12

### Added
- Initial deployment: webhook trigger, CNJ extraction, judicial API call, single-layer AI classification, CRM ticket creation
- Header Auth webhook security
- CNJ regex handling for both masked and unmasked formats
- Google Drive document storage linked to CRM ticket

---
