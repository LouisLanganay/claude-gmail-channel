# fynex-channels — Gmail channel for Claude Code

A Claude Code **marketplace** exposing a single plugin: `gmail`, a one-way
inbound channel that surfaces new Gmail INBOX emails directly in a Claude Code
session (notifications labelled `source="gmail"`).

## Why a GitHub marketplace

A `directory`-sourced (local) marketplace is treated as *development* and
requires `--dangerously-load-development-channels`, which conflicts with
`--dangerously-skip-permissions`. A **git-sourced** marketplace loads with no
dangerous flag — same as the official `discord`/`telegram`/`figma` plugins.

## Architecture

```
Gmail API (Pub/Sub push)
  → gmail-webhook (systemd, HTTP :3002)          # fynex agent repo
  → POST http://127.0.0.1:3006/gmail-notify      # this plugin's bridge
  → mcp.notification({ method: 'notifications/claude/channel', meta.chat_id: 'gmail' })
  → Claude Code session
```

The plugin is a minimal MCP server declaring the experimental `claude/channel`
capability. The channel name (`gmail`) comes from the `mcpServers` key in
`gmail/.mcp.json`. It runs an HTTP bridge on `127.0.0.1:3006`
(`GMAIL_BRIDGE_PORT` env overrides the port — used for isolated testing).

## Install

```bash
claude plugin marketplace add LouisLanganay/claude-gmail-channel
claude plugin install gmail@fynex-channels
```

Then launch Claude with the channel:

```bash
claude --channels plugin:gmail@fynex-channels ...
```

## Feeding the bridge

The plugin does not talk to Gmail itself: something has to POST each new email
to the bridge. In my setup that is a small webhook receiving Gmail Pub/Sub push
notifications (not part of this repository). Any process on the same machine
can do it:

```bash
curl -X POST http://127.0.0.1:3006/gmail-notify \
  -H 'Content-Type: application/json' \
  -d '{"from": "Jane <jane@example.com>", "subject": "Hello", "snippet": "First line of the email"}'
```

`from`, `subject` and `snippet` are all optional. Each POST becomes one channel
notification in the running Claude Code session.

## Notes

- The bridge listens on `127.0.0.1` only. Nothing is exposed to the network.
- The channel is one-way: Claude can read notifications but cannot reply
  through it. Use a Gmail MCP server to read full messages or answer.
- The server's instructions tell Claude to treat email content as untrusted
  data (no following instructions found in a subject, sender or snippet).
- Notification text is in French (`Nouvel email de …`), edit `gmail/server.ts`
  to change it.
