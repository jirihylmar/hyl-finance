# Session Notes

## Session 1 — 2026-04-20

### Task 0.1 — Read input materials and explore examples

**Status:** work done; BLOCKED on requirements gap before proceeding to 0.2.

#### Input materials breakdown

Folder `input/` contains exactly one file.

| File | Type | Size | Content |
|------|------|------|---------|
| `input/requirements.txt` | Requirements (plaintext) | 2 lines, 132 bytes | See below |

##### `input/requirements.txt` — extracted facts

Literal contents:
1. "This is the local worker. No aws connection needed for now."
2. "Primary gov related and tax issues will be processed and automated here."

Requirements extracted:
- **R1 — Deployment surface:** Local machine only. No AWS resources in scope (explicitly deferred with "for now" — may change later, but current scope excludes cloud).
- **R2 — Domain:** Government-related and tax processing/automation. No jurisdiction stated; given project author (`projekt1@hub440.cz`) and the "hyl-finance" name, Czech Republic gov/tax is the likely context but not confirmed in writing.
- **R3 — Role:** "Worker" — implies a processing/automation role (batch jobs, scheduled tasks, or on-demand scripts), not an interactive application or API server.

Constraints extracted:
- **C1 — No cloud dependency.** Any design must run locally without AWS.
- **C2 — No language, framework, runtime, or package manager specified.**
- **C3 — No data sources or inputs specified.** (Which tax forms? Which gov portals? Which file formats? None stated.)
- **C4 — No deliverables or outputs specified.** (Filed returns? Generated reports? Scraped data? CSVs? None stated.)
- **C5 — No schedule, volume, or SLA specified.**

Decisions extracted: **none** (no explicit decisions recorded in input).

External URLs to fetch via WebFetch: **none** (file contains no URLs).

#### Examples exploration

Repo `syndicate-playbooks-examples/` offers two playbooks:

| Playbook | Stack / shape | Fit for hyl-finance |
|---|---|---|
| `playbook-aws-serverless-multirepo` | AWS CDK + Lambda + API GW + DynamoDB + S3, multi-repo | **Poor** — violates R1/C1 (no AWS). |
| `playbook-mcp-mono-repo` | Mono-repo, documentation → MCP instance configs, exploration-first | **Poor** — this project is a local automation worker, not MCP config generation. |

**Observation:** neither existing playbook matches. hyl-finance is a "local automation / scripts" project, a category not covered by current playbooks. The README lists no planned "local-scripts" playbook either.

Two reasonable paths forward for Task 0.2:
- **Path A — Adapt `_project-template` directly** (skip picking a domain playbook). The core-rules workflow (progress.json, session protocol, atomic tasks, session_notes.md) still applies; only the domain expansion is custom.
- **Path B — Borrow structure from `mcp-mono-repo`** (mono-repo, exploration-first) since it's closest in shape (single repo, no deployment target), then strip MCP-specific conventions.

#### Gap / blocker before Task 0.2

Requirements are too thin to draft a meaningful `IMPLEMENTATION_PLAN.md` in Task 0.3 without speculation — and the playbook rules (CLAUDE.md §"Spec Discipline") forbid speculative content in the spec. Before proceeding, the user needs to fill gaps on at least:

1. **Jurisdiction + specific gov/tax processes in scope** — which forms, portals, or filings?
2. **Data sources** — manual file drops? scraping? API? bank exports? accounting software?
3. **Deliverables** — what does the worker *produce*? (Filed submissions, draft PDFs, CSVs for accountant, database records, email notifications?)
4. **Language / runtime preference** — Python? TypeScript/Node? Other?
5. **Trigger model** — cron, manual CLI invocation, file-watcher?
6. **Persistence** — local SQLite? flat files? none?
7. **Secrets handling** — where do gov-portal credentials live?

Path forward proposed: ask the user these questions, then complete Task 0.2 (template = adapted `_project-template`, rationale documented) and begin Task 0.3.

#### Files touched this task
- `session_notes.md` (created, this file)

#### Gap-filling answers from user (2026-04-20)

User elected to answer gap questions in-session rather than expand `input/`. Authoritative answers (to be treated as primary requirements alongside `input/requirements.txt`):

