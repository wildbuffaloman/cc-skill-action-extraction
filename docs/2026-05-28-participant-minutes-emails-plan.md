# Participant Minutes Emails — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an interactive-only Step to `action-extraction` that drafts (never sends) one personalized, encoding-safe Venezuelan-Spanish minutes email per selected attendee — with the meeting's key decisions, that person's assigned tasks, and Trello card links — and asks them to confirm receipt + acceptance.

**Architecture:** Reuse the proven `send_email.py` MIME builder by adding a `create_draft()` twin to `send_email()` (shared raw-message builder → `gws gmail users drafts create`). The skill (SKILL.md prose) gains a new interactive Step 7 that resolves recipients, builds per-person bodies from the already-extracted Ownership Split, links/creates Trello cards, and calls `send_email.py --draft`. Cron mode is untouched.

**Tech Stack:** Python 3 (`email.message.EmailMessage`, `base64`), `gws` CLI (Gmail drafts API), Trello MCP, Vault Contacts markdown table.

**Repos touched (commit separately):**
- `05 AI` repo → `05 AI/SHARED/scripts/send_email.py` + new test
- `action-extraction` standalone repo → `SKILL.md`, `docs/`, `_bundled/`

---

## File Structure

| File | Repo | Responsibility | Action |
|------|------|----------------|--------|
| `05 AI/SHARED/scripts/send_email.py` | 05 AI | MIME-safe draft creation (`create_draft`) sharing the builder with `send_email` | Modify |
| `05 AI/SHARED/scripts/test_send_email_draft.py` | 05 AI | Real end-to-end test: create accented-Spanish draft, read back, assert no mojibake, delete | Create |
| `05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md` | action-extraction | New interactive Step 7 (emails) + Trello Step 4.5 + renumbered report + mode-scoped boundary + version/description | Modify |
| `.../action-extraction/docs/2026-05-28-participant-minutes-emails-design.md` | action-extraction | Spec (already written) | (exists) |

---

## Task 1: Add `create_draft()` to send_email.py (shared MIME builder)

**Files:**
- Modify: `05 AI/SHARED/scripts/send_email.py`
- Test: `05 AI/SHARED/scripts/test_send_email_draft.py`

### Rationale
`send_email()` already builds a correct `EmailMessage`, base64url-encodes it, and posts `{'raw': ...}` to `users.messages.send`. The Gmail drafts API is identical except the body is `{'message': {'raw': ...}}` posted to `users.drafts.create`. Extracting the message-building into one helper avoids a second, divergent encoder (the whole point — Spanish accents must not mojibake).

- [ ] **Step 1: Write the failing test**

Create `05 AI/SHARED/scripts/test_send_email_draft.py`:

