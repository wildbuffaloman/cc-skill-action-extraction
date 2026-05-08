---
name: action-extraction
version: "0.2.0"
description: "Propagate action items, waiting-for items, and decisions from today's meeting minutes into the relevant project/program briefs. Vault-only — no external notifications."
user-invocable: true
argument-hint: "optional: 'cron' for non-interactive scheduled run"
---

Take the meeting minutes created today by `meeting-minutes` and fan the structured outputs (Next Actions, Waiting For, Decisions) into the project/program briefs each meeting relates to. This is the handoff step between "capture" and "GTD pipeline" — without it, minutes rot in INBOX.

## Philosophy

- **Minutes are the source.** Do not re-read the Granola transcript. The `meeting-minutes` skill already extracted structured outputs; this skill routes them.
- **Briefs are the destination.** Each meeting maps to zero, one, or more project/program briefs. Actions land in the matching brief's `## Next Actions`; waiting-for items land in `## Waiting For`; decisions go to `## Log`.
- **No notifications.** Vault-only. No Discord, no Slack, no Telegram.

## Vault Exception

This skill MAY modify existing files in `01 PROJECTS/` and `02 AREAS/` — specifically project/program briefs. It appends to the Next Actions, Waiting For, and Log sections. It does not delete or rewrite existing content.

It does NOT:
- Create new briefs
- Modify meeting minutes files themselves
- Touch anything outside `01 PROJECTS/` and `02 AREAS/`

## Inputs

- **No argument** — interactive mode. Present candidate updates per meeting and ask for confirmation before each brief edit.
- **"cron"** — non-interactive. Execute updates automatically using the matching heuristic below. No user prompts.

## Cron Mode

When invoked with `cron`, the skill runs without any user interaction.

### Hard Rules (cron mode only)

1. **Today-only source set.** Process only meeting minutes files in `00 HUB/00 INBOX/` whose filename date suffix matches today (`YYYY-MM-DD`). Older minutes are skipped.
2. **Brief-match confidence floor.** A brief is updated only when the meeting→brief match hits ≥70% confidence via the Matching Heuristic. Ambiguous matches are logged and skipped, not resolved interactively.
3. **Idempotency.** Before appending, check whether the action item text already exists in the target brief's Next Actions / Waiting For. Skip exact-string duplicates.
4. **No new briefs.** If a meeting matches no existing brief above the threshold, the meeting is logged as "unrouted" and skipped — never spawn a new brief.
5. **No notifications.** Cron mode writes a terminal-only summary.

## Workflow

### Pre-flight: Synthesis-Status Check

Before parsing any Meeting Minutes file as input, check its frontmatter for `synthesis_status`. The `/meeting-minutes` skill writes this flag to indicate whether the LLM extraction step (Action Items / Waiting For / Decisions / Summary synthesis) ran to completion.

**Behavior:**

| `synthesis_status` value | Action |
|--------------------------|--------|
| `complete` | Proceed normally — file is a valid input. |
| `pending` (or any non-`complete` value) | **Refuse** — emit a clear error: `"synthesis incomplete for [[<wikilink>]] — re-run /meeting-minutes <granola_id> first, then re-invoke /action-extraction"` and skip the file. |
| Frontmatter field absent | Treat as `complete` for backward compatibility (legacy files written before the flag was introduced) — but log a soft warning recommending the user rerun `/meeting-minutes` if the file's `## Next Actions` section is suspiciously empty (e.g., literal `- None` with a 100KB+ raw transcript in `## Summary`). |

**Why this matters:** without this gate, `/action-extraction` cannot distinguish "synthesis incomplete (extraction step rate-limited)" from "synthesis ran cleanly and the meeting genuinely had no actions." Both produce a file with empty Next Actions / Waiting For. Treating the first case as the second silently drops every action item from a rate-limited meeting and leaves the project briefs out of sync with reality.

**Cron mode:** in cron, log refused files to the unrouted summary alongside `synthesis_status` reason. Continue processing remaining valid files. Do NOT crash or block the rest of the run.

