# servicenow-vtb-lane-move

Claude skill. Moves every card in one lane to another on the **Endpoint
Services** Visual Task Board in ServiceNow, also called the DE or Desktop
Engineering board.

Covers the two sprint-close moves:

1. **Done!** to **Victory Board**
2. **Work in progress** to **Sprint Backlog**

It counts the cards on both sides of a move, shows you the count, and waits for
explicit approval before writing anything. Approval is requested separately for
each move.

Moves every card in the lane regardless of assignee, task class, or state. It
is not filtered to one person's work. On demand only, never scheduled.

## Install

The repo root is the skill directory:

```bash
git clone https://github.com/jgerdom-hbs/servicenow-vtb-lane-move.git \
  ~/.claude/skills/servicenow-vtb-lane-move
```

For claude.ai, download the `.skill` bundle from
[Releases](https://github.com/jgerdom-hbs/servicenow-vtb-lane-move/releases)
and upload it in settings.

## Sharing

Shareable across EPS. Works for any standard ITIL user, not just one person.
