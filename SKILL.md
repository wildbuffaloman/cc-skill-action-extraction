---
name: action-extraction
version: "0.5.1"
description: "Propagate action items, waiting-for items, and decisions from today's meeting minutes into the relevant project/program briefs. Cron: vault-only. Interactive: may also draft (never send) personalized minutes emails per participant with their decisions, assigned tasks, and Trello card links. v0.5.0: Step 1.5 delegates `{@ ...}` comment harvesting to /meeting-minutes harvest on today's minutes BEFORE routing, so action-reroute corrections apply before propagation."
user-invocable: true
argument-hint: "optional: 'cron' for non-interactive scheduled run"
---

Take the meeting minutes created today by `meeting-minutes` and fan the structured outputs (Next Actions, Waiting For, Decisions) into the project/program briefs each meeting relates to. This is the handoff step between "capture" and "GTD pipeline" — without it, minutes rot in INBOX.

## Philosophy

- **Minutes are the source.** Do not re-read the Granola transcript. The `meeting-minutes` skill already extracted structured outputs; this skill routes them.
- **Harvest corrections before routing.** Before parsing, hand today's minutes to `/meeting-minutes harvest` (Step 1.5) so the user's inline `{@ ...}` comments — especially `action-reroute` ("this is JC's not mine") — are applied first. Routing the *corrected* minutes prevents propagating an action to the wrong owner or brief, and lets durable corrections feed memory + AREA files.
- **Briefs are the destination.** Each meeting maps to zero, one, or more project/program briefs. Actions land in the matching brief's `## Next Actions`; waiting-for items land in `## Waiting For`; decisions go to `## Log`.
- **No notifications in cron.** Cron mode is vault-only: no Discord, no Slack, no Telegram, no email. **Interactive mode** MAY additionally create Gmail **drafts** (never sends) for participants — see Step 7. The draft path is gated behind explicit per-meeting user selection and never runs in cron.

## Vault Exception

This skill MAY modify existing files in `01 PROJECTS/` and `02 AREAS/` — specifically project/program briefs. It appends to the Next Actions, Waiting For, and Log sections. It does not delete or rewrite existing content.

It does NOT directly:
- Create new briefs
- Edit meeting minutes files with its own hands — **but** in Step 1.5 it delegates `{@ ...}` comment harvesting to `/meeting-minutes harvest`, which may edit today's minutes (apply + remove resolved comments) and additively enrich AREA/wiki files. The minutes edit is owned by `/meeting-minutes`; action-extraction only invokes it.
- Touch anything outside `01 PROJECTS/` and `02 AREAS/` (except via the delegated Step 1.5 harvest above)

## Inputs

- **No argument** — interactive mode. Present candidate updates per meeting and ask for confirmation before each brief edit.
  Interactive mode also offers Step 7 (Participant Minutes Emails) — per-meeting attendee selection and Gmail draft creation.
- **"cron"** — non-interactive. Execute updates automatically using the matching heuristic below. No user prompts.
  Cron NEVER drafts emails or touches Trello — Steps 4.5 and 7 are interactive-only.

## Cron Mode

When invoked with `cron`, the skill runs without any user interaction.

### Hard Rules (cron mode only)

1. **Today-only source set.** Process only meeting minutes files in `00 HUB/00 INBOX/` whose filename date suffix matches today (`YYYY-MM-DD`). Older minutes are skipped.
2. **Brief-match confidence floor.** A brief is updated only when the meeting→brief match hits ≥70% confidence via the Matching Heuristic. Ambiguous matches are logged and skipped, not resolved interactively.
3. **Idempotency.** Before appending, check whether the action item text already exists in the target brief's Next Actions / Waiting For. Skip exact-string duplicates.
4. **No new briefs.** If a meeting matches no existing brief above the threshold, the meeting is logged as "unrouted" and skipped — never spawn a new brief.
5. **No notifications.** Cron mode writes a terminal-only summary.
6. **Harvest before routing (Step 1.5).** Run the `{@ ...}` harvest on today's `synthesis_status: complete` minutes before Step 2, delegating to `/meeting-minutes harvest`. Apply clear directives + additive AREA enrichments automatically; surface ambiguous/structural/conflicting ones in the terminal summary. Never prompt. This guarantees `action-reroute` corrections apply before propagation even in the unattended `/close-day` Phase-Chain run.

