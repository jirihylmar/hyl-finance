---
description: Ingest invoice PDFs (přijaté/vydané), classify per Czech VAT rules, update prijate_faktury.tsv / vydane_faktury.tsv / DAP.tsv / KH_A4 / KH_B2 (project)
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

# Parse Invoices

> Ingest a folder of invoice PDFs (received and/or issued), classify per Czech VAT rules, and append rows to the four ledger/aggregate TSVs under `dph-dap/`. No Python parser — read PDFs with the `Read` tool and apply LLM judgment plus the rules below.

**Arguments (optional free text after `/parse-invoices`):**
- `--period 2026_2` — Filing period in `YYYY_Q` format. **Required** unless obvious from the folder name. The user's filing-period convention **overrides DUZP-derived quarter** (user directive). If omitted, ask.
- `--received <folder>` — Folder of received-invoice PDFs. Default: most recent `dph-dap/Přijaté Faktury/YYYYMM/` not yet processed.
- `--issued <folder>` — Folder of issued-invoice PDFs. Default: `dph-dap/Vydané faktury/YYYY/`.
- `--no-merge` — Stop after producing the workbench (skip ledger append + aggregates). Useful for dry-run review.

If both flags missing, ask user which tranche(s) to process.

---

## Files touched (all gitignored under `dph-dap/`)

| File | Purpose | Updated how |
|---|---|---|
| `dph-dap/.workbench-<tag>.tsv` | Batch working file — one row per PDF classified this run | Written fresh |
| `dph-dap/Fakturace - prijate_faktury.tsv` | Received-invoices ledger (primary input) | Rows **appended** (and revised in-place when FX rate for a prior month becomes authoritative) |
| `dph-dap/Fakturace - vydane_faktury.tsv` | Issued-invoices ledger | Rows appended |
| `dph-dap/Fakturace - DAP.tsv` | Quarterly VAT-return aggregate (DAP line totals per period) | Lines 1, 5, 10, 12, 40, 43 for the target period updated in place |
| `dph-dap/Fakturace - KH_A4_vydane_nad_10k.tsv` | KH A.4 sample (issued domestic ≥10k CZK) | Rows appended for qualifying issued invoices |
| `dph-dap/Fakturace - KH_B2_prijate_nad_10k.tsv` | KH B.2 sample (received domestic ≥10k CZK) | Rows appended for qualifying received invoices |

**Never commit `dph-dap/*`.** It is gitignored via `.gitignore` line `dph-dap/`.

---

## Classification rules (tuned during the initial intake tranches)

### `Typ` (transaction type)

| `Typ` | When | Example suppliers |
|---|---|---|
| **Tuzemsko** | Supplier has a CZ DIČ (`CZ...`) AND place of supply is CZ. Also covers CZ branches of foreign entities (e.g. AWS EMEA Czech Branch CZ685187560 — even though invoice shows EUR) | O2, BDO, STARNET, AWS EMEA CZ Branch, Forpsi, IKEA |
| **Zahraničí** | EU-established supplier providing services with place of supply in CZ per § 9(1) (services to a VAT-registered business) — buyer reverse-charges under § 9(1) | Google Cloud EMEA (IE…) |
| **Reverse charge** | Non-EU supplier (US, UK after Brexit, etc.). Buyer reverse-charges under § 108 (import of services) | Anthropic PBC (US) |

**Signals to read from PDF:**
- Supplier country + VAT number presence. CZ DIČ or CZ-issued tax certificate → Tuzemsko.
- Note on invoice like "Services subject to the reverse charge - VAT to be accounted for by the recipient as per Article 196 of Council Directive 2006/112/EC" → Zahraničí (EU §9(1)).
- US/non-EU supplier with no CZ/EU VAT number → Reverse charge (§108).

### `Zařazení_KH` (Control Statement section)

