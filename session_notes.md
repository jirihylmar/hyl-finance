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

