---
name: servicenow-vtb-lane-move
description: >-
  Move every card in one lane to another on the "Endpoint Services" Visual Task Board (VTB) in
  ServiceNow (also called the DE VTB or Desktop Engineering board). Covers two sprint-close moves:
  (1) "Done!" to "Victory Board" and (2) "Work in progress" to "Sprint Backlog". Counts cards on
  both sides of a move, shows the count, and waits for explicit approval before writing —
  separately for each move. Moves every card in the lane regardless of assignee, task class, or
  task state; not filtered to one person's work. Use whenever a teammate mentions closing out a
  sprint, clearing the Done lane, moving completed work to the Victory Board, sprint rollover, or
  moving Work in Progress cards back to Sprint Backlog, even if they don't name the skill directly.
  Always runs against the Endpoint Services board — never ask which board. On-demand only, never
  scheduled. Shareable across the EPS team: works for any standard ITIL user, not just one person.
---

# ServiceNow VTB Lane Move (HBS TSS — Endpoint Services board)

## What this does

A two-part sprint-close workflow on the Endpoint Services Visual Task Board:

1. **"Done!" → "Victory Board"** — completed work moves to the celebration lane.
2. **"Work in progress" → "Sprint Backlog"** — unfinished work carries into the next sprint.

Both moves act on **lane membership only**. Every card sitting in the source lane moves, regardless
of who it's assigned to, its task class (Private Task, Incident, Universal Request, Universal Task,
Catalog Task, Incident Task, etc.), or whether the underlying task record is actually closed. This
mirrors what the team does by hand when dragging cards between columns. Not filtering by assignee,
task type, or task state is a deliberate design choice for this skill — don't add filtering later
without checking with the team first.

Built for any Endpoint Services team member as a **standard ITIL user** — no ServiceNow admin rights
needed. All reads and writes go through the Table API from inside a live, already-authenticated
ServiceNow tab, so the runner's own session cookie rides along the way it does for any in-browser
request. No token or credential is ever extracted, stored, or reused outside that tab.

This file is self-contained. Every call below was verified live against production on 2026-08-31
(moved 71 cards Done!→Victory Board in a real run). Do not re-derive a constant unless the "If a
lane sys_id changes" section below says to, and do not consult other files in this skill directory.

## Verified constants

| Constant | Value |
|---|---|
| Instance | `https://hbs.service-now.com` (production only, never the test instance) |
| Board name | `Endpoint Services 🍕` (owned by Jessie Taylor) |
| Board sys_id | `12ce8fb81b753f0025d642e58d4bcbc0` |

### Lane sys_ids

| Lane (exact name) | sys_id |
|---|---|
| Backlog | `dece8fb81b753f0025d642e58d4bcbc0` |
| Sprint Backlog | `56ce8fb81b753f0025d642e58d4bcbc2` |
| Work in progress | `9ece8fb81b753f0025d642e58d4bcbc2` |
| Testing | `cfee4f381b753f0025d642e58d4bcb58` |
| Implement | `d4fe83f81b753f0025d642e58d4bcb44` |
| Done! | `0cea2f38db71bb007a381f83059619cb` |
| Victory Board | `68995789dbc63f007a381f8305961997` |

**Lane names are exact, including punctuation and case.** It's `Done!` (with the exclamation
point) and `Work in progress` (lowercase "p" in "progress"), not "Work In Progress." Match the
literal string.

These sys_ids came from the board's DOM (`id="lane_<sys_id>"` on each `li.v-lane-header-item`),
**not** from a card's `lane` reference field. That matters: deriving a lane's sys_id from one of
its cards only works if the lane currently has at least one card. The DOM lookup works even when a
lane is empty (e.g., right after this skill clears out "Done!"), so it's the more durable method.
See "If a lane sys_id changes" below to re-run it.

## Non-negotiable guardrails

- **On-demand only.** Never schedule, background, or repeat this without the runner explicitly asking each time.
- **Whole lane, every assignee, every task type.** Do not filter by assignee, task class, or task state. That's a deliberate design choice — don't "improve" it by adding filters.
- **Always show lane counts on both sides of a move and get an explicit go-ahead before writing — separately for each of the two moves.** Approving Done!→Victory Board is not approval for Work in progress→Sprint Backlog; ask again.
- **Never touch anything but `lane` and `order` on `vtb_card`.** Don't modify the underlying task, tags, or `swim_lane`.
- **No credential handling.** Never ask for, store, or write out a session token, cookie, or API key, in chat or in a file.
- **This is a whole-team write, not a personal one.** It touches every technician's cards, not just the person running it. Make sure the runner actually looked at the counts before approving — this can't be undone by the skill itself.

