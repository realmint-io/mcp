# Realmint MCP

Agent-native access to the Realmint catalog of tokenized real-world assets (RWAs): scoring, market data, route support, price history, and a keyless **x402 buy on-ramp** — exposed as native tools over the [Model Context Protocol](https://modelcontextprotocol.io).

The MCP surface is **key-safe by design**. The catalog is read-only. Two trading
paths move funds only under your own authorization and never see a raw key:

- **x402 keyless buy** — an autonomous agent with its own EVM key signs an EIP-3009
  authorization **client-side** to fund + buy (`realmint_x402_buy_challenge`).
- **Sign-in + delegated trade** — a human/connector signs in via OAuth and delegates
  their Privy embedded wallet once; Realmint then signs buy/sell/withdraw
  **server-side** under a per-trade spending cap. Privy holds the wallet; no key
  touches Realmint.

| | |
|---|---|
| **Endpoint** | `https://mcp.realmint.io/mcp` |
| **Transport** | Streamable HTTP |
| **Auth** | Mixed (lazy): catalog is open; trade tools trigger OAuth on first use |
| **Tools** | 8 catalog (read-only) + 6 trade (OAuth-gated) |
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
| **Fund & buy, agent-native** | "Fund my wallet with 25 USDC over x402 and buy 5 of TSLAx" |

---

## Add it to your client

### Claude.ai (custom remote connector)

1. **Settings → Connectors → Add custom connector**
2. **Server URL:** `https://mcp.realmint.io/mcp`
3. **Auth:** none to connect — the catalog works immediately. The first time you
   call a trade tool, Claude runs the OAuth sign-in (Realmint → Privy) and you
   delegate your wallet once.
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
- Auth: none to connect; trade tools trigger OAuth on first use

### Programmatic

```bash
curl -X POST https://mcp.realmint.io/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Unauthenticated you see the 8 catalog tools (all `readOnlyHint=true`). After OAuth
sign-in, the 6 trade tools appear too (`destructiveHint=true`).

---

## Catalog tools (read-only, no auth)

| Tool | Purpose |
|---|---|
| `realmint_list_assets` | Enumerate the full catalog with schemas |
| `realmint_list_market` | Spot price, 24h change, volume, market cap, history |
| `realmint_get_asset` | Full schema for one asset (by lowercased ticker) |
| `realmint_get_asset_history` | Versioned metadata change history |
| `realmint_get_score` | Six-dimension Risk & Rights Score + composite |
| `realmint_get_route_support` | Routable venues + per-venue scores |
| `realmint_get_price_history` | Time-series prices (5m/1h/1d auto-cascading) |
| `realmint_x402_buy_challenge` | x402 funding challenge — the keyless, agent-native USDC on-ramp to buy an asset (returns the 402; the agent signs client-side) |

## Trade tools (sign in via OAuth)

Calling one of these when signed out returns a `401` that triggers the OAuth flow
(Realmint's Ory Hydra, federated to Privy login). You sign in and **delegate your
wallet once** (`app.realmint.io/mcp/delegate`); Realmint then signs each trade
**server-side** via that delegation, under a per-trade spending cap — no key
touches the MCP, no per-trade browser step.

| Tool | Purpose |
|---|---|
| `realmint_whoami` | Confirm the signed-in identity (Privy DID) |
| `realmint_my_wallet` | Your wallets, smart account, and balances |
| `realmint_deposit` | Where to send USDC to add buying power |
| `realmint_buy` | Buy an RWA (your smart account must already hold USDC) |
| `realmint_sell` | Sell an RWA back to USDC on Injective |
| `realmint_withdraw` | Withdraw USDC from your smart account to any address |

The complete tool catalog with descriptions, inputs, and titles is published in the [discovery card](https://app.realmint.io/.well-known/mcp/server-card.json).

### Buying with `realmint_x402_buy_challenge`

The on-ramp is agent-native and keyless — an agent buys with only an EVM key, the same two signatures a human gives, all **client-side**:

1. Call `realmint_x402_buy_challenge` with your `owner_eoa`, the `amount_usdc`, and the `asset_id` you intend to buy. It returns an HTTP 402 with x402 payment requirements (`accepts[0]`): `payTo` is **your** Base smart account derived from `owner_eoa`, `maxAmountRequired` the USDC atomic amount, `asset` the Base USDC contract, `extra` the EIP-712 domain.
2. **Verify `payTo` derives from your own `owner_eoa`**, then sign an EIP-3009 `transferWithAuthorization` over that domain, base64-encode the x402 `PaymentPayload`, and re-POST it to `/v1/route/x402-buy` in the `X-PAYMENT` header. On settlement the USDC bridges custody-free to your Injective smart account.
3. Buy any RWA from that balance via `POST /v1/route/intent` (signing the route as usual). The asset rests in your own smart account — `send` it elsewhere whenever you want, exactly like a human user.

See the [x402 agent-buy guide](https://realmint.io/developers/x402) for the full walkthrough.

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

- **Key-safe.** The catalog is read-only. Trading moves funds only under your own authorization and never exposes a raw key: x402 buys are signed client-side (EIP-3009), and the OAuth trade tools sign server-side via your Privy **delegation** under a per-trade spending cap — Privy holds the embedded wallet; no key touches Realmint.
- **No PII collected by the MCP server.** The `x-api-key` header is used for upstream rate-limit identification; trade-tool identity comes from the OAuth (Hydra) token.
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

The MCP server source lives in a private repository; the hosted endpoint at `https://mcp.realmint.io/mcp` is the supported public interface.