```python
"""End-to-end test for send_email.create_draft().

Creates a real draft on the bufalinda account with accent-dense Venezuelan
Spanish, reads it back via the Gmail API, asserts the decoded body and subject
contain the exact accented characters (no mojibake), then deletes the draft.

Run: python3 "05 AI/SHARED/scripts/test_send_email_draft.py"
Drafts are never sent — safe to run against the live account.
"""
import base64
import json
import os
import subprocess
import sys

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from send_email import create_draft, GWS_BIN, ACCOUNT_CONFIGS  # noqa: E402

ACCENTED_SUBJECT = "Minuta y tareas — Reunión de Promoción — 2026-05-28"
ACCENTED_BODY = (
    "Hola Alondra,\n\n"
    "Acuerdos clave\n"
    "• Definición del posicionamiento y presupuesto por categoría.\n\n"
    "¿Confirmas recibido? Por favor responde aquí. ¡Gracias!\n"
)


def _read_draft_message(draft_id: str, account: str) -> dict:
    env = os.environ.copy()
    env['GOOGLE_WORKSPACE_CLI_CONFIG_DIR'] = os.path.expanduser(ACCOUNT_CONFIGS[account])
    out = subprocess.run(
        [GWS_BIN, 'gmail', 'users', 'drafts', 'get',
         '--params', json.dumps({'userId': 'me', 'id': draft_id, 'format': 'full'})],
        capture_output=True, text=True, env=env, timeout=15,
    )
    assert out.returncode == 0, f"drafts get failed: {out.stderr}"
    return json.loads(out.stdout)


def _delete_draft(draft_id: str, account: str) -> None:
    env = os.environ.copy()
    env['GOOGLE_WORKSPACE_CLI_CONFIG_DIR'] = os.path.expanduser(ACCOUNT_CONFIGS[account])
    subprocess.run(
        [GWS_BIN, 'gmail', 'users', 'drafts', 'delete',
         '--params', json.dumps({'userId': 'me', 'id': draft_id})],
        capture_output=True, text=True, env=env, timeout=15,
    )


def main() -> int:
    result = create_draft(
        to='afernandez@bufalinda.com',
        subject=ACCENTED_SUBJECT,
        body=ACCENTED_BODY,
        account='bufalinda',
    )
    draft_id = result['id']
    try:
        data = _read_draft_message(draft_id, 'bufalinda')
        payload = data['message']['payload']
        headers = {h['name']: h['value'] for h in payload.get('headers', [])}
        subject = headers.get('Subject', '')
        assert 'Ã' not in subject and 'Â' not in subject, f"mojibake in subject: {subject!r}"

        # Walk parts to find the text/plain body and decode it
        def find_text(p):
            if p.get('mimeType') == 'text/plain' and p.get('body', {}).get('data'):
                return p['body']['data']
            for sub in p.get('parts', []) or []:
                d = find_text(sub)
                if d:
                    return d
            return None

        b64 = find_text(payload)
        assert b64, "no text/plain body found in draft"
        decoded = base64.urlsafe_b64decode(b64 + '===').decode('utf-8')
        assert 'Reunión' in subject, f"accent lost in subject: {subject!r}"
        assert '¿Confirmas recibido?' in decoded, f"accent/punct lost in body: {decoded!r}"
        assert '¡Gracias!' in decoded, f"inverted-mark lost in body: {decoded!r}"
        assert 'categoría' in decoded, f"accent lost in body: {decoded!r}"
        print("PASS — draft encoding clean (no mojibake)")
        return 0
    finally:
        _delete_draft(draft_id, 'bufalinda')


if __name__ == '__main__':
    sys.exit(main())
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python3 "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/SHARED/scripts/test_send_email_draft.py"`
Expected: FAIL with `ImportError: cannot import name 'create_draft' from 'send_email'`

- [ ] **Step 3: Refactor the MIME builder out of `send_email()`**

In `send_email.py`, add this helper above `send_email()` (after `_verify_wire_level`). It is the exact message-building logic currently inline in `send_email()` (lines ~274–339), lifted verbatim and returned instead of sent:

```python
def _build_raw_message(
    to: str,
    subject: str,
    body: str,
    account: str,
    cc: str | None,
    bcc: str | None,
    reply_to: str | None,
    from_addr: str | None,
    attachments: list[str] | None,
    in_reply_to: str | None,
) -> tuple[str, str | None, list[str]]:
    """Build a base64url RFC822 message shared by send_email() and create_draft().

    Returns (raw_b64url, thread_id_or_None, thread_warnings). Encoding is handled
    by Python's email library: RFC 2047 subjects, MIME-Version, charset=utf-8,
    quoted-printable bodies — the same path that fixed the gws +send mojibake bug.
    """
    msg = EmailMessage()
    msg['To'] = to
    msg['From'] = from_addr or ACCOUNT_FROM[account]
    msg['Subject'] = subject
    if cc:
        msg['Cc'] = cc
    if bcc:
        msg['Bcc'] = bcc
    if reply_to:
        msg['Reply-To'] = reply_to

    thread_warnings: list[str] = []
    thread_id_for_send: str | None = None
    if in_reply_to:
        thread_info = _fetch_thread_info(in_reply_to, account)
        rfc_id = thread_info.get('rfc_message_id', '')
        if rfc_id:
            msg['In-Reply-To'] = rfc_id
            prev_refs = thread_info.get('references', '').strip()
            msg['References'] = f'{prev_refs} {rfc_id}'.strip() if prev_refs else rfc_id
            thread_id_for_send = thread_info.get('thread_id') or None
        else:
            thread_warnings.append(
                f'threading requested (in_reply_to={in_reply_to!r}) but failed '
                'to fetch original message — sent as new thread'
            )

    msg.set_content(body, charset='utf-8')

    if attachments:
        import mimetypes
        for path_str in attachments:
            path = Path(path_str)
            if not path.exists():
                raise FileNotFoundError(f'Attachment not found: {path_str}')
            ctype, encoding = mimetypes.guess_type(str(path))
            if ctype is None or encoding is not None:
                if path.suffix.lower() == '.md':
                    ctype = 'text/markdown'
                else:
                    ctype = 'application/octet-stream'
            maintype, subtype = ctype.split('/', 1)
            data = path.read_bytes()
            if maintype == 'text':
                msg.add_attachment(data.decode('utf-8', errors='replace'),
                                   subtype=subtype, filename=path.name)
            else:
                msg.add_attachment(data, maintype=maintype, subtype=subtype,
                                   filename=path.name)

    raw_b64url = base64.urlsafe_b64encode(msg.as_bytes()).decode('ascii').rstrip('=')
    return raw_b64url, thread_id_for_send, thread_warnings
```

