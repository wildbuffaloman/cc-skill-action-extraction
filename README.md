# action-extraction

> Propagate action items, waiting-for items, and decisions from today's meeting minutes into the relevant project/program briefs. Vault-only — no external notifications.

**Version:** 0.1.0

## Installation

```bash
cd ~/.claude/skills
git clone https://github.com/wildbuffaloman/cc-skill-action-extraction.git action-extraction
```

## Usage

Invoke in Claude Code:

```
/action-extraction          # interactive: review matches before propagating
/action-extraction cron     # non-interactive: auto-propagate today's minutes
```

## How It Works

This skill is the handoff step between *capture* (`meeting-minutes`) and the GTD pipeline:

1. Reads today's meeting-minutes files from `00 HUB/00 INBOX/`.
2. Extracts the structured `## Next Actions`, `## Waiting For`, and `## Meeting Notes ### Decisions` sections from each.
3. Matches each meeting to one or more project/program briefs in `01 PROJECTS/` or `02 AREAS/`.
4. Routes each item: actions → matching brief's `## Next Actions`; waiting-for → `## Waiting For`; decisions → `## Log`.
5. Idempotent — safe to re-run; will not duplicate already-propagated items.

## Dependencies

- **`brief-updater`** shared node (bundled in `_bundled/nodes/`).
- **Upstream:** `meeting-minutes` skill — produces the minutes files this skill consumes.
- **Downstream (optional):** `followup` skill — picks up the Waiting For items this skill adds.

## Files

- `SKILL.md` — the skill instructions Claude Code reads
- `_bundled/nodes/brief-updater.md` — shared node dependency (vault → bundled → skip runtime fallback)
- `_bundled/manifest.json` — bundle metadata + source hashes