| Section | For | Threshold |
|---|---|---|
| **A.2** | Received services from abroad with reverse charge (both EU §9(1) and non-EU §108 go here) | any amount |
| **A.4** | Issued invoices, domestic, `Celkem_CZK ≥ 10000` | ≥10 000 CZK incl. VAT |
| **A.5** | Issued invoices, domestic, `Celkem_CZK < 10000` | <10 000 CZK — aggregated totals, no per-row detail |
| **B.2** | Received invoices, domestic (`Typ = Tuzemsko`), `Celkem_CZK ≥ 10000` | ≥10 000 CZK incl. VAT |
| **B.3** | Received invoices, domestic (`Typ = Tuzemsko`), `Celkem_CZK < 10000` | <10 000 CZK — aggregated totals, no per-row detail |

**Received-side rule of thumb:**
- `Typ = Tuzemsko` + `Celkem_CZK ≥ 10000` → B.2
- `Typ = Tuzemsko` + `Celkem_CZK < 10000` → B.3
- `Typ = Zahraničí` OR `Typ = Reverse charge` → A.2

**Issued-side rule of thumb:**
- `Typ = Tuzemsko` + `Celkem_CZK ≥ 10000` → A.4
- `Typ = Tuzemsko` + `Celkem_CZK < 10000` → A.5

### DAP line mapping (received)

Used to regenerate `Fakturace - DAP.tsv` for the target period.

| DAP line | Description | Source rows |
|---|---|---|
| **1** | Dodání zboží/služby v tuzemsku (output domestic) | Sum of issued rows where `Typ = Tuzemsko`, with period = target |
| **5** | Přijetí služby § 9(1) z EU | Sum of received rows where `Typ = Zahraničí` (EU services with RC) |
| **10** | § 92a domestic reverse charge | Sum of rows with §92a classification — currently none in data; placeholder |
| **12** | Ostatní zdanitelná plnění § 108 | Sum of received rows where `Typ = Reverse charge` (non-EU, e.g. Anthropic) |
| **40** | Z přijatých zdanitelných plnění od plátců (input VAT from CZ payers) | Sum of received rows where `Typ = Tuzemsko` |
| **43** | Ze zdanitelných plnění vykázaných na řádcích 3 až 13 | = Line 5 + Line 12 (+ Line 10 when present) |

