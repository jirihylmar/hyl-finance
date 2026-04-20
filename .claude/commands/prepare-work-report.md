---
description: Prepare an invoice-attachment work report from one or more source code repositories. Captures the Customer-facing capabilities of the delivered systems — not a catalogue of code.
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Agent
  - AskUserQuestion
  - WebFetch
---

# Prepare Work Report

> Produce an invoice-attachment work report for a specific issued invoice. Output is a short, Customer-facing protocol describing what the delivered systems **do**, not a catalogue of code, git commits, or phase numbers.

The point of the attachment is to give the Customer's accountant or manager a credible, precise, easily-read picture of the work that was performed — at the level of system capabilities and business outcomes, grounded in authoritative live data (MCP) wherever available. Every line in the output should pass the **"is this already on GitHub?"** test: if the answer is "yes, trivially", the line belongs on GitHub, not in the attachment.

---

## Arguments (free-text after `/prepare-work-report`)

- `--invoice <number>` — Invoice number (e.g. `20260003` or `20260003-2`). Required. Determines output file names.
- `--customer <name>` — Customer legal name. Required unless obvious from context.
- `--period <from-to>` — Work period (e.g. `2025-09-01..2026-04-17`). Required.
- `--repos <path1,path2,...>` — Absolute paths to the source repositories to analyse. Required.
- `--workspace <dir>` — Output workspace. Default `projects/<customer-slug>/etapa-N/` (gitignored).
- `--language <en|cs>` — Output language. Default `en`. **Do not mix.**

If any required argument is missing, ask the user via `AskUserQuestion`.

---

## Files produced (all gitignored under `projects/`)

For invoice number `<N>`, three files land in the workspace:

| File | Purpose |
|---|---|
| `Vydaná faktura - <N> - příloha Výkaz prací.md` | Source (editable, version-controlled locally) |
| `Vydaná faktura - <N> - příloha Výkaz prací.pdf` | A4 PDF rendered via pandoc + xelatex + Latin Modern Roman |
| `Vydaná faktura - <N> - příloha Výkaz prací.docx` | Word version for optional Customer-side edits |

The invoice PDF itself (if already drafted) lives alongside them and is **not** modified.

---

## Clarification questions to ask up front (DO NOT skip)

These six questions prevent 80 % of the ping-pong in later iterations:

1. **Maturity**: is the delivery **prototype / semi-production / production**? → drives wording ("uvedena do poloprovozu" vs. "deployed" vs. "brought into production"). Default assumption: prototype with integration work continuing.
2. **Access URLs**: for each public URL the attachment will cite (app, MCP, API), is the URL OK to disclose or is it mapped inside the Customer's private accounts (e.g. Anthropic business)? → some URLs must be redacted with a note "provisioned in the Customer's ... accounts".
3. **GitHub org ownership**: who owns/operates the GitHub organisation? If it is the Supplier's (often pseudonymous), disclose that in the repo list.
4. **Customer-friendly URLs vs. raw cloud URLs**: which form does the Customer want? Some Customers prefer the raw Amplify / CloudFront domain over a Supplier-branded vanity domain.
5. **Scope figures**: should we cite live counts (from MCP queries) or bands ("hundreds", "thousands")? Bands stay valid across drift; exact counts date quickly.
6. **Consulting and testing clause**: confirm the exact wording. Default: *"The price invoiced for the reporting period also includes consulting with the Customer on architecture, specifications and functional design, and ongoing testing (unit, integration and live) carried out by the Supplier on its own infrastructure. These activities are an integral part of the deliverable and are not invoiced separately."*

---

## Workflow

### Phase A — Gather facts

**Prefer live MCP over source-file inference** whenever the platform exposes an MCP gateway.

1. **If the Customer's platform has an MCP gateway:**
   - Check for `.mcp.json` in the platform's main repo. If none, add one to this project root and note that a Claude Code restart is required for tools to load.
   - Call `get_solution_overview` + `get_help` on every exposed connector. This is authoritative, precise, reflects live data, and bypasses any drift in the source README.
   - Record: exact connector inventory, scope counts, legislation/source lists, language coverage caveats.

2. **Else (or in addition, when full coverage matters):**
   - Spawn one `Explore` agent **per repository** in parallel. Scope each agent to a focused read: README, top-level docs, architecture, config, a sampling of representative source files, a look at rendered outputs if on disk.
   - Ask each agent for: **(a) what the system does in business terms (2–3 sentences), (b) end-to-end workflow (max 6 steps), (c) 5–8 user-visible capabilities, (d) scope indicators (products, languages, thesauri, API endpoints), (e) what it replaces if relevant (informational only, will not appear in output).**
   - Do **not** ask for commit lists, phase numbers, or file catalogues.

