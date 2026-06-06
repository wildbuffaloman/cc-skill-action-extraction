---
name: correction-capture
description: "Diff a triage skill's proposed dispositions against the user's actual choices, log corrections to a per-skill ledger, codify clear property→target mappings into a learned-overrides store (auto) or propose ambiguous ones, and surface what was learned — closing the self-improvement loop for inbox-clear / read-review / email-triage and other manifest-or-classification triage skills"
version: 2
---

Capture the gap between what a triage skill **proposed** and what the user **chose**, then turn recurring corrections into fast-path rules the skill reads on its next run. This is the missing wire in the vault's learning loop: the proposed-vs-chosen diff is produced every execution and otherwise discarded.

Implements the active, deterministic side of [[Memory on Triage]] — instead of *hoping* a skill saves a rule when the user corrects it, the consuming skill calls this node at the end of every execution phase.

## Design contract

The node is **skill-agnostic**. The consuming skill is responsible for translating its own correction surface (a manifest's Approve column, a Living Index, a conversational reclassification) into a **normalized correction list**. The node owns everything after that: ledger, tiering, scope routing, writing, surfacing.

This is what lets a manifest-based skill (`inbox-clear`, `read-review`) and a conversational skill (`email-triage`) share one node.

## Per-skill data files

Two files live in the consuming skill's directory (`05 AI/CLAUDE CODE/skills/<skill>/`):

- **`_corrections-log.md`** — append-only, pipe-delimited ledger (raw signal; grep-friendly; mirrors `read-review/_routing-log.md`).
- **`learned-overrides.md`** — curated fast-path rules the skill's classification step reads **before** its decision tree (the behavior change; low-volume; loaded into agent prompts).

Neither carries `category:` frontmatter — both are skill-infra data files, not vault content notes. Bootstrap with a header comment on first write.

Cross-skill rules additionally write a global `feedback_*.md` memory + `MEMORY.md` pointer per [[Memory on Triage]].

### Ledger schema (`_corrections-log.md`)

```
# <skill> Corrections Log — appended by correction-capture (mode: capture). Rows >1yr → _corrections-log-archive.md.
# date | item | proposed | chosen | destination | signal | codified
2026-05-28 | [[Morning Briefing 2026-05-28]] | ROUTE TO READ_REVIEW | ROUTE TO REFERENCE | 03 REFERENCE/PERSONAL/LIFE OS/DAILY NOTES/ | disposition-override | auto
2026-05-28 | [[Valdeagua quote]] | ROUTE TO READ_REVIEW | ROUTE TO AREA | 02 AREAS/06 BUSINESS/ | disposition-override | proposed
```

- `signal ∈ { disposition-override, destination-override, reclassification, rejection, draft-voice, auto-action-rollback }`
- `codified ∈ { auto, proposed, no }`
- `auto-action-rollback` (inbox-clear Phase 1.5): the user rolled back an auto-executed move. `proposed` = the auto-disposition that fired, `chosen` = INBOX (or the `{@ ROLLBACK to:}` override). Ledger-only per run; ≥2 rollbacks of the same allowlist rule at consolidation → propose tightening/removing that rule.

### Learned-overrides schema (`learned-overrides.md`)

```
# <skill> Learned Overrides — read by classification BEFORE the decision tree.
# Auto-written by correction-capture; promote/edit via /retro consolidation. Remove an entry to revert.

### filename ~ "morning briefing" | "daily note" | "startup day"
- Apply: ROUTE TO REFERENCE → `03 REFERENCE/PERSONAL/LIFE OS/DAILY NOTES/`
- Scope: cross-skill → see [[feedback_inbox_routing_rules]]
- Provenance: learned 2026-05-28 from [[Morning Briefing 2026-05-28]] · instances: 4 · tier: auto
- Status: active
```

- `tier ∈ { auto, proposed, seed }` (`seed` = migrated from a skill's prior manual overrides)
- `Status ∈ { active, provisional, conflict }`

## mode: capture

**Used by:** the consuming skill at the end of its execution phase (inbox-clear Phase 2, read-review `--execute`, email-triage Step 8).
**Model:** runs in the skill's main conversation (judgment-bearing; not a mechanical sub-agent task).
**Interactive:** No — writes auto-tier rules silently and surfaces them; proposes the rest.

### Inputs (the skill provides)

- `skill` — selects the data-file paths.
- `corrections[]` — normalized list; each item:
  - `item` — the note/email/row (wikilink where applicable)
  - `proposed` — the disposition/classification the skill proposed
  - `chosen` — what the user actually chose
  - `destination` — chosen target path/handling (optional)
  - `signal` — one of the signal values above
  - `generalizable_key` — optional property the rule keys on (e.g. `domain:mpps.gob.ve`, `filename~"morning briefing"`, `source-type:x.com`, `sender:tonyfrangie@substack.com`). If absent, the node attempts to infer one; if none is inferable, the correction is treated as one-off (ledger-only).
- `report` — the run's report/section to append surfacing blocks to.

### Process (per correction)

1. **Append a ledger row** — always, regardless of what follows.
2. **One-off test** ([[Memory on Triage]] scope test): would this rule save effort or prevent a mistake *next* time? If clearly one-off (typo, single-note quirk, no generalizable key) → `codified: no`, stop.
3. **Rejection handling:** `signal = rejection` → ledger-only, `codified: no`. Never auto-codified (rejection is ambiguous — "not now" vs. "wrong"). Recurrence is handled at consolidation.
4. **Tier test:**
   - Deterministic *property→target* mapping (a source property — domain / filename pattern / frontmatter category / source-type / sender — maps cleanly to a destination / disposition / classification) → **AUTO**.
   - Content-semantic or judgment call → **PROPOSE**.
5. **Scope test** (per [[Memory on Triage]] + retro Gate-2 scope): would this rule apply across triage skills, or only this one?
   - Routes a *source property* to a vault location/handling any triage skill would face (e.g. "morning briefings → DAILY NOTES", "MPPS senders are delegated") → **cross-skill**.
   - Tied to this skill's own dispositions/columns/surface → **skill-local**.
6. **Apply:**
   - **AUTO + skill-local** → write a rule block to `learned-overrides.md`. Add an undo line to the report.
   - **AUTO + cross-skill** → write a `feedback_*.md` memory + `MEMORY.md` pointer; write a thin pointer rule in `learned-overrides.md` referencing the memory. List under "Memory Updates (saved this session)".
   - **PROPOSE** (either scope) → emit a `[x]`-able row in the report's "Proposed Learnings" block; mark ledger `codified: proposed`. Applied on the user's next run or an explicit apply pass.
7. **Conflict guard:** if an AUTO write would contradict an existing `active` rule (same key, different target), do **NOT** overwrite. Write the new rule as `Status: conflict` and surface it for the user to resolve. Prevents thrashing between runs.

### Output (append to the run's report)

```markdown
### 🧠 Learned This Run
**Auto-applied (skill-local):**
- `filename ~ "morning briefing"` → ROUTE TO REFERENCE · DAILY NOTES — from [[note]]
  · undo: remove this entry from <skill>/learned-overrides.md
**Proposed (tick [x] to apply next run):**
- [ ] `domain:valdeagua.com` → ROUTE TO AREA · 02 AREAS/06 BUSINESS/ — from [[note]] (judgment call)
**Conflicts (resolve manually):**
- `source-type:x.com` proposed → REFERENCE but active rule says READ_REVIEW — kept active; see learned-overrides.md

### 💾 Memory Updates (saved this session)
- feedback_morning_briefings_daily_notes.md (cross-skill) — + MEMORY.md pointer
```

If `corrections[]` is empty: append `_No corrections this run — 0 learned._` and touch no stores.

## mode: consolidate

**Used by:** `/retro` via the "Triage Correction Consolidation" step in [[vault-self-improvement-protocol]] (and therefore `/session-close`'s retro evaluation).
**Model:** sonnet.
**Interactive:** Yes — present a consolidation manifest; apply only on user approval (retro rule: never auto-apply).

### Process (per consuming skill)

1. Read `<skill>/_corrections-log.md`.
2. Group rows by `(signal, normalized-pattern)`.
3. **Promote:** any group with `codified: no` or `codified: proposed` and **count ≥ 2** → propose codification into `learned-overrides.md` (skill-local) or a `feedback_*.md` memory (cross-skill) per the scope test. This is the safety net that catches what the per-run capture left as PROPOSE or one-off-but-now-recurring.
4. **Rejections:** recurring rejected-proposal patterns (≥2) → surface for a manual decision ("stop proposing X?"). Never auto-applied.
5. **General learnings** (not routing — workflow/tooling insight) → route to `lessons.md` / `decision-traces` via [[migrate-lessons]].
6. **Housekeeping:** move ledger rows older than 1 year to `<skill>/_corrections-log-archive.md`. Flag `learned-overrides.md` entries never matched in N runs as stale (advisory only).
7. Present the consolidation manifest (→ INBOX). Apply approved items; update ledger `codified` flags.

### Output (consolidation manifest, → INBOX)

```markdown
## Triage Correction Consolidation — YYYY-MM-DD

### Promotions (≥2 recurrences, un-codified)
| Approve | Skill | Pattern | Proposed Rule | Scope | Recurrences |
|---------|-------|---------|---------------|-------|-------------|
| [ ] | inbox-clear | domain:valdeagua.com | → ROUTE TO AREA · BUSINESS | skill-local | 3 |

### Recurring Rejections (manual decision)
| Approve | Skill | Proposal repeatedly rejected | Recurrences | Suggested action |
|---------|-------|------------------------------|-------------|------------------|

### General Learnings → lessons.md / decision-traces
- ...

### Housekeeping
- Archived N ledger rows >1yr · Stale learned-overrides flagged: ...
```

## Usage notes

- The consuming skill loads `learned-overrides.md` and applies matching rules **before** its decision tree (highest-priority override). State this explicitly in the skill's classification agent prompt.
- Capture runs in the skill's **main conversation**, not a mechanical sub-agent — tiering and scope routing are judgment calls.
- Auto-writes are silent but always surfaced with an undo path — the user is never surprised by a behavior change they can't see or reverse.
- When bundled for sharing, copy this node into the consuming skill's `_bundled/nodes/` per [[Shared Node Bundling]]; the per-skill data files (`_corrections-log.md`, `learned-overrides.md`) are runtime artifacts, not bundled.

## Consumers

**Tier 1** (manifest/conversational, full surface):
- [[inbox-clear]] — mode: capture (Phase 2), consolidate (retro)
- [[read-review]] — mode: capture (`--execute`), consolidate (retro)
- [[email-triage]] — mode: capture (Step 8, conversational), consolidate (retro)

**Tier 2** (rolled out 2026-05-28):
- [[reminders-sync]] — capture (Phase 2 Step 9; destination-override on placement)
- [[followup]] — capture (Phase 2; row-by-row disposition/destination/reclassification)
- [[clean-projects]] — capture (Phase 2; reclassification/rejection — instance-specific re-parents stay ledger-only)
- [[clean-areas]] — capture (Phase 2; disposition/reclassification on linking — mostly ledger-only)
- [[clean-reference]] — capture (Phase 2; destination-override — seeded with the archive-default rule)
- [[meeting-agenda]] — capture (Step 4.5; harvests inline `{@ ...}` comments on recurring agendas. Extends the signal set with `content-correction` / `content-add` / `generation-instruction` / `fact` / `one-off`; `generation-instruction` codifies to learned-overrides, cross-cutting `fact` enriches the vault AREA/wiki source of truth, rolled out 2026-06-06)
- [[meeting-minutes]] — capture (Step 6.5; harvests inline `{@ ...}` comments on existing minutes. Same signal set as meeting-agenda plus `action-reroute` (corrects a Next-Action/Waiting-For owner before `/action-extraction` propagates it). Learned `generation-instruction`s are preloaded by Step 4 so future minutes inherit corrections; `/action-extraction` is the recommended automatic harvest trigger, rolled out 2026-06-06)

**Tier 3** (rolled out 2026-05-28):
- [[startup-day]] — capture (Step 5 Q1 override; daily picks ledger-only, only recurring task→priority patterns codify)
- [[close-day]] — capture (Phase 2; proposed-creation rejection is the codifiable surface, Q1/Deep-Work picks ledger-only)
- [[task-sync]] — capture (Phase 2; conflict disposition-override — deterministic skill, value is the ledger + consolidation)
- [[action-extraction]] — capture (Step 4 INTERACTIVE mode only; skipped in cron; post-run brief-edit reroutes are out of scope — detached-surface limitation noted in-skill)

All Tier 2/3 consumers also run mode: consolidate via /retro.

**Deferred:**
- aiac-triage — lives in the `aiac-pipeline` plugin (separate repo/lifecycle), not core `skills/`. Wiring deferred to a plugin-scoped ship.
