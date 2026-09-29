<p align="center"><img src="logo.png" width="96" alt="trip1 logo"></p>

# trip1 agent skills and MCP server

[![smithery badge](https://smithery.ai/badge/trip1/trip1)](https://smithery.ai/servers/trip1/trip1)

A plugin that lets agents book hotels on [trip1](https://trip1.com) through the trip1 MCP server: about 3 million properties in 200+ countries, paid in USDC on Base over [x402](https://x402.org).

- **Endpoint:** `https://trip1.com/api/mcp` (Streamable HTTP)
- **Auth:** none, no account, API key or OAuth
- **Docs:** https://trip1.com/agents
- **MCP Registry:** `com.trip1/mcp`

Ships one skill, `hotel-booking`, which activates when the user wants to find, compare, or book a hotel.

## What's in the box

- `.claude-plugin/plugin.json` — Claude Code / Claude Desktop plugin manifest
- `.mcp.json` — wires the trip1 remote MCP server so the plugin is self-contained
- `plugin.json` and `mcp.json` — the same plugin in the [Agent Plugins](https://open-plugins.com) format, for Cursor and other compatible clients
- `skills/hotel-booking/SKILL.md` — the skill that orchestrates the full booking flow
- `server.json` — the [official MCP Registry](https://registry.modelcontextprotocol.io) entry
- `llms-install.md` — install instructions for agents such as Cline that set servers up themselves

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

### Cursor, Windsurf, Cline

Add the server to the client's MCP config (`~/.cursor/mcp.json` for Cursor; *MCP Servers → Configure → Remote Servers* in Cline):

```json
{
  "mcpServers": {
    "trip1": {
      "type": "streamable-http",
      "url": "https://trip1.com/api/mcp"
    }
  }
}
```

Cursor also loads this repo as a plugin through `plugin.json` and `mcp.json`. Clients that only speak stdio can bridge with `npx -y mcp-remote https://trip1.com/api/mcp`.

### VS Code

```bash
code --add-mcp '{"name":"trip1","type":"http","url":"https://trip1.com/api/mcp"}'
```

### Codex CLI, Gemini CLI, any SKILL.md-compatible agent

```bash
npx skills add trivial-corp/agents
```

This installs the skill. Add the MCP server separately via your client's MCP config (same JSON as above).

### ChatGPT

Search for **trip1** in the ChatGPT apps directory. Or add it as a custom connector in *Settings → Connectors → Add*, pointing at:

```
https://trip1.com/api/mcp
```

The skill doesn't apply here; ChatGPT doesn't load `SKILL.md` files. The tool descriptions in the MCP server carry enough intent for ChatGPT to drive the flow on its own.

### Cowork and other MCP-aware clients

Any client that speaks remote MCP (Streamable HTTP) can add trip1 as a connector. Paste the same `mcpServers` block above, or the bare URL `https://trip1.com/api/mcp` if the client accepts URLs directly.

### Local development

```bash
git clone https://github.com/trivial-corp/agents
cd agents
claude --plugin-dir .
```

## What the skill does

The skill is instructions, not code. It tells the agent:

- when a hotel-booking intent is present and the skill should activate
- which trip1 MCP tools to call, in what order, with which arguments
- how to handle the x402 payment handshake and the CoinGate fallback
- how to report results and recover from rate drops, payment failures, and polling timeouts

## Paying on the agent's behalf

For fully hands-off agent payments, load an x402-capable wallet MCP alongside trip1. The simplest option:

```bash
npx @coinbase/payments-mcp
```

### Paying from code with the x402 SDK

`@x402/fetch` refuses any payment above **$1** by default, and a hotel booking always costs more, so raise the cap. Pin the recipient too, so the client only ever pays trip1:

```ts
import { wrapFetchWithPaymentFromConfig } from "@x402/fetch";
import { ExactEvmScheme } from "@x402/evm";
import { privateKeyToAccount } from "viem/accounts";

const TRIP1_PAY_TO = "0x4eD818663D9040461Ee1f6E8f618Df1770dddDc1"; // see https://trip1.com/.well-known/x402.json

const fetchWithPayment = wrapFetchWithPaymentFromConfig(fetch, {
  schemes: [{ network: "eip155:8453", client: new ExactEvmScheme(privateKeyToAccount(process.env.EVM_PRIVATE_KEY)) }],
  spendControls: { maxAmountPerPayment: "$2000" },
  policies: [(_version, reqs) => reqs.filter((r) => r.payTo.toLowerCase() === TRIP1_PAY_TO.toLowerCase())],
});

const response = await fetchWithPayment(paymentUrl); // payment_url from purchase_hotel
```

Then poll `get_order_details` until `ready` is true.

### Without a wallet

Call `purchase_hotel` with `payment_service: "coingate"`. It returns a CoinGate checkout URL that a human finishes in a browser. CoinGate takes 120+ cryptocurrencies, including USDC.

## Publishing to the MCP Registry

Bump `version` in `server.json`, then run `mcp-publisher publish` after `mcp-publisher login dns --domain trip1.com`. The DNS key lives with the trip1 team.

## Releasing

Bump `version` in `.claude-plugin/plugin.json` and merge to `main`. The `release.yml` workflow tags `vX.Y.Z`, packages `agents.zip` (full plugin bundle) plus one zip per skill under `skills/`, and publishes them to a GitHub release.

To cut a release from an existing commit instead, push a tag manually: `git tag -a vX.Y.Z -m "..." && git push origin vX.Y.Z`. The same workflow handles it.

## Links

- Landing page: https://trip1.com/agents
- Server card: https://trip1.com/.well-known/mcp.json
- x402 metadata: https://trip1.com/.well-known/x402.json
- MCP Registry entry: [`com.trip1/mcp`](https://registry.modelcontextprotocol.io/?q=com.trip1)
- x402 protocol: https://x402.org
- Agent Skills spec: https://agentskills.io/specification

## License

[MIT](LICENSE)
