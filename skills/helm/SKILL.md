---
name: helm
description: Drive the `helm` CLI. Use whenever the user mentions `helm` or any `helm <area>` command.
---

# helm CLI

`helm` is a daemon-backed CLI: the CLI is thin; the work happens in a long-lived daemon process
(`helm daemon`) that owns sessions, workspaces, cron, channels, providers, skills, and plugins.
Most "helm doesn't work" problems are really "the daemon isn't running" or "the wrong daemon
state dir is in scope".

## Discover, don't guess

The CLI is self-describing — ask it instead of relying on prose (including this document):

1. `helm daemon status` — always first. Exit code `2` from any `helm` invocation means daemon
   unreachable; `helm start` brings up daemon + server (background, idempotent). After a CLI
   upgrade or config change, `helm restart` re-spawns both.
2. `helm <area> --help`, then `helm <area> <sub> --help --json` for the machine schema: flags,
   args, output shape, exit codes, examples. That schema is the source of truth for flags —
   never guess a flag from memory.
3. Pass `--json` whenever you parse output. `Output: bare` = one JSON object; `jsonl` =
   newline-delimited (parse with `jq -c` per line); `stream` = SSE-style chunks on stdout
   (`session send`, the `*tail` commands) — don't buffer the whole stream.
4. `helm daemon logs`, or merge all processes by time:
   `cat $HELM_HOME/logs/*.log | jq -s 'sort_by(.ts)'`.
5. `helm doctor [--fix] [--json]` for daemon health checks.

## Exit codes (uniform)

`0` ok · `1` error (validation/business/assertion) · `2` daemon unreachable · `130` SIGINT.
Extras: `eval` uses `1` = assertion failed, `2` = infra error; `issue move` adds `3` = conflict,
`4` = upstream daemon unreachable. `helm daemon stop|restart` and the `helm restart` umbrella
exit `2` when loops or cron runs are in flight (stderr lists them; `--force` aborts and
proceeds). Sessions/PTYs never trip that gate.

## Command map

<!-- BEGIN GENERATED: command-map — regenerate with `bun packages/cli/scripts/gen-skill-command-map.ts`; do not edit by hand -->

| Area         | Purpose                                                                                       | Subcommands                                                                                                                                     |
| ------------ | --------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `start`      | Bring up daemon + server (background, idempotent).                                            | —                                                                                                                                               |
| `restart`    | Bounce daemon + server (background).                                                          | —                                                                                                                                               |
| `daemon`     | Manage the helm daemon.                                                                       | `start` `stop` `status` `logs` `reset` `restart` `pair` `tokens`                                                                                |
| `terminals`  | Manage the persistent pty-host that keeps terminals alive across daemon restarts              | `status` `kill-all`                                                                                                                             |
| `server`     | The optional HTTPS proxy + pairing for remote/browser access to daemons.                      | `start` `stop` `restart` `status` `stats` `add` `ca` `ls` `rm`                                                                                  |
| `session`    | Live agent conversations.                                                                     | `ls` `show` `rm` `rename` `archive` `pin` `new` `send` `attach` `cancel` `approve` `tail` `search` `branch` `rewind` `stats` `routing` `export` |
| `workspace`  | Manage Helm workspaces — named bundles of projects + agent policy + a meta-chat session.      | `ls` `show` `new` `update` `rm` `chat` `stats` `overview` `tail`                                                                                |
| `cron`       | Manage workspace-scoped scheduled tasks (cron).                                               | `new` `ls` `show` `update` `rm` `enable` `disable` `run` `logs`                                                                                 |
| `channel`    | Manage chat-platform channels (telegram/dingtalk/wechat) per workspace.                       | `status` `configure` `enable` `disable` `rm` `restart` `pair`                                                                                   |
| `doctor`     | Run health checks against the daemon.                                                         | —                                                                                                                                               |
| `eval`       | Run eval specs against ephemeral sessions.                                                    | `run` `ls` `show`                                                                                                                               |
| `issue`      | Manage workspace-scoped issues / requirements.                                                | `new` `ls` `show` `update` `rm` `move` `start`                                                                                                  |
| `loop`       | Drain a workspace queue of todo issues sequentially                                           | `start` `stop` `status`                                                                                                                         |
| `issue-sync` | Bidirectional sync between Helm issues and external trackers (GitHub, Linear)                 | `bind` `list` `unbind` `now` `test`                                                                                                             |
| `provider`   | Manage LLM providers (Anthropic-compat).                                                      | `ls` `templates` `add` `rm` `enable` `disable` `default` `test`                                                                                 |
| `setup`      | First-run / rerunnable onboarding wizard (provider, workspaces, per-project config, channel). | `provider` `workspace` `channel`                                                                                                                |
| `skill`      | Install and manage Claude Agent skills (git, npm, clawhub, well-known URL sources).           | `install` `ls` `show` `rm` `enable` `disable` `update`                                                                                          |
| `cc-plugin`  | Install and manage Claude Code plugins from registered marketplaces.                          | `install` `ls` `info` `rm` `enable` `disable` `update` `marketplace`                                                                            |
| `plugin`     | Manage Helm runtime plugins (TS modules registering hooks/tools/commands inside the daemon).  | `list` `info` `enable` `disable` `install` `uninstall` `update`                                                                                 |
| `update`     | Update helm-agent to the latest version (or a specific tag/version).                          | —                                                                                                                                               |
| `util`       | Agent-bridge utilities.                                                                       | `send-media`                                                                                                                                    |

