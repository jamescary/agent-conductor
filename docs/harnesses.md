# Harness setup

The conductor and every lane are Herdr agents started with:

```bash
herdr agent start <name> --kind <KIND> --pane <pane> -- <ARGS...>
```

`AGENT_KIND` and `AGENT_ARGS` in `run.env` configure lanes. `CONDUCTOR_KIND` and `CONDUCTOR_ARGS` configure the conductor. If `CONDUCTOR_ARGS` is empty, the conductor gets `AGENT_ARGS` only when its kind equals `AGENT_KIND`. To put a single lane on another harness, write its kind and args in the Kind cell of the Lanes table in `CONDUCTOR.md`, for example `codex --args "--sandbox workspace-write …"`. The conductor then runs `conductor launch-lane <run> <LANE> --kind <k> --args "<a>"`. A lane whose kind differs from `AGENT_KIND` and has no `--args` starts with no extra args, so it gets no permission flags.

`run.env` is sourced by bash. `$RUN` is set before sourcing and `$HOME` expands, so both work inside values. `$REPO` works only after the `REPO=` line. `AGENT_ARGS`, `CONDUCTOR_ARGS` and `--args` are split on whitespace. There is no quoting, `~` expansion or globbing, so a path in an argument can't contain spaces.

Where this page comes from: flags marked *local help* were checked against the installed CLIs in 2026-10 (herdr 0.9.5, Claude Code 2.1.304, Codex CLI 0.160.0, Gemini CLI 0.35.3, omp 18.12.6). Facts marked *observed* come from a smoke run on 2026-10-22: herdr 0.9.5, Claude Code 2.1.304 lanes, fish as the shell in the lane panes. OpenCode wasn't installed, so its facts come only from its docs. Every other fact links its source. Anything that couldn't be confirmed is marked **unverified**.

## Safety note

A bypass flag (`--dangerously-skip-permissions`, `--dangerously-bypass-approvals-and-sandbox`, `--yolo`, `--auto`, `--approval-mode=yolo`, …) turns off that harness's approvals, and in some harnesses its sandbox too. The agent can then run any command as your user, read anything your user can read, and use the network. The prompts tell agents the rules: local only, `$REPO` read-only, never print secrets. Nothing enforces them.

The conductor's approval handling is only a fallback. When a lane stops on a prompt, Herdr reports it as `blocked`. The conductor answers yes/no prompts about actions inside that lane's worktree, its ticket worktrees under `$RUN/wt/`, `$RUN` or the dev slot, if `LANE_RULES.md` allows the action. Every other prompt is recorded and the lane is left stopped for you. The conductor does not sandbox anything. Prefer a sandboxed mode with extra writable directories where the harness has one, and use bypass flags only on a machine and account where you accept that risk.

## What lanes and the conductor need

| Need | Lane | Conductor | Why |
|---|---|---|---|
| Own cwd | lane worktree (path chosen by Herdr) | `$REPO` pane | the harness's workspace root |
| Write `$RUN` | yes | yes | `out/<LANE>/…` (STATUS.md, PR.md, shots), `wt/<TICKET>/` ticket worktrees; conductor writes `out/CONDUCTOR.md`, `out/REPORT.md` and runs `$GATES` in `wt/<TICKET>/` |
| Write `$REPO/.git` | yes | `git fetch` only | `git -C $REPO worktree add` and commits in linked worktrees write the index, refs and objects under the main repo's `.git` |
| Write `${XDG_STATE_HOME:-$HOME/.local/state}/agent-conductor/` | yes, if `DEV_CMD` is set | yes (`slot status/force-release`) | dev slot lock |
| Herdr socket | yes, if `DEV_CMD` is set | yes | conductor: `herdr worktree/agent/pane` commands; lanes: `herdr pane split/run/wait-output` to run the dev server in a sibling pane |
| Network | yes | yes | `$INSTALL_CMD` runs inside `conductor worktree` under the lane's harness; dev server binds `$DEV_PORT`; browser checks |
| Image viewing | yes | yes | lanes check their screenshots; the conductor opens every PNG to verify them |
| Browser | if `DEV_CMD` is set | no | the harness's browser tool, or a Playwright script |