Then **replace** the inline build in `send_email()` (the block from `msg = EmailMessage()` through `raw_b64url = base64.urlsafe_b64encode(...)`, i.e. current lines ~274–332, INCLUDING the `thread_warnings`/`thread_id_for_send`/`if in_reply_to:` block) with:

```python
    raw_b64url, thread_id_for_send, thread_warnings = _build_raw_message(
        to, subject, body, account, cc, bcc, reply_to, from_addr,
        attachments, in_reply_to,
    )
```

Leave the rest of `send_email()` (the `env`, `send_body`, `subprocess.run([... 'messages', 'send' ...])`, verify, return) unchanged.

- [ ] **Step 4: Add `create_draft()` and a `_verify_draft()` helper**

Add after `send_email()`:

```python
def _verify_draft(draft_id: str, account: str) -> list[str]:
    """Fetch a created draft back and inspect its headers for mojibake / missing
    MIME headers. Mirror of _verify_wire_level but for the drafts.get envelope
    ({id, message: {payload: {...}}}). Never raises — advisory only."""
    env = os.environ.copy()
    env['GOOGLE_WORKSPACE_CLI_CONFIG_DIR'] = os.path.expanduser(ACCOUNT_CONFIGS[account])
    try:
        result = subprocess.run(
            [GWS_BIN, 'gmail', 'users', 'drafts', 'get',
             '--params', json.dumps({'userId': 'me', 'id': draft_id, 'format': 'full'})],
            capture_output=True, text=True, env=env, timeout=10,
        )
    except subprocess.TimeoutExpired:
        return ['draft verification timed out']
    if result.returncode != 0:
        return [f'draft verification fetch failed: {result.stderr.strip()[:200]}']
    try:
        data = json.loads(result.stdout)
    except json.JSONDecodeError:
        return ['draft verification returned non-JSON']
    payload = data.get('message', {}).get('payload', {})
    headers = {h['name']: h['value'] for h in payload.get('headers', [])}
    warnings: list[str] = []
    subject = headers.get('Subject', '')
    if 'Ã' in subject or 'Â' in subject:
        warnings.append(f'mojibake in draft Subject header: {subject!r}')
    if 'MIME-Version' not in headers:
        warnings.append('draft missing MIME-Version header')
    content_type = headers.get('Content-Type', '')
    if not content_type:
        warnings.append('draft missing Content-Type header')
    elif 'charset' not in content_type.lower() and 'multipart' not in content_type.lower():
        warnings.append(f'draft Content-Type missing charset: {content_type!r}')
    return warnings


def create_draft(
    to: str,
    subject: str,
    body: str,
    account: str = 'bufalinda',
    cc: str | None = None,
    bcc: str | None = None,
    reply_to: str | None = None,
    from_addr: str | None = None,
    attachments: list[str] | None = None,
    in_reply_to: str | None = None,
    verify: bool = True,
) -> dict:
    """Create a Gmail DRAFT (never sends) via gws drafts API with proper MIME
    encoding. Same signature family as send_email(). Returns the Gmail
    drafts.create response dict ({'id', 'message': {...}}). When verify=True
    (default), reads the draft back and appends result['warnings'] if the
    encoding looks wrong. Raises RuntimeError on a non-zero gws exit.
    """
    if account not in ACCOUNT_CONFIGS:
        raise KeyError(f'Unknown account {account!r}. Must be one of {list(ACCOUNT_CONFIGS)}')

    raw_b64url, thread_id, thread_warnings = _build_raw_message(
        to, subject, body, account, cc, bcc, reply_to, from_addr,
        attachments, in_reply_to,
    )

    message_obj: dict = {'raw': raw_b64url}
    if thread_id:
        message_obj['threadId'] = thread_id

    env = os.environ.copy()
    env['GOOGLE_WORKSPACE_CLI_CONFIG_DIR'] = os.path.expanduser(ACCOUNT_CONFIGS[account])
    result = subprocess.run(
        [GWS_BIN, 'gmail', 'users', 'drafts', 'create',
         '--params', '{"userId":"me"}',
         '--json', json.dumps({'message': message_obj})],
        capture_output=True, text=True, env=env,
    )
    if result.returncode != 0:
        raise RuntimeError(
            f'gws drafts create failed (rc={result.returncode})\n'
            f'STDOUT: {result.stdout}\nSTDERR: {result.stderr}'
        )
    response = json.loads(result.stdout)

    warnings = list(thread_warnings)
    if verify and response.get('id'):
        warnings += _verify_draft(response['id'], account)
    if warnings:
        response['warnings'] = warnings
    return response
```