| # | Topic | Answer |
|---|---|---|
| Q1 | Jurisdiction | **Czech Republic** (primary) **+ EU-wide** (VAT OSS, Intrastat) as later scope. v1 processes are CZ-only; EU-wide is reserved for later phases. |
| Q2 | Language / runtime | **Python** |
| Q3 | Trigger model | **Manual CLI invocation** (no cron, no file-watcher in v1) |
| Q4 | Deliverables | **Filled forms (PDF + XML)** and **reports for accountant / human review**. No direct portal submission in v1. |
| Q5 | Processes in v1 | **CZ VAT return (P\u0159izn\u00e1n\u00ed k DPH)** + **Kontroln\u00ed hl\u00e1\u0161en\u00ed** (EPO XML); **CZ Income tax (DPPO / DPFO)** |
| Q6 | Data sources | **Bank statements** (CSV / ABO / camt.053) and **manual CSV / Excel** drops |
| Q7 | Persistence | **Flat files only** (JSON / CSV on disk). No database. |
| Q8 | Portal credentials | **Output-only in v1.** Plan a secrets approach but do not implement portal login/submission. Portal automation is a future phase. |

Consolidated requirements after gap-filling:
- **R1** Local-only Python worker, manual CLI.
- **R2** Produces CZ tax artefacts: DPH+KH EPO XML, DPPO/DPFO returns (PDF and/or XML per what FS \u010cR accepts), plus human-readable reports (CSV/Excel/PDF) for accountant review.
- **R3** Consumes bank exports (CSV / ABO / camt.053) and hand-maintained CSV/Excel inputs.
- **R4** No database \u2014 everything on the filesystem.
- **R5** No portal credentials stored in v1; secrets design deferred to a later phase.
- **R6** EU-wide processes (VAT OSS, Intrastat) are scoped but not part of v1.

