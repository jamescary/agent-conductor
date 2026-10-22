<!--
Starter lane prompt. Copy it to `lanes/<LANE>-<slug>.md` once per lane (e.g. `lanes/L2-billing-copy.md`),
replace every `<…>` and every TICKET-n, and delete this comment. Delete this example file if no lane uses it.
LANE ids match [A-Za-z0-9]+. Two lanes must never own the same file unless it is listed under "Shared files"
in both lane files with non-overlapping hunks.
-->
# Lane L1: <short lane title>

LANE = `L1`. Read `{{RUN}}/LANE_RULES.md` first and follow it. Then run the tickets below.

- **Base for ticket branches:** `{{BASE_REF}}` unless a ticket below says otherwise.
- **Lane worktree:** your current directory (the conductor gave you its path). Use it to read code and run read-only commands. Don't implement tickets here; each ticket gets its own worktree under `{{RUN}}/wt/<TICKET>/`.
- **Specs:** find a ticket's section with `grep -n '^## TICKET-1:' {{RUN}}/specs/SPECS.md`, then read the whole section. Its Conductor notes override everything else, except the Hard limits in `LANE_RULES.md`.

## Files you own
<List the paths and globs this lane may change. Be specific; anything not listed is off limits.>
- `<src/features/example/**>`
- `<src/components/ExampleWidget/**>`
- new `<src/lib/example-helpers.ts>` and its test file

## Shared files (exact hunks only)
<Files more than one lane must touch. Name the hunk this lane may change; touch nothing else in the file. Delete this section if none.>
- `<src/routes.ts>`: only the `<example>` route entry.
- `<src/i18n/en.json>`: only keys under `<example.*>`.

## Tickets, in this order
Each ticket gets its own branch, worktree, commit, PR.md and STATUS line.

1. **TICKET-1**: <one-line summary of the change>. Branch `<feat/TICKET-1-short-slug>`, base `{{BASE_REF}}`.
   - <Key acceptance points or decisions the lane must not reinterpret.>
   - Screenshots: before and after of <the exact screen and state>.
2. **TICKET-2** (stacked on TICKET-1): <one-line summary>. Branch `<fix/TICKET-2-short-slug>`, base `<feat/TICKET-1-short-slug>`.
   - Start it only after TICKET-1 is committed. If you amend TICKET-1 later, rebase this branch onto it and re-run its gates.
   - <Key acceptance points.>
   - Screenshots: none, not user-visible; browser `n.a.`. (Or: after only, of <the exact screen and state>.)

## Test data
<Which test account/team to use (credentials come from the lane env, never from this file), what data each ticket needs, and how to create it through the UI if missing. Record any data you create in PR.md.>
- Use only the test account and team named in the specs or env.
- <TICKET-1 needs …. Create it through the UI if it is missing.>

## Out of scope and stop conditions
- <Behavior or areas this lane must not change, even if it looks related.>
- <Hot or risky areas: if a change would touch <…>, stop and mark the ticket `blocked: <reason>`.>
- If a ticket needs a file another lane owns, record `[blocked] needs <file> (owned by <lane>)` and move on.

## Merge notes for PR.md
- <Shared files and the hunks touched; which other lanes touch nearby code; stacking order.>