3. **Check git history scope**: `git log --reverse --format="%ad" --date=short | head -1` on each repo. If the first commit falls within the reporting period, the whole system is in scope — do **not** report only the "last few phases"; the work is foundational.

### Phase B — Draft

Structure every attachment as follows (sections in this order):

```
# Work Report
Attachment to invoice no. <N>

## Identification
  (one-line paragraphs, blank line between each — Markdown requires this for line breaks)
  Supplier: ...
  Customer: ...
  Period: ...
  Subject: ...

## Summary
  (one paragraph on what the platform does + maturity statement if prototype)
  Access points:
    - Web application: <raw cloud URL unless user said otherwise>
    - MCP gateway: <description; do not publish the URL if it is provisioned in Customer's private accounts>
    - AWS account: <id>
    - AWS region: <region>
  Source code repositories (... — disclose pseudonymous Supplier orgs here):
    - repo1 — short role
    - repo2 — short role
    - repo3 — short role

## 1. <System A> — <short-name>
  Purpose: <one short paragraph>
  System functions:
    - 5–8 bullets describing user-visible capabilities
  Scope as of <date>: <quantitative fact>

## 2. <System B>  ... same structure

## 3. <System C>  ... same structure

## 4. Inter-system integration
  1. <what A provides to B and C>
  2. <what B provides to A and C>
  3. <what C provides to A and B>
  Single AWS account. Only genuinely shared infrastructure mentioned here.

## 5. Consulting and testing
  <confirmed clause>

Issued on <date>.
```

Each system section must be **roughly the same depth** (± 2 bullets is fine). Do not let one section balloon.

### Phase C — Ping-pong review

Present the draft. Invite inline notes using `/*` comments anywhere in the MD. Classes of feedback expected:

- **Marketing claims to remove** (examples found in past iterations: "from days to minutes", "revolutionary", "one click publishes everything")
- **Absolute claims to soften** ("all formats" → "selected formats"; "one click" → "on user request")
- **Untrue claims to correct** (often "shared Athena database" or similar invented integration)
- **Specifics to generalise** (concrete examples that are only examples → state they are examples; the system does more)
- **Negative framing to remove** ("what this replaces: ad-hoc Excel sheets..." — do **not** write this; it is unprofessional and adds nothing for the Customer)
- **URLs to redact** (any URL mapped in the Customer's private accounts)
- **Language issues** (calques, mixed CS/EN terminology, awkward translations)

Apply all notes in one pass. Strip every `/*` marker when done.

### Phase D — Render (ONLY after content is accepted)

```bash
pandoc <SOURCE.md> -o <OUTPUT.pdf> \
  --pdf-engine=xelatex \
  -V mainfont="Latin Modern Roman" \
  -V monofont="Latin Modern Mono" \
  -V sansfont="Latin Modern Sans" \
  -V geometry:margin=2cm \
  -V lang=en \
  -V papersize=a4 \
  -V colorlinks=true -V linkcolor=blue -V urlcolor=blue

pandoc <SOURCE.md> -o <OUTPUT.docx> -V lang=en
```

**Do not render until the user explicitly asks.** Early rendering burns feedback cycles on things the user can't judge from the draft anyway.

### Phase E — File naming and filing

Rename output files to the convention:
`Vydaná faktura - <invoice-number> - příloha Výkaz prací.{md,pdf,docx}` (Czech diacritics in the filename are fine).

Ensure `projects/` is in `.gitignore` (privacy parity with `dph-dap/`). The output PDFs and DOCX contain real client names, URLs, and scope figures — they belong on disk, not in git.

---

## Content rules — what to write and what NOT to write

### DO

- Write for the Customer's accountant/manager. Not the engineer.
- Describe **capabilities** (what the system *does*), not implementation (what files exist).
- Prefer bands ("thousands of descriptors") over exact counts that drift.
- Use examples ("for example internal resale specification, customer specification, supervisory label, packaging label") for anything template-driven or configurable — say "for example" explicitly.
- Cite legislation and authoritative sources with full official names and CELEX/ELI where relevant. Regulatory databases are usually the most valuable line for the accountant to see.
- State the consulting-and-testing clause explicitly; it must be present.

### DON'T

- **No marketing**: "from days to minutes", "revolutionary", "seamless", "cutting-edge", "world-class". None.
- **No "what this replaces" sections**. Negative framing comparing the new system to the legacy Excel process is unprofessional in an invoice attachment.
- **No absolute claims** unless literally true. "One click publishes everything in all languages" is never literally true; "publication on user request" with a 5-step pipeline description is.
- **No dates, timelines, or promises** beyond the reporting period.
- **No internal phase numbers** (Phase 15, Phase 51, etc.). The Customer doesn't care and the last few phases are not the same as "what was built". If the whole system was built from scratch, describe the whole system.
- **No commit counts, file paths, Lambda function names, stack class names.** Those are on git and add noise.
- **No tables with missing headers** (`| | |`) — they render inconsistently. Use bulleted lists for key-value content.
- **No mixed CS + EN terminology**. Pick one output language and stick to it.

### Anti-patterns found in past iterations

| Anti-pattern | Correction |
|---|---|
| "brought into production" when it's a prototype | "prototype; integration work continues" |
| "shared Athena database across all three systems" | "runs under a single AWS account" (drop the false claim) |
| "four output document types" (when template-driven) | "arbitrary output document types through dynamic Handlebars templates — for example ..." |
| "two product-name rules" (when there are many) | "a wide set of business and regulatory rules — for example ..." |
| "5 items or 29 s triggers async mode" (internal detail) | "batch processing of hundreds of products through an asynchronous-execution mode, transparent to the user" |
| "Co nahrazuje: Excel tabulky bez verzování ..." | Delete the section entirely |
| "Phase 45 → 56 of BPSS" | Do not cite phase numbers; describe the system as a whole |
| `https://mcp.hub440.cz/...` published in the attachment | Redact: "provisioned in the Customer's Anthropic business accounts" |
| `https://bmpss.digital-horizon.cz` (Supplier vanity) | Use raw cloud URL (`https://main.dzx5nm1pynwm7.amplifyapp.com/`) unless Customer says otherwise |
| "GitHub organisation DigitalHorizonCz" (no disclosure) | "... a pseudonymous organisation owned and operated by the Supplier" |
| Markdown table with empty header `\| \| \|` | Bulleted list |
| Single-newline line breaks in Identification block | Blank line between each item (Markdown requires this) |

---

## MCP-first playbook (when the platform has an MCP gateway)

This section is a concrete shortcut for platforms that expose MCP — use it in preference to reading source code when possible.

1. Inspect the platform's main repo for `.mcp.json`. Copy the `mcpServers` block to this project's root `.mcp.json` **before any analysis**. Restart Claude Code. Verify tools load via `ToolSearch` for the server name.
2. Call `<prefix>_help_get_solution_overview` first — this yields the authoritative system map: instance inventory, scope counts, language coverage, data model, cross-instance workflows, boundaries.
3. For each instance, call its `_get_help` tool. This gives precise rules, registered legislation with CELEX/ELI, language coverage caveats, exact notes partitions, and translation provenance.
4. Synthesise the attachment from these outputs. The connector inventory and the legislation list are the two most valuable artefacts for the attachment — they answer "what did the work produce" at the right level.

---

## Privacy and disclosure

- `projects/` is gitignored (parity with `dph-dap/`). Output files stay local.
- Redact URLs mapped in Customer's private accounts (Anthropic business, Google Workspace, private IdPs, etc.).
- Disclose Supplier-operated GitHub organisations that could otherwise be mistaken for Customer orgs.
- If the Customer's data contains supplier DIČs or real invoice numbers in tracked files, only do so in a **private** repo — verify before committing.

---

## Acceptance checklist

Before rendering the final PDF/DOCX:

- [ ] All `/*` inline notes resolved and stripped from the source.
- [ ] No marketing vocabulary, no "what this replaces" sections, no absolute claims.
- [ ] All URL / phase-number / file-path content that fails the "is this on GitHub" test is removed.
- [ ] Identification block uses blank-line-separated paragraphs (renders as separate lines).
- [ ] Every system section is roughly the same depth.
- [ ] Legislation list is complete and authoritative (cross-check with MCP if available).
- [ ] Consulting-and-testing clause is present and verbatim.
- [ ] Output language is consistent throughout (no CS/EN mixing).
- [ ] File names follow `Vydaná faktura - <N> - příloha Výkaz prací.{md,pdf,docx}`.
- [ ] `projects/` is in `.gitignore`.

---

## Example run (compressed)

```
user: /prepare-work-report --invoice 20260003-2 --customer "BRAINMARKET s.r.o." \
      --period 2025-09-01..2026-04-17 \
      --repos ~/aps-brm-products-playbook,~/app-brm-manufacturing-dictionary,~/app-brm-manufacturing-products

model: [asks the 6 clarification questions]

user:  prototype; MCP URL private; raw Amplify URL; DigitalHorizonCz is Supplier's;
       use bands not exact counts; default consulting clause

model: [spawns 3 Explore agents; when MCP becomes available, re-queries all connector _get_help]
       [drafts .md with 5 sections + Identification + Summary + consulting clause]
       [presents draft for inline /* review]

user:  [adds /* notes inline, requests generalisations, redactions, Czech cleanup]

model: [applies all notes in one pass; strips /* markers]
       [renders PDF + DOCX only after user says "render"]
       [renames files to the Vydaná faktura convention]
```
