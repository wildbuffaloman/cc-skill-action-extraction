---
name: brief-updater
description: "Surgical project brief section editing — insert tasks, toggle checkboxes, update waiting-for, append logs, write continuation prompts. v2: log-entry mode names the consumed spec/plan/manifest as a [[wikilink]] so downstream consumption checks (inbox-clear Spec/Plan Gate) can detect executed artifacts."
version: 2
---

Canonical edit operations for Obsidian project briefs. Every mode uses the Edit tool for surgical changes -- never rewrite the entire file. Always match existing formatting of the target brief.

## Core Logic

### Section Location
Sections are H2 headers: `## Next Actions`, `## Waiting For`, `## Log`. The `### Continuation Prompt` subsection lives at the top of `## Next Actions`. Locate by scanning for the exact header string.

### Canonical Insertion Point
The `### Continuation Prompt` block always sits at the **top** of `## Next Actions`, immediately under the `## Next Actions` header. New `- [ ]` tasks go **below the Continuation Prompt block**: after the last existing `- [ ]` line in the target section, and before the next `## ` header (`## Waiting For` / `## Log`). If the section has no tasks yet, insert immediately after the Continuation Prompt block. **Never** insert a task above the Continuation Prompt, and never place it inside the CP's `>` blockquote.

### Edit Tool Usage
Always use Edit with `old_string`/`new_string`. To append after a line, use that line as `old_string` and same line + new line as `new_string`. Never rewrite sections wholesale.

### Priority Emoji Mapping

Apple Reminders' UI exposes 4 levels (None/Low/Medium/High = 0/9/5/1). They map intuitively to Eisenhower quadrants — High = Q1 (today's fire), Medium = Q2 (strategic), Low = Q3 (quick admin), None = Q4 (defer). See `feedback_apple_reminders_q_mapping_intuitive.md` for the canonical rule.

| Apple UI | Apple int | Emoji | Quadrant |
|----------|-----------|-------|----------|
| High | 1 (1-4) | 🔺 | Q1 |
| Medium | 5 | ⏫ | Q2 |
| Low | 9 (6-9) | 🔼 | Q3 |
| None | 0 | *(omit)* | Q4 |

---

## Modes

### task-insert
**Used by:** inbox-clear, reminders-sync, meeting-minutes

Add `- [ ]` line to `## Next Actions` at the canonical insertion point.

**Format:** `- [ ] Title [priority emoji] [📅 due date] [🔁 recurrence]`

Marker order: title -> priority -> due date -> recurrence. Omit any absent marker.
```
- [ ] Call supplier about delivery 🔺 📅 2026-03-20
- [ ] Monthly Love Letter to Niki ⏫ 📅 2026-03-19 🔁 every month
- [ ] Buy office supplies
```
Rules: priority emoji only if not Normal; due date with `📅` prefix; recurrence with `🔁` prefix (`🔁 every N units`); title max 200 chars. If `## Next Actions` missing, flag to user.

