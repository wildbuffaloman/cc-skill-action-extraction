# action-extraction Learned Overrides — read by Step 3 (Match Each Meeting to Briefs) BEFORE the Matching Heuristic, in BOTH interactive and cron mode.
# Auto-written by correction-capture (mode: capture, interactive runs only); promote/edit via /retro consolidation. Remove an entry to revert.
# A rule block keys on a meeting property (title pattern / attendee) → a target brief, OR on action-item phrasing → Next Actions vs Waiting For ownership.

<!-- No rules yet — bootstrapped empty. The flywheel fills this as you reroute proposed meeting→brief assignments, or re-own action items (Next Actions ↔ Waiting For), during an interactive run.
Examples of the shapes auto-learned rules will take:

### meeting-title ~ "promocion bufalinda" | attendee:Alondra
- Apply: ROUTE TO BRIEF → `01 PROJECTS/Promocion Bufalinda/Promocion Bufalinda.md`
- Scope: skill-local
- Provenance: learned YYYY-MM-DD from [[Meeting Minutes — ... — YYYY-MM-DD]] · instances: 2 · tier: auto
- Status: active

### action-phrasing ~ "JC Zerpa" | "Philippe"
- Apply: OWN AS → Waiting For (assigned fully to others)
- Scope: skill-local
- Provenance: learned YYYY-MM-DD from [[Meeting Minutes — ... — YYYY-MM-DD]] · instances: 2 · tier: auto
- Status: active
-->