<!-- END GENERATED: command-map -->

## The mental model

A **workspace** is the policy + project bundle; a **session** is a live agent conversation.
Each workspace has one or more **meta-chat** sessions (`helm workspace chat`); the **active**
one is the inbound-channel (Telegram/DingTalk/WeChat) target and the CLI default. Cron,
channels, issues, and loops are workspace-scoped; providers and skills are global. A new
session inherits policy defaults from its workspace; CLI flags then whole-field-replace per
field.

Branch vs rewind: **branch** (`session branch`) when you want a clean policy/state slate or to
keep both branches; **rewind** (`session rewind`) to undo recent turns on the same session.

## State dirs

- `HELM_HOME` overrides `~/.helm` (the dev desktop uses `~/.helm-dev`). Pair with
  `CLAUDE_CONFIG_DIR` to fully isolate state (e.g. E2E tests).
- Dev shells often alias `helm` to `bun <repo>/packages/cli/src/bin.ts` — confirm with
  `which helm` and `helm -v` before debugging.

## Destructive commands — confirm with the user first

- `helm workspace rm <name>` cascades: it also deletes all of the workspace's meta-chat
  sessions.
- `helm daemon reset` blows away daemon state — treat it like `rm -rf $HELM_HOME`.
- `helm terminals kill-all` — interactive terminals live in a persistent pty-host and survive
  `helm daemon stop/restart`, `helm restart`, and upgrades; `kill-all` is the ONLY way to
  intentionally end them.

## Gotchas the CLI won't tell you

- Daemon restart wipes in-memory session policy (`--allowed-tool` / `--permission-mode` /
  `--skill` chosen at creation). The session resumes without them — re-pass on the next
  `session new`, or rely on workspace defaults.
- `helm session send <id> -` reads the prompt from stdin — use it for multi-line or piped
  content instead of `"$(cat file.md)"`.
- `helm util send-media` only works inside a running agent turn (it reads `$HELM_SESSION_ID`).
- WeChat outbound is best-effort: it needs a `context_token` harvested from an inbound message,
  held in memory and lost on daemon restart — a proactive cron/loop push may silently not
  arrive until the recipient messages the bot again.
- `helm workspace update --default-deliver-to` clobbers any existing `failureDestination`
  (whole-object set; the desktop UI preserves it). To keep both from the CLI, hand-edit
  `workspace.json`.
- `--rm-project` matches the stored path casing exactly (`--project`/`--add-project` are
  canonicalized on darwin/win32; removal is not). If removal no-ops, check
  `helm workspace show <name>` for the stored casing.
- The `codex` provider template uses local ChatGPT OAuth (`codex login` first); passing
  `--api-key` with it is a hard error. Its `--fast` flag opts into the priority tier and is
  billing-affecting.
- `helm-agent` publishes `latest` and `canary` npm dist-tags. Plain `helm update` on a canary
  build auto-tracks the canary channel — switch back to stable with `helm update --tag latest`;
  track canary explicitly with `helm update --canary` (mutually exclusive with `--tag`).
- `helm update --json`: restart progress prints human lines on stdout — parse the JSON envelope
  from the **last** non-empty stdout line.
- `helm daemon status` human output distinguishes stale state (dead pid — `start` self-heals)
  from a hung daemon (pid alive but `/health` dead — needs restart); the `--json` envelope does
  not carry that distinction.
- Issue sync (GitHub/Linear) is polling-only, last-write-wins by `updatedAt`; deleting a local
  issue unlinks without touching the remote. `issue move` strips remote sync links — re-bind in
  the target workspace.
- A few surfaces are daemon-route/UI-only with no CLI: background-task stop/background
  (`POST /sessions/:id/tasks/...`) and AskUserQuestion resolution
  (`POST /sessions/:id/questions`). Read the matching `docs/designs/` doc before driving those
  routes directly.

## Reaching `helm server` from another device

Default bind is loopback (`127.0.0.1:8220`). By IP: `helm server start --bind 0.0.0.0:8220`.
By domain / Tailscale MagicDNS name: add `--hostname <name>` (repeatable) — it adds a cert SAN,
allows that Origin, and prints a per-name `#code=` login URL. On loopback no code is needed by
default (`HELM_LOOPBACK_NOAUTH`, default on; set `0` on shared machines). Trust the self-signed
CA on clients via `helm server ca` (PEM to stdout; `--path` prints `~/.helm/tls/ca.pem`), or
bring your own cert with `--tls-cert`/`--tls-key` (e.g. from `tailscale cert`). `--bind` /
`--hostname` / `--tls-*` persist to `start-args.json` and are replayed by restarts and
`helm update`.

## When you're stuck

1. `helm daemon status`. Always.
2. `helm <area> <sub> --help --json` for the exact schema.
3. `helm daemon logs` (or the merged-log one-liner above) to see what the daemon actually did.
4. Wedged session: `helm session cancel <id>`, confirm idle via `helm session tail <id>`,
   escape bad state with `helm session branch <id>`.
5. Wedged daemon: `helm daemon restart` (exit 2 with an in-flight summary → re-run with
   `--force`).