> **Chained execution context (added 2026-05-27 per [[2026-05-27-cloud-cron-migration-v2-redirected]]):** When invoked from `/close-day` Phase Chain Phase C, this skill runs immediately after Phase A's `/meeting-minutes cron` writes new minutes to `00 HUB/00 INBOX/`. The existing Rule 1 ("Today-only source set") already handles this correctly — the freshly-written INBOX files match today's date suffix and are picked up by Step 1 naturally. No behavior change needed for the Phase Chain integration; this note exists for traceability so future maintainers see the dependency.

## Learned Overrides

Stable routing preferences learned from user corrections live in this skill's **`learned-overrides.md`**, managed by the [[correction-capture]] node. The matching step (Step 3) MUST load that file and apply any matching rule **before** the Matching Heuristic scores candidates — a matching learned override is the highest-priority routing decision. This load applies in **both interactive and cron mode**: applying learned overrides improves the automatic match too.

The file grows automatically from **in-run interactive overrides** — when the user reroutes a proposed meeting→brief assignment or re-owns an action item (Next Actions ↔ Waiting For) during the interactive run, Step 4's Correction Capture diffs proposed-vs-chosen and codifies clear corrections. This is the active form of [[Memory on Triage]].

> **Detached-surface limitation:** post-run brief-edit reroutes are out of scope for this capture; catching them would require a separate post-hoc brief-diff mechanism (deferred design — not wired here). The user most often corrects action-extraction *after* the run by editing a brief to move or re-own an action item days later, and an end-of-run capture cannot see those edits. The wired surface is the load step (both modes) plus in-run interactive overrides only.

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

### Step 1.5 — Harvest `{@ ...}` Comments on Today's Minutes (pre-routing)

Before parsing, run the `/meeting-minutes` harvest over today's minutes so the user's inline `{@ ...}` corrections are applied **first** — this is the automatic daily trigger for `/meeting-minutes` Step 6.5. The point is ordering: an `action-reroute` comment (`{@ this is JC's not mine}`, `{@ goes to the Cusica project}`) must change the item's owner/target **before** Step 4 propagates it, or the action lands in the wrong brief.

1. **Scope to today's complete minutes only.** From the Step 1 file list, keep those with `synthesis_status: complete` (or absent — legacy) per the Pre-flight check; never harvest a `pending` stub.
2. **Skip if no comments.** Grep the set for `{@` — if none contain a comment, skip this step silently (the common case; near-zero cost).
3. **Delegate the harvest.** For the files that do contain `{@ ...}` spans, invoke `/meeting-minutes harvest` scoped to today's minutes (the action-extraction-invoked path in `/meeting-minutes` Harvest Mode / Step 6.5). It applies each comment to the minutes file, re-owns/re-routes action & waiting-for items per the ownership-split rule, additively enriches AREA/wiki files with `fact`-intent comments, captures the learnings via [[correction-capture]] (writing `meeting-minutes/learned-overrides.md` + ledger), and removes each resolved comment. **action-extraction does not edit the minutes itself** — it only invokes the harvest (see Vault Exception).
4. **Auto/surface split.** Headless (cron / Phase-Chain): auto-apply clear directives + additive enrichments, surface the rest in this run's summary — never prompt. Interactive: `/meeting-minutes` surfaces ambiguous comments for the user.
5. **Then re-read.** Step 2 parses the **post-harvest** minutes, so the routed actions reflect the corrected owners/targets.

> **Why before Step 2, not after Step 4:** routing first and correcting later would propagate the wrong owner into a brief, then require a second pass to undo it. Harvesting up front makes the corrected minutes the single source the rest of the run reads. Comments left unresolved (surfaced) stay in the minutes file for the next interactive pass.

### Step 2 — Parse Each Minutes File

For each file, extract the structured sections:
- **Frontmatter** — `AREA`, `SUB-AREA`, `attendees`, `agenda` (if set), `title`.
- **Summary** — one paragraph.
- **Next Actions** — the `- [ ] ... — @Owner` items under `## Next Actions`.
- **Waiting For** — the rows under `## Waiting For` (Owner / Action / Due).
- **Decisions** — under `### Decisions` inside `## Meeting Notes`.
- **Linked agenda** — the `agenda:` frontmatter value, if set.

### Step 3 — Match Each Meeting to Briefs (Matching Heuristic)

