# Mira Claude Code channel (test-claude)

Mira is a plugin for Claude Code that lets you talk to a coding assistant from your phone. Messages typed or spoken in the Mira iOS app (a smart glasses companion app) arrive in a Claude Code session running on your computer, and Claude's short spoken-style answer is sent back to the phone.

This repository is a test and development copy of that plugin, used to try out Claude Code channels, hooks, a plugin marketplace and plugin auto-update.

## What it does

- Runs as an MCP "channel" server inside Claude Code (`plugins/mira/server.ts`) and listens for chat requests on `127.0.0.1:3141`.
- Pushes each incoming chat message into the active Claude Code session as a `<channel source="mira">` tag, along with optional context such as the user's first name, local time and location.
- Uses Claude Code hooks (`plugins/mira/hooks/`) to send the final answer, and live "tool is running" status updates, back to the waiting app request as a server-sent event stream.
- Opens a Cloudflare tunnel so the phone can reach your computer from anywhere. The tunnel is provisioned by a Mira staging backend, so this part only works for people with access to that backend.
- Checks this GitHub repository for a newer plugin version and tries to update itself.
- Includes a default agent, also called Mira, with a short and dry spoken style (`plugins/mira/agents/mira.md`).

## Tech stack

- TypeScript running on Bun
- Model Context Protocol SDK (`@modelcontextprotocol/sdk`), `zod`, `localtunnel`
- Claude Code plugin, marketplace and hooks system
- Cloudflare tunnels (`cloudflared`, downloaded on first run)

## Status

Prototype. The local bridge can be tried without the iOS app, using curl (see below). The tunnel and the iOS connection need the private Mira backend and app, so they are not usable by outside visitors. The three large Markdown files at the top level (`channels.md`, `hooksinfo.md`, `plugininfo.md`) are copies of Claude Code documentation kept as reference notes. They belong to Anthropic and are not part of the plugin.

## Run it

Requirements: [Claude Code](https://code.claude.com) v2.1.80 or later and [Bun](https://bun.sh). Dependencies are installed by Claude Code and Bun, you do not run the server yourself. If you want to install them by hand, run `bun install` inside `plugins/mira`.

1. Start Claude Code and add this repository as a marketplace, then install the plugin:

   ```sh
   claude
   # inside the Claude Code session:
   /plugin marketplace add maxawad/test-claude
   /plugin install mira@mira-marketplace
   ```

   Optional: enable auto-update with `/plugin` then Marketplaces, then `mira-marketplace`, then Enable auto-update. The marketplace does not set an explicit plugin version, so Claude Code uses the git commit to detect updates. After an update, run `/reload-plugins` or restart the session.

2. Quit, then relaunch Claude Code with the development channels flag. This plugin is not on Anthropic's approved channel list, so the flag is required:

   ```sh
   claude --dangerously-load-development-channels plugin:mira@mira-marketplace
   ```

   Confirm the prompt. Claude Code starts `server.ts` over stdio and the HTTP listener starts on `127.0.0.1:3141`.

3. In the Mira iOS app, open Settings, then Claude Code, and turn Local Integration on. This step needs the Mira app.

4. Send a message from the app. It appears in your Claude Code session, and the answer goes back to the app.

### Try it with curl (no iOS app needed)

With Claude Code running as in step 2, in another terminal:

```sh
curl -X POST http://127.0.0.1:3141/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"messages":[{"speaker":0,"content":"hello from curl"}],"user_local_time":"now","user_timezone":"UTC","location":null}'
```

The request stays open as a server-sent event stream until Claude finishes, with a 120 second timeout. The stream sends `{"text": "..."}`, then `{"sources": []}`, then `[DONE]`.

### Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `MIRA_PORT` | `3141` | Port of the local listener |
| `MIRA_HOST` | `127.0.0.1` | Address the listener binds to. Keep it on loopback unless you know why you need otherwise. |
| `MIRA_LOG` | `/tmp/mira.log` | Log file path |
| `MIRA_SSE_HEARTBEAT_MS` | `15000` | Keep-alive interval for the event stream |
| `CLAUDE_BIN` | `~/.local/bin/claude` | Claude Code binary used by the auto-update step |

No API keys are needed to run the local bridge. See `.env.example` for the variable names.

## Project layout

```
.claude-plugin/marketplace.json          marketplace catalog
plugins/mira/
  .claude-plugin/plugin.json             plugin manifest, declares the MCP server
  server.ts                              MCP channel and Bun HTTP listener
  cloudflare.ts                          tunnel provisioning and cloudflared process
  device.ts                              per-machine device id
  plugin_update.ts                       update check against this repository
  hooks/                                 SessionStart, PreToolUse, PostToolUse and Stop hooks
  agents/mira.md                         default Mira agent prompt
channels.md, hooksinfo.md, plugininfo.md reference copies of Claude Code docs
```

## Credits

Built on Claude Code's channels, hooks and plugin features from Anthropic. The Mira iOS app and its backend belong to the Mira team.