Open items that surfaced:
- **O1** Specific output format for DPPO/DPFO is unconfirmed (FS \u010cR accepts EPO XML for DPPO/DPFO too, but user hasn't stated preference).
- **O2** Chart-of-accounts / categorization source: bank statements arrive raw. How transactions get classified (VAT rate, expense category, DPPO line) isn't specified \u2014 candidate for Phase 1 exploration.
- **O3** Multi-year / period handling (single tax year vs. rolling) not specified.

These three open items are acceptable for exploration-first treatment in Phase 1 and do not block Task 0.2.

#### Task 0.1 status
Ready to mark complete. `session_notes.md` now contains a per-file breakdown of all input material (1 file) and the gap-filling requirements that round out the spec basis.

#### Mid-session discovery: `resources/`

During Session 1, a `resources/` folder appeared on disk (not present at session start):

```
resources/
├── Přijaté Faktury/   (received invoices)
│   ├── 202501/  202502/  202503/  202504/  202601/  archive/  desktop.ini
└── Vydané faktury/    (issued invoices)
    ├── 2025/  2026/  archive/  desktop.ini
```

Actions taken:
- Added `resources/` to `.gitignore` — this is real financial data and must not be committed.
- Opened new gap task **0.1a** ("Confirm invoice data-source handling") to resolve before drafting the spec in 0.3. Questions to close: is invoice parsing in-scope for v1, what file formats live in these folders (PDF, ISDOC, XML, images), and do invoices feed VAT/KH calculation or are they archival-only?

This data source was not covered by Q6 (which asked about bank statements and manual CSV/Excel). It changes the probable shape of v1 meaningfully — for CZ DPH + Kontrolní hlášení, received/issued invoices are usually the *primary* source, not bank statements. The spec cannot be drafted until 0.1a is resolved.

---

### Session 1 closure

**Completed this session**
- Task 0.1 — Read input materials and explore examples ✓

**Verification results**
| Task | Verify | Result |
|---|---|---|
| 0.1 | `session_notes.md` has per-file breakdown + requirements | PASSED |

**New tasks added**
- 0.1a — Confirm invoice data-source handling (pending, depends on 0.1)

**Artifacts created**
- `session_notes.md` (this file)
- `.gitignore` entry: `resources/`

**Key decisions**
- `resources/` is gitignored (financial data, not for git).
- Requirements gap closed via in-session Q&A rather than expanding `input/` (user choice).
- Lead template candidate for 0.2: adapt `_project-template` directly — neither `aws-serverless-multirepo` nor `mcp-mono-repo` fits a local Python tax worker.

**Issues encountered**
- `input/requirements.txt` was far too sparse to draft a non-speculative spec; resolved by structured Q&A.
- `resources/` appeared mid-session, invalidating the assumption that data sources were fully covered by Q6; resolved by opening 0.1a and gitignoring the folder.

**Context for next session**
- Current task: **0.1a** (invoice handling), then **0.2** (template selection), then **0.3** (spec sections 1–3).
- Authoritative requirements are: (a) `input/requirements.txt` + (b) the 8 gap-filling answers recorded above + (c) whatever 0.1a determines about invoices.
- No AWS work authorized; no portal credentials; Python; CLI trigger; flat files.

**Git status**
| Repo | Status |
|---|---|
| orchestration (`.`) | needs_commit (progress.json, session_notes.md, .gitignore changed) |

---

## Session 2 — 2026-04-20

User directive: **finish Phase 0 in one session, then delete `syndicate-playbooks-examples/`**. The formal 0.5 blocking-approval checkpoint is collapsed into a single end-of-session review covering the full set of Phase 0 deliverables.

### Task 0.1a — Confirm invoice data-source handling

**New data drop:** `dph-dap/` folder (gitignored) containing 5 TSV files plus the same invoice subdirectories seen in `resources/`. TSVs are Google Sheets exports (filename pattern `Fakturace - <tabname>.tsv`) — the user maintains a curated Google Sheet as the source-of-truth ledger.

#### File-by-file breakdown of `dph-dap/`

| File | Role | Shape (header columns) |
|---|---|---|
| `Fakturace - prijate_faktury.tsv` | **Primary input**: full received-invoices ledger (34 rows observed) | `Typ`, `Dodavatel`, `DIČ_dodavatele`, `Číslo_faktury`, `DUZP`, `Částka_orig`, `Měna`, `Základ_daně_CZK`, `DPH_21%_CZK`, `Celkem_CZK`, `Zařazení_KH`, `Poznámka`, `období` |
| `Fakturace - vydane_faktury.tsv` | **Primary input**: issued-invoices ledger (3 rows observed) | `Typ`, `Odběratel`, `DIČ_odběratele`, `Číslo_faktury`, `DUZP`, `Částka_orig`, `Měna`, `Základ_daně_CZK`, `DPH_21%_CZK`, `Celkem_CZK`, `Zařazení_KH`, `Poznámka`, `Období` |
| `Fakturace - DAP.tsv` | **Sample output**: per-period DAP line-number aggregation — the format v1 must produce | `Řádek_DAP`, `Popis`, `Základ_daně`, `DPH_21%`, `Poznámka`, `období` |
| `Fakturace - KH_A4_vydane_nad_10k.tsv` | **Sample output**: KH section A.4 extract (issued, domestic, >10k CZK incl. VAT) | same as vydane + `Oddíl` prefix and section description |
| `Fakturace - KH_B2_prijate_nad_10k.tsv` | **Sample output**: KH section B.2 extract (received, domestic, >10k CZK) with a totals row | same as prijate |

#### Domain facts extracted from real data

- **Filing cadence:** quarterly. Period format `YYYY_Q` (e.g. `2025_3` = Q3 2025). Observed periods: `2025_3`, `2025_4`, and zero-filled placeholders for `2026_1`–`2026_4`.
- **DAP line numbers seen:** `1` (domestic supply), `5` (services received from EU §9(1)), `10` (domestic reverse charge §92a, supplier side), `12` (other taxable supplies with obligation to pay tax on receipt §108), `40` (input tax from taxable supplies from VAT payers), `43` (input tax from lines 3–13).
- **KH sections seen:** `A.2` (received services from abroad / reverse charge), `A.4` (issued domestic >10k CZK), `B.2` (received domestic >10k CZK), `B.3` (received domestic <10k CZK, aggregated).
- **Transaction `Typ` values seen:** `Tuzemsko` (domestic), `Reverse charge` (services from abroad), `Zahraničí` (foreign supplier with §9(1) place of supply).
- **VAT rate seen:** 21% (only). Reduced rates (12% current, historically 15%/10%) are not yet represented in data but must be supported in code.
- **Currencies:** CZK and EUR. When currency ≠ CZK, the sheet already contains CZK-converted base and VAT — v1 trusts the sheet's CZK values rather than doing FX itself.
- **DIČ handling:** Reverse-charge suppliers (Anthropic) have empty DIČ field; domestic and EU suppliers always have DIČ.

#### Decision for Task 0.1a

1. **In scope for v1 — primary inputs:** the two ledger TSVs (`prijate_faktury.tsv`, `vydane_faktury.tsv`). v1 reads these as input.
2. **In scope for v1 — outputs:** per-period DAP aggregation (matching the format of `DAP.tsv`), per-period KH section extracts (at minimum A.4 and B.2 + B.3 totals and A.2), EPO-compatible XML for both DAP and KH.
3. **Out of scope for v1:** parsing raw invoice PDFs in `resources/Přijaté Faktury/` and `resources/Vydané faktury/`. These remain archival; v1 does not read them. (Future phase may add PDF extraction as a cross-check.)
4. **Out of scope for v1:** direct Google Sheets API integration. v1 consumes the TSV files that the user exports manually. (Future phase may add Sheets API pull.)
5. **Downgrade of Session 1 answer Q6:** bank statements (CSV / ABO / camt.053) are **removed** as a v1 input. The TSV ledgers supersede them. Bank statements may return in a later phase for reconciliation.
6. **Folder convention established:**
   - `input/` — project-requirements docs (read-only, git-tracked).
   - `dph-dap/` — curated tabular ledger inputs (gitignored; real data).
   - `resources/` — invoice PDFs archive (gitignored; real data).
   - `output/` — generated DAP/KH TSVs, EPO XML, accountant reports (gitignored; will be added).
7. **Open item update:** O1 (DPPO/DPFO output format) and O2 (categorization source for income tax) remain open and will be handled exploration-first in a later phase. VAT-centric TSVs do not cover income-tax data yet.

**Task 0.1a verification:** session_notes.md now records invoice-handling decision, file formats, and role of each TSV. **PASSED.**

### Task 0.2 — Playbook template selection

**Selected:** adapt `_project-template` directly, borrowing the exploration-first pattern from `playbook-mcp-mono-repo`.

**Rationale:**
- `playbook-aws-serverless-multirepo` — rejected (R1/C1 forbid AWS; multi-repo is overkill for a single Python CLI).
- `playbook-mcp-mono-repo` — partial fit (mono-repo, exploration-first, no deployment target) but purpose mismatch (generates MCP configs, not a CLI worker).
- **Chosen path**: use `_project-template` as the structural baseline (progress.json bootstrap, 8 default commands, CLAUDE.md template shape) and add a small domain expansion:
  - Single-repo layout (no `infrastructure/`, `backend/`, `frontend/`, `testing/` sub-repos).
  - Exploration-first Phase 1 (pattern borrowed from mcp-mono-repo) because the real TSVs are our ground truth but the EPO XML schema still needs validation against FS ČR's current XSD.
  - No AWS sections in CLAUDE.md / no `/check-aws` usage in v1.
  - `tasks/phase_*.md` format borrowed directly from the aws-serverless playbook.

**Task 0.2 verification:** template choice and rationale documented above. **PASSED.**

### Session 2 pivot — add-work lifecycle adopted

At end of Session 2, after reviewing my draft deliverables (IMPLEMENTATION_PLAN.md, CLAUDE.md, tasks/phase_1..5.md), user reversed direction:

> "delete all you wrote. ad hoc tasks will be coming from me and we will formalise them one by one. just get on track add work lifecycle"

**Actions taken:**
- Deleted `IMPLEMENTATION_PLAN.md`, `CLAUDE.md`, `tasks/` folder (with all phase files) from disk.
- `progress.json` updated:
  - Tasks 0.3, 0.4, 0.5, 0.6, 0.7, 0.8 marked `superseded` (they produced deliverables that were subsequently discarded).
  - Phase 1–5 task blocks renamed `phase_N_*_superseded` with status `superseded`; all contained tasks marked `superseded`. Kept in file per append-only rule — audit trail only, do not treat as live work.
  - Tasks 0.1, 0.1a, 0.2 remain `complete` (understanding tasks, no discarded deliverable).
  - 0.9 left `pending` (user did not instruct deletion of examples folder under the new plan).
  - `current_task` → `null`. New top-level field `workflow: "add-work lifecycle"` documents the shift.
- `syndicate-playbooks-examples/` left in place (deletion was tied to the now-retracted full-Phase-0 plan).
- `input/requirements.txt` deletion and `projects/brainmarket/` folder — user declined to address in this session; left untouched.

**Forward workflow (confirmed):**
1. User proposes a task.
2. `/add-work` registers it in `progress.json` under a new phase or as a direct task.
3. Work is done.
4. `/update-progress` closes it.
5. No upfront spec; no pre-planned phases.

**Session 2 closure**

| Repo | Status |
|---|---|
| orchestration (`.`) | needs_commit — `.gitignore`, `progress.json`, `session_notes.md` modified; `input/requirements.txt` deleted; spec/CLAUDE.md/tasks pending delete-commit. |

Context for next session: `/start-session` sees `current_task: null` → expect user to specify the first add-work task. `dph-dap/` real data is available and understood (see Task 0.1a breakdown above) if needed.