**Evidence:** 2026-04-29 `/close-day` Phase 2E.4 — `/meeting-minutes` cron was rate-limited mid-synthesis, leaving `00 HUB/00 INBOX/Meeting Minutes — Gestion y Planificación — 2026-04-29.md` with raw transcript in `## Summary` and `## Next Actions: - None`. `/action-extraction` correctly refused (philosophy: "Minutes are the source. Do not re-read the Granola transcript") but the refusal was based on inferring malformed structure rather than reading an explicit flag. The synthesis_status gate makes this future-proof.

### Step 1 — Gather Today's Minutes

1. Resolve the vault root from the CLAUDE.md chain.
2. List files in `00 HUB/00 INBOX/` matching `Meeting Minutes — * — YYYY-MM-DD.md` where the date suffix matches today.
3. If zero files found, write a one-line summary and exit.

### Step 2 — Parse Each Minutes File

For each file, extract the structured sections:
- **Frontmatter** — `AREA`, `SUB-AREA`, `attendees`, `agenda` (if set), `title`.
- **Summary** — one paragraph.
- **Next Actions** — the `- [ ] ... — @Owner` items under `## Next Actions`.
- **Waiting For** — the rows under `## Waiting For` (Owner / Action / Due).
- **Decisions** — under `### Decisions` inside `## Meeting Notes`.
- **Linked agenda** — the `agenda:` frontmatter value, if set.

### Step 3 — Match Each Meeting to Briefs (Matching Heuristic)

For each meeting, find the best-matching project or program brief. Score candidates 0-100:

1. **Agenda linkage (strongest signal, +60)** — If `agenda:` frontmatter is set, read the agenda and check its `## Key References` for project/program wikilinks. Each linked brief is a direct candidate.
2. **Title/content keyword match (+20 to +40)** — Compute word-set overlap between meeting title + summary and the brief's title + outcome. Normalize (lowercase, strip stop words).
3. **AREA / SUB-AREA frontmatter match (+10 to +20)** — If the meeting's `AREA` and `SUB-AREA` match a brief's AREA and SUB-AREA frontmatter.
4. **Attendee match (+10)** — If 2+ meeting attendees appear in a brief's Stakeholders section.

**Confidence floor:** 70. Ambiguous matches (below 70) are flagged and skipped in cron mode.

**Multi-match:** A meeting can update multiple briefs if several score ≥70. Route each action item / waiting-for item to the brief it most directly relates to (based on text analysis).

### Step 4 — Propagate to Briefs

**Ownership split at propagation** (per [[Brief Template Compliance]] convention, added 2026-04-21): split each minutes item by owner before routing.

- Items where I (@Alberto) am **at least one of the owners** — including mixed ownership like `@Alberto / @Alondra` — route to the target brief's `## Next Actions`.
- Items assigned **fully to others** (e.g., `@Alondra`, `@JC Zerpa / @Philippe`) route to the target brief's `## Waiting For` table, with the responsible person populated in the Owner column.
- **Never** place @others-only items in Next Actions — this is a systematic bug that caused ~90% misclassification in the 2026-04-21 Promocion Bufalinda propagation before the rule was codified.

For each matched brief:

**`## Next Actions`** — append new action items from the minutes. Format:
```
- [ ] {action text} — @{owner} — [[Meeting Minutes — TITLE — YYYY-MM-DD]]
```
The wikilink to the source minutes is MANDATORY — it follows the "Actionable output navigation" rule from the vault CLAUDE.md.

**`## Waiting For`** — append new waiting-for rows. Format:
```
| @{owner} | {action} | {due-date-or-blank} | [[Meeting Minutes — TITLE — YYYY-MM-DD]] |
```

**`## Log`** — append a dated entry summarizing the meeting outcome. Format:
```
- {YYYY-MM-DD} — Meeting: {Meeting Title} — {N} actions added, {N} WF items, {N} decisions. See [[Meeting Minutes — TITLE — YYYY-MM-DD]]
```
One entry per meeting per brief, never more.

### Step 5 — Idempotency Check

Before appending to Next Actions or Waiting For, scan the target section for exact-string duplicates. Skip any item whose action text already exists in the brief.

### Step 6 — Handle Unmatched Items

