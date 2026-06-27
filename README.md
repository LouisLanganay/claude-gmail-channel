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