## Ground rules for every call

1. **Origin.** `javascript_tool` runs in the active tab. From any other origin there is no session cookie and CORS blocks the request, which reads as an auth failure. Navigate to the board URL first.
2. **`X-UserToken` on every request.** `window.g_ck` is the CSRF token. GETs tolerate its absence; writes do not. Send it always so read and write paths are identical.
3. **`sysparm_display_value=all`** on every read. It returns `{value, display_value}` per field, which removes any guessing about labels versus backend codes.
4. **Filter client-side, never with dot-walked queries.** One wide read plus in-memory filtering is deterministic. Dot-walked *fields* in `sysparm_fields` are fine; dot-walked `sysparm_query` filters on `vtb_card` are not reliable.
5. **Never treat `200` + `result: []` as a permissions problem.** See Response triage.
6. **No identity check needed.** This skill isn't scoped to one person, so there's no need to confirm the runner's own sys_id — any authenticated EPS teammate can run it as themselves.

## Shared helper

```js
const HOST = 'https://hbs.service-now.com';
const H = { 'Accept': 'application/json', 'Content-Type': 'application/json', 'X-UserToken': window.g_ck };

async function snFetch(path, method = 'GET', body) {
  const r = await fetch(HOST + path, {
    method,
    headers: H,
    credentials: 'include',
    body: body ? JSON.stringify(body) : undefined
  });
  const ct = r.headers.get('content-type') || '';
  if (!ct.includes('json')) {
    return { status: r.status, nonJson: true, snippet: (await r.text()).slice(0, 200) };
  }
  return { status: r.status, body: await r.json() };
}
const dv = (row, f) => (row[f] && row[f].display_value) || '';
```

A `nonJson` result means wrong origin or an expired session. Stop and re-run preflight rather than retrying.

## Workflow

### Step 0 — Preflight

Navigate to `https://hbs.service-now.com/$vtb.do?sysparm_board=12ce8fb81b753f0025d642e58d4bcbc0`, then confirm:

```js
{
  origin: location.origin,                          // must be "https://hbs.service-now.com"
  hasToken: typeof window.g_ck !== 'undefined' && !!window.g_ck
}
```

### Step 1 — Read the board, count "Done!" and "Victory Board"

```js
const BOARD = '12ce8fb81b753f0025d642e58d4bcbc0';
const FIELDS = ['sys_id','lane','order','task','task.number','task.short_description','task.sys_class_name','task.state','task.assigned_to'].join(',');
const res = await snFetch(
  '/api/now/table/vtb_card'
  + '?sysparm_query=' + encodeURIComponent('board=' + BOARD + '^removed=false')
  + '&sysparm_fields=' + FIELDS
  + '&sysparm_display_value=all&sysparm_limit=2000'
);
const rows = res.body.result || [];
const done = rows.filter(r => dv(r,'lane') === 'Done!');
const victory = rows.filter(r => dv(r,'lane') === 'Victory Board');
// report: done.length, victory.length
```

Present: *"Done!: N cards. Victory Board: M cards."* Optionally break `done` down by state (Open /
Closed / etc.) if the runner wants context — the underlying task's state doesn't affect whether a
card moves, so this is informational only. **Stop and wait for explicit approval before Step 2.**

### Step 2 — Move "Done!" → "Victory Board" (only after approval)

```js
const doneSorted = done.slice().sort((a,b) => Number(dv(a,'order')) - Number(dv(b,'order')));
const vbOrders = victory.map(r => Number(dv(r,'order'))).filter(n => !isNaN(n));
const maxVbOrder = vbOrders.length ? Math.max(...vbOrders) : -1;   // -1 so an empty lane starts at order 0

const t0 = performance.now();
const results = [];
for (let i = 0; i < doneSorted.length; i++) {
  const card = doneSorted[i];
  const r = await snFetch(
    '/api/now/table/vtb_card/' + card.sys_id.value + '?sysparm_fields=sys_id,lane,order&sysparm_display_value=all',
    'PATCH',
    { lane: '68995789dbc63f007a381f8305961997', order: String(maxVbOrder + 1 + i) }
  );
  results.push({ number: dv(card,'task.number'), title: dv(card,'task.short_description'), ok: r.status === 200, status: r.status });
}
const elapsedSec = Number(((performance.now() - t0) / 1000).toFixed(2));
```