**Dedup guard (mirrors log-entry's "never duplicate").** Before inserting, semantic-match the new task against existing items in BOTH `## Next Actions` and `## Waiting For`. If a matching item already exists, update it in place (refresh due date / priority / owner) instead of adding a second line. Only insert when no equivalent item is present. Wording may differ across capture sources — match on intent, not exact string. This is the rule that keeps Next Actions from filling with re-discovered duplicates across sessions.

---

### wf-update
**Used by:** followup

Modify items in `## Waiting For`. Find target item by semantic match on description text.

- **Follow-up marker:** Append `📨 YYYY-MM-DD` after sending a message
- **RESCHEDULE:** Update or add `📅 YYYY-MM-DD` on the item line
- **REDIRECT/ASSIGN:** Replace `@Person` with `@NewName`, or add `@Name` prefix to unassigned items

Example: `- [ ] Waiting on @Carlos for quote 📅 2026-03-15` -> after RESCHEDULE:2026-04-01 + SEND -> `- [ ] Waiting on @Carlos for quote 📅 2026-04-01 📨 2026-04-10`

---

### checkbox-toggle
**Used by:** followup (COMPLETE), task-sync

Change `- [ ]` to `- [x]` in `## Next Actions` or `## Waiting For`. Find item by semantic match -- wording may differ between systems.

**With attribution (task-sync):** Also append a log entry (detect format first):
- Table: `| YYYY-MM-DD | Task | [Item text] (synced from [Trello/Reminders]) |`
- List: `- YYYY-MM-DD -- Task: [Item text] (synced from [Trello/Reminders])`

---

### log-entry
**Used by:** followup, session-close, close-day, meeting-minutes

Append row to `## Log`. Detect existing format first:
- **Table:** `| YYYY-MM-DD | Type | Description |` -- new row at end of table body
- **List:** `- YYYY-MM-DD -- Type: Description` -- after last list item

Type values: `Task`, `Milestone`, `Follow-up`, or as specified by calling skill. One item per row. Never duplicate same date + summary. Match existing format exactly.

Example: `| 2026-04-10 | Follow-up | Sent via Slack to @Carlos re: supplier quote. Channel: DM. |`

**Name the consumed artifact (added v2).** When the session **executed / consumed / shipped a spec, plan, design-doc, implementation-plan, or manifest** that lives (or lived) in `00 HUB/00 INBOX/`, the Description MUST name it as a `[[wikilink]]` (and say it was executed/shipped). This is the authoritative signal downstream skills use to tell that an INBOX spec/plan is safe to archive — `/inbox-clear`'s Spec/Plan Consumption Gate greps project `## Log` sections for exactly this. Plain prose that omits the artifact name leaves the consumption check blind.
- Good: `| 2026-06-05 | Milestone | Executed [[2026-06-05-cron-mode-plan]] — shipped startup-day v0.5.0 cron mode; 19/19 tests green. |`
- Weak (avoid): `| 2026-06-05 | Milestone | Finished the cron work. |` (no artifact named → consumption check can't confirm).

---

### continuation-prompt
**Used by:** session-close, close-day, meeting-minutes, propagate-amendment

Update or create `### Continuation Prompt` at top of `## Next Actions`. Only the latest is kept -- overwritten each session. **Hard rule:** A brief must contain exactly ONE Continuation Prompt block. There are only two valid operations on the CP, and they are mutually exclusive: **replace** it (overwrite in place) or **preserve** it (make zero CP edits). Writing a new CP block while an old one survives — even renamed to `_legacy` or `(Archive)` — is forbidden; that is the accumulation bug. If multiple are detected (legacy state from older skill versions or hand-edits), consolidate to one at the canonical location (top of `## Next Actions`, before any `- [ ]` items). Common drift: stray prompt placed AFTER `## Log` heading -- move it back to canonical location during update.

```markdown
### Continuation Prompt
> **Date:** YYYY-MM-DD
> **Mode:** <Vault Management / Programming Projects>
> **Context:** <1-2 sentences>
>
> **Resume from:** <specific next step>
> **Key decisions this session:** <critical choices>
> **Blockers/open questions:** <unresolved>
> **Files touched:** <key files modified>
```

**Replace procedure (one Edit, never insert-on-top):**

1. Read the brief. Count existing `### Continuation Prompt` headings — include drift variants (`### _legacy Continuation Prompt`, `### Continuation Prompt (… Archive …)`).
2. Bound the current block: from its `### Continuation Prompt` heading through its last contiguous `>` line. Replace that whole span in ONE Edit (old header+body → new header+body).
3. **Merge carryover:** copy any still-relevant items from the old block's Resume-from / Blockers / Waiting sections into the new one, so nothing requires reading old text to recover.
4. If you cannot cleanly bound the existing block (ambiguous formatting, merged-in content from another brief), STOP and consolidate by hand. **Never** fall back to inserting a second block above the old one.
5. Remove any extra `_legacy`/archive CP headings in the same update. Demote genuinely-instructional example CPs (e.g. participant-facing templates) to plain text or a fenced code block so they are no longer headings.

**End state:** exactly ONE `### Continuation Prompt` heading in the brief (verifiable: `grep -c '^### .*Continuation Prompt' <brief>` outside code fences == 1). If missing entirely: insert directly under `## Next Actions`, before any `- [ ]` items.

---

### template-compliance
**Used by:** clean-projects

Add missing frontmatter fields and required sections (`## Outcome`, `## Next Actions`, `## Log`) per Brief Template. Do not modify existing content -- only add what is absent.
