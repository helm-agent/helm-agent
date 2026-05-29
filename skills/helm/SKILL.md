---
name: helm
description: Drive the `helm` CLI. Use whenever the user mentions `helm` or any `helm <area>` command.
---

# helm CLI

`helm` is a daemon-backed CLI. The CLI is thin; the work happens in a long-lived daemon process (`helm daemon`) that owns sessions, workspaces, cron, channels, providers, skills, and plugins. Internalize this model first — most "helm doesn't work" problems are really "the daemon isn't running" or "the wrong daemon state dir is in scope".

## Always do this first

1. `helm daemon status` — confirm the daemon is up. Exit code `2` from any `helm` invocation means daemon unreachable; start it with `helm start` (brings up daemon + server, background, idempotent). After a CLI upgrade or config change, run `helm restart` to bounce both daemon and server (not idempotent; always re-spawns).
2. `helm <area> --help` — discoverability. Every area (`session`, `workspace`, `cron`, `channel`, `provider`, `skill`, `plugin`, `eval`, `issue`, `loop`, `setup`, `daemon`, `server`, `util`) has its own help. Add `--help --json` to get a machine schema instead of prose. The top-level `helm start` / `helm restart` umbrellas have no `--help` of their own; see `helm --help`.
3. Default to `--json` whenever you need to parse output. JSONL is used for list outputs (`helm session ls`, `helm cron logs`). Stream outputs from `helm session send` / `session tail` are SSE-style chunks on stdout.

## Exit codes (uniform across the CLI)

