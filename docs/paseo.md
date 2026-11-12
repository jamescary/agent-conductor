# Paseo backend

`BACKEND="paseo"` in `run.env` (set by `conductor new --backend paseo`) runs the conductor and every lane as [Paseo](https://paseo.sh/docs) agents instead of Herdr agents. The run directory, specs, lane files, ticket worktrees, dev slot, `verify` and the report are the same. `conductor new` keeps only the Paseo instructions in the rendered `CONDUCTOR.md`, `LANE_RULES.md`, `README.md` and `run.env`.

Where this page comes from: facts marked *observed* come from a smoke run on 2026-11-12 with Paseo 0.15.4 (desktop daemon) and omp lanes (`omp/anthropic/claude-haiku-4-5`, mode `full`): `new --backend paseo`, `launch-lane` (env from `LANE_ENV_FILE` reached the lane), `watch`, `status`, `prompt-lane`, `start`, `start --prompt-only`, a `blocked` lane answered with `paseo permit allow`, `worktree` and `verify`; the conductor ran the Paseo preflight from `CONDUCTOR.md`. Mode ids come from the daemon's provider inspection. Claude and Codex lanes weren't run end to end.

## Requirements

- Paseo 0.15.4 or later, its daemon running, and `paseo` on `PATH` for whoever runs `conductor` (you, and the conductor agent). `paseo ls` must work.
- At least one provider that `paseo provider ls` reports `available`.
- `jq`, `git`, `bash` as for Herdr. `conductor start` runs from any shell that reaches the daemon, including a Paseo agent's shell; it doesn't have to run inside Paseo.

## How each step maps

| Step | Herdr | Paseo |
|---|---|---|
| Preflight | `herdr agent list` reachable | `paseo ls --json` succeeds |
| Conductor placement | pane split from yours, or `start --pane <id>` | new local workspace at `$REPO` titled `<run-name>-conductor`, or `start --workspace <id>` |
| Lane workspace | `herdr worktree create --branch --base --label` | `paseo workspace create --isolation worktree --mode branch-off --new-branch <b> --base <ref> --worktree-slug <run-name>-<lane> --title <run-name>-<lane>` |
| Agent identity | Herdr name `<run-name>-<lane>` | title `<run-name>-<lane>` plus labels `agent-conductor.run=<run-name>` and `agent-conductor.role=<LANE>` (or `conductor`); the agent id is the handle in `lanes.tsv` and every command |
| Start + prompt | `herdr agent start`, settle, `herdr agent prompt` | `paseo run --no-wait --provider <kind> --workspace <ws> … "<prompt>"`: one call creates the agent with its first prompt |
| Re-prompt (`prompt-lane`, `start --prompt-only`) | `herdr agent prompt` | `paseo send <id> --no-wait --prompt "<prompt>"` |
| `LANE_ENV_FILE` | exported into the pane shell before the agent starts | each variable it sets passed as `paseo run --env KEY=value` |
| States | `herdr agent list` `agent_status` | `paseo inspect`: pending permission request → `blocked`; `running`/`initializing` → `working`; `idle`, `error`, `closed` as is; archived or deleted → `gone` |
| State change marker | `state_change_seq` | `UpdatedAt` (moves while the agent works, stays put while it is idle; *observed*) |
| Read an agent | `herdr agent read <name>` | `paseo logs <id> --tail 40` |
| Answer an approval | `herdr agent send-keys` | `paseo permit ls --json`, `paseo permit allow <id> <request-id>` |
| Dev server | Herdr pane: `split`, `run`, `wait-output`, `send-keys ctrl+c`, `close` | Paseo terminal: `terminal create --workspace <ws> --cwd <worktree>`, `send-keys -l "$DEV_CMD"` + `Enter`, poll `terminal capture -S` for `DEV_READY_REGEX`, `send-keys C-c`, `kill` (*observed*) |
| Cleanup | `herdr worktree remove --workspace <id>` | `paseo workspace archive <id>` (removes the Paseo-owned worktree) |

The lane prompt also names the lane's Paseo workspace id, which the lane needs for `paseo terminal create`. Lanes launched by the conductor are its Paseo subagents (Paseo reads `PASEO_AGENT_ID` from the conductor's shell) and appear as tabs in their own workspaces.

## Kind and args

