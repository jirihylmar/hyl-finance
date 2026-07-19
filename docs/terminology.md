# Terminology Registry (ratified)

One grep-friendly canonical name per concept. New or edited content anywhere in this repo must
use these names. Bootstrapped from the concepts touched by the first repo-hygiene pass; extend as
concepts are ratified, never by local drift in one file.

## Canonical names

| Concept | Canonical name | Grounded in |
|---|---|---|
| Received-invoices ledger | `Fakturace - prijate_faktury.tsv` | `dph-dap/Fakturace - prijate_faktury.tsv` (13 columns, `období` lowercase) |
| Issued-invoices ledger | `Fakturace - vydane_faktury.tsv` | `dph-dap/Fakturace - vydane_faktury.tsv` (13 columns, `Období` uppercase) |
| VAT-return aggregate | `Fakturace - DAP.tsv` | DAP line totals per period; base/VAT columns are `Základ_daně` / `DPH_21%` (no `_CZK` suffix) |
| Control-statement A.4 sample | `Fakturace - KH_A4_vydane_nad_10k.tsv` | 15 columns, leading `Oddíl` |
| Control-statement B.2 sample | `Fakturace - KH_B2_prijate_nad_10k.tsv` | 12 columns, no `Oddíl` |
| Batch working file | workbench (`dph-dap/.workbench-<tag>.tsv`) | written fresh per intake tranche, gitignored |
| Filing period | `období` in `YYYY_Q` format (e.g. `2026_1`) | user's filing period; overrides DUZP-derived quarter |
| Transaction type | `Typ` ∈ {`Tuzemsko`, `Zahraničí`, `Reverse charge`} | values verbatim as stored in the ledgers |
| Intake tranche | tranche | one batch of invoice PDFs processed by `/parse-invoices` |
| Customer work workspace | `projects/<customer-slug>/etapa-N/` | gitignored; e.g. `projects/brainmarket/etapa-1/` |
| Invoice-attachment work report | `Vydaná faktura - <N> - příloha Výkaz prací.{md,pdf,docx}` | output naming of `/prepare-work-report` |

## Bans (bare ambiguous words)

- Bare **"ledger"** — always say *received ledger* or *issued ledger* (or the full filename).
- Bare **"period"** when the filing period is meant — write **filing period** or `období`; DUZP-derived
  calendar quarter is a different thing and must be named as such.
- Bare **"report"** — distinguish the *work report* (invoice attachment) from VAT filings (DAP / KH).