- `0` — success
- `1` — error (validation, business-logic, assertion failures)
- `2` — daemon unreachable (don't retry without restarting the daemon)
- `130` — SIGINT

The `eval` subcommands also use `1` for "assertion failed" and `2` for "infra/setup error".

`helm daemon stop`, `helm daemon restart`, and `helm restart` (the umbrella) ALSO exit `2` when the daemon has any running loops or in-flight cron runs. Stderr lists what's in-flight. Pass `--force` to abort and proceed anyway. Sessions/PTYs are intentionally not counted (every open chat tab would otherwise trip the gate). `helm server stop/restart` is unaffected — the server hosts no loops/crons.

## State dirs

- `HELM_HOME` overrides `~/.helm`. Pair with `CLAUDE_CONFIG_DIR` if you need to fully isolate state (e.g., E2E tests).
- Dev shells often alias `helm` to `bun /Users/chencheng/Projects/Helm/packages/cli/src/bin.ts`. Confirm with `which helm` and `helm -v` before debugging.
- Logs live under `$HELM_HOME/logs/`. Merge across processes by time: `cat ~/.helm/logs/*.log | jq -s 'sort_by(.ts)'`.

## Command map (what each area is for)

| Area         | What it manages                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Top-of-mind subcommands                                                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `daemon`     | The background process itself. Speaks HTTPS by default with the shared CA at `~/.helm/tls/ca.pem`; BYO via `--tls-cert <path>` `--tls-key <path>` on `start`. The Electron desktop app's embedded daemon opts back to plain HTTP via `HELM_DAEMON_TLS=0` set by the supervisor — no operator action needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | `start`, `stop`, `restart`, `status`, `logs`, `reset`, `pair`, `tokens`                                                                    |
| `server`     | The optional `helm server` (HTTPS proxy + pairing). Default bind: `127.0.0.1:8220` with random-port fallback if 8220 is taken. Override with `--bind <host>:<port>`. Speaks HTTP/2 over TLS to bypass Chrome's 6-per-origin H1 socket cap. Auto-mints a leaf via `mkcert` if installed, otherwise self-signed; trust anchor at `~/.helm/tls/ca.pem`. Auto-leaf SANs include the bind host (so `--bind <tailscale-ip>:<port>` "just works"); on first non-loopback bind the selfsigned anchor rotates once to extend SANs (TOFU-pinned clients must re-run `server add`). BYO via `--tls-cert <path>` `--tls-key <path>`. `start` backgrounds by default; pass `--foreground` to attach. `restart` preserves the bearer token for ~5 min. **Persistent start args:** `--bind` / `--tls-cert` / `--tls-key` are persisted to `~/.helm-server/start-args.json` on successful start and replayed by `helm update` / a no-arg `helm server restart`. **`add` accepts `--ca <path>`** to override the default TOFU peer-chain harvest with an operator-supplied CA; without `--ca`, the SHA-256 fingerprint of the pinned chain is printed to stderr for out-of-band verification. | `start`, `stop`, `restart`, `status`, `add [--ca <path>]`, `ls`, `rm`                                                                      |
| `start`      | Top-level umbrella: bring up daemon + server (both detached). No subcommands. Idempotent — re-running is safe and short-circuits already-running processes. `--bind` / `--tls-cert` / `--tls-key` are forwarded to **server only** (daemon stays on its loopback default). `--foreground` is rejected with a hint; run `helm daemon start --foreground` / `helm server start --foreground` in separate shells if you need attached lifecycles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | (no subcommands)                                                                                                                           |
| `session`    | Live agent conversations                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | `new`, `send`, `tail`, `ls`, `show`, `cancel`, `approve`, `attach`, `fork`, `rewind`, `export`, `search`, `rename`, `archive`, `pin`, `rm` |
| `workspace`  | Named bundles of projects + policy + meta-chat session                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | `new`, `ls`, `show`, `update`, `chat`, `tail`, `stats`, `overview`, `rm`                                                                   |
| `cron`       | Workspace-scoped scheduled agent runs                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | `new`, `ls`, `show`, `update`, `enable`, `disable`, `run`, `logs`, `rm`                                                                    |
| `channel`    | Chat-platform adapters per workspace (telegram / dingtalk / wechat)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | `status`, `configure`, `enable`, `disable`, `restart`, `pair`, `rm`                                                                        |
| `provider`   | LLM providers (Anthropic-compatible)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | `ls`, `templates`, `add`, `test`, `default`, `enable`, `disable`, `rm`                                                                     |
| `skill`      | Claude Agent skills installed into the daemon                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | `install`, `ls`, `show`, `enable`, `disable`, `update`, `rm`                                                                               |
| `cc-plugin`  | Claude Code plugins + marketplaces (installer for `.claude-plugin/plugin.json` repos)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | `install`, `ls`, `info`, `enable`, `disable`, `update`, `rm`, `marketplace`                                                                |
| `plugin`     | Helm runtime plugins (TS modules registering hooks/tools/commands inside the daemon)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | `list`, `info`, `enable`, `disable`                                                                                                        |
| `setup`      | First-run / rerunnable onboarding wizard                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | (interactive) or `provider` / `workspace` / `channel`                                                                                      |
| `eval`       | Run eval specs against ephemeral sessions                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | `run`, `ls`, `show`                                                                                                                        |
| `issue`      | Workspace-scoped issues with agent context                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | `new`, `ls`, `show`, `update`, `start`, `rm`, `move`                                                                                       |
| `loop`       | Workspace background loops                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | `start`, `stop`, `status`                                                                                                                  |
| `issue-sync` | Bidirectional issue sync with GitHub Issues + Linear (polling, KISS plain fetch)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | `bind`, `list`, `unbind`, `now`, `test`                                                                                                    |
| `util`       | Agent-bridge helpers callable **only from inside a running agent** (`$HELM_SESSION_ID`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | `send-media`                                                                                                                               |
| `update`     | Self-update the `helm-agent` npm package                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | (top-level, with `--tag`, `--canary`, `--dry-run`, `--pm`)                                                                                 |
| `doctor`     | Daemon health checks                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | (top-level, with `--fix`, `--json`)                                                                                                        |

## The mental model

A **workspace** is the policy + project bundle. A **session** is a live agent conversation. Workspaces have a singleton **meta-chat** session (`helm workspace chat`). Cron, channels, and issues are workspace-scoped. Providers and skills are global.

When you create a session with `helm session new --workspace dev`, the session inherits policy defaults from the workspace, and CLI flags then whole-field-replace per field. **Policy flags are session-fixed** — they persist across daemon re-inits within a daemon lifetime, but are lost on daemon restart. If you need a clean policy slate, fork the session.

## Common workflows

### First-run setup

The fastest path is the wizard:

```
helm setup
```

Or drive it non-interactively (single workspace, no per-project config):

```
helm setup --provider anthropic --api-key $ANTHROPIC_API_KEY --workspace dev \
  --project /path/to/repo --telegram-bot-token $TG_BOT_TOKEN
```

Pin model defaults at setup time (both flags validate the id against the resolved provider's catalog and exit 1 with `usage` on miss, leaving config untouched):

```
helm setup --provider anthropic --api-key $ANTHROPIC_API_KEY --workspace dev \
  --default-model claude-sonnet-4-6 \
  --workspace-model claude-sonnet-4-6
```

- `--default-model <id>` — writes top-level `HelmConfig.defaultModel` (used by every session unless overridden). Requires `--provider` OR a pre-existing default provider; mutually exclusive with `--no-default-provider` and unsupported with custom-URL providers.
- `--workspace-model <id>` — writes `sessionPolicy.model` on the workspace. Validated against the workspace's resolved provider (`--workspace-provider` → existing `sessionPolicy.provider` → `cfg.defaultProvider`).
- `--workspace-provider <id>` — writes `sessionPolicy.provider` on the workspace. The daemon validates that the provider id is configured.

For WeChat headless, write the config and pair separately from a TTY:

```
helm setup --provider anthropic --api-key $ANTHROPIC_API_KEY --workspace dev \
  --project /path/to/repo --wechat --skip-key-test
helm channel pair dev wechat
```

Piece-by-piece flow if the user wants control:

```
helm setup provider           # configure provider + key, optionally as default
helm setup workspace          # create default workspace, optionally attach one project
helm setup channel            # attach a Telegram / DingTalk / WeChat channel to a workspace
```

### Talking to a session

```
helm session new --cwd /repo --workspace dev --permission-mode acceptEdits
# prints SessionSummary; capture .id (use --json for clean parsing)
SID=$(helm session new --cwd /repo --json | jq -r '.id')

helm session send "$SID" "fix the parser bug" --effort high
echo "summarise this file" | helm session send "$SID" -    # '-' reads from stdin
helm session send "$SID" "review" --attach att_1 --attach att_2:image/png
helm session tail "$SID"                                    # subscribe to SSE
helm session cancel "$SID"                                  # cancel active run
```

Approvals: when a tool needs human approval, the SSE stream emits an `approval` event with an approvalId. Resolve with `helm session approve <approvalId> <allow|deny>`.

AskUserQuestion: when the agent invokes the `AskUserQuestion` tool against a UI-bound session (web UI / desktop / meta-chat sessions reserved with `supportsInteractiveQuestions: false`), the SSE stream emits `tool.question_required`. There is no `helm session` CLI for resolving these from the terminal — the dialog is UI-only. The daemon route `POST /sessions/:id/questions` with body `{ toolCallId, answers: ({ labels: string[] } | null)[] }` commits the answers; `answers[i]` corresponds to `questions[i]` from the event. On resolve, `tool.question_resolved` carries `answers: { question, header, chosen: string | null }[]` so UI clients can render a chip with the chosen labels (`null` = dismissed) — see `docs/designs/2026-05-18-askuser-error-chip.md`. CLI-only sessions (created via `helm session new` without the flag) keep the SDK's native AskUserQuestion behavior — empty answers, run completes.

### Listing and slicing JSON output

```
helm session ls --json                          # JSONL
helm session ls --json --fields id,title,cwd    # project only the fields you need
helm session ls --cwd /repo --limit 20 --sort title
helm session search "auth bug" --deep           # body search, not just title
```

`--fields` accepts comma-separated dotted paths and is the right way to keep parsing cheap.

Each row in `helm session ls --json` carries optional `pinned?: boolean` and `archived?: boolean` flags from the per-session sidecar at `~/.helm/session-meta/<sessionId>.json`. Flip them from the terminal with `helm session archive <id> [<id>...] [--unarchive]` and `helm session pin <id> [<id>...] [--unpin]` (variadic, JSONL one line per id, per-id failures don't abort the batch). The underlying daemon route `PATCH /sessions/:id` with `{ title?, pinned?, archived? }` is also still callable directly. Deleting a session removes the sidecar. Rows also carry optional `jsonlPath?: string` — the absolute path to the SDK-written transcript on disk, honoring `CLAUDE_CONFIG_DIR`. Absent when the daemon can't locate the file. The renderer uses it for "Copy JSONL path"; CLI callers can pipe it into `jq` / `cat` without re-encoding the cwd.

Rows also carry optional `currentModel?: string` and `currentProviderId?: string` — the currently-bound model and provider for the session. Sourced from the active session map (live SDK binding) when running, with fallback to the meta sidecar's `lastProviderId` / `lastModel` (written by the daemon after every successful provider resolution). Survives daemon restart for idle sessions. Use them to render a routing chip on session rows without an extra fetch.

### Switching provider/model mid-session

`helm session routing <id> --provider <id> --model <name>` sets the new default routing for the session. Applies after the current turn finishes (no in-flight cancel). No-op when both fields match the current binding. Direct daemon call: `PATCH /sessions/:id/routing` with body `{ providerId, model }` → `{ ok: true, changed: boolean }`.

For one-off per-turn overrides, send a message with `providerOverride` (and/or the existing `modelOverride`) in the request body: `POST /sessions/:id/messages { text, providerOverride?, modelOverride? }`. The session reverts to its pinned routing on the next turn.

The daemon route `GET /sessions` also accepts `?dirs=<a>,<b>,…` (comma-separated, URL-encoded) as a multi-root alternative to `?cwd=`. The two are mutually exclusive (400 `cwd_and_dirs_exclusive`); `dirs` is soft-capped at 32 entries. The sessions rail uses this to push the active workspace's project filter down to the SDK; CLI callers keep using `--cwd`.

### Exporting a transcript as Markdown

`helm session export <id>` renders a human-readable transcript. Default destination is stdout.

```
helm session export sess_abc                                 # to stdout
helm session export sess_abc --last 50                       # only the last 50 messages
helm session export sess_abc --since <uuid>                  # messages after a uuid
helm session export sess_abc -o transcript.md                # write to a file
helm session export sess_abc -o transcript.md --with-attachments
#                                                              writes transcript.md plus transcript.md.attachments/att-1.png …
```

- `--last N` and `--since <uuid>` are mutually exclusive (same semantics as `helm session show`).
- `--with-attachments` requires `-o`; without `-o` it exits 1 with a `usage` error envelope.
- Tool calls render as collapsed `<details>` blocks (GitHub-friendly); images render as a placeholder by default, or as sidecar files with `--with-attachments`.
- The route response is JSON: `{ markdown, attachments?: [{filename, mime, bytesBase64}] }`. When driving the route directly, replace `{{ATTACHMENTS_DIR}}` in `markdown` with the directory you write the attachments to.

### Aggregated stats

```
helm session stats <id>                  # prose summary
helm session stats <id> --json           # SessionStats envelope
helm workspace stats <name>              # rolled-up aggregate
helm workspace stats <name> --per-session
helm workspace stats <name> --json --per-session
```

Stats are computed from the on-disk SDK transcript: tokens (input/output/cache-read/cache-creation), per-model breakdown, message counts (user / assistant / tool_use / tool_result), tool-use frequency, duration (prefers `system.turn_duration` events; falls back to wall-clock), and estimated cost (USD) from a hardcoded `MODEL_PRICES` table. `costUSD` is a **partial sum** across priced models — the total for whichever models resolved. To detect incompleteness, branch on `missingModels.length > 0`, not on `costUSD === null` (kept nullable in the schema for forward compat; runtime always emits a number). The aggregator excludes the `<synthetic>` SDK sentinel from pricing so it never appears in `missingModels`. Model-id normalization tries: direct hit → strip `<provider>/` prefix + dots→dashes → strip `-YYYYMMDD` date suffix. Assistant-line de-duplication is applied (the SDK splits one API response across N content-block lines with identical `usage`). Workspace scope: sessions whose `cwd ∈ ws.projects[]` plus `ws.metaSessionId`, deduped by sessionId.

`SessionStats` also carries optional `contextUsage: { used, max, percent }` — the prompt-side occupancy of the model's context window for the LAST assistant turn (`used = input + cacheRead + cacheCreation`). `max` is looked up from a static table keyed by normalized model id (same normalization as `costUSD`). Omitted when the model is unknown or no assistant turn has happened yet — clients should branch on `contextUsage !== undefined` before reading.

`SessionStats` also carries optional `cacheWindow: { anchorTs, ttlSeconds: 300 | 3600 }` — backing the live prompt-cache TTL countdown ("cache 51m07s / COLD"). `anchorTs` is the most recent assistant turn's timestamp (cache last refreshed); `ttlSeconds` is read from the most recent assistant with a non-zero `cache_creation` bucket (`ephemeral_1h_input_tokens > 0 → 3600`, else `→ 300`; legacy flat `cache_creation_input_tokens` folds to 300). Omitted when there is no assistant turn or no observed `cache_creation` bucket — clients branch on `cacheWindow !== undefined` and compute `remaining = ttlSeconds − max(0, (now − anchorTs)/1000)` (≤ 0 ⇒ COLD).

### Workspace overview (counts/inventory)

```
helm workspace overview <name>                # prose rollup
helm workspace overview <name> --json         # WorkspaceOverview envelope
helm workspace overview <name> --by-project   # include issues.byProject map
```

`workspace overview` is the counts-focused sibling of `workspace stats` (which is cost-focused). The envelope is `{ overview: WorkspaceOverview }` where `WorkspaceOverview` covers: `projects[]` with `exists` flags, `metaSessionId`, `sessions { total, metaIncluded }`, `issues { total, byStatus { triage, todo, in_progress, done, cancelled }, bySource { human, agent }, byProject? }`, `crons { total, enabled, disabled, lastRun { name, ts, status } | null, failuresLast24h }`, `loops { running }`, and `channels { telegram, dingtalk, wechat }` (enabled booleans only — no secrets). Daemon-direct callers can pass `?by_project=1` on `GET /workspaces/:name/overview` to add the `issues.byProject` path→count map. Responses are served from a 5s in-process TTL cache keyed by workspace name (cron-history tails are the only non-O(1) work); clients listening to issue/cron/loop SSE streams should re-fetch overview on those events rather than expect their own SSE feed. Failure tallies (`failuresLast24h`) count rows where `status ∈ {failed, timeout, cancelled_by_crash}`; `cancelled_by_shutdown` is a clean stop and `skipped_overlap` did not run, so neither contributes.

### Workspaces

```
helm workspace new dev --add-project /repo --description "Day-job repo" --permission-mode acceptEdits
helm workspace new mono --add-project /a --add-project /b --skills-all --agent reviewer
helm workspace show dev --json                  # full envelope, includes SOUL.md path + exists flag
helm workspace chat dev "what's in flight?"     # talk to the workspace meta-chat
helm workspace tail dev                         # SSE stream of meta-chat
```

`--no-soul` disables SOUL.md bootstrap and identity injection on a workspace at creation time. To toggle identity injection on an existing workspace, use `helm workspace update <name> --inject-soul` / `--no-inject-soul` (parity with `--inject-helm-context` / `--inject-child-results`).

**Meta-session provider · model** — the workspace's meta-chat resolves provider + model as `ws.metaProvider ?? ws.sessionPolicy.provider` and `ws.metaModel ?? ws.sessionPolicy.model`. Edits take effect on the next meta-session reservation (no live mid-turn flip — restart the meta-chat to apply, e.g. via the `/new` slash command).

```
helm workspace update dev --meta-provider anthropic       # explicit meta-provider override
helm workspace update dev --unset-meta-provider           # inherit from sessionPolicy.provider
helm workspace update dev --meta-model claude-opus-4-7    # (already documented; unchanged)
```

Setting `--meta-provider` to a concrete id whose catalog does not contain the workspace's current `metaModel` auto-clears `metaModel` to `null` so the dropdown stays coherent. `--unset-meta-provider` (inherit) does NOT auto-clear `metaModel`.

**Tabs · open behavior** — controls the desktop Sessions view when the user clicks a session from the sidebar / Cmd+K / issue detail sheet and the active tab is already an `agent-session`:

```
helm workspace update <name> --tab-open-behavior <replace|insertAfter>
helm workspace update <name> --unset-tab-open-behavior   # resets to 'replace'
```

- `replace` (default) — new session takes over the active tab.
- `insertAfter` — new session is opened as a new tab immediately after the active tab.

When the active tab is a terminal, the new session/terminal is **always** inserted after the active tab regardless of this setting — so terminals are never silently clobbered.

On case-insensitive filesystems (darwin/win32), `--project` / `--add-project` values are rewritten to the canonical on-disk casing before they're stored in `workspace.projects[]`. `--rm-project` is **not** canonicalized so wrong-case stale entries can still be removed by exact match. If `--rm-project` produces no removal, inspect `helm workspace show <name>` to confirm the stored casing and re-run with that value.

#### Issue-execution config (loop + issue runs)

Workspaces carry five `IssueExecutionConfig` leaves — `extraPromptTemplate`, `preScript`, `postScript`, `permissionMode`, `loopAuto` — used by `helm loop` and `helm issue start`. Each leaf can be set per-project, with a workspace-level default as fallback.

**Per-project leaves** (apply to one project; flag value is `<absPath>=<value>`):

```
helm workspace update dev --set-project-issue-prompt-template /repo='Title: {{title}}\n\n{{description}}'
helm workspace update dev --set-project-issue-pre-script /repo='bun install'
helm workspace update dev --set-project-issue-post-script /repo='bun run check'
helm workspace update dev --set-project-issue-permission-mode /repo=acceptEdits
helm workspace update dev --set-project-loop-auto /repo=true

# unset variants take just the path:
helm workspace update dev --unset-project-issue-pre-script /repo
helm workspace update dev --unset-project-loop-auto /repo
```

Multiple `--set-project-*` flags can be combined in one PATCH; the project path must already be in `workspace.projects[]`.

**Workspace-level defaults** (apply to every project that has no per-project override; flag value is a bare scalar):

```
helm workspace update dev --default-issue-prompt-template 'Title: {{title}}\n\n{{description}}'
helm workspace update dev --default-issue-pre-script 'bun install'
helm workspace update dev --default-issue-post-script 'bun run check'
helm workspace update dev --default-issue-permission-mode acceptEdits
helm workspace update dev --default-issue-loop-auto true        # value is 'true' or 'false'

# unset variants take no value:
helm workspace update dev --unset-default-issue-pre-script
helm workspace update dev --unset-default-issue-loop-auto
```

Resolution cascade per leaf is: per-project value → workspace default → `null` / `undefined` / `'bypassPermissions'` / `false` depending on the leaf. `permissionMode` additionally falls back to `sessionPolicy.permissionMode` between the workspace default and the hard `'bypassPermissions'` default.

**Default notification delivery** (workspace-level; applies to every cron run + every loop iteration event):

> Note: the CLI **clobbers** any existing `failureDestination` field when `--default-deliver-to` runs (cron-parity, whole-object set). The desktop UI **preserves** `failureDestination`. If you need to set `failureDestination` from the CLI, hand-edit `workspace.json`.

```
helm workspace update dev --default-deliver-to webhook:https://hooks.example.com/x
helm workspace update dev --default-deliver-to telegram:12345 --default-deliver-best-effort
helm workspace update dev --default-deliver-to dingtalk:<conversationId>
helm workspace update dev --unset-default-delivery
```

**Loop-delivery filters** (workspace-level + per-project; gate which loop SSE events reach the delivery sink — the UI loop view is unaffected and keeps showing every event):

```
# Event-type gate. Defaults to 'all' when unset.
helm workspace update dev --default-issue-delivery-events completion           # only issue_finished
helm workspace update dev --default-issue-delivery-events completion-and-stopped
helm workspace update dev --default-issue-delivery-events none                 # mute everything
helm workspace update dev --unset-default-issue-delivery-events

helm workspace update dev --set-project-delivery-events /repo=completion
helm workspace update dev --unset-project-delivery-events /repo

# Status gate — applies only to issue_finished events; other event types pass through. Defaults to 'any' when unset.
# 'failures-only' counts failed_agent / failed_resolve / cancelled_by_crash; cancelled_by_user / cancelled_by_shutdown / skipped are NOT failures.
helm workspace update dev --default-issue-delivery-status-filter failures-only
helm workspace update dev --default-issue-delivery-status-filter success-only
helm workspace update dev --unset-default-issue-delivery-status-filter

helm workspace update dev --set-project-delivery-status-filter /repo=failures-only
helm workspace update dev --unset-project-delivery-status-filter /repo
```

Both gates must pass for an event to deliver. With both leaves unset, behavior is identical to pre-filter — every loop event reaches the configured `defaultDelivery` sink.

### Cron, channels, issues, loop

```
helm cron new dev --schedule "0 9 * * *" --prompt "daily standup digest"
helm cron new dev digest --cwd /repo --cron "0 9 * * *" --instruction "..." --permission-mode plan \
  --deliver-to dingtalk:<conversationId>            # per-task DingTalk delivery
helm cron run <cronId>                          # async by default
helm cron logs <cronId>

helm channel status                             # all workspaces, all platforms
helm channel status --json                      # ApiError-shaped envelope; each
                                                # PlatformReport may include an
                                                # optional `linkState`: 'open' |
                                                # 'reconnecting' for adapters
                                                # with a live-transport concept
                                                # (DingTalk websocket). Absent =
                                                # open or not applicable.
helm channel configure dev telegram --token $TG_BOT_TOKEN
helm channel configure dev telegram --peer-bot helpful_bot --respond-on-mention true
                                                # peer-bot opt-in + group mention gate (Telegram)
helm channel restart dev telegram               # idempotent; useful after network blips
helm channel pair wechat                        # QR-pairing flow (wechat only)

helm issue new dev --title "ship 0.0.13"
helm issue start <issueId>                      # creates a session pre-loaded with the issue
helm issue move 42 --from-workspace dev --from-daemon localA \
                   --to-daemon prod --to-workspace main
                                                # Move an issue to another daemon/workspace.
                                                # Preserves attachments + audit log; resets
                                                # status to triage, sessionIds/cronNames to [].
                                                # Requires helm-server running. Exit codes:
                                                # 0 ok · 1 validation/business · 2 helm-server
                                                # unreachable · 3 conflict (source changed
                                                # during move) · 4 upstream daemon unreachable.

helm loop start dev                             # fire-and-forget
helm loop status dev
```

### Issue sync (GitHub Issues + Linear)

Helm can mirror local issues to/from external trackers. v1 is polling-only (no webhooks — the daemon has no public ingress) and syncs three fields: title + description + status. Auth is a PAT (GitHub) or API key (Linear) stored under `~/.helm/secrets.json` at `secrets["<workspace>/issue-sync/<source>/token"]`. One token per (workspace, source) is shared across all bindings.

```bash
# Bind a project to a GitHub repo (prompts/uses HELM_ISSUE_SYNC_TOKEN, or --token)
helm issue-sync bind dev \
  --source github --project /repos/foo \
  --owner acme --repo foo \
  --token ghp_xxx

# Bind to a Linear team
helm issue-sync bind dev \
  --source linear --project /repos/foo \
  --team-id 11111111-2222-... \
  --interval-ms 120000

helm issue-sync list dev                # list bindings
helm issue-sync list dev --project /repos/foo --json
helm issue-sync test dev <bindingId>    # probe auth + scope
helm issue-sync now  dev <bindingId>    # force a pull right now
helm issue-sync unbind dev <bindingId>  # delete binding + strip local remoteLinks
```

Default status maps (overridable via `PATCH /workspaces/:ws/issue-sync/bindings/:id`):

- **GitHub** — `open ↔ triage`, `closed ↔ done`. Local `todo` / `in_progress` push as `open`.
- **Linear** — `Backlog ↔ triage`, `Todo ↔ todo`, `In Progress ↔ in_progress`, `Done ↔ done`, `Cancelled ↔ cancelled`.

Local delete = unlink-on-delete (the remote is NOT touched). Conflict policy is last-write-wins by `updatedAt`. Unmapped remote states fall back to `triage` and write a `syncError` onto the link. PRs are filtered out of GitHub list results (the `pull_request` field discriminator).

### Peer-bot communication (Telegram)

Telegram bots can message other bots. Helm gates peer-bot inbound on a per-workspace opt-in list, and exposes an agent tool the workspace meta-session can use to originate messages to peer bots.

CLI flags on `helm channel configure <ws> telegram`:

- `--peer-bot <username>` — repeatable. Allow a peer bot (by `@username`, lowercased; leading `@` stripped) to DM/group this workspace's bot. Inbound from a bot whose username is not on the list is dropped with `[channel-telegram] peer-rejected-not-opted-in`.
- `--respond-on-mention <true|false>` — in group chats, drop messages that don't `@mention` this bot's own username. DMs are unaffected. Defaults to `false` (current behavior preserved).

Headless equivalents on `helm setup` / `helm setup channel`:

- `--telegram-peer-bot <username>` (repeatable)
- `--telegram-respond-on-mention <true|false>`

Agent tool (workspace meta-session only): `mcp__helm_workspace__send_to_peer_bot({ username, text })`. Returns `{ ok: true, chatId, messageId }` on success, `{ ok: false, error }` on refusal. Error codes:

- `peer_not_opted_in` — username not on `peerBotOptIns`
- `telegram_not_configured` — no running Telegram adapter for this workspace
- `peer_send_failed` — grammy raised; raw message under `detail`

The returned `chatId` is the `@username` target string, not a numeric chat id — do not expect equality with future inbound `chatId` values. Outbound carries a `[helm:hop=N]` loop-guard trailer where N = `(last observed inbound hop for this peer) + 1` (or 1 with no prior inbound). Inbound at `hop > 3` is dropped with `[channel-telegram] peer-loop-dropped`; duplicate `(chatId, messageId)` is dropped with `[channel-telegram] peer-dupe-dropped`. Peer-bot replies route into the workspace meta-session; there is no dedicated child session per peer.

### Providers and skills

```
helm provider templates                         # what built-ins exist
helm provider add anthropic --api-key $KEY
helm provider add --template codex              # Codex via local ChatGPT OAuth — no API key
helm provider add --template codex --fast       # Codex with priority tier (billing-affecting)
helm provider add --template claude-code --claude-code-path /opt/claude/bin/claude  # pin the SDK's Claude Code CLI binary
helm provider test anthropic                    # 1-token probe; cheap auth check
helm provider default anthropic                 # writes HelmConfig.defaultProvider

```

**Codex provider note.** The `codex` template is special: it uses local ChatGPT OAuth from `~/.codex/auth.json` (or `$CODEX_HOME/auth.json`) instead of an API key. Prereq: run `codex login` before `helm provider add --template codex`. Passing `--api-key` to `helm provider add --template codex` exits 1 with a hard error. The Helm daemon embeds the `codexthropic` proxy in-process (lazy-started on first resolution, listening on a random localhost port) — no separate process to run. `helm provider test codex` round-trips through the embedded server and exercises both OAuth refresh and upstream reachability.

**Codex `--fast` flag.** Opts into priority tier (billing-affecting); maps to `service_tier: 'priority'` on the upstream Codex request. Persisted per provider — multiple codex providers can have independent settings. Only templates with `supportsFast` accept `--fast` (today: codex only); the daemon returns `400 validation_failed` otherwise. Flip from the CLI by re-running `helm setup` (the codex branch asks "Use fast (priority) tier?" with the current value as the default), via the desktop Settings → Providers toggle, or with a direct `PATCH /config/providers/<id>` body `{"fast": false}`. Toggling invalidates live sessions bound to the provider.

**`--claude-code-path <path>` flag.** Optional per-provider path to the Claude Code CLI binary the Claude Agent SDK should spawn (the SDK's top-level `pathToClaudeCodeExecutable` option — NOT an env var). Applies to ALL provider types (claude-code, codex, plugin, custom). Pass-through string; no fs existence check (the SDK fails loudly at spawn if wrong). Set it at `add` time, or edit later via `PATCH /config/providers/<id>` body `{"pathToClaudeCodeExecutable": "/path"}` (set/overwrite only — there is no clear-to-unset). `helm provider ls --json` echoes the field under `pathToClaudeCodeExecutable` so you can confirm what's set. Changing it invalidates live sessions bound to the provider.

```
helm skill install owner/repo                   # GitHub shorthand
helm skill install owner/repo#main
helm skill install npm:@scope/pkg@1.2.3
helm skill install clawhub:my-skill
helm skill install /local/path --scope project --cwd $(pwd)
helm skill install https://example.com          # well-known (.well-known/agent-skills/index.json)
helm skill install wellknown:https://example.com  # force well-known route
helm skill install git:https://git.sr.ht/~u/r   # `git:` prefix for self-hosted git hosts
helm skill disable <name>                       # renames SKILL.md → SKILL.md.disabled
helm skill update <name>                        # re-fetch from recorded source
```

**Helm runtime plugins** (separate from `helm cc-plugin`; these are TS modules the daemon dynamically imports — hooks, providers, channels, slash commands):

```
helm plugin list                                                # discovered + gating state
helm plugin info @owner/helm-plugin-foo                         # scoped npm package keys are supported
helm plugin install @owner/helm-plugin-foo                      # fetch latest from default registry
helm plugin install @owner/helm-plugin-foo@1.2.3                # exact version
helm plugin install @owner/helm-plugin-foo@beta                 # dist-tag
helm plugin install @owner/helm-plugin-foo --registry https://x # private registry
helm plugin update @owner/helm-plugin-foo                       # noop when up to date; refetch via sidecar
helm plugin uninstall @owner/helm-plugin-foo                    # rm dir + drop from plugins.enabled/disabled
helm plugin enable @owner/helm-plugin-foo                       # toggle without (re)installing
helm plugin disable <name>
```

`install` writes a `.helm-install.json` sidecar recording `{ pkg, registry, spec, version, installedAt }`. Default `--registry` resolves to `configStore.npmRegistry`, falling back to `https://registry.npmjs.org`. Tarball fetch + extract is direct HTTPS — no `npm` on PATH required. Two-rename atomic swap rolls back to the prior install on failure. Auto-enables the key in `~/.helm/config.json` (strips a matching `disabled` entry too); the daemon picks it up on next restart. Restart hint is printed after every install / update. `--json` envelope on every subcommand carries machine-readable `outcome` codes (`installed`, `replaced`, `updated`, `noop`, `uninstalled`, or error codes `not-installed`, `no-sidecar`, `sidecar-mismatch`, `not-a-plugin`, `network`, `extract`, `validation-failed`, `internal`). See `docs/designs/2026-05-27-helm-plugin-npm-install.md` and `docs/designs/2026-05-27-helm-plugin-scoped-npm.md`.

### Health checks (`helm doctor`)

Fail-fast probe against the daemon. One check today (`no_provider`); more
will be added.

```
helm doctor                  # run all checks
helm doctor --fix            # apply each rule's non-interactive remediation
helm doctor --json           # single-line JSON envelope on stdout
```

JSON envelope:

```json
{
  "ok": true,
  "checks": [
    {
      "id": "no_provider",
      "ok": true,
      "severity": "error",
      "message": "provider configured",
      "fixed": true
    }
  ]
}
```

Each check carries `id`, `ok`, `severity` (`error` | `warn`), `message`,
optional `hint` (only when failing), optional `fixed: true` (only when
`--fix` flipped a failing check to passing on the same run).

Exit codes: `0` = all error-severity checks pass; `1` = any error-severity
check fails or snapshot probe errored; `2` = daemon unreachable.

The `no_provider` rule fires when no enabled provider qualifies — a
provider qualifies if it's enabled AND either has a stored key OR is a
`noApiKey` template (`codex`, `claude-code`). `--fix` POSTs
`/config/bootstrap-claude-code` with `{ setDefault: true }` (single
transactional daemon write that ensures the `claude-code` provider
exists, enables it, and sets it as default). See
`docs/designs/2026-05-23-claude-code-provider.md`.

`preventAppNap` is another `/config/defaults` boolean (default `false`). When
toggled `true` on macOS, the daemon spawns `caffeinate -i -s -w <pid>` to
suppress App Nap and system sleep during long-running work; toggling is
hot-reload (no daemon restart). There is **no CLI flag** — toggle only via
`POST /config/defaults { preventAppNap: true }` (e.g. from the desktop
Settings UI). On non-darwin platforms the toggle is accepted but inert.

### Updating helm-agent

`helm-agent` publishes two npm dist-tags: `latest` (stable) and `canary`
(prerelease, versions like `0.0.17-canary.3`). `helm update` self-updates
the globally-installed `helm-agent` and restarts the daemon and
helm-server (if either was running) so they pick up the new bundle.

```
helm update                          # default channel; auto-switches to
                                     # canary when the current build is
                                     # itself a canary prerelease, so
                                     # canary users aren't stranded
                                     # comparing against a stale `latest`
helm update --canary                 # Track the canary channel explicitly
helm update --tag latest             # Switch back to stable from canary
helm update --tag 0.0.6              # Pin a specific version (honors downgrades)
helm update --dry-run                # Print the install plan, don't run it
helm update --pm pnpm --json         # Force the package manager + machine output
```

Notes:

- `--canary` and `--tag` are mutually exclusive — `--canary` is shorthand
  for `--tag canary` with cleaner help/discovery.
- Plain `helm update` on a canary build defaults to the canary channel
  without mutating `tagExplicit`, so it still won't downgrade you to an
  older canary build.
- The daemon's background update-notification poll follows the same
  channel rule: prerelease `helm-agent` → polls `canary`; otherwise
  `latest`. Channel switches invalidate the 24h poll cache on the next
  tick.
- `--json` success envelope (post-install): `{ ok: true, from, to, pm,
restartedDaemon, restartedServer, restartError?, restartServerError? }`.
  Both `restartedDaemon` and `restartedServer` are always present (false
  when the corresponding process wasn't running, or when the restart
  attempt failed); the matching `restartError` / `restartServerError`
  string is only set on a failed restart. A failed restart does NOT
  change the exit code — only install failures exit non-zero. Note that
  `runServerRestart` prints human progress lines on stdout, so callers
  must parse the JSON envelope from the **last** non-empty line on
  stdout, not the only line.

### Forking and rewinding

- `helm session fork <id> [--at <messageUuid>]` — branch into a new sessionId. Use for cross-tool clean state.
- `helm session rewind <id> --files|--transcript|--both` — rewind in place. Transcript-only is a Helm soft anchor; files goes through the SDK.

When unsure between fork and rewind: **fork** when you want a clean policy/state slate or you intend to keep both branches; **rewind** when you just want to undo recent turns on the same session.

## Gotchas worth remembering

- Daemon restart wipes the in-memory session policy. The session resumes but `--allowed-tool` / `--permission-mode` / `--skill` choices set at creation are gone. Re-pass them on the next `session new`, or rely on the workspace defaults.
- `helm util send-media` only works **inside** a running agent turn (it reads `$HELM_SESSION_ID`). It targets the active turn's Telegram chat — don't call it from a regular shell.
- `helm session send <id> -` reads the prompt from stdin. Use it for multi-line prompts or piped content; don't try to `helm session send "$SID" "$(cat file.md)"` for big files.
- `--help --json` returns the per-command schema. This is the most reliable way to discover flags programmatically without scraping prose.
- Workspaces cascade on delete: `helm workspace rm dev` also drops the meta-chat session. Confirm with the user before running it.
- `helm daemon reset` blows away daemon state. Treat it like `rm -rf` on `$HELM_HOME` — confirm first.

## Output conventions

- `Output: bare (<Type>)` in `--help` means one JSON object (with `--json`) or one human line (without).
- `Output: jsonl (<Type>)` means newline-delimited JSON when `--json` is set. Parse with `jq -c` per line, not as a single document.
- `Output: stream` means SSE-style chunks on stdout (used by `session send` and the `*tail` commands). Don't buffer the whole stream into memory.

## When you're stuck

1. `helm daemon status` first. Always.
2. `helm <area> <sub> --help --json` for the exact flag schema.
3. `helm daemon logs` (or merge across processes with `cat $HELM_HOME/logs/*.log | jq -s 'sort_by(.ts)'`) to see what the daemon actually did.
4. If a session is wedged: `helm session cancel <id>`, then `helm session tail <id>` to confirm it returned to idle. If state is bad, `helm session fork <id>` to escape.
5. If the daemon is wedged: `helm daemon restart` (force-kills a stale daemon, then starts). If a loop/cron is in-flight you'll get exit 2 with an in-flight summary — re-run with `--force` to abort and continue.