Every harness reads its own instruction file from the lane worktree, which is a checkout of your repo. If the repo has only `AGENTS.md`, `CLAUDE.md` or `GEMINI.md`, point the harness at that file (see each section).

## Lane startup

`conductor launch-lane` exports `LANE_ENV_FILE` into the lane pane's shell before the agent starts: it sources the file in `bash` with `set -a` and then re-executes your `$SHELL`, so it doesn't depend on which shell the pane runs. The file must therefore be valid bash (normally `KEY=value` lines), any code in it runs as you, and its path must not contain quotes or backslashes. With fish as the pane shell, the lane agent saw the exported variable (*observed*).

A permission prompt only stalls one lane. A first-start dialog is worse: workspace trust, bypass-mode acceptance or a browser consent prompt blocks the lane before it reads its prompt. Claude Code showed its folder-trust dialog ("Is this a project you created or one you trust?") for the first Herdr lane worktree of a repo it hadn't trusted. A second lane worktree of the same repo started without it, and a fresh clone showed it again (*observed*). Trust therefore appears to be per repository, not per worktree; the mechanism is **unverified**.

**How the conductor handles it.** `launch-lane` (and `start`) retry `herdr agent start` up to 4 times, 3 s apart, because the new pane may not be at a shell prompt yet, then wait up to 120 s for the agent to become ready. If it is still in any state other than `idle` or `done` (for example `blocked` on a dialog, or `unknown`), `launch-lane` still appends the lane's `out/lanes.tsv` row, does not send the lane prompt, prints the pane id and `herdr agent read <agent> --source recent-unwrapped --lines 40`, and exits 4. The conductor reads the screen. If the dialog asks to trust the lane's own worktree, it picks the trust option with `herdr agent send-keys` and runs `conductor prompt-lane <run> <LANE>`, which sends the same lane prompt `launch-lane` would have sent. Any other dialog is logged in `out/CONDUCTOR.md` and the lane is left for you. To do the same by hand, read the screen first: in the smoke run, `herdr pane send-keys <pane> down enter` accepted Claude Code's trust dialog, after which the agent went `idle` and accepted the lane prompt (*observed*). The keys depend on the harness version. Sending right after the trust dialog isn't enough: with Herdr 0.9.5 and Claude Code 2.1.304 (2026-10-22), a lane prompt sent right after accepting the trust dialog was dropped. `herdr agent prompt` reported success, but Claude Code was still initializing, the input box stayed empty, and the agent stayed `idle` with an unchanged `state_change_seq` for 240 s (*observed*). Herdr's installed agent skill says submission success doesn't prove the agent started a turn (*local help*). So `launch-lane`, `prompt-lane` and `start` settle first (the agent must be `idle` or `done` on two checks about 2 s apart, within 30 s), record `state_change_seq`, send, and confirm the turn started (`working`, or a changed `state_change_seq`, within 45 s). Unconfirmed prompts are never resent automatically: once `herdr agent prompt` has succeeded, any later failure in the confirm step (including a failed `herdr agent list`) exits 5, never 1, and the command prints the pane id, a `herdr agent read` command and a reminder that the prompt was already sent. If the prompt text sits unsent in the input box, press Enter with `herdr agent send-keys <agent> enter`; if the box is empty, run `conductor prompt-lane <run> <LANE>` (or `conductor start <run> --prompt-only` for the conductor) again.

**How to avoid it.** Start each harness once by hand with the same flags, in a scratch worktree of the same repo, and accept its one-time dialogs (bypass-mode acceptance, browser consent, login, folder trust) before a real run. Harnesses that trust per folder still ask in every new lane worktree; use their own trust settings where they have them. For Gemini CLI, trust the directory that holds Herdr's worktrees (see its section). For Claude Code, expect exit 4 and a `prompt-lane` at least for the first lane in a repo it hasn't trusted yet.