`AGENT_KIND`, `CONDUCTOR_KIND` and `--kind` take a Paseo provider or `provider/model` (`claude`, `claude/opus`, `codex/gpt-5.4`, `omp/anthropic/claude-opus-5-5`). `AGENT_ARGS`, `CONDUCTOR_ARGS` and `--args` are extra `paseo run` flags, split on spaces: `--mode <id>`, `--thinking <id>`, `--model <id>`. The same rule as Herdr applies: a lane on a non-default kind gets `AGENT_ARGS` only through an explicit `--args`.

Paseo has no extra-writable-dir flag. Lanes write `$RUN/out`, `$RUN/wt` and (through git) `$REPO/.git`, all outside the lane worktree, so a sandboxed or asking mode turns those writes into permission requests: the lane goes `blocked` until the conductor allows an in-scope request or you answer it.

Mode ids reported by Paseo 0.15.4 on this machine:

| Provider | Unattended | Others |
|---|---|---|
| `claude` | `bypassPermissions` | `auto` (classifier), `acceptEdits`, `default` (always ask), `plan` |
| `codex` | `full-access` | `auto-review`, `auto` (default permissions; sandboxed) |
| `omp` | `full` | `write` (write approval), `ask` |

For other providers, run `paseo agent mode <agent-id> --list` on one of their agents. The trade-offs per harness in [harnesses.md](harnesses.md) still apply: a bypass mode lets the agent run anything as you, and the prompt rules are instructions, not enforcement. The Claude caveat holds too: an `auto`-mode conductor may be refused when it launches bypass-mode lanes.

```bash
BACKEND="paseo"
AGENT_KIND="claude"
AGENT_ARGS="--mode bypassPermissions"
CONDUCTOR_ARGS="--mode bypassPermissions"
```

## Differences to know

- **No exit 4.** Paseo drives providers directly, so there are no first-start dialogs (folder trust, bypass acceptance) and no agent without a prompt. `launch-lane` and `start` exit 0, 1 or 5.
- **Exit 5** means the agent was created (or messaged) but no turn was confirmed within 45 s. Read `paseo logs <id> --tail 40`: a `[Message received]` entry means the agent has the prompt; only re-run `prompt-lane` / `start --prompt-only` when it doesn't.
- **Provider errors look idle.** A provider that fails to authenticate ends its turn and goes `idle`, not `error` (*observed*: Claude with an expired login). `watch` reports the lane settled; the conductor reads the timeline and records it.
- **Permission requests may say little.** `paseo permit ls` showed an omp request only as `OMP select` with description `-` (*observed*). Read the lane's timeline (`paseo logs <id> --tail 40`, or `--filter permissions`) before allowing anything.
- **Duplicate names.** `launch-lane` and `start` refuse when an unarchived agent of the run already has the title. Archive it (`paseo archive <id>`) after review to reuse the name. Archived agents read as `gone`.
- **Worktree setup runs too.** If `$REPO` has a `paseo.json` with `worktree.setup` or a `.worktreeinclude`, Paseo applies them to the lane worktree as well as `COPY_FILES` and `INSTALL_CMD`, so dependencies may install twice. Whether setup has finished when `paseo workspace create` returns is **unverified**. Ticket worktrees under `$RUN/wt/` are plain `git worktree add` and get only `COPY_FILES` and `INSTALL_CMD`.
- **`LANE_ENV_FILE` values are stored.** `paseo run --env` puts them in the agent's record (`~/.paseo/agents/<project>/<agent-id>.json`, key `launchEnv`; the Paseo home is `0700`) until the agent is deleted, and on the `paseo` command line while it runs (*observed*). The file is sourced by bash under `set -a`, and every variable it assigns or exports is passed, even when your shell already has the same value: the daemon, not your shell, spawns the lane. `HOME`, `PATH`, `USER`, `LOGNAME`, `SHELL`, `TMPDIR`, `LANG` and `TERM` are passed only when the file changes them. Variables your shell exports that the file doesn't touch are never passed. The path may contain any character.
- **Archiving deletes worktrees.** `paseo workspace archive` on a lane workspace runs the repo's `worktree.teardown` and removes the worktree once no other workspace uses it. Branches and ticket worktrees under `$RUN/wt/` stay. Nothing in a run archives anything.
- **Finish notices.** Because lanes are the conductor's subagents, Paseo may post their completions into the conductor's conversation. `CONDUCTOR.md` tells the conductor to treat them as hints and keep using `conductor watch`.
