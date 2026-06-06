# action-extraction

> Propagate action items, waiting-for items, and decisions from today's meeting minutes into the relevant project/program briefs. Cron: vault-only. Interactive: may also draft (never send) personalized minutes emails per participant with their decisions, assigned tasks, and Trello card links. v0.5.0: Step 1.5 delegates `{@ ...}` comment harvesting to /meeting-minutes harvest on today's minutes BEFORE routing, so action-reroute corrections apply before propagation.

**Version:** 0.5.1

## Installation

```bash
cd ~/.claude/skills
git clone https://github.com/wildbuffaloman/cc-skill-action-extraction.git action-extraction
```

## Usage

Invoke in Claude Code:

```
/action-extraction optional: 'cron' for non-interactive scheduled run
```

## Files

- `SKILL.md`
- `_bundled/manifest.json`
- `_bundled/nodes/brief-updater.md`
- `_bundled/nodes/correction-capture.md`
- `docs/2026-05-28-participant-minutes-emails-design.md`
- `docs/2026-05-28-participant-minutes-emails-plan.md`
- `learned-overrides.md`