## Summary

| Harness | `--kind` | Unattended | Extra writable dirs | Instruction file | Subagents | Image viewing | Browser |
|---|---|---|---|---|---|---|---|
| Claude Code | `claude` | `--permission-mode auto` or `--dangerously-skip-permissions` | `--add-dir` | `CLAUDE.md` (`AGENTS.md` if no `CLAUDE.md`) | yes (Agent tool) | yes (Read) | Claude in Chrome (`--chrome`) |
| Codex CLI | `codex` | `--sandbox workspace-write` + `--ask-for-approval on-request`/`--approve-for-me`, or `--dangerously-bypass-approvals-and-sandbox` | `--add-dir` | `AGENTS.md` | yes (ask explicitly) | yes (`view_image`) | not in the CLI; Playwright / MCP |
| OpenCode | `opencode` | `--auto` or `permission` config | `permission.external_directory` | `AGENTS.md` (`CLAUDE.md` fallback) | yes (`general`, `explore`, …) | yes (read tool, vision model) | MCP / Playwright |
| omp | `omp` | `--approval-mode=yolo` or `--auto-approve` | `--add-dir=` | `AGENTS.md` | yes (`task`) | yes (`read`) | built-in `browser` tool |
| Gemini CLI | `gemini` | `--approval-mode yolo` (`-y`) | `--include-directories`; `SANDBOX_MOUNTS` with `-s` | `GEMINI.md` (configurable) | yes, v0.36.0+ (`generalist`) | yes (`read_file`) | `browser_agent` (opt-in) / Playwright |

## Claude Code (`claude`)

