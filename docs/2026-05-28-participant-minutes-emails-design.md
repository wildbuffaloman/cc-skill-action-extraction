# Design — action-extraction v0.3.0: Participant Minutes Emails

**Date:** 2026-05-28
**Skill:** `action-extraction`
**Version:** 0.2.0 → 0.3.0
**Status:** Design — pending user review

---

## 1. Problem & Goal

Today `action-extraction` is **vault-only**: it routes a meeting's Next Actions /
Waiting For / Decisions into the matching project/program briefs and stops there.
Participants never receive a personalized record of what was decided and what they
now own. The handoff to *people* is manual.

**Goal:** in interactive mode, after propagation, draft one personalized minutes
email per selected attendee containing the meeting's key decisions, that person's
assigned tasks (with Trello card links), and an explicit request to confirm
receipt and acceptance. Drafts only — never auto-sent.

**Non-goals (v1):**
- No change to cron behavior — cron stays 100% vault-only, no emails.
- No attachment of the full minutes `.md` (the email *is* the personalized minutes).
- No reply-tracking / acceptance-reconciliation automation (that is a future skill).

---

## 2. Decisions (from brainstorming, 2026-05-28)

| Decision | Choice |
|----------|--------|
| Recipient scope | **Ask per meeting** — list attendees, user selects who gets a draft. |
| Sending account | **Bufalinda gws** — `aeduhau@bufalinda.com`. |
| Language | **Venezuelan Spanish (tuteo)** — per [[Venezuelan Spanish (tuteo)]]. |
| Trello linking | **Match existing card, else create + assign + link.** |
| Acceptance mechanism | **Reply-to-confirm** — plain Spanish ask, no tooling. |
| Send vs draft | **Draft only, never auto-send** — per [[Email Sending Protocol]]. |
| Mode | **Interactive only** — cron skips the entire email step. |

---

## 3. Boundary Change

The current `## Philosophy` and `## Rules → Output Rules` assert "No notifications.
Vault-only. No Discord, no Slack, no Telegram." This becomes mode-scoped:

> **Cron mode:** vault-only, no notifications (unchanged).
> **Interactive mode:** MAY additionally create (never send) Gmail drafts on the
> Bufalinda account. All drafts require explicit user send action outside this skill.

The skill's `description` frontmatter is updated to reflect the new interactive
capability while affirming cron stays vault-only.

This is the only relaxation. All other boundary rules are preserved verbatim:
- Read-only on minutes files.
- Append-only on briefs.
- Never create briefs.

---

## 4. Workflow Insertion

New **Step 7 — Participant Minutes Emails**, placed after Step 6 (Handle Unmatched
Items). The existing Report step is renumbered to **Step 8** so the report can
summarize drafts created.

```
... Step 4 Propagate → Step 5 Idempotency → Step 6 Unmatched
    → Step 7 Participant Minutes Emails  [interactive only]   (NEW)
    → Step 8 Report (was Step 7; now includes drafts summary)
```

The new email step is **skipped entirely in cron mode** (the existing cron Hard
Rules already forbid notifications; the step adds an explicit guard:
`if mode == cron: skip`).

---

## 5. Data Flow (reuse, no re-parse)

Step 4 already computes, per meeting, the **Ownership Split**: each owner mapped to
their owned action items, plus the meeting's shared Decisions list. Step 8 consumes
this **in-memory** structure. It does NOT re-read the Granola transcript and does
NOT re-parse the minutes file beyond what Step 2 already extracted.

Per meeting, the data available to Step 8:
- `meeting.title`, `meeting.date`, `meeting.attendees[]`
- `meeting.decisions[]` (shared)
- `ownership[owner] = [action items owned by that owner]`
- Trello links produced during Step 4.5 (see §7) keyed by action item

---

## 6. Recipient Resolution

For each meeting, for each attendee in `attendees`:

1. **Resolve email** in priority order:
   1. Vault Contacts (`03 REFERENCE/IMPORTANT DOCS/Vault Contacts.md` — match by name)
   2. Google Contacts (gws)
   3. Granola attendee email (if present in meeting metadata)
2. Attendees with no resolvable email are listed as `— no email — skipped`.

**Interactive selection:** present the attendee list per meeting. Default-selected =
attendees who own ≥1 action item from that meeting. Pure observers are listed but
unselected (user can opt them in). User confirms the selection before any draft is
built.

---

## 7. Trello: Match-Else-Create (Step 4.5)

Runs as a sub-step during/after propagation so links are ready for Step 8.