Actions that can't be routed to any brief (no match ≥70) are reported in the terminal summary as "unrouted actions." They remain in the minutes file — the user can route them manually during their next review.

### Step 7 — Report

Terminal-only summary:

```
═══════════════════════════════════════════════════════════
 ACTION EXTRACTION — YYYY-MM-DD
═══════════════════════════════════════════════════════════

 Meetings processed: N
 ─────────────────────────────────────────────────────────
 • {Meeting Title} → {Brief Name}: +{N} actions, +{N} WF, +{N} decisions
 • {Meeting Title} → {Brief Name}: +{N} actions, +{N} WF
 • {Meeting Title} → UNROUTED (no brief match ≥70)

 Briefs updated: N
 ─────────────────────────────────────────────────────────
 [[Brief Name]] — +{N} actions, +{N} WF
 [[Brief Name]] — +{N} actions

 Flags: N unrouted actions, N unowned actions
═══════════════════════════════════════════════════════════
```

## Rules

### Boundary Rules
- **Read-only on minutes files.** Never modify files in `00 HUB/00 INBOX/`.
- **Append-only on briefs.** Never delete or rewrite existing Next Actions, WF items, or Log entries. Only add new rows.
- **Never create briefs.** If no match, flag and skip — do not spawn new files.
- **Preserve formatting.** Match the existing indentation, bullet style, and table format of each target brief.

### Matching Rules
- **Confidence floor: 70.** Below threshold → log as unrouted, skip.
- **Agenda linkage wins.** If a matching agenda exists and it links to a brief, that brief is always a ≥70 match.
- **No fuzzy re-matching.** If a minutes file has been processed before (Log entry exists in target brief referencing the same `[[Meeting Minutes — ...]]` wikilink), skip.

### Output Rules
- **Every added item includes a wikilink to the source minutes** — required by the vault's "Actionable output navigation" convention.
- **Terminal-only.** No Discord, Telegram, Slack, or INBOX report. Briefs and Log entries are the durable output.

### Granularity Preservation Rule (added 2026-04-30)

When a meeting's outputs propagate across **brief + tracking sheet + minutes** (or any analogous multi-artifact governance system), treat the three artifacts as **mutually-projecting views** of the same work at different granularities — never normalize them to identical row counts.

| Artifact | Granularity | Optimized for |
|----------|-------------|---------------|
| Project / Program brief | Executive summary — consolidate by person | Sponsor / CEO / area lead reading |
| Team-facing tracking sheet | Operational tracking — separate by OKR slot (multiple rows per person OK) | Area lead + individual contributor weekly check-ins |
| Meeting minutes | Source of truth — full Ownership Split per item | Auditability + downstream extraction |

**Behavior:**
- When propagating to a brief, consolidate per-person items even if the source had multiple per-person entries split by OKR slot.
- When propagating to a sheet, preserve the OKR-slot granularity — don't collapse to the brief's row count.
- When the user reports "row X is missing in the brief but present in the sheet" or vice versa, **don't auto-realign** — verify the divergence is intentional granularity (likely) before changing anything.

**Evidence:** 2026-04-30 G&P trimestral session — synthesizer over-eagerly normalized the brief's Q2 table to match the meeting minutes' 13-row ownership-split detail (including responsables who didn't have OKR-tracked Q2 commitments yet). The user had to correct twice: first to align brief to canonical V3 (11 rows), then to add back two members with proper Q2 task assignments. The right mental model was "three projecting views" not "three copies of the same table."

### Cron Mode
- Prefaced with "CRON MODE — execute immediately, no confirmation needed." in the trigger payload.
- Non-interactive. All ambiguity resolves to "skip + flag," never "ask user."
- Runs after `meeting-minutes` in the daily pipeline. Depends on `meeting-minutes` having completed its run.

## Dependencies

Shared capabilities from `_shared/nodes/` (resolve via `{{VAULT_ROOT}}/05 AI/CLAUDE CODE/skills/_shared/nodes/{name}.md`):
- [[brief-updater]] — mode: task-insert, wf-update, log-entry

## Related Skills
- [[meeting-minutes]] — upstream: creates the minutes files this skill consumes.
- [[followup]] — downstream: picks up the Waiting For items we add.