- **Unattended.** `--permission-mode` takes `acceptEdits`, `auto`, `bypassPermissions`, `manual`, `dontAsk` and `plan` (*local help*). `acceptEdits` auto-approves file edits and `mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed` inside the working directory and added directories. Every other shell command (`git commit`, `pnpm test`, `conductor …`) still prompts ([permission modes](https://code.claude.com/docs/en/permission-modes)). In the smoke run a lane in `acceptEdits` went `blocked` on every Bash command it ran (*observed*). That makes `acceptEdits` an attended mode: the conductor or you must answer nearly every step. The practical unattended choices, each with its trade-off:
  - `--permission-mode auto` approves almost everything after a background classifier check, where your plan and model offer it. It keeps a check on each action, but a model makes that call, not you, and it isn't a sandbox. It can also refuse legitimate actions (see Caveats).
  - `--dangerously-skip-permissions` (same as `--permission-mode bypassPermissions`) skips all checks, so nothing stops any command the agent decides to run. It shows a one-time warning dialog and refuses to run as root. ([permission modes](https://code.claude.com/docs/en/permission-modes))
- **Writable dirs.** Use `--add-dir $RUN`. Added directories become readable and editable under the current mode ([permissions](https://code.claude.com/docs/en/permissions#working-directories)). Don't add `$REPO`: that would let the file tools edit the read-only main checkout. `--add-dir $REPO/.git` gains nothing either. `.git` is a protected path, so file-tool writes there prompt in every mode except `bypassPermissions` ([protected paths](https://code.claude.com/docs/en/permission-modes#protected-paths)). Lanes write git metadata through `git` in Bash anyway. Without Claude's Bash sandbox, `--add-dir` scopes only Claude's own file tools, not which files Bash commands write, so git writes to `$REPO/.git` are gated only by Bash command approval. With the sandbox on, a linked worktree may write the main repo's shared `.git` (except `hooks/` and `config`). Other paths need `sandbox.filesystem.allowWrite` in `~/.claude/settings.json`, for example the slot lock dir, and the Herdr socket needs `sandbox.network.allowUnixSockets` ([sandboxing](https://code.claude.com/docs/en/sandboxing)). The exact socket path is **unverified**. `--add-dir` takes several values (`--add-dir <directories...>`, *local help*), so put it last in `AGENT_ARGS` or repeat the flag.
- **Instruction file:** `CLAUDE.md`. Since v2.1.277 it reads `AGENTS.md` instead when there is no `CLAUDE.md` in the working directory or above it ([memory](https://code.claude.com/docs/en/memory)).
- **Subagents:** built-in `general-purpose`, `Explore` and `Plan`, plus custom agents in `.claude/agents/`. Claude delegates through its Agent tool. Call them "subagents" ([subagents](https://code.claude.com/docs/en/sub-agents)).
- **Images:** Read returns PNG, JPEG, GIF and WebP as visual content. Large screenshots get downscaled ([tools reference](https://code.claude.com/docs/en/tools-reference)).
- **Browser:** `--chrome` turns on Claude in Chrome (*local help*). It needs the Chrome extension, a direct Anthropic plan and `/login` auth. It uses your browser's existing login state and pauses for you on login pages and CAPTCHAs ([chrome](https://code.claude.com/docs/en/chrome)). For unattended lanes a Playwright script is more predictable.
- **Caveats:**
  - In auto mode the classifier blocks, by default, "launching an autonomous agent loop that runs without human approval or a sandbox, such as one started with `--dangerously-skip-permissions`" ([permission modes](https://code.claude.com/docs/en/permission-modes)). A Claude conductor in auto mode can therefore be refused when it runs `conductor launch-lane` with bypass lane args. Give such a conductor bypass args too, or run it with `acceptEdits` and answer its prompts yourself, or give its lanes auto args instead.
  - Folder trust: the first lane worktree of a repo Claude hasn't trusted shows the trust dialog; a second worktree of the same repo didn't (*observed*). When it shows, `launch-lane` exits 4 and the conductor accepts it and runs `conductor prompt-lane`. See [Lane startup](#lane-startup).
  - Claude's own worktree isolation checks (blocking git commands aimed at the main checkout) apply to sessions started with `--worktree` or entered with `EnterWorktree` ([worktrees](https://code.claude.com/docs/en/worktrees#how-claude-code-enforces-isolation)). Don't pass `--worktree` in `AGENT_ARGS`: Herdr already created the lane worktree, and the lane has to run `git -C $REPO worktree add`.

```bash
AGENT_KIND="claude"
# Unattended, classifier check on each action (needs a plan and model that offer auto mode):
AGENT_ARGS="--permission-mode auto --add-dir $RUN"
# Unattended, no checks at all:
# AGENT_ARGS="--dangerously-skip-permissions --add-dir $RUN"
# Attended: edits auto-approved, nearly every shell command prompts; the conductor answers in-scope ones, you the rest:
# AGENT_ARGS="--permission-mode acceptEdits --add-dir $RUN"
# Browser checks through Claude in Chrome: add --chrome before --add-dir.
CONDUCTOR_ARGS="--permission-mode auto --add-dir $RUN"
```

## Codex CLI (`codex`)

- **Unattended.** The sandbox modes are `read-only`, `workspace-write` and `danger-full-access`. The approval policies are `on-request` and `never` ([sandboxing](https://learn.chatgpt.com/docs/sandboxing)). `--sandbox workspace-write --ask-for-approval on-request` keeps the sandbox, and escalations surface as Herdr `blocked` prompts for the conductor. `--approve-for-me` sends approval requests to automatic review under the workspace-write sandbox (*local help*; [auto-review](https://learn.chatgpt.com/docs/sandboxing/auto-review)). With `--ask-for-approval never`, a sandbox failure goes straight back to the model with no prompt. `--dangerously-bypass-approvals-and-sandbox` removes both.
- **Writable dirs.** Use `--add-dir $RUN` ("additional directories that should be writable alongside the primary workspace", *local help*). The same key in `config.toml` is `sandbox_workspace_write.writable_roots`. Network is off in workspace-write by default, and `conductor worktree` runs `$INSTALL_CMD` inside the lane, so add `-c sandbox_workspace_write.network_access=true` ([advanced config](https://learn.chatgpt.com/docs/config-file/config-advanced)). When `DEV_CMD` is set, also add the slot lock dir (`${XDG_STATE_HOME:-$HOME/.local/state}/agent-conductor`). Create it first with `mkdir -p`, since Codex may reject a missing path (**unverified**).
- **Instruction file:** `AGENTS.md`. Other names can be listed in `project_doc_fallback_filenames`. Combined instructions are capped at `project_doc_max_bytes` (32 KiB by default) ([AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)).
- **Subagents:** enabled by default in current releases. Codex spawns them only when asked directly or when `AGENTS.md`/skill instructions ask, and they inherit the sandbox policy. Call them "subagents" and ask Codex to "spawn" them ([subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)). The `multi_agent` feature is stable and on in 0.160.0 (`codex features list`).
- **Images:** the built-in `view_image` tool can inspect images already on disk ([openai/codex#12439](https://github.com/openai/codex/issues/12439), maintainer reply). The same issue reports that in sandboxed modes the model sometimes refuses unless it was given an explicit full path, so give it absolute screenshot paths. `-i/--image` attaches images to the initial prompt (*local help*).
- **Browser:** "Browser isn't available in Codex CLI" ([browser](https://learn.chatgpt.com/docs/browser)). Use a Playwright script, or a browser MCP server added with `codex mcp add` (*local help*).
- **Caveats:**
  - **`.git` is read-only in workspace-write.** "In workspace-write mode, some environments keep `.git/`, `.agents/`, `.codex/`, and `.aws/` read-only even when the rest of the workspace is writable", so `git commit` may need approval ([advanced config](https://learn.chatgpt.com/docs/config-file/config-advanced)). Lanes commit from linked worktrees whose metadata lives in `$REPO/.git/worktrees/<name>`. On Linux that gitdir is mounted read-only even when an ancestor is writable ([openai/codex#19786](https://github.com/openai/codex/issues/19786), open). It is **unverified** whether `--add-dir $REPO/.git` lifts this on macOS or Linux. Expect commit and `git worktree add` escalations with `on-request`, and failures with `never`.
  - **Herdr socket.** `codex sandbox` has an `--allow-unix-socket` option (*local help*), which implies the sandbox restricts AF_UNIX sockets by default **[inference]**. A sandboxed Codex conductor may not reach the Herdr server, and neither may a sandboxed lane that starts the dev server through `herdr pane …`. Run the conductor with `--dangerously-bypass-approvals-and-sandbox`, and check lanes first with `codex sandbox -C ~/src/my-app -- herdr agent list`. If that fails, expect an escalation prompt for every `herdr` call.

```bash
AGENT_KIND="codex"
# Sandboxed; escalations (commits, worktree add) become prompts for the conductor.
AGENT_ARGS="--sandbox workspace-write --ask-for-approval on-request --add-dir $RUN --add-dir $HOME/.local/state/agent-conductor -c sandbox_workspace_write.network_access=true"
# Unattended, no sandbox:
# AGENT_ARGS="--dangerously-bypass-approvals-and-sandbox"
CONDUCTOR_ARGS="--dangerously-bypass-approvals-and-sandbox"
```

## OpenCode (`opencode`)

Not installed locally. Everything below comes from the docs at opencode.ai/docs. The docs also announce an OpenCode v2. Whether these flags and keys carry over to v2 unchanged is **unverified**.

- **Unattended.** `--auto` auto-approves every permission request that isn't explicitly denied ([cli](https://opencode.ai/docs/cli/), [permissions](https://opencode.ai/docs/permissions/)). Without it, most permissions default to `allow`, `external_directory` and `doom_loop` default to `ask`, and reads of `.env` files are denied ([permissions](https://opencode.ai/docs/permissions/)). The permission rules gate tool calls. The docs describe no OS-level sandbox.
- **Writable dirs.** `external_directory` governs any tool that touches paths outside the start directory. Give `$RUN` and `$REPO/.git` explicit `allow` rules (patterns use `*` and `?`, last matching rule wins) ([permissions](https://opencode.ai/docs/permissions/)). Don't put that config in `$REPO`. Point `OPENCODE_CONFIG` at a file outside the repo ([config](https://opencode.ai/docs/config/)), for example `$RUN/opencode.json` (`runs/` is gitignored), and export it from `LANE_ENV_FILE`, which `launch-lane` exports into the lane shell before the agent starts. `LANE_ENV_FILE` doesn't reach the conductor, so for the conductor use the global `~/.config/opencode/opencode.json`. `OPENCODE_PERMISSION` takes inline permission JSON ([cli](https://opencode.ai/docs/cli/)).

  ```json
  {
    "$schema": "https://opencode.ai/config.json",
    "permission": {
      "external_directory": {
        "/abs/path/to/agent-conductor/runs/my-run/*": "allow",
        "/abs/path/to/my-app/.git/*": "allow"
      }
    }
  }
  ```

- **Instruction file:** `AGENTS.md`. If there is no `AGENTS.md`, it falls back to `CLAUDE.md`. More files can be listed in the `instructions` array in `opencode.json` ([rules](https://opencode.ai/docs/rules/)).
- **Subagents:** built-in `general` (full tools, for parallel units of work), `explore` and `scout` (both read-only). Primary agents invoke them through the Task tool, and the `task` permission controls which ones are allowed. Call them "subagents" ([agents](https://opencode.ai/docs/agents/)).
- **Images:** the read tool passes PNG, JPEG, GIF, WebP and PDF files to the model, up to 20 MiB ([tools, v2 docs](https://opencode.ai/v2/docs/tools/)). The model must accept image input. Several issues report reads failing on models configured without it ([anomalyco/opencode#5359](https://github.com/anomalyco/opencode/issues/5359)).
- **Browser:** no built-in browser tool is documented. Add a browser MCP server (`opencode mcp add`, [cli](https://opencode.ai/docs/cli/)) or use a Playwright script.
- **Caveats:** a per-run `OPENCODE_CONFIG` replaces nothing. It layers between the global config and the project config, and the project's own `opencode.json` still applies on top ([config](https://opencode.ai/docs/config/)).

```bash
AGENT_KIND="opencode"
AGENT_ARGS="--auto"
# In LANE_ENV_FILE:  OPENCODE_CONFIG=/abs/path/to/agent-conductor/runs/my-run/opencode.json
CONDUCTOR_ARGS="--auto"
```

## omp (`omp`)

Source: [can1357/oh-my-pi](https://github.com/can1357/oh-my-pi); flags from `omp --help` (*local help*).

- **Unattended.** `--approval-mode=yolo`, or `--auto-approve` ("auto-approve all tool calls"). The other modes are `always-ask`, `always-ask-cached` and `write`. Exactly what `write` approves is **unverified**.
- **Writable dirs.** `--add-dir=<dir>` ("add a workspace directory beyond the working directory", repeatable). No OS-level sandbox flag shows up in `omp --help`. Whether one exists is **unverified**.
- **Instruction file:** `AGENTS.md`.
- **Subagents:** the `task` tool.
- **Images:** the `read` tool returns PNG/JPEG as image content (seen in an omp 18.12 session). `omp @image.png "…"` attaches an image at start (*local help*).
- **Browser:** a built-in `browser` tool (Puppeteer). `omp browser-relay` lets it drive your own Chrome tabs (*local help*).

```bash
AGENT_KIND="omp"
AGENT_ARGS="--approval-mode=yolo --add-dir=$RUN"
CONDUCTOR_ARGS="--approval-mode=yolo --add-dir=$RUN"
```

## Gemini CLI (`gemini`)

- **Unattended.** `--approval-mode yolo` or `-y/--yolo` approves all tools. `auto_edit` approves edits only, and `plan` is read-only (*local help*). With Trusted Folders enabled (off by default), an untrusted folder disables tool auto-acceptance even in yolo ([trusted folders](https://geminicli.com/docs/cli/trusted-folders/)). Trust the directory that holds Herdr's worktrees and the run directory.
- **Writable dirs.** `--include-directories` adds workspace directories, comma-separated or repeated (*local help*). The sandbox is off unless you pass `-s/--sandbox`, set `GEMINI_SANDBOX`, or set `tools.sandbox`. When on, it confines writes to the project directory, and other paths are mounted through `SANDBOX_MOUNTS=from:to:opts`, read-only by default ([sandbox](https://geminicli.com/docs/cli/sandbox/)). If you use the sandbox, mount `$RUN`, `$REPO/.git` and the slot lock dir with `rw` through `LANE_ENV_FILE`.
- **Instruction file:** `GEMINI.md`. Set `context.fileName` (for example `["AGENTS.md", "GEMINI.md"]`) in `~/.gemini/settings.json` to read other names ([GEMINI.md](https://geminicli.com/docs/cli/gemini-md/)).
- **Subagents:** enabled from v0.36.0 ([release notes](https://github.com/google-gemini/gemini-cli/discussions/24548)). On the 0.35.3 tested here they're **unverified**. Each subagent is exposed as a tool with its own name. The built-in `generalist` has the main agent's full tool access, and custom subagents go in `.gemini/agents/` ([subagents](https://geminicli.com/docs/core/subagents/)). Call them "subagents".
- **Images:** `read_file` supports text, images, audio and PDF ([file system tools](https://geminicli.com/docs/tools/file-system/)).
- **Browser:** the `browser_agent` subagent is disabled by default. It needs Chrome 144+ and `agents.overrides.browser_agent.enabled: true`, shows a consent dialog on first use, and always asks for confirmation on form filling whatever the approval mode ([subagents](https://geminicli.com/docs/core/subagents/)). Form-based sign-in therefore blocks an unattended lane, so use a Playwright script.
- **Caveats:** the Gemini CLI docs say that for unpaid-tier and Google One users, Gemini CLI was replaced by Antigravity CLI on 2026-06-18 ([docs banner](https://geminicli.com/docs/core/subagents/)). Herdr has an `agy` kind. That it maps to Antigravity CLI is **unverified**.

```bash
AGENT_KIND="gemini"
AGENT_ARGS="--approval-mode yolo --include-directories $RUN"
CONDUCTOR_ARGS="--approval-mode yolo --include-directories $RUN"
```

## Other Herdr kinds

Any `herdr agent start --kind` value works. The CLI keeps no per-harness table. It passes `AGENT_KIND` and `AGENT_ARGS` through to Herdr, and the conductor relies only on Herdr's agent states (`idle`, `working`, `blocked`, `done`, `unknown`). Kinds in herdr 0.9.5: `pi`, `claude`, `codex`, `gemini`, `cursor`, `devin`, `agy`, `cline`, `omp`, `mastracode`, `opencode`, `copilot`, `kimi`, `kiro`, `droid`, `amp`, `grok`, `hermes`, `kilo`, `qodercli`, `qwen`, `letta`, `maki`, `muse`.

For any harness, work out: the flag or config that removes approval prompts, how to make `$RUN` (and `$REPO/.git`, if it sandboxes git) writable, which instruction file it reads, whether it can view a PNG from disk, and what browser tooling it has. A harness that can't view images still works. Lanes and the conductor mark screenshots `unverified` in STATUS.md, PR.md and REPORT.md.