**Board resolution per matched brief:**
- Read the brief's declared Trello board (frontmatter `trello_board` or a Working
  Notes reference). Respect [[Trello List Filtering]] (active lists only).
- If a matched brief declares **no** board: ask the user once which board to use,
  or skip Trello for that meeting (user choice). Never guess a board.

**Per owned action item:**
1. Fuzzy-match the action text against the board's cards (normalized title overlap).
2. **Match found** → use that card's `https://trello.com/c/...` URL.
3. **No match** → create a card on the board (title = action text, due = item due
   date if present), then:
   - Resolve the member via `get_board_members` (name match). If the owner is a
     board member → assign them. If not → create + link the card but **skip
     assignment and flag it** in the report.
   - Use the new card's URL.

Card URLs are plain `https://trello.com/c/...` strings (recipients can click;
wikilinks would render as raw text in email).

---

## 8. Email Composition

One draft per selected attendee. Body is **plain Venezuelan-Spanish text, no
wikilinks**, structured:

```
Asunto: Minuta y tareas — {Meeting Title} — {YYYY-MM-DD}

Hola {Nombre},

Te comparto el resumen de lo que acordamos en {Meeting Title} ({fecha}) y las
tareas que quedaron a tu cargo.

Acuerdos clave
• {decisión 1}
• {decisión 2}
...

Tus tareas asignadas
• {acción 1} — vence {fecha o "sin fecha"} — {trello URL si aplica}
• {acción 2} — ...

¿Confirmas recibido?

Por favor responde a este correo confirmando que recibiste la minuta y que estás
de acuerdo con los acuerdos y tus tareas asignadas. Si algo no refleja lo
conversado, respóndelo aquí y lo corregimos.

Gracias,
Alberto

—
Borrador asistido por Claude Code
```

- If the attendee owns **zero** tasks but was selected, the "Tus tareas asignadas"
  section reads `• (sin tareas asignadas en esta reunión)`.
- Decisions section always included (shared context). If the meeting had no
  decisions, the section is omitted.

---

## 9. Send Mechanism + Primary Risk

Drafts are created on `aeduhau@bufalinda.com`.

**#1 implementation risk — encoding.** User memory documents gws gmail mojibake on
the *send* path (`feedback_email_mime_encoding`), and tuteo Spanish is dense with
`á é í ñ ¿ ¡`. Mitigation:

1. Prefer the `send_email.py` helper in a **draft** mode if it supports one
   (investigate during planning — it currently targets sending).
2. Otherwise create the draft via gws gmail drafts, then **read the draft back**
   ([[Wire-Level Verification]]) and confirm accented characters render correctly
   before reporting success.
3. This is validated on a **real draft during testing**, before any git push
   ([[Patch Ship Discipline]]).

If neither path produces correctly-encoded drafts, Step 8 fails loudly with the
raw bytes shown — it never silently produces mojibake drafts.

---

## 10. Report Addition

The Report step (renumbered to Step 8) gains a Drafts block:

```
 Participant drafts: N created, M skipped (no email)
 ─────────────────────────────────────────────────────────
 • {Nombre} <{email}> — {K} tareas, {T} Trello cards [draft link]
 • {Nombre} — SKIPPED (no email resolved)

 Trello: {created} cards created, {linked} linked, {unassigned} unassigned (not board member)
```

---

## 11. Safety Properties (invariants)

- **Drafts only** — zero auto-send anywhere in the skill.
- **Cron path unchanged** — Step 8 guarded `if cron: skip`; no new cron output.
- **Briefs append-only**, **minutes read-only** — unchanged.
- **No new briefs** — unchanged.
- **Eisenhower / due-date** handling unchanged (due dates only surfaced in emails).
- **Encoding-safe or fail-loud** — never emit mojibake drafts.

---

## 12. Dependencies (additions)

- gws CLI — Gmail draft creation + Google Contacts lookup (Bufalinda account).
- Trello MCP — `get_board_members`, card search/create, member assignment.
- Vault Contacts — recipient email + name resolution.
- Conventions invoked: [[Email Sending Protocol]], [[Venezuelan Spanish (tuteo)]],
  [[Wire-Level Verification]], [[Trello List Filtering]], [[Patch Ship Discipline]],
  [[Eisenhower Priority Mapping]] (unchanged).

---

## 13. Out of Scope / Future

- Acceptance reply tracking & reconciliation (a separate downstream skill).
- Cron-mode drafting (deliberately excluded — violates review-before-send).
- Full-minutes attachment.
- Non-Bufalinda / English recipient routing (current scope assumes Bufalinda team).
