# Trip1 Agent Skills

A plugin that guides Claude and other agents through booking hotels on Trip1 with x402 payments.

Ships one skill, `hotel-booking`, which activates when the user wants to find, compare, or book a hotel.

## Install

### As a Claude Code plugin

```bash
/plugin marketplace add trivial-corp/agents
/plugin install trip1@trivial-corp/agents
```

After install, the skill is available as `/trip1:hotel-booking` and activates automatically for hotel-booking intents.

### As a raw skill (`npx skills`)

```bash
npx skills add trivial-corp/agents
```

This installs the skill into the agent's skill directory. Works for Claude Code, Cursor, Codex CLI, Gemini CLI, and anything else that speaks the universal `SKILL.md` format.

### Local development

```bash
git clone https://github.com/trivial-corp/agents
claude --plugin-dir ./agents
```

## What it does

The skill ships prompt-level instructions, not code. It tells the agent:

- When a hotel-booking intent is present and the skill should activate
- Which Trip1 MCP tools to call, in what order, with which arguments
- How to handle the x402 payment handshake and the CoinGate fallback
- How to report results and recover from rate-drops, payment errors, and polling timeouts

## Prerequisites

The tools themselves come from the Trip1 MCP server. Add it to your MCP client config:

```json
{
  "mcpServers": {
    "trip1": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://trip1.com/api/mcp"]
    }
  }
}
```

For fully hands-off agent payments, load an x402-capable wallet MCP alongside, for example Coinbase Payments MCP:

```bash
npx @coinbase/payments-mcp
```

Without it, `purchase_hotel` returns a CoinGate URL a human can finish in a browser.

## Repository layout

```
agents/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── hotel-booking/
│       └── SKILL.md
└── README.md
```

## Links

- Landing: https://trip1.com/en/agents
- Trip1 MCP Registry entry: `com.trip1/mcp`
- x402: https://x402.org