0. **Apply learned overrides** (both modes) — Load `action-extraction/learned-overrides.md`. If a rule matches this meeting (by meeting title / attendee → brief, or by action-item phrasing → Next Actions vs Waiting For ownership), apply it at **HIGH** confidence (treated as ≥70, clearing the confidence floor) and record `Learned Override: <rule>` alongside the routing decision. The Matching Heuristic below only scores candidates the loaded rules do not already resolve. This load runs in **both interactive and cron mode** — learned overrides improve the automatic match too.

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

#### Step 4 — Correction Capture (interactive mode only)

**`if mode == cron: skip this step entirely.`** In cron/automatic mode there is no manifest and nothing to diff — routing was automatic — so Correction Capture does NOT run. The step is interactive-only; it only fires when the user approved/edited per-brief assignments in this run.

This is what makes the skill learn. After the user has confirmed all per-brief assignments (and any re-ownership of action items), build a normalized `corrections[]` by diffing each routed item's **proposed** disposition (the meeting→brief match + Next Actions/Waiting For ownership the skill proposed in Step 3/4) against the **chosen** disposition (what the user actually approved):
- User routed an action item to a different brief than the heuristic proposed (or resolved an ambiguous <70 match the skill would have skipped) → `signal: destination-override` (`destination:` = chosen brief path).
- User moved an item across the ownership split — Next Actions ↔ Waiting For — overriding the proposed owner placement → `signal: reclassification`.
- Assignment accepted as proposed → confirmation; NOT logged.

Set `generalizable_key` where one is inferable (e.g. `meeting-title~"..."`, `attendee:<name>`, `action-phrasing~"..."`).

Then invoke the [[correction-capture]] node (`mode: capture`) with `skill: action-extraction` and this list. It appends rows to `action-extraction/_corrections-log.md`, auto-applies clear meeting/attendee→brief and action-phrasing→ownership rules to `action-extraction/learned-overrides.md` (and writes a `feedback_*.md` memory + MEMORY.md pointer for cross-skill rules), proposes ambiguous ones, and returns "🧠 Learned This Run" + "💾 Memory Updates" blocks. Append both to the Step 8 report. Auto-applied rules change the **next** run's Step 3 routing immediately — including the next cron run, since Step 3's load step is mode-agnostic.

> **Detached-surface limitation (repeated for the maintainer reading this step):** this capture sees only in-run interactive overrides. Post-run brief-edit reroutes — the user moving or re-owning an action item by editing a brief days later — are NOT visible here and are out of scope; catching them would require a separate post-hoc brief-diff mechanism (deferred — not wired).

### Step 4.5 — Trello Card Linking (interactive only)

**Skipped in cron mode.** For each matched brief that has Trello, resolve and link
(or create) a card per owned action item so Step 7 can include card links.

1. **Resolve the board.** Scan the matched brief's body for a Trello board URL of
   the form `https://trello.com/b/<boardId>/...` (briefs store the board as a
   markdown link, e.g. `[Trello Board CxC parent](https://trello.com/b/txNqZkxM/...)`).
   - If found, extract `<boardId>` and call `mcp__trello__set_active_board` with it.
   - If the brief has **no** board URL, ask the user once: which board (paste a
     board URL/id) or skip Trello for this meeting. Never guess a board.
2. **Per owned action item** (from the Step 4 Ownership Split):
   1. List the board's cards (`mcp__trello__get_lists` → `mcp__trello__get_cards_by_list_id`,
      active lists only per [[Trello List Filtering]] — skip Backlog/Listo/Cerrado/OKR/Recursos).
   2. **Fuzzy-match** the action text against card names (lowercase, strip stop
      words, ≥60% token overlap). If a card matches → record its
      `https://trello.com/c/<shortLink>` URL.
   3. **No match** → create a card (`mcp__trello__add_card_to_list` on the brief's
      default "in-progress"/"Doing" active list; title = action text; due = the
      item's due date if present). Record the new card's URL.
   4. **Assign the owner:** `mcp__trello__get_board_members` → match by name. If the
      owner is a board member → `mcp__trello__assign_member_to_card`. If NOT a board
      member → leave the card unassigned and flag it (`{owner} not on board`) for
      the Step 8 report.
3. Keep an in-memory map `{action_item → trello_url}` for Step 7. Cards are plain
   `https://trello.com/c/...` URLs (never wikilinks — they go into email bodies).

**Idempotency:** if a card already matches (step 2.2), do not create a duplicate.

### Step 5 — Idempotency Check

Before appending to Next Actions or Waiting For, scan the target section for exact-string duplicates. Skip any item whose action text already exists in the brief.

### Step 6 — Handle Unmatched Items

Actions that can't be routed to any brief (no match ≥70) are reported in the terminal summary as "unrouted actions." They remain in the minutes file — the user can route them manually during their next review.

### Step 7 — Participant Minutes Emails (interactive only)

**Skipped entirely in cron mode** (`if mode == cron: skip` — no recipient selection,
no drafts). This step creates Gmail **drafts** (never sends) on the Bufalinda account,
one per selected attendee, in Venezuelan Spanish (tuteo) per [[Venezuelan Spanish (tuteo)]].

**For each processed meeting:**

1. **Read decisions** from the minutes' `## Key Decisions` section (current
   meeting-minutes template). These are shared context included in every draft for
   that meeting.

