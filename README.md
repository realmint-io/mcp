# Realmint MCP

Agent-native access to the Realmint catalog of tokenized real-world assets (RWAs): scoring, market data, route support, and price history — exposed as native tools over the [Model Context Protocol](https://modelcontextprotocol.io).

The MCP surface is **read-only by design**. Trade execution lives in the Realmint HTTP API and requires client-side wallet signing — keys never touch the MCP server.

| | |
|---|---|
| **Endpoint** | `https://mcp.realmint.io/mcp` |
| **Transport** | Streamable HTTP |
| **Auth** | None for the public catalog |
| **Discovery card** | [`/.well-known/mcp/server-card.json`](https://app.realmint.io/.well-known/mcp/server-card.json) |
| **Status** | Live since May 2026 |

---

## What you can ask it

| Capability | Example prompt |
|---|---|
| **Score risk & rights** | "Score Tether Gold and tell me if it's safe for our DAO treasury" |
| **Search the catalog** | "Find tokenized treasuries on Ethereum with no freeze capability" |
| **Track issuer changes** | "Has anything in my watchlist changed control parameters this month?" |
| **Map routable venues** | "Where can I acquire 100k of OUSG, ranked by liquidity?" |
| **Pull market & price data** | "Show me PAXG's spot price, 24h volume, and 6-month chart" |

---

## Add it to your client

### Claude.ai (custom remote connector)

1. **Settings → Connectors → Add custom connector**
2. **Server URL:** `https://mcp.realmint.io/mcp`
3. **Auth:** none required
4. Save and start asking the example prompts above.

### Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "realmint": {
      "transport": {
        "type": "streamable-http",
        "url": "https://mcp.realmint.io/mcp"
      }
    }
  }
}
```

Restart Claude Desktop.

### Cursor / Cline / Continue / any MCP client

Configure a remote MCP server with:

- URL: `https://mcp.realmint.io/mcp`
- Transport: `streamable-http`
- Auth: none

### Programmatic

```bash
curl -X POST https://mcp.realmint.io/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

You should see 7 tools, each with `annotations.title` and `annotations.readOnlyHint=true`.

---

## The 7 tools

| Tool | Purpose |
|---|---|
| `realmint_list_assets` | Enumerate the full catalog with schemas |
| `realmint_list_market` | Spot price, 24h change, volume, market cap, history |
| `realmint_get_asset` | Full schema for one asset (by lowercased ticker) |
| `realmint_get_asset_history` | Versioned metadata change history |
| `realmint_get_score` | Six-dimension Risk & Rights Score + composite |
| `realmint_get_route_support` | Routable venues + per-venue scores |
| `realmint_get_price_history` | Time-series prices (5m/1h/1d auto-cascading) |

All tools are read-only. Annotations follow the MCP spec:

```
readOnlyHint:    true
destructiveHint: false
idempotentHint:  true
openWorldHint:   true
```

The complete tool catalog with descriptions, inputs, and titles is published in the [discovery card](https://app.realmint.io/.well-known/mcp/server-card.json).

### The Risk & Rights Score

`realmint_get_score` returns a 0–100 composite plus six per-dimension scores:

- **backing** — proof of reserves, audit cadence, backing mechanism.
- **enforceability** — governing law, issuer verification, legal documents.
- **control** — issuer power risk: freeze, mint, burn, blacklist, upgrade.
- **exit** — redemption, transferability, KYC/whitelist, settlement, holding periods.
- **liquidity** — market data, venue count, evidence docs.
- **social** — recency-weighted Twitter activity; unverified handles penalized.

Suggested decision thresholds:

- composite **≥ 75** → typical case, proceed.
- **55 ≤ composite < 75** → elevated friction (gates, missing audits); review the asset's flags before acting.
- composite **< 55** → high risk; abort or require explicit user override.

Tokens with a non-null `metadata.lifecycle` (migrated, discontinued, abandoned) are score-capped at 49 by policy.

---

## Troubleshooting

**`HTTP 403 — "Invalid Origin header"`**
The server enforces Origin-header validation as required by the Anthropic Connectors Directory. `claude.ai`, `chatgpt.com`, `mcp.realmint.io`, `app.realmint.io`, and `realmint.io` are pre-allowed. If you operate from a different origin and need it added, email us at the address below.

**Tool returns an empty list**
The Realmint catalog is curated — at any given time it indexes ~30 assets. Run `realmint_list_assets` to see the current set before asking about a specific ticker.

**Asset ID format**
Asset IDs are lowercased tickers — `xaut`, not `XAUT`; `paxg`, not `PAXG`.

**`metadata.lifecycle` is non-null**
The token has been migrated, discontinued, or abandoned. Trading is disallowed and the score is capped at 49. The lifecycle field tells you why and points to the replacement (if any).

**Price history returns coarser buckets than asked**
The server cascades to coarser buckets (5m → 1h → 1d) when the requested bucket is empty for that asset. Look at `effective_granularity` in the response for what was actually returned.

---

## Security & privacy

- **Read-only.** No write tools, no trade execution, no destructive operations.
- **No PII collected by the MCP server.** The optional `x-api-key` header is used solely for upstream rate-limit identification.
- **HTTPS** termination via Traefik.
- **Origin / Host validation** enabled at the FastMCP layer (DNS rebinding protection).
- **Privacy policy:** <https://realmint.io/privacy>
- **Terms of service:** <https://realmint.io/terms>

---

## Versioning & changelog

This connector is versioned alongside the Realmint platform. Substantive changes to tool surfaces or output shapes will be announced in this README and reflected in the [discovery card](https://app.realmint.io/.well-known/mcp/server-card.json) (`serverInfo.version`).

---

## Contact & support

- **General:** <contact@realmint.io>
- **Realmint web app:** <https://realmint.io>
- **API docs:** <https://api.realmint.io/docs>
- **Discovery card:** <https://app.realmint.io/.well-known/mcp/server-card.json>

---

Realmint is operated by 0xFútbol Inc, a BVI Business Company (registration 2169115), trading as Realmint. The MCP server source lives in a private repository; the hosted endpoint at `https://mcp.realmint.io/mcp` is the supported public interface.