- [ ] **Step 5: Add `--draft` flag to the CLI**

In `_cli()`, after the `--no-verify` arg, add:

```python
    parser.add_argument('--draft', dest='draft', action='store_true', default=False,
                        help='Create a Gmail draft instead of sending (never sends).')
```

Replace the `result = send_email(...)` call in `_cli()` with a branch:

```python
    fn = create_draft if args.draft else send_email
    common = dict(
        to=args.to, subject=args.subject, body=body, account=args.account,
        cc=args.cc, bcc=args.bcc, reply_to=args.reply_to, from_addr=args.from_addr,
        in_reply_to=args.in_reply_to, attachments=args.attachments,
    )
    try:
        result = fn(**common) if args.draft else fn(verify=args.verify, **common)
    except (RuntimeError, KeyError, FileNotFoundError) as e:
        print(f'ERROR: {e}', file=sys.stderr)
        return 1
```

(`create_draft` defaults `verify=True`; the send path keeps its `verify=args.verify` auto-by-domain behavior.)

- [ ] **Step 6: Run the test to verify it passes**

Run: `python3 "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/SHARED/scripts/test_send_email_draft.py"`
Expected: `PASS — draft encoding clean (no mojibake)` and the test draft is auto-deleted. (Requires the bufalinda gws config to be authenticated; if it prints an auth error, surface it — do not mark the task done.)

- [ ] **Step 7: Commit (05 AI repo)**

```bash
cd "/Users/albertoduhau/Documents/Obsidian Vault/05 AI"
git add SHARED/scripts/send_email.py SHARED/scripts/test_send_email_draft.py
git commit -m "feat(send_email): add create_draft() + --draft CLI for MIME-safe Gmail drafts

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```
If a local hook blocks commit-to-main, route via `gh pr create` + `gh pr merge --merge --delete-branch` per [[feedback_gh_pr_merge_bypasses_commit_hook]].

---

## Task 2: SKILL.md — mode-scope the boundary + bump metadata

**Files:**
- Modify: `05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md`

- [ ] **Step 1: Bump frontmatter**

Replace lines 3–6 (`version` and `description`):

```yaml
version: "0.3.0"
description: "Propagate action items, waiting-for items, and decisions from today's meeting minutes into the relevant project/program briefs. Cron: vault-only. Interactive: may also draft (never send) personalized minutes emails per participant with their decisions, assigned tasks, and Trello card links."
```

- [ ] **Step 2: Mode-scope the Philosophy "No notifications" bullet**

Replace the `## Philosophy` line `- **No notifications.** Vault-only. No Discord, no Slack, no Telegram.` with:

```markdown
- **No notifications in cron.** Cron mode is vault-only: no Discord, no Slack, no Telegram, no email. **Interactive mode** MAY additionally create Gmail **drafts** (never sends) for participants — see Step 7. The draft path is gated behind explicit per-meeting user selection and never runs in cron.
```

- [ ] **Step 3: Amend the "Terminal-only" Output Rule**