2. **Resolve attendee emails.** For each `attendees` entry (strip `[[ ]]`, match by
   name — first name or full name): look up an email via
   **Vault Contacts (`03 REFERENCE/IMPORTANT DOCS/Vault Contacts.md`, Name/Last Name → Email column)
   → Google Contacts (gws) → Granola attendee metadata**. Attendees with no
   resolvable email are listed as `— no email — skipped`.

3. **Interactive selection.** Present the attendee list for this meeting. Pre-select
   attendees who own ≥1 action item (from the Step 4 Ownership Split). Pure observers
   are listed but unselected. The user confirms the selection. Build drafts only for
   selected, email-resolved attendees.

4. **Compose each draft** (plain text, no wikilinks, tuteo Spanish):

   ```
   Asunto: Minuta y tareas — {Meeting Title} — {YYYY-MM-DD}

   Hola {Nombre},

   Te comparto el resumen de lo que acordamos en {Meeting Title} ({fecha}) y las
   tareas que quedaron a tu cargo.

   Acuerdos clave
   • {decisión 1}
   • {decisión 2}

   Tus tareas asignadas
   • {acción 1} — vence {fecha o "sin fecha"} — {trello_url si aplica}
   • {acción 2} — ...

   ¿Confirmas recibido?

   Por favor responde a este correo confirmando que recibiste la minuta y que
   estás de acuerdo con los acuerdos y tus tareas asignadas. Si algo no refleja lo
   conversado, respóndelo aquí y lo corregimos.

   Gracias,
   Alberto

   —
   Borrador asistido por Claude Code
   ```

   - Tasks come from the owner's slice of the Ownership Split; Trello URLs from the
     Step 4.5 map. If the selected attendee owns no tasks, write
     `• (sin tareas asignadas en esta reunión)`.
   - If the meeting had no decisions, omit the `Acuerdos clave` section.

5. **Create the draft** (never send). Write the body to a temp file and call the
   shared helper in draft mode:

   ```bash
   python3 "{{VAULT_ROOT}}/05 AI/SHARED/scripts/send_email.py" \
       --draft \
       --to "{recipient_email}" \
       --subject "Minuta y tareas — {Meeting Title} — {YYYY-MM-DD}" \
       --body-file "{temp_body_path}" \
       --account bufalinda
   ```

   The helper builds RFC-2047/UTF-8/quoted-printable MIME (no mojibake) and
   verifies the created draft. **If the JSON result contains a `warnings` array,
   surface it and treat the draft as suspect** — do not report it as clean.

6. Collect per-draft results (draft id, recipient, task count, Trello cards touched,
   any warnings) for the Step 8 report.

**Boundary:** drafts only. This step NEVER calls `send`. The user reviews and sends
each draft from Gmail per [[Email Sending Protocol]].

### Step 8 — Report

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

 Comments harvested (Step 1.5): {A} applied, {S} surfaced
 ─────────────────────────────────────────────────────────
 • {Meeting Title}: {A} applied (corrections/action-reroutes/facts), {S} surfaced; AREA enriched: {file or —}

 Briefs updated: N
 ─────────────────────────────────────────────────────────
 [[Brief Name]] — +{N} actions, +{N} WF
 [[Brief Name]] — +{N} actions

 Flags: N unrouted actions, N unowned actions

 Participant drafts: {N} created, {M} skipped (no email)
 ─────────────────────────────────────────────────────────
 • {Nombre} <{email}> — {K} tareas, {T} Trello cards [draft: {draft_id}]
 • {Nombre} — SKIPPED (no email resolved)

 Trello: {created} cards created, {linked} linked, {unassigned} unassigned (not board member)

 ⚠ Draft warnings: {any encoding/threading warnings, or "none"}