Each line aggregates two numbers per period — the `Základ_daně` (base) and `DPH_21%` (VAT) columns of `Fakturace - DAP.tsv` (note: the DAP file's headers carry no `_CZK` suffix, unlike the ledgers).

---

## FX rate for EUR invoices

**Method: AWS EMEA Czech Branch monthly-invoice rate.** The AWS invoice for the period `YYYY/MM` ships with a line `VAT in CZK (1 EUR = X.XXX CZK)`. This `X.XXX` is the month-specific rate the user applies to **all** EUR invoices whose DUZP lands in that same calendar month — including Google Cloud EMEA (reverse-charge §9(1)) and Anthropic (§108 reverse charge), neither of which prints a CZK rate.

| DUZP month | Rate source |
|---|---|
| AWS invoice in batch for month M | Use that rate for every EUR invoice with DUZP in month M |
| AWS invoice for month M not yet available | Temporarily carry over prior month's rate; flag in `Poznámka` as "FX provisional"; **re-run `/parse-invoices --revise-fx YYYY_MM`** when the AWS invoice arrives |

**Formula for EUR row (reverse-charge recipient):**
- `Základ_daně_CZK = Total_EUR_net × rate` (for AWS where net is printed) OR `Celkem_EUR × rate / 1.21 → round` (Anthropic/Google, which don't print VAT; use the total as base since there is no VAT on the foreign invoice).
- Actually for Google/Anthropic the "Total" EUR is the net (VAT 0% on invoice), so `Základ_daně_CZK = EUR × rate`, then `DPH_21%_CZK = Základ × 0.21`, then `Celkem_CZK = Základ × 1.21`.
- For AWS: PDF prints EUR_net and EUR_VAT separately; apply rate to each: `Základ_daně_CZK = EUR_net × rate`, `DPH_21%_CZK = EUR_VAT × rate`, `Celkem_CZK = EUR_total × rate`.

**Round to 2 decimals.** Known tolerance: rounding can make `base + VAT != total` by ≤0.02 CZK — match the user's existing pattern (two-decimals, bank rounding).

### Revising FX retroactively

When a later AWS invoice reveals the authoritative rate for a prior month and the rows for that month used a provisional rate, **update in place** in `Fakturace - prijate_faktury.tsv` (use `Edit` on the exact rows) and re-run aggregates for that period. Example: March 2026 rows for Anthropic `A0WDEPAH-0013` and Google `5527595008` were initially written at rate 24.245 (Feb carry-over) then revised to 24.515 when `EUINCZ26-57569.pdf` arrived.

---

## Filing-period assignment

`období` / `Období` = filing period in `YYYY_Q` format (e.g. `2026_1` = Q1 2026).

**Rule (user-ratified):** the user's filing period **overrides** DUZP-derived quarter. When running this skill, if `--period` is supplied, **all rows in this batch** get that period regardless of their DUZP. Example: invoices with DUZP `07.11.2025`, `30.11.2025`, `07.12.2025`, `31.12.2025` can still be assigned `období = 2026_1` if the user is processing them in their Q1-2026 filing.

If `--period` is not supplied, ask the user explicitly. Do NOT guess from DUZP.

---

## Duplicate check

Before appending to `prijate_faktury.tsv` or `vydane_faktury.tsv`:

```bash
awk -F'\t' 'NR>1 && $0!~/^$/{print $2"|"$4}' "<ledger>.tsv" | sort | uniq -d
```

Duplicates are keyed on `(Dodavatel or Odběratel) + Číslo_faktury`. If the check returns any row, stop and show the duplicates to the user.

If the same invoice is discovered in a new tranche (e.g. AWS EUINCZ26-16150 turned up as a duplicate in an earlier tranche), **exclude it from the workbench** with a note; do not append.

---

## Procedure

### 1. Confirm arguments

- Resolve `--period`. If missing, `AskUserQuestion` for it.
- Resolve folders. Default to `dph-dap/Přijaté Faktury/YYYYMM/` (pick the most recent non-empty) and `dph-dap/Vydané faktury/YYYY/`.
- List PDFs in each folder with `Bash ls`. Report count.

### 1a. Monthly-pattern completeness check (BEFORE reading PDFs)

Six suppliers bill monthly; a missing month is almost always a not-yet-downloaded invoice, and for
AWS it also means a missing authoritative FX rate. Derive the expectation from the ledger, not from
memory: for each supplier below, list its DUZP months already in `prijate_faktury.tsv`, then check
the batch folder covers every month from the last recorded DUZP up to the filing period's end.

| Supplier | Cadence (DUZP) | Invoice-number pattern |
|---|---|---|
| AWS EMEA Czech Branch | last day of month | `EUINCZ<YY>-<seq>` (also carries the month's FX rate) |
| Anthropic, PBC | ~6th of month | `A0WDEPAH-<seq>` (sequential, one per month) |
| Google Cloud EMEA | last day of month | 10-digit, ascending |
| O2 Czech Republic | ~7th of month | 10-digit, ascending |
| STARNET, s.r.o. | mid-month | `26XXXXXXXX`, ascending |
| BDO Euro-Trend (T-Mobile) | last day of month | `2026XXXXX`, ascending |

Report any gaps to the user BEFORE classification (they may want to download the missing PDFs
first). A genuinely skipped month happens (e.g. April 2026 had no AWS/O2/STARNET/Anthropic invoice
at all) — record the gap in the review summary so it is a documented fact, not a silent hole.

### 2. Read every PDF

- Use the `Read` tool on each PDF file. It extracts text and pages (handles Czech UTF-8 correctly).
- For repetitive suppliers (O2, BDO, AWS, Google, Anthropic, STARNET) the layout is stable — extract `Číslo`, `DUZP`, `Částka_orig`, `Měna`, base, VAT, total, supplier `DIČ`.
- For first-time suppliers, read the whole invoice and classify with judgment. If ambiguous (e.g. Polish OSS supplier charging CZ VAT to a B2C buyer — the SKI case), **flag inline in Poznámka** with `[FLAG: ...]` and surface to user in the review step.

### 3. Build the workbench

Create `dph-dap/.workbench-<tag>.tsv` (filename example: `.workbench-202602.tsv` for Feb 2026 batch; `.workbench-2026_1-vydane.tsv` for issued-side batch). Use the same column order as `prijate_faktury.tsv` for received or `vydane_faktury.tsv` for issued:

**Received columns (13):** `Typ, Dodavatel, DIČ_dodavatele, Číslo_faktury, DUZP, Částka_orig, Měna, Základ_daně_CZK, DPH_21%_CZK, Celkem_CZK, Zařazení_KH, Poznámka, období`

**Issued columns (13):** `Typ, Odběratel, DIČ_odběratele, Číslo_faktury, DUZP, Částka_orig, Měna, Základ_daně_CZK, DPH_21%_CZK, Celkem_CZK, Zařazení_KH, Poznámka, Období`

Note `období` lowercase in received, `Období` uppercase in issued (matches existing column headers).

For issued CZK-denominated invoices, leave `Částka_orig` and `Měna` **empty** (existing BRAINMARKET/GOLD SPORT rows follow this pattern).

### 4. Present for review (BLOCKING)

Summarize the batch:
- Total row count, split by supplier and by `Zařazení_KH`.
- Monthly-pattern gaps found in step 1a (missing supplier-months), stated explicitly.
- List each `[FLAG]` row with the concern.
- Period-assignment distribution (should be a single period).
- FX rate source used for EUR rows per month.

Use `AskUserQuestion` with options: `Approved — merge`, `Edits needed`, `Stop for manual review`.

Do NOT proceed past step 4 without explicit approval.

### 5. Merge into ledgers (after approval)

**Duplicate-safe append** using the awk check above. If the `prijate_faktury.tsv` source ends without a trailing newline, insert one before appending (common with files edited on Windows — they have CRLF or no final `\n`).

For FX-revision rows (same invoice, different CZK values than existing row), use `Edit` with the exact old line → new line. Do NOT append a second row for the same invoice.

### 6. Regenerate aggregates

For each affected period:

1. Filter `prijate_faktury.tsv` to rows where `období == <period>`.
2. Filter `vydane_faktury.tsv` to rows where `Období == <period>`.
3. Compute the 6 DAP line totals (1, 5, 10, 12, 40, 43) per the mapping above.
4. Use `Edit` to update the corresponding 6 rows in `Fakturace - DAP.tsv` (leave existing periods alone).

### 7. Regenerate KH section samples

- For received rows with `Celkem_CZK ≥ 10000` AND `Typ = Tuzemsko` AND `Zařazení_KH = B.2`: append to `Fakturace - KH_B2_prijate_nad_10k.tsv`. **The B.2 file ends with a totals row** (empty first 7 columns, then Σ Základ / Σ DPH / Σ Celkem) — insert new data rows before it and recompute the sums.
- For issued rows with `Celkem_CZK ≥ 10000` AND `Typ = Tuzemsko` (→ A.4): append to `Fakturace - KH_A4_vydane_nad_10k.tsv`. In existing A.4 rows the `Zařazení_KH` and `Období` columns are left **empty** — mirror that.
- Match each file's existing header exactly — the two layouts differ: `KH_A4_vydane_nad_10k.tsv` uses a 15-column layout with a leading `Oddíl` column (value `A.4`) and a description column; `KH_B2_prijate_nad_10k.tsv` uses a 12-column layout (the received-ledger columns minus `období`, no `Oddíl`). Read the header before appending.

### 8. Verify

Run these checks and show results to user:
```bash
wc -l "dph-dap/Fakturace - prijate_faktury.tsv"
wc -l "dph-dap/Fakturace - vydane_faktury.tsv"
awk -F'\t' 'NR>1 && $0!~/^$/{print $2"|"$4}' "dph-dap/Fakturace - prijate_faktury.tsv" | sort | uniq -d
awk -F'\t' 'NR>1 && $0!~/^$/{print $2"|"$4}' "dph-dap/Fakturace - vydane_faktury.tsv" | sort | uniq -d
```

Expected: row-count delta matches batch size; both `uniq -d` checks return empty.

### 9. Close with /update-progress

Mark the invoked tasks complete in `progress.json` (invoke `/update-progress` or edit directly if the tasks were added ad-hoc). Never commit anything under `dph-dap/` (gitignored real financial data); the repo itself is a private GitHub repo and tracked files (progress.json, session_notes.md, skills) push normally.

---

## Known supplier playbook (pattern quick-reference)

| Supplier | DIČ / ID | Typ | KH | FX source |
|---|---|---|---|---|
| O2 Czech Republic a.s. | CZ60193336 | Tuzemsko | B.3 (always <10k) | — CZK native. Invoice total includes a non-VAT installment portion (splátkový prodej): record `Základ/DPH/Celkem` from the VAT portion only (e.g. 659.5/138.5/798), full invoice total goes to `Částka_orig` (e.g. 966) |
| BDO Euro-Trend s.r.o. | CZ61500542 | Tuzemsko | B.3 (always <10k) | — CZK native |
| STARNET, s.r.o. | CZ26041561 | Tuzemsko | B.3 | — CZK native |
| AWS EMEA Czech Branch | CZ685187560 | Tuzemsko | B.3 | Rate printed on invoice |
| Google Cloud EMEA Limited | IE3668997OH | Zahraničí (§9(1)) | A.2 | Monthly AWS rate |
| Anthropic, PBC | — (US, no VAT) | Reverse charge (§108) | A.2 | Monthly AWS rate |
| INTERNET CZ, a.s. (Forpsi) | CZ26043319 | Tuzemsko | B.3 | — CZK native |
| GOLD SPORT s.r.o. | CZ01596497 | Tuzemsko | A.4 (if ≥10k) | — CZK native, issued-side |
| BRAINMARKET s.r.o. | CZ03488578 | Tuzemsko | A.4 | — CZK native, issued-side |
| TRAILERS & FACILITY, Orcik Tech, Bayo, IKEA, HP TRONIC | various CZ | Tuzemsko | B.2 (≥10k) or B.3 (<10k) | — CZK native |

When a **new supplier** appears, do not guess — read the PDF fully, check DIČ country code and VAT treatment note, and decide. If ambiguous, FLAG and ask.

## Anti-patterns (learned the hard way)

- **Don't use DUZP-derived quarter for `období`.** It is the user's filing period, which can differ.
- **Don't guess FX rate from ČNB or elsewhere.** Always use the same-month AWS invoice rate. If unavailable, carry over + flag.
- **Don't commit any file under `dph-dap/`.** Real financial data; gitignored for a reason.
- **Don't classify a Polish-supplier-with-CZ-VAT row without confirming.** OSS distance sales to B2C consumers are non-deductible for the CZ buyer. Always ask when buyer's DIČ is blank on an invoice that charges CZ VAT from a non-CZ supplier.
- **Don't append when you should revise.** If a prior-month EUR row exists with a provisional FX rate and the authoritative rate is now known, edit-in-place — do not append a second row.

## File-format landmines (learned the hard way)

- **Legacy rows contain bare carriage returns INSIDE the Poznámka field** (iDoklad copy-paste; e.g. the 2025 BRAINMARKET rows in `vydane_faktury.tsv` and `KH_A4`). awk (records = `\n`) sees one row; Python's `splitlines()` fragments it into several.
- **Never round-trip these TSVs through Python's default `read_text()`/`write_text()`** — universal-newline translation silently converts embedded bare CRs (and CRLF endings) into real newlines, corrupting row structure on rewrite. Read bytes or pass `newline=""`; split records on `\n` only; strip a trailing `\r` per line.
- **After any programmatic rewrite, verify structure**: every record has the expected column count (`awk -F'\t' 'NF!=13'` for ledgers). If rows got fragmented, repair by rejoining consecutive fragments until the column count is reached (join with a space).
- The ledgers mix line endings (CRLF on old Windows-edited rows, LF on appended ones) — this is harmless; append with plain `\n` and never "normalize" the whole file.