Replace the `## Rules → Output Rules` bullet `- **Terminal-only.** No Discord, Telegram, Slack, or INBOX report. Briefs and Log entries are the durable output.` with:

```markdown
- **Cron output is terminal-only.** No Discord, Telegram, Slack, or INBOX report in cron. Briefs and Log entries are the durable vault output. Interactive mode additionally creates Gmail drafts (Step 7) — drafts are never auto-sent; the user reviews and sends them outside this skill per [[Email Sending Protocol]].
```

- [ ] **Step 4: Verify the edits**

Run: `grep -n "0.3.0\|No notifications in cron\|Cron output is terminal-only" "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md"`
Expected: three matches on the new lines.

---

## Task 3: SKILL.md — add Step 4.5 (Trello match-else-create)

**Files:**
- Modify: `05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md`

- [ ] **Step 1: Insert Step 4.5 after Step 4 (Propagate to Briefs), before Step 5 (Idempotency Check)**

Insert this section:

````markdown
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
````

- [ ] **Step 2: Verify insertion order**

Run: `grep -n "### Step 4 \|### Step 4.5 \|### Step 5 " "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md"`
Expected: Step 4, then Step 4.5, then Step 5, in ascending line order.

---

## Task 4: SKILL.md — add Step 7 (Participant Minutes Emails) + renumber Report to Step 8

**Files:**
- Modify: `05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md`

- [ ] **Step 1: Renumber the existing Report step**

Change the heading `### Step 7 — Report` to `### Step 8 — Report`.

- [ ] **Step 2: Insert the new Step 7 immediately before Step 8 (Report)**

Insert this section (between Step 6 and the renumbered Step 8):

````markdown
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
````

- [ ] **Step 3: Add the Drafts block to Step 8 (Report)**

Inside the Step 8 report template, after the `Flags:` line and before the closing
`═` rule, add:

```
 Participant drafts: {N} created, {M} skipped (no email)
 ─────────────────────────────────────────────────────────
 • {Nombre} <{email}> — {K} tareas, {T} Trello cards [draft: {draft_id}]
 • {Nombre} — SKIPPED (no email resolved)

 Trello: {created} cards created, {linked} linked, {unassigned} unassigned (not board member)

 ⚠ Draft warnings: {any encoding/threading warnings, or "none"}
```

- [ ] **Step 4: Verify headings**

Run: `grep -n "### Step 6 \|### Step 7 — Participant\|### Step 8 — Report" "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md"`
Expected: Step 6, Step 7 (Participant Minutes Emails), Step 8 (Report), ascending.

---

## Task 5: SKILL.md — update Dependencies + Inputs + cron guard note

**Files:**
- Modify: `05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md`

- [ ] **Step 1: Note the interactive-only email capability under Inputs**

In `## Inputs`, append to the **No argument** bullet:

```markdown
  Interactive mode also offers Step 7 (Participant Minutes Emails) — per-meeting attendee selection and Gmail draft creation.
```

And append to the **"cron"** bullet:

```markdown
  Cron NEVER drafts emails or touches Trello — Steps 4.5 and 7 are interactive-only.
```

- [ ] **Step 2: Extend the Dependencies section**

In `## Dependencies`, after the existing `[[brief-updater]]` line, add:

```markdown

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
```

- [ ] **Step 3: Verify**

Run: `grep -n "send_email.py\|Venezuelan Spanish\|Does NOT Require" "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/CLAUDE CODE/skills/action-extraction/SKILL.md"`
Expected: matches in the Dependencies section.

---

## Task 6: End-to-end verification on a real meeting (the keystone test)

**Files:** none modified — this is the live validation per [[Patch Ship Discipline]].

- [ ] **Step 1: Pick the real fixture**

Use `00 HUB/00 INBOX/Meeting Minutes — Repaso Plan de Promocion Alondra — 2026-05-27.md`
(attendees `[[Alberto Duhau]]`, `[[Alondra]]`; Alondra → `afernandez@bufalinda.com`;
`related_project: [[Refrescamiento e Implementación de Plan de Promoción]]`).
Note: its date suffix is 2026-05-27 — invoke action-extraction in **interactive mode**
and point it at this file explicitly (interactive mode is not date-gated like cron).

- [ ] **Step 2: Run the skill interactively**

