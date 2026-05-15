<div align="center">

🌐 **English** · [中文](./README.zh-CN.md)

# Drillr · The Financial Research Data Backend for Agents

The financial MCP for AI agents. Scan markets. Build conviction. Track every signal. Cite every claim.

[![License](https://img.shields.io/badge/License-MIT-0969DA?style=flat)](./LICENSE)
[![MCP](https://img.shields.io/badge/MCP-Streamable_HTTP-F97316?style=flat)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/🛠_Tools-2EA44F?style=flat)](./docs/tools.md)
[![REST API](https://img.shields.io/badge/🔧_REST_API-0EA5E9?style=flat)](./docs/rest-api.md)
[![Pricing](https://img.shields.io/badge/💰_Pricing-F59E0B?style=flat)](./docs/pricing.md)
[![Docs](https://img.shields.io/badge/🌐_Docs-8B5CF6?style=flat)](https://drillr.ai/developer/docs)
[![Issues](https://img.shields.io/badge/💬_Issues-EC4899?style=flat)](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues)

</div>

One API key. Five tools for every research workflow: thesis search engine, standardized financial data (statements, ratios, earnings, insider, ownership), live signals, paragraph-cited SEC filing search, alt-data.

> ⭐ **If drillr helps your agent, star us — that's how we know to keep building this in the open.**

## Quick Start

1. Sign up at [drillr.ai](https://drillr.ai)
2. Get an `external`-scope API key at [drillr.ai/developer/keys](https://drillr.ai/developer/keys)
3. Drop into your host's mcp.json — one endpoint exposes all the tools below.

### Option A: Manual mcp.json (any MCP host — recommended)

#### Claude Code / Claude Agent SDK / Cursor / VS Code

```jsonc
{
  "mcpServers": {
    "drillr": {
      "type": "http",
      "url": "https://gateway.drillr.ai/mcp/data",
      "headers": { "Authorization": "Bearer ${DRILLR_API_KEY}" }
    }
  }
}
```

For Cursor, paste the block into `~/.cursor/mcp.json`. For VS Code (GitHub Copilot Chat), run `MCP: Add Server` from the Command Palette and paste the block. Or use one-click install:

[![Install in Cursor](https://img.shields.io/badge/Install_in-Cursor-171717?style=for-the-badge&logo=cursor&logoColor=white)](https://cursor.com/en/install-mcp?name=drillr&config=eyJ1cmwiOiJodHRwczovL2dhdGV3YXkuZHJpbGxyLmFpL21jcC9kYXRhIiwiaGVhZGVycyI6eyJBdXRob3JpemF0aW9uIjoiQmVhcmVyICR7RFJJTExSX0FQSV9LRVl9In19) [![Install in VS Code](https://img.shields.io/badge/Install_in-VS_Code-0078D4?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=drillr&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A//gateway.drillr.ai/mcp/data%22%2C%22headers%22%3A%7B%22Authorization%22%3A%22Bearer%20%24%7BDRILLR_API_KEY%7D%22%7D%7D)

After install, replace `${DRILLR_API_KEY}` in the generated config with your real `drl_*` key.

#### Hermes Agent

```yaml
mcp_servers:
  drillr:
    url: 'https://gateway.drillr.ai/mcp/data'
    headers: { Authorization: 'Bearer ${DRILLR_API_KEY}' }
```

#### Other hosts

Any MCP-compatible host (OpenClaw, ChatGPT MCP, etc.) — same Streamable HTTP transport, same Bearer header. Authentication is Bearer API key only.

### Option B: Smithery one-line

```bash
npx -y @smithery/cli install drillr/drillr --client claude
```

Smithery prompts for your `drl_*` API key on first install and writes it into your client's mcp.json automatically.

Listing: https://smithery.ai/servers/drillr/drillr

### Option C: Claude Code plugin

This repo doubles as its own single-plugin marketplace. From Claude Code:

```
/plugin marketplace add Little-Grebe-Inc/drillr-mcp-server
/plugin install drillr
```

Then set `DRILLR_API_KEY` in your environment (or paste it into the generated config) and you're done.

## Hello World

Once configured, ask your agent something like:

> _"Pull NVDA's last 10-Q gross margin and compare it to AMD's same quarter — flag any divergence in segment mix."_

What happens under the hood:
1. Your host routes the question to the `drillr` MCP server
2. The agent picks the right tools — typically `sec_report_search` (10-Q content) and `run_sql` (financial_statements for margins)
3. You get back a markdown answer with sources cited, typically in 8-15 seconds
4. Check your remaining credit balance at [drillr.ai/developer/keys](https://drillr.ai/developer/keys); REST clients additionally get an inline `{ "data": ..., "_credits": ... }` envelope on every 2xx (see [Pricing](./docs/pricing.md) and [REST API › Response Envelope](./docs/rest-api.md#response-envelope)) — MCP responses follow standard JSON-RPC and do not carry per-call credit info inline

## One Toolkit, 8 Tools

drillr exposes a single MCP endpoint with 8 tools — an all-in-one toolkit for most financial research workflows:

| Tool                | Purpose                                                                                                          |
| ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `run_sql`           | Standardized financial data over 90+ tables — statements, ratios, earnings, insider, ownership, prices, alt-data |
| `sec_report_search` | Paragraph-level semantic search over 10-K / 10-Q / 20-F / 6-K / S-1 / DEF 14A filings                            |
| `sec_report_list`   | List filings by ticker, form type, and date range                                                                |
| `company_search`    | Resolve tickers, segments, peers, market-cap and listing filters                                                 |
| `signal_list`       | Live cross-asset signal feed across ~6,900 tickers                                                               |
| `list_tables`       | Discover available SQL tables                                                                                    |
| `get_table_schema`  | Inspect columns and types for any SQL table                                                                      |
| `fiscal_utility`    | Fiscal-period helpers (FY/FQ resolution across companies with non-calendar years)                                |

Full tool reference: [`docs/tools.md`](./docs/tools.md).

## What's Covered

- **Global equities**: US + Japan, plus ADRs of Chinese / Korean / European companies. Hong Kong / A-shares / Korea native listings coming soon.
- **Ontology-based Company Search**: Search over the universe of equities with business model descriptions, supply chain positions, growth vector or thematic fit. 
- **Fundamentals**: financials back to the 1980s, 90+ structured tables (income statement, balance sheet, cash flow, ratios, growth, valuation)
- **SEC filings**: 10-K / 10-Q / 20-F / 6-K / S-1 / DEF 14A with paragraph-level semantic search
- **Earnings**: call transcripts with AI-structured summaries (guidance, risks, segments, Q&A), full estimate-vs-actuals history
- **Markets**: equities, ETFs, indices (incl. Nikkei 225 / TOPIX), forex, crypto, commodities
- **Analyst coverage**: 565K rating events from 519 firms
- **News + signals**: continuously-updating feed across ~6,900 tickers, cross-asset (equities, macro, geopolitics, commodities, crypto)
- **AI value chain alt-data**: 24 categories spanning energy, chips, compute pricing, LLM token economics, model benchmarks, AI company financials, app usage, web traffic, patents, papers, government contracts, trade flows, etc.

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

## License

MIT — see [`LICENSE`](./LICENSE).