═══════════════════════════════════════════════════════════
```

## Organization Card Integration

When extracted actions reference an organization (e.g., "Follow up with [Acme] on contract", "Send Q3 report to [Distribuidora XYZ]"), call `/organization-card` in **Silent Update mode only** — never Create or Quick Add. The action-extraction skill enriches existing org cards with new context (last_contact, related-to project wikilinks, relationship state) but does NOT propose new org cards. New-org discovery belongs to `/meeting-minutes` Step 4.5 and `/session-close` Phase 2.4. If the referenced org has no existing card, leave it unenriched — the next session-close cycle will catch it.

## Rules

### Boundary Rules
- **Read-only on minutes files (with its own hands).** action-extraction never edits files in `00 HUB/00 INBOX/` directly. The sole exception is Step 1.5, which *delegates* `{@ ...}` harvesting to `/meeting-minutes harvest` — that skill owns the edit (apply directive + remove the resolved comment); action-extraction only invokes it, then re-reads the corrected file in Step 2.
- **Append-only on briefs.** Never delete or rewrite existing Next Actions, WF items, or Log entries. Only add new rows.
- **Never create briefs.** If no match, flag and skip — do not spawn new files.
- **Preserve formatting.** Match the existing indentation, bullet style, and table format of each target brief.

### Matching Rules
- **Confidence floor: 70.** Below threshold → log as unrouted, skip.
- **Agenda linkage wins.** If a matching agenda exists and it links to a brief, that brief is always a ≥70 match.
- **No fuzzy re-matching.** If a minutes file has been processed before (Log entry exists in target brief referencing the same `[[Meeting Minutes — ...]]` wikilink), skip.

### Output Rules
- **Every added item includes a wikilink to the source minutes** — required by the vault's "Actionable output navigation" convention.
- **Cron output is terminal-only.** No Discord, Telegram, Slack, or INBOX report in cron. Briefs and Log entries are the durable vault output. Interactive mode additionally creates Gmail drafts (Step 7) — drafts are never auto-sent; the user reviews and sends them outside this skill per [[Email Sending Protocol]].
- **Self-improving.** Step 3 loads `learned-overrides.md` and applies matching meeting/attendee→brief and action-phrasing→ownership rules **before** the Matching Heuristic, in **both modes**. Correction Capture ([[correction-capture]] mode: capture) runs at the end of Step 4 **interactive mode only** — `if mode == cron: skip`, since cron has no manifest to diff — logging proposed-vs-chosen `destination-override` / `reclassification` corrections to `_corrections-log.md` and auto-codifying clear ones into `learned-overrides.md`. **Detached-surface limitation:** post-run brief-edit reroutes (the user re-owning/moving an item days later) are out of scope — catching them needs a separate post-hoc brief-diff mechanism (deferred, not wired). Recurring un-codified corrections (≥2) are promoted at `/retro` (mode: consolidate). This is the active form of [[Memory on Triage]].

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
- [[correction-capture]] — mode: capture (Step 4 interactive-mode self-improvement loop); mode: consolidate (run from /retro)

### Interactive-mode (Step 4.5 + Step 7) additional dependencies
- `05 AI/SHARED/scripts/send_email.py` — `create_draft()` / `--draft` for MIME-safe Gmail draft creation (Bufalinda account). See [[Email Sending Protocol]] and [[Wire-Level Verification]].
- Trello MCP — `set_active_board`, `get_lists`, `get_cards_by_list_id`, `add_card_to_list`, `get_board_members`, `assign_member_to_card`. Honors [[Trello List Filtering]].
- Vault Contacts (`03 REFERENCE/IMPORTANT DOCS/Vault Contacts.md`) + Google Contacts (gws) — recipient email resolution.

### Vault Conventions
- [[Email Sending Protocol]] — drafts only; user reviews/sends.
- [[Venezuelan Spanish (tuteo)]] — all participant-email copy.
- [[Wire-Level Verification]] — verify the created draft's bytes; never emit mojibake.
- [[Trello List Filtering]] — active lists only.
- [[Eisenhower Priority Mapping]] — due-date handling unchanged.

### Does NOT Require
- No send capability (drafts only). No Slack/Discord/Telegram. No new-brief creation.

## Related Skills
- [[meeting-minutes]] — upstream: creates the minutes files this skill consumes; Step 1.5 also invokes its `harvest` mode (Step 6.5) on today's minutes before routing, so `{@ ...}` corrections (esp. `action-reroute`) apply first.
- [[followup]] — downstream: picks up the Waiting For items we add.
