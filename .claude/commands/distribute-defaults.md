---
description: Distribute updated default commands to all playbook projects (local + remote workspaces), scoped-commit + robust FF-push everywhere by default (central)
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Write
---

# Distribute Defaults

Push the latest versions of the 10 default commands from this repo to all playbook projects, **respecting per-project local customizations** via overlay files, then **scoped-commit only the files this run changed** so nothing is left hanging — without disturbing other agents' concurrent work.

**Push policy — robust and self-governing; the operator decides nothing and remembers no flag.** A zero-flag run **deploys**: it updates + scoped-commits the canonical and every project, then pushes only what it changed to each origin and propagates to every remote workspace. This is robust, not reckless — every push is **fast-forward-only** and the commits are **scoped** (only the changed default files), and any repo that can't FF (dirty / diverged / stale checkout) is **skipped and reported, never forced**. Note that `git push` publishes whatever a repo is ahead by — this run's scoped commits *plus any pre-existing unpushed commits* on that branch (a deploy's job is to get everything to origin); a repo with nothing ahead of upstream is a no-op. The rare `--no-push` stages commits without publishing.

**This command runs from `syndicate-playbooks-examples` on the local WSL machine only.** It is NOT distributed to projects. It orchestrates remote workspaces over SSH; it never runs from a box.

You (the operator) call `/distribute-defaults` with *what to change*. This skill owns *how* it propagates everywhere.

---

## What changed vs. the old single-host version

The old skill walked only `/home/hylmarj/*/` and never committed. It now:

1. **Multi-location.** Distributes to **all workspaces**: local (`$HOME/*/`) plus every remote box in `~/.syndicate-remote-secrets/box.json` (and any future boxes in `~/.syndicate-remote-secrets/workspaces.json`). Nothing is hardcoded to `/home/hylmarj`.
2. **Scoped commit + robust FF-push by default.** After distributing, it commits **only the specific default files it changed** in each project (`git add -- <those files>` then `git commit -- <those files>` — a partial commit that ignores anything else in the index) and pushes FF-only. Concurrent agents' uncommitted/staged work is never swept in; any non-FF repo is skipped + reported, never forced.
3. **Delivery is via ORIGIN — nothing runs on, or is installed on, a box.** See below.

---

## Delivery is via origin (the rule that makes "all repos, wherever they are" true)

**The promise: EVERY playbook project receives the canonical, wherever it lives. That is not conditioned on whether the project happens to be checked out on this machine** — local existence is an accident of who cloned what, not a fact about the project.

**Origin is the universal bus.** Every playbook project has one, and every host already FF-pulls from it (`/start-session` Step 0). So distribution reduces to **making origin correct**:

| Project | How it is served |
|---|---|
| **Local** (`$HOME/*/` here) | Step 1: local engine writes + scoped-commits + FF-pushes to its origin. |
| **Shared** (here *and* on a box) | Same as local — one push, every host pulls it. |
| **Remote-only** (on a box, no local checkout) | Step 3: `git clone <its origin>` into a **transient** temp dir, then `bash $ENGINE --workspace $TMPWS --apply --commit --push`, then `rm -rf`. The clone is throwaway; the push is what lasts. |

Then the box is brought current with a **FF-pull** of every playbook project there — never a force, never a write of our own content. That pass also arms `core.hooksPath` per project (repo-local config that cannot travel via git — the one thing a push cannot deliver). A project is skipped + reported only if git itself refuses the fast-forward (it would clobber a dirty file) or the branch has diverged; unrelated dirt is **not** a reason to withhold the canonical.

**Why remote-only projects are served from here rather than by an engine on the box.** `syndicate-playbooks-examples` and `syndicate-remote` are **LOCAL-ONLY by policy and must never live on a box**. A box-side engine would require the examples repo there — so the engine goes to the projects instead of the repo going to the box. This also removes a whole failure class: no bootstrap, no box-side install, nothing to drift, nothing to clean up.

> **History lesson — do not re-introduce this.** An earlier revision read the examples repo's *absence* from the box as a bug and **bootstrap-cloned it**. The absence is the policy. Worse, auto-cloning made the policy unenforceable by deletion: remove the repo, and the next deploy silently re-planted it. If you find yourself adding a clone-to-box path, you are solving the wrong problem — use `--workspace` and push to origin.

---

## The mechanics live in two scripts (not inline bash)

| Script | Role | Runs on |
|---|---|---|
| `scripts/distribute-defaults.sh` | **Engine** — discover projects in a workspace (`$HOME/*/`, or `--workspace <dir>`), classify, apply overlays, scoped commit/push. | **Local WSL only.** It never runs on a box — remote-only projects are served from here via a transient clone. |
| `scripts/distribute-defaults-all.sh` | **Orchestrator** — drives local → examples-repo push → each remote workspace (deploy by default; `--no-push` stages). | Local WSL only. |
| `scripts/apply-overlay.py` | Bakes canonical + a project's overlay. | Local WSL only (alongside the engine). |

All three scripts stay on the local authority host. Nothing is shipped to, installed on, or executed in a box workspace. The box-side footprint is exactly three `ssh` calls: a reachability probe, a **read-only** project enumeration, and a **FF-pull pass** — which also arms `core.hooksPath` and `chmod +x`es the delivered hook, because that is repo-local config that **cannot travel via git** and is therefore the one thing a push cannot replace.

### Engine flags (`scripts/distribute-defaults.sh`)

```
--report            classify + print drift (always implied; default action)
--apply             write canonical(+overlay) into stale/overlay-stale/missing targets
--commit            scoped-commit ONLY the default files this run changed
--push              push each committed project's branch
--pull-first        FF-pull each project before classify (never forces)
--workspace <dir>   enumerate projects from <dir> instead of $HOME
--sync-internal     copy canonical -> this repo's internal command copies (authority host only)
--dry-run           simulate; write/commit/push nothing
--json              machine-readable summary on stdout
```
`--workspace` is what lets the LOCAL engine serve projects that are not checked out on this machine (see **Delivery is via origin** below). Only project *enumeration* moves; `REPO_ROOT` still resolves from the engine's own location, so the canonical and **its git history** stay local — which the `stale` classifier requires. **Never override `$HOME` to do this**: git's `store` credential helper resolves `~/.git-credentials` via `$HOME`, so a `HOME=` override silently breaks auth and every push fails with an unexplained `push-fail`.
Exit 3 = blocking drift (`divergent`/`broken-overlay`) present and `--apply` was requested → nothing applied (a `--dry-run` warns and continues instead).

### Orchestrator flags (`scripts/distribute-defaults-all.sh`)

```
(no flags)      deploy: update + commit + FF-push everywhere + propagate to remotes (robust; skips conflicts)
--no-push       stage only: commit everywhere, push nothing, no remotes
--push          explicit deploy (already the default); accepted for clarity
--dry-run       simulate everything end to end
--report-only   classify on every host (local + remotes, read-only), change nothing
--local-only    never contact remote workspaces
```

---

## The 10 Default Commands

Canonical defaults maintained in `_project-template/.claude/commands/`:

1. `add-work.md`
2. `check-aws.md`
3. `generate-architecture.md`
4. `generate-phases.md`
5. `repo-hygiene.md`  ← periodic consolidation pass; triggered by the clock gate in `start-session` Step 2.7 + the phase-close hook in `update-progress` Step 3a
6. `syndicate-refresh-remote.md`  ← paired with the `syndicate-refresh-remote` CLI installed from `~/syndicate-remote/scripts/install.sh`
7. `setup-workflow-only.md`
8. `setup.md`
9. `start-session.md`
10. `update-progress.md`

`distribute-defaults.md` (this file) is **not** one of the 10 and is never distributed.

---

## Commit guard (distributed beyond the 10 commands)

Besides the 10 command files, the engine also distributes a **mechanical commit guard** and a `.gitignore` baseline so `git add -A` can never silently sweep a build artifact into history (the failure that motivated it):

- `_project-template/.claude/hooks/pre-commit` — canonical guard. **Single centrally-owned script, overwrite-on-difference** (no overlay, no `divergent`/`broken-overlay` state — projects must never hand-edit it). Blocks a commit that *newly adds* a build-artifact-pattern file or a >25 MiB blob; never inspects already-tracked files. Synced by `sync_hook()` (chmod +x **before** `git add` so the index records mode `100755`).
- `_project-template/.claude/hooks/artifact-guard.allow` — per-project allowlist (gitignore-style globs). **Seed-if-missing only**, then project-owned — this is the sole per-project variance point.
- `_project-template/.gitignore` artifact baseline — merged into each project's `.gitignore` under a `# >>> syndicate artifact-guard baseline >>>` managed marker block by `sync_gitignore()` (append-if-absent, idempotent; skip + report on a concurrent unstaged `.gitignore`).

On `--apply` the engine also runs `git config core.hooksPath .claude/hooks` per project, so the **delivering run arms the guard** (closing the fresh-clone-on-box window — no wait for the next `/start-session`). The engine's own scoped commits use `--no-verify`, so a project's possibly-broken hook can never deny distribution. `/setup` installs+arms it for new projects; `/start-session` Step 0.5 re-arms `core.hooksPath` every session (it is repo-local config that does not travel with a clone).

---

## Skills (a third payload type, added 2026-07-16)

The engine ships **three** kinds of payload, not one. The 10 command `.md` files go through the full
classify/overlay/divergent machinery; the commit guard is a single centrally-owned file; and a
**skill** is a *directory* of files — `SKILL.md` plus whatever it needs beside it (a Python script, a
template, reference docs).

- **Source:** `_project-template/.claude/skills/<name>/` → delivered to `<project>/.claude/skills/<name>/`.
- **Discovered, never listed.** There is no `SKILLS=` variable. Adding a skill means adding a
  directory — **no edit to `distribute-defaults.sh`**. This is deliberate: a hardcoded list is the
  same disease as the hardcoded sub-repo loop and the hardcoded command catalog, both of which this
  estate has already had to cure. It is also the generality proof — verified 2026-07-16 by dropping a
  throwaway second skill in and watching it ride with zero code change (`missing` went 32 → 64, one
  per project per skill, and back to 32 on delete).
- **Single-owner, overwrite-on-difference** — same contract as the commit guard, no overlay. The
  splice/overlay machinery is markdown-anchor based and cannot bake a Python script; projects never
  hand-edit a skill. States are `identical` / `missing` / `stale` in the same summary vocabulary as
  commands. Source file mode is preserved, so an executable helper stays executable.
- **The development record does not ship.** Only what a project needs goes in the skill directory;
  verification evidence and provenance live under `docs/` in this repo. Fifty checkouts do not need
  the audit trail.
- **Dogfood:** the authority host syncs the same skills into its own `.claude/skills/`, so this repo
  runs what it ships.

> **Precedence trap — the reason a delivery can silently do nothing.** Claude Code resolves skills
> **Enterprise > Personal > Project**. A personal skill at `~/.claude/skills/<name>/` **shadows** the
> distributed project copy: the files land, every drift check passes, the run reports success — and
> the host keeps running the old personal copy in every project, forever. If a skill previously lived
> at user level on any host, **retire that copy**, and verify behaviourally (does the *distributed*
> one run?) rather than by file presence.

---

## Workspaces (locations)

Discovered in order, deduped by host:

1. **Local** — always; `$HOME/*/` on this WSL machine.
2. `~/.syndicate-remote-secrets/box.json` — the single object form (today's box).
3. `~/.syndicate-remote-secrets/workspaces.json` — optional **array** of `{host,user,workspace,ssh_key}` for additional/future boxes.

Each remote entry needs `host`, `user`, `workspace`, `ssh_key`. Unreachable or non-fast-forwardable workspaces are **skipped and reported**, never forced.

---

## Local Customization Mechanism (unchanged)

Each project may keep customizations in `<project>/.claude/local-overlays/`:

- `<command>.md` — **splice-based overlay** (additive). Directives, anchored to exact canonical lines:
  ```markdown
  <!-- splice-before: "EXACT CANONICAL LINE" -->
  <!-- splice-after:  "EXACT CANONICAL LINE" -->
  <!-- splice-append -->
  ```
  Lines before the first directive are an ignored header comment. First match wins; a missing anchor is a `broken-overlay` (blocks).
- `.skip` — newline-separated canonical filenames this project has fully forked. Listed files are **never written**. Use sparingly — each entry is a permanent fork. `#` comments and blanks allowed.

---

## Classification (per target file)

| Status | Meaning | Action on `--apply` |
|--------|---------|---------------------|
| `identical` | target == canonical | none |
| `overlay-ok` | target == canonical + overlay (baked) | none |
| `missing` | no target file | write canonical(+overlay) |
| `stale` | target matches a historical canonical (sha256, robust to line shifts) | overwrite |
| `subset` | target is canonical minus some lines — zero target-only content, provably lossless | overwrite |
| `overlay-stale` | overlay present, baked != target | overwrite with canonical+overlay |
| `skipped` | file in `.skip` | none (report) |
| `divergent` | differs, no overlay, no historical match, has target-only content | **BLOCK** — needs resolution |
| `broken-overlay` | overlay parse/anchor error | **BLOCK** — fix overlay |

`stale` detection walks `git log` of the canonical (robust to line shifts; not `--follow`, so a canonical version from before a file rename is not matched). `subset` covers a target that is an older/truncated distribution (canonical-minus-some-lines with no target-only lines) — overwrite-safe. `divergent` means the target has content that was never a canonical version and isn't expressed as an overlay — it must be resolved (overlay, `.skip`, or explicit overwrite) before a real apply.

---

## Procedure the skill follows

### 1. Verify host + repo
```bash
basename "$(pwd)"          # must be syndicate-playbooks-examples
```
Refuse if run from a box (the orchestrator also guards on hostname `ip-172-31-*`).

### 2. Make the canonical change
Edit the relevant file(s) in `_project-template/.claude/commands/` per the operator's "what to change" request. This is the only creative step.

### 3. Dry-run report across all hosts (always do this first)
```bash
bash scripts/distribute-defaults-all.sh --dry-run
```
Read the per-host drift. Present a summary table to the operator. If any `divergent` / `broken-overlay` appears, **STOP** and resolve (overlay, `.skip`, or explicit operator approval to overwrite) — a real apply will refuse otherwise.

### 4. Gate on blockers only (do not ask the operator to weigh push consequences)
From the dry-run: if any `divergent` / `broken-overlay` appears, **STOP** and resolve (overlay, `.skip`, or fix). That is the only thing that blocks. Do **not** ask the operator "publish now or later?" — the deploy is self-governing and safe (FF-only, scoped, skip+report). Just report what will change, then proceed.

### 5. Execute — deploy (default)
```bash
bash scripts/distribute-defaults-all.sh
```
This runs, in order:
1. **Local**: engine `--sync-internal --apply --commit --push` (distribute + scoped-commit + FF-push local projects).
2. **Examples repo**: scoped commit + push of canonical + internal copies.
3. **Each remote workspace**: enumerate its playbook projects over SSH; any with **no local checkout** is served from here via a transient clone (`--workspace`) pushed to its own origin; then FF-pull every playbook project on the box so it is immediately current. **Nothing is installed on the box.**

Every push is FF-only + scoped; any repo that can't fast-forward is skipped and reported (never forced). Use `--no-push` only if you explicitly want to stage commits without publishing.

### 6. Verify
```bash
bash scripts/distribute-defaults-all.sh --report-only
```
Every project on every host should be `identical`, `overlay-ok`, or `skipped`. Re-deploy if any repo was skipped on a transient conflict (or resolve it with `/syndicate-refresh-remote`).

---

## Safety guarantees

- **Only the 10 default filenames plus the commit-guard surface** are ever touched — never project-specific commands. The guard surface is `.claude/hooks/pre-commit` (overwrite-on-difference), the `.claude/hooks/artifact-guard.allow` seed (only if absent), the managed `.gitignore` block, and `core.hooksPath` (armed per project on `--apply`).
- **Scoped commits.** `git add -- <changed defaults>` then `git commit -- <changed defaults>` commits a snapshot of only those paths; any other staged/unstaged work by a concurrent agent is left exactly as-is.
- **No forcing.** Remotes only fast-forward; dirty/diverged repos (project or examples) are skipped and reported, with a pointer to `/syndicate-refresh-remote` to resolve.
- **`.skip` files** are never written. **Overlays** are baked, not overwritten canonical-only.
- **Blockers gate real applies.** `divergent`/`broken-overlay` abort a real `--apply` (engine exit 3); `--dry-run` continues and reports.
- **Fail-safe overlay bakes.** A broken/stale overlay anchor is reliably detected — `apply-overlay.py` exits 2 on any bake error (in every mode, emitting no partial output) and the classifier reads that code directly — so it classifies `broken-overlay` and BLOCKS rather than silently overwriting. Writes are atomic: `write_target` bakes to a temp file and promotes only on success, so a failed bake never truncates or partially-writes a target. (Hardened after a latent bug silently dropped a customization block from stale-anchored overlays following a canonical heading renumber.)
- **Deploy is self-governing.** Full deploy is the default; the operator is never asked to weigh push timing. Safety comes from *how* it pushes (FF-only, scoped, skip+report), not from withholding the push. `--no-push` stages; `--report-only` is read-only.

---

## Notes

- For a dry run, use `--dry-run` (full simulation) or `--report-only` (classify only).
- The examples-repo canonical commit is scoped to `_project-template/.claude/commands` + the internal copies. The engine/orchestrator scripts and this skill `.md` are infrastructure — committed by their author, not by routine runs.
- **A fresh box needs no bootstrap and no install — ever.** Nothing is placed on a box: not the examples repo, not the engine. A new box is served the moment its projects are reachable by origin; its box-side footprint is a reachability probe, a read-only enumeration, and a FF-pull pass (which also arms `core.hooksPath`). (Superseded design: the skill used to claim "first-ever deploy bootstraps itself", which was false for months — there was no clone path — and was then briefly implemented as a clone-to-box, which violated the local-only policy. Both are gone.)

### Exit codes (orchestrator)

| Code | Meaning |
|---|---|
| `0` | Everything the run attempted succeeded, **and every configured remote workspace was updated**. |
| `1` | Usage / setup error (unknown flag, engine missing, run from the box). |
| `3` | Local host has blocking drift (`divergent`/`broken-overlay`) — nothing applied. Resolve, re-run. |
| `4` | Local + origin updated, but **one or more remote workspaces were SKIPPED** (unreachable). Their projects' drift state is UNKNOWN, not clean. Re-run after resolving. |

**Why `4` exists.** A deploy that reached zero remotes used to be indistinguishable from a complete one: the skip printed two indented lines mid-scrollback, then the run printed `=== complete ===` and exited `0`. The loop also ran in a pipeline subshell, so no failure state could have escaped it even if one had been set. The run now ends with an explicit `N of M updated, M skipped` tally and exits `4` on any skip — **a silent partial deploy is the failure this command must never produce**, because a box running stale defaults looks exactly like a box running current ones.

## Cross-reference: refresh-remote skill + runtime

`syndicate-refresh-remote.md` (one of the distributed defaults) is the markdown front-end for the `syndicate-refresh-remote` CLI. The CLI is **not** distributed by this command — it lives in `~/syndicate-remote/scripts/syndicate-refresh-remote.sh`, installed per-machine via `~/syndicate-remote/scripts/install.sh` (binary at `~/.local/bin/`), configured by `~/.syndicate-remote-secrets/box.json`. To bump the skill markdown: edit the canonical and run `/distribute-defaults`. To bump the CLI: edit the shell source in `syndicate-remote` and re-run its installer.