Invoke `/action-extraction` (no args). Walk: brief match → propagation preview →
Trello Step 4.5 (the promo project's board, or skip if none declared) → Step 7
attendee selection (select Alondra) → draft creation.

- [ ] **Step 3: Verify the draft exists and is clean**

Run: `python3 - <<'PY'` reading the latest bufalinda draft, OR open Gmail drafts for
`aeduhau@bufalinda.com`. Confirm:
- Subject and body show correct accents (`Promoción`, `¿Confirmas recibido?`, `¡...`).
- "Tus tareas asignadas" lists Alondra's owned items with Trello URLs (or the
  no-tasks line).
- "Acuerdos clave" reflects `## Key Decisions`.
- The acceptance paragraph is present.
- The skill reported **0 draft warnings**.

Expected: a clean Spanish draft addressed to `afernandez@bufalinda.com`, unsent.

- [ ] **Step 4: Confirm cron is unaffected**

Run: `/action-extraction cron` (or dry-read the cron path) and confirm Steps 4.5 and
7 are skipped — terminal-only summary, no drafts created, no Trello calls.

- [ ] **Step 5: Decide the test draft's fate**

The Alondra draft is a genuine, useful artifact — leave it for the user to review/send,
or delete it if the user prefers. Ask before deleting.

---

## Task 7: Commit the skill + sync bundle

**Files:**
- `action-extraction/SKILL.md`, `docs/`, possibly `_bundled/`

- [ ] **Step 1: Re-bundle shared nodes (if changed)**

The skill's only `_shared/nodes` dep (`brief-updater`) is unchanged, so no re-bundle
is strictly required. If `/push-skills` bundling runs, confirm `_bundled/nodes/brief-updater.md`
still matches `_shared/nodes/brief-updater.md` (no drift). `send_email.py` is NOT a
shared node — it stays referenced by absolute path, not bundled.

- [ ] **Step 2: Commit the skill repo**

```bash
cd "/Users/albertoduhau/Documents/Obsidian Vault/05 AI/CLAUDE CODE/skills/action-extraction"
git add SKILL.md docs/2026-05-28-participant-minutes-emails-design.md docs/2026-05-28-participant-minutes-emails-plan.md
git commit -m "feat: v0.3.0 — interactive participant minutes emails (drafts) + Trello linking

- New interactive Step 7: per-attendee Gmail drafts (Venezuelan Spanish, drafts-only)
- New Step 4.5: Trello match-else-create + assign, card links in emails
- Mode-scoped the no-notifications boundary (cron unchanged)
- Reuses send_email.create_draft() for MIME-safe encoding

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

> **Background-job note:** if a worktree-isolation guard trips on this vault skill-repo
> commit, follow [[feedback_bg_job_worktree_skill_edits]] — EnterWorktree, edit, then
> `git merge --ff-only` to the symlinked-live main, PR-route the push, ExitWorktree.

- [ ] **Step 3: Sync plugin cache (if the skill is mirrored to a local plugin)**

Per [[feedback_plugin_cache_sync]], if action-extraction is registered in a local
plugin cache, sync the updated SKILL.md so the running session picks up v0.3.0.

---

## Self-Review (completed during authoring)

- **Spec coverage:** §3 boundary → Task 2. §4 workflow insertion → Tasks 3–4. §5 data
  flow → Step 7.4 (reuses Ownership Split, reads `## Key Decisions`). §6 recipient
  resolution → Step 7.2–7.3. §7 Trello → Task 3. §8 email composition → Step 7.4.
  §9 send mechanism/encoding → Task 1 (create_draft + test) + Step 7.5. §10 report →
  Task 4 Step 3. §11 invariants → cron guards in Tasks 3/4 + Task 6 Step 4. §12
  dependencies → Task 5. All covered.
- **Placeholder scan:** code steps contain real code; prose steps contain the exact
  markdown to insert. No TBD/TODO.
- **Type consistency:** `create_draft`, `_build_raw_message`, `_verify_draft` used
  consistently across Tasks 1, 4, 5; CLI `--draft` flag matches the Step 7.5 invocation.
- **Spec-vs-reality fixes folded in:** Trello board = brief-body URL (not frontmatter);
  decisions = `## Key Decisions` (not `### Decisions`). Both reflected in Tasks 3 & 4.
