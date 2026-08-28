<div align="center">

🌐 **English** · [中文](./README.zh-CN.md)

# Drillr · The Financial Research Data Backend for Agents

The financial MCP for AI agents. Scan markets. Build conviction. Track every signal. Cite every claim.

[![License](https://img.shields.io/badge/License-MIT-0969DA?style=flat)](./LICENSE)
[![MCP](https://img.shields.io/badge/MCP-Streamable_HTTP-F97316?style=flat)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/🛠_Tools-2EA44F?style=flat)](./docs/tools.md)
[![REST API](https://img.shields.io/badge/🔧_REST_API-0EA5E9?style=flat)](./docs/rest-api.md)
[![Docs](https://img.shields.io/badge/🌐_Docs-8B5CF6?style=flat)](https://drillr.ai/developer/docs)
[![Issues](https://img.shields.io/badge/💬_Issues-EC4899?style=flat)](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues)

</div>

Browser sign-in. No API key to copy. Nine tools for agent research: standardized financial data, company discovery, semantic news and event search, paragraph-cited company filings, and alt-data.

> ⭐ **If drillr helps your agent, star us — that's how we know to keep building this in the open.**

## Quick Start

1. Sign up at [drillr.ai](https://drillr.ai)
2. Add `https://gateway.drillr.ai/mcp/data` to an OAuth-capable MCP client
3. Sign in and approve the named client in your browser. No secret is displayed or copied.

### Claude Code

```bash
claude mcp add --scope user --transport http drillr \
  https://gateway.drillr.ai/mcp/data
claude mcp login drillr
```

### Codex CLI

```bash
codex mcp add drillr --url https://gateway.drillr.ai/mcp/data
```

Codex starts browser sign-in during setup. For an existing entry, run `codex mcp login drillr`.

### Claude Desktop / OAuth-capable hosts

```jsonc
{
  "mcpServers": {
    "drillr": {
      "type": "http",
      "url": "https://gateway.drillr.ai/mcp/data"
    }
  }
}
```

Restart the host after saving, then approve Drillr in the browser when prompted.

### Cursor / VS Code

One click writes the server entry; your editor then signs you in through the browser.

[![Install in Cursor](https://img.shields.io/badge/Install_in-Cursor-171717?style=for-the-badge&logo=cursor&logoColor=white)](https://cursor.com/en/install-mcp?name=drillr&config=eyJ1cmwiOiJodHRwczovL2dhdGV3YXkuZHJpbGxyLmFpL21jcC9kYXRhIn0=) [![Install in VS Code](https://img.shields.io/badge/Install_in-VS_Code-0078D4?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=drillr&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fgateway.drillr.ai%2Fmcp%2Fdata%22%7D)

### API-key fallback

Use this only for REST or an MCP host that does not support browser OAuth. Create an `external` key at [drillr.ai/developer/keys](https://drillr.ai/developer/keys), store it as a secret, and add:

```jsonc
"headers": { "Authorization": "Bearer <YOUR_DRILLR_API_KEY>" }
```

#### Hermes Agent fallback

```yaml
mcp_servers:
  drillr:
    url: 'https://gateway.drillr.ai/mcp/data'
    headers: { Authorization: 'Bearer <YOUR_DRILLR_API_KEY>' }
```

#### Other hosts

Any MCP-compatible host uses the same Streamable HTTP endpoint. Omit `headers` and use browser OAuth whenever the host supports it; otherwise use the API-key fallback above. Never configure OAuth and a static bearer header on the same server entry.

### Smithery fallback

```bash
npx -y @smithery/cli install drillr/drillr --client claude
```

Smithery offers the API key as an optional field. Leave it empty on an OAuth-capable client and sign in through the browser instead.

Listing: https://smithery.ai/servers/drillr/drillr

### Claude Code plugin fallback

This repo doubles as its own single-plugin marketplace. From Claude Code:

```
/plugin marketplace add Little-Grebe-Inc/drillr-mcp-server
/plugin install drillr
```

The plugin installs the server without a key. Restart Claude Code, run `/mcp`, pick `drillr`, and choose `Authenticate`.

## Hello World

Once configured, ask your agent something like:

> _"Pull NVDA's last 10-Q gross margin and compare it to AMD's same quarter — flag any divergence in segment mix."_

What happens under the hood:
1. Your host routes the question to the `drillr` MCP server
2. The agent picks the right tools — typically `sec_report_search` (10-Q content) and `run_sql` (financial_statements for margins)
3. You get back a markdown answer with sources cited, typically in 8-15 seconds
4. Check your remaining credit balance at [drillr.ai/developer/keys](https://drillr.ai/developer/keys); REST clients additionally get an inline `{ "data": ..., "_credits": ... }` envelope on every 2xx (see [REST API › Response Envelope](./docs/rest-api.md#response-envelope)) — MCP responses follow standard JSON-RPC and do not carry per-call credit info inline

## One Toolkit, 9 Tools

drillr exposes a single MCP endpoint with 9 tools — an all-in-one toolkit for most financial research workflows:

| Tool                | Purpose                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `run_sql`           | Standardized financial data over 90+ tables — statements, ratios, earnings, insider, ownership, prices, alt-data |
| `sec_report_search` | Paragraph-level semantic search across company filings in the US, Japan, Hong Kong, and China A-shares            |
| `sec_report_list`   | List a ticker's indexed filings by filing type                                                                   |
| `company_search`    | Four-market qualitative discovery by business model, supply chain, peers, or theme                              |
| `news_search`       | Semantic search over news, market events, and attributed claims, grouped into storylines                        |
| `ticker_lookup`     | Resolve a company name, brand, or ticker substring to ticker history                                             |
| `list_tables`       | Discover available alt-data SQL tables by category                                                               |
| `get_table_schema`  | Inspect columns and types for any SQL table                                                                      |
| `fiscal_utility`    | Fiscal-period helpers (FY/FQ resolution across companies with non-calendar years)                                |

Full tool reference: [`docs/tools.md`](./docs/tools.md).

## What's Covered

- **Core equity coverage**: US, Japan, Hong Kong, and China A-shares. Ticker formats: `AAPL`, `6758.T`, `00700.HK`, `600519.SH` / `300750.SZ`.
- **Ontology-based Company Search**: Search over the universe of equities with business model descriptions, supply chain positions, growth vector or thematic fit. 
- **Fundamentals**: `financial_statements`, `company_snapshot`, and `price_volume_history` cover all four core markets; financial history reaches back to the 1980s
- **Company filings**: SEC EDGAR, Japan EDINET, HKEX, and China A-share reports with paragraph-level semantic search
- **Earnings**: call transcripts with AI-structured summaries and estimate-vs-actuals history; this specialized dataset covers US + Japan
- **Markets**: equities, ETFs, indices, forex, crypto, commodities
- **Specialized US datasets**: analyst ratings, ownership, executives, 8-K events, and extended-hours quotes
- **News + events**: continuously updating four-market and cross-asset search with story grouping and attributed claims
- **AI value chain alt-data**: energy & power, data centers, semiconductors, compute pricing, AI models / companies / benchmarks, LLM token pricing, macro & trade, prediction markets, critical minerals

Full data dictionary: [`docs/tools.md`](./docs/tools.md).

## REST API

Every MCP tool has a 1:1 REST endpoint. Same `drl_*` key, same data, same billing. See [`docs/rest-api.md`](./docs/rest-api.md).

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/run_sql \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sql":"SELECT ticker, close FROM price_volume_history WHERE ticker='\''AAPL'\'' AND time_frame='\''daily'\'' ORDER BY period_end DESC LIMIT 5"}'
```

## Out of Scope

We're upfront about edges so your agent doesn't waste research loops:

- Private / unlisted companies (we cover public-listed only)
- On-chain crypto metrics — we have CEX prices (BTCUSD / ETHUSD / SOLUSD etc.), not TVL / holders / wallets
- Options chains, real-time order book, intraday tick data
- Retail brokerage actions (placing orders, managing positions)
- drillr does not produce its own price forecasts — we surface analyst consensus

## Community

Building something with drillr, hit a rough edge, or want early-access drops? Come say hi — scan to join, or click the heading link.

<table>
  <tr>
    <td align="center" width="50%"><a href="https://discord.gg/YAh96nw5Vh"><b>Discord</b></a></td>
    <td align="center" width="50%"><b>WeChat</b></td>
  </tr>
  <tr>
    <td align="center"><img src="https://gateway.drillr.ai/qr/discord.svg" width="160" alt="Drillr Discord QR" /></td>
    <td align="center"><img src="https://gateway.drillr.ai/qr/wechat.svg" width="160" alt="Drillr WeChat group QR" /></td>
  </tr>
  <tr>
    <td align="center">Devs building agentic research products — office hours, debugging help, early access.</td>
    <td align="center">Chinese-speaking dev community — fastest product feedback.</td>
  </tr>
</table>

## License

MIT — see [`LICENSE`](./LICENSE).
