# helm-agent

Public feedback repository for **Helm** — an Electron desktop + CLI daemon wrapping the Claude Agent SDK.

This repo is **feedback-only**. Source code lives elsewhere; please use Issues here to report bugs, request features, or share usage feedback.

## What is Helm?

Helm groups projects into **workspaces**, each with a singleton orchestrator chat and per-project child sessions. It runs as a background daemon controllable from the desktop app, the `helm` CLI, or external channels (Telegram, WeChat).

Highlights:

- Workspace-scoped sessions across multiple project directories
- Cron tasks, issue tracking, and channel adapters per workspace
- Plugin & skill marketplace for extending agent behavior
- Multi-provider model routing (Anthropic, OpenAI-compatible, custom)

## Filing Feedback

Open an [issue](../../issues/new/choose) using the appropriate template:

- **Bug report** — something is broken or behaves unexpectedly
- **Feature request** — propose a new capability
- **Question / discussion** — usage help or design discussion

Please include:

- Helm version (`helm --version` or About panel)
- OS and version
- Reproduction steps (for bugs)
- Relevant logs from `~/.helm/logs/` (redact secrets)

## Links

- Issues: https://github.com/helm-agent/helm-agent/issues
- Discussions: https://github.com/helm-agent/helm-agent/discussions

## License

Feedback submitted here is contributed under the repository's terms; see [LICENSE](LICENSE) once added.
