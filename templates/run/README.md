# Run: {{RUN_NAME}}

What this run does: *(one line — e.g. "TICKET-1 – TICKET-4 in my-app, two lanes")*.

- Run dir: `{{RUN}}`
- Target repo (read-only main checkout): `{{REPO}}`
- Lane agents: `{{AGENT_KIND}}`, branches from `{{BASE_REF}}` unless a lane says otherwise.

## Before launching (human)
1. Edit `run.env`: `GATES`, `INSTALL_CMD`, `DEV_CMD` / `DEV_PORT` / `DEV_READY_REGEX`, `COPY_FILES`, `AGENT_ARGS` / `CONDUCTOR_ARGS` (permission mode, extra writable dirs for `{{RUN}}` and `{{REPO}}/.git`; see the comments in `run.env`).
2. Write the ticket specs in `specs/SPECS.md`: base, branch, owned files, acceptance, and any "Conductor notes".
3. Write one lane prompt per lane in `lanes/` (copy `lanes/L1-example.md` to `lanes/<LANE>-<slug>.md`).
4. Fill the Lanes table in `CONDUCTOR.md` (delete the example row) and list cross-lane shared files.
5. Every `COPY_FILES` entry exists in `{{REPO}}`. They are copied into each worktree; this dir stores no copies.
6. Optional: write `LANE_ENV_FILE` (`KEY=value` lines, e.g. test-account creds) outside the repo or under this run dir, and `chmod 600` it.
7. Nothing listens on `DEV_PORT` (`lsof -nP -iTCP:<port> -sTCP:LISTEN` prints nothing).

## Launch
From inside a Herdr pane:
```bash
{{CONDUCTOR_HOME}}/bin/conductor start {{RUN_NAME}}
```
(Use `{{RUN}}` instead of the name if this run lives outside the default runs dir.) It runs preflight, opens a pane for the conductor, and prompts it with `CONDUCTOR.md`.

A harness's first-start dialog (folder trust) blocks an agent until answered. For a lane, `{{CONDUCTOR_HOME}}/bin/conductor prompt-lane {{RUN}} <LANE>` then re-sends the lane prompt. If it blocks the conductor (`start` exits 4), answer it in the conductor pane, then run `{{CONDUCTOR_HOME}}/bin/conductor start {{RUN_NAME}} --prompt-only`.

Watch from the Herdr sidebar: each lane appears as its own workspace. Progress:
- `out/CONDUCTOR.md` — conductor log;
- `out/lanes.tsv` — one row per launched lane (agent, workspace, pane, worktree, branch);
- `out/<LANE>/STATUS.md` — one line per ticket;
- `out/REPORT.md` — final report. The conductor finishes with `CONDUCTOR DONE {{RUN}}/out/REPORT.md`.

`{{CONDUCTOR_HOME}}/bin/conductor status {{RUN_NAME}}` prints a snapshot at any time.

## After the run (human)
1. Review the branches in the merge order from `out/REPORT.md`. Ticket worktrees are in `wt/<TICKET>/`; lane worktrees are in the `worktree` column of `out/lanes.tsv`.
2. PR bodies: `out/<LANE>/<TICKET>/PR.md`.
3. Screenshots: `out/<LANE>/<TICKET>/shots/`. They are not committed; upload them to the PR.
4. Push and open PRs yourself. Update your tracker from the report's human follow-ups; the run never writes to it.

## Cleanup
```bash
herdr worktree remove --workspace <id>             # each lane, workspace column of out/lanes.tsv
git -C {{REPO}} worktree remove {{RUN}}/wt/<TICKET> # each ticket worktree
```

## Files
- `README.md` — this file.
- `run.env` — run config.
- `CONDUCTOR.md` — conductor prompt.
- `LANE_RULES.md` — rules every lane reads first.
- `lanes/*.md` — one prompt per lane.
- `specs/SPECS.md` — ticket specs.
- `out/` — logs, statuses, PR bodies, screenshots, gates logs, report.
- `wt/` — per-ticket git worktrees.
