# Trip1 Agent Skills

[![smithery badge](https://smithery.ai/badge/trip1/trip1)](https://smithery.ai/servers/trip1/trip1)

A plugin that lets agents book hotels on [Trip1](https://trip1.com) through the Trip1 MCP server, paid in USDC on Base over [x402](https://x402.org).

Ships one skill, `hotel-booking`, which activates when the user wants to find, compare, or book a hotel.

## What's in the box

- `.claude-plugin/plugin.json` — Claude Code / Claude Desktop plugin manifest
- `.mcp.json` — wires the Trip1 remote MCP server so the plugin is self-contained
- `skills/hotel-booking/SKILL.md` — the skill that orchestrates the full booking flow

## Install

### Claude Code

```bash
/plugin marketplace add trivial-corp/agents
/plugin install trip1@trivial-corp/agents
```

The MCP server and the skill come in together. Restart the session so Claude Code picks up the new MCP connection.

### Claude Desktop

Two options.

**As a local plugin.** Download `agents.zip` from the [latest release](https://github.com/trivial-corp/agents/releases/latest), then in Claude Desktop go to *Customize → Personal plugins → Upload local plugin* and drop the zip in.

**As a connector.** If you only want the MCP server without the skill, add the connector directly:

```json
{
  "mcpServers": {
    "trip1": {
      "type": "http",
      "url": "https://trip1.com/api/mcp"
    }
  }
}
```

### Cursor, Codex CLI, Gemini CLI, any SKILL.md-compatible agent

```bash
npx skills add trivial-corp/agents
```

This installs the skill. Add the MCP server separately via your client's MCP config (same JSON as above).

### ChatGPT

ChatGPT supports remote MCP connectors. In *Settings → Connectors → Add*, point it at:

```
https://trip1.com/api/mcp
```

The skill doesn't apply here; ChatGPT doesn't load `SKILL.md` files. The tool descriptions in the MCP server carry enough intent for ChatGPT to drive the flow on its own.

### Cowork and other MCP-aware clients

Any client that speaks remote MCP (Streamable HTTP) can add Trip1 as a connector. Paste the same `mcpServers` block above, or the bare URL `https://trip1.com/api/mcp` if the client accepts URLs directly.

### Local development

```bash
git clone https://github.com/trivial-corp/agents
cd agents
claude --plugin-dir .
```

## What the skill does

The skill is instructions, not code. It tells the agent:

- when a hotel-booking intent is present and the skill should activate
- which Trip1 MCP tools to call, in what order, with which arguments
- how to handle the x402 payment handshake and the CoinGate fallback
- how to report results and recover from rate drops, payment failures, and polling timeouts

## Paying on the agent's behalf

For fully hands-off agent payments, load an x402-capable wallet MCP alongside Trip1. The simplest option:

```bash
npx @coinbase/payments-mcp
```

Without it, `purchase_hotel` returns a CoinGate URL that a human finishes in a browser. CoinGate takes USDC and 50+ other cryptocurrencies.

## Releasing

Bump `version` in `.claude-plugin/plugin.json` and merge to `main`. The `release.yml` workflow tags `vX.Y.Z`, packages `agents.zip` (full plugin bundle) plus one zip per skill under `skills/`, and publishes them to a GitHub release.

To cut a release from an existing commit instead, push a tag manually: `git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z`. The same workflow handles it.

## Links

- Landing page: https://trip1.com/en/agents
- MCP Registry entry: [`com.trip1/mcp`](https://registry.modelcontextprotocol.io/?q=com.trip1)
- x402 protocol: https://x402.org
- Agent Skills spec: https://agentskills.io/specification

## License

[MIT](LICENSE)