Report: attempted / succeeded / failed counts, `elapsedSec` (this is API write time only — it does
not include the read, the diff, or the runner's approval time), and the number + title of every
card moved so the runner can spot-check the board. List any failures by number and status code.

### Step 3 — Read the board again, count "Work in progress" and "Sprint Backlog"

Same pattern as Step 1, filtering on `dv(r,'lane') === 'Work in progress'` and `'Sprint Backlog'`.
Present the counts. **Stop and wait for a second, separate approval before Step 4** — do not treat
approval of the Done! move as approval for this one.

### Step 4 — Move "Work in progress" → "Sprint Backlog" (only after approval)

Same mechanics as Step 2: sort the source lane by current `order`, compute append order from the
target lane's current max (or -1 if empty), loop the PATCH, target lane sys_id
`56ce8fb81b753f0025d642e58d4bcbc2`, time the loop, report counts + elapsed seconds + moved titles.

### Step 5 — Final summary

One combined report: both moves' attempted/succeeded/failed counts and elapsed times. Remind the
runner to glance at the board to confirm.

## If a lane sys_id changes

If a write 400s/404s referencing an unrecognized lane, or the board's visible lane list doesn't
match the table above (a lane was renamed, removed, or added), re-derive the current sys_ids from
the DOM rather than guessing:

```js
[...document.querySelectorAll('li.v-lane-header-item')].map(el => ({
  sysId: el.id.startsWith('lane_') ? el.id.slice(5) : null,
  name: (el.querySelector('[class*="lane-name"], [class*="lane-title"], .v-lane-header-name, h3, .title')?.textContent || '').trim()
}))
```

This works even for a lane with zero cards, unlike reading a lane's sys_id off one of its cards.
Update the constants table above once confirmed.

## Response triage

| Symptom | Cause | Action |
|---|---|---|
| Response is not JSON, or is HTML | Wrong origin, or session redirected to login | Re-run preflight. Navigate to the board URL, confirm `location.origin`. |
| `401` | Missing or stale `X-UserToken` on a write | Re-read `window.g_ck` from the current page and retry once. |
| `403` | A real ACL denial for that user's role on `vtb_card` | Report it, don't retry with a different query. Some standard ITIL users may not have write access to every board's cards — check with a board owner if this happens consistently. |
| `200` with `result: []` on `vtb_card` | Almost always the query, not permissions | Confirm the `board=` filter is present and the sys_id matches the constant. |
| PATCH references an unrecognized lane | Lane sys_id constant is stale | Re-derive via the DOM method above. |
| A lane's count is 0 | Could be genuinely empty, or removed entirely | The DOM lookup still finds an empty-but-existing lane; a truly removed lane needs the runner to check the board directly. |

## Tables that do not work over the Table API

| Table | Result | Workaround |
|---|---|---|
| `vtb_board` | `200`, empty | Board sys_id is a hardcoded constant. |
| `vtb_lane` | `200`, empty | Lane sys_ids come from the board's DOM (see above), not this table. |

## Known limitations

- There is no reverse reference from `vtb_task` to its card, so reads always start from `vtb_card`.
- This is a write operation across the **whole team's** board, not one person's cards — there's no
  scoped "undo." Reversing a move means running the skill again in the opposite direction or fixing
  lanes by hand.
- The elapsed-time metric measures only the PATCH loop, not the read/count/approval steps around it.

## Example triggers

- **"Move Done to Victory Board"** → Step 1 (count), wait, Step 2 (move) after approval. Stop there unless asked to continue.
- **"Close out the sprint" / "run the sprint close" / "roll the sprint over"** → full sequence: Step 1, approve, Step 2, Step 3, approve, Step 4, Step 5.
- **"How many cards are in Done! right now?"** → Step 1 only, no write.
- **"Move Work in Progress to Sprint Backlog"** → Step 3 (count), wait, Step 4 (move) after approval.

## Changelog

- **2026-08-31** — Created and verified live: Done!→Victory Board move (71 production cards, 0
  failures, ~37.6s API time), lane sys_id resolution via board DOM (works for empty lanes), PATCH
  timing methodology.
