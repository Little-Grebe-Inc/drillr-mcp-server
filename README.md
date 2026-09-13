<div align="center">

🌐 **English** · [中文](./README.zh-CN.md)

# drillr · The financial data and research MCP for AI agents

Filings, statements, earnings, ownership, events, executives, analyst data, company discovery and research signals for US, China and Japan equities — every figure traced to the document it came from.

[![License](https://img.shields.io/badge/License-MIT-0969DA?style=flat)](./LICENSE)
[![MCP](https://img.shields.io/badge/MCP-Streamable_HTTP-F97316?style=flat)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/🛠_10_tools-2EA44F?style=flat)](https://drillr.ai/docs/mcp)
[![REST API](https://img.shields.io/badge/🔧_REST_API_v2-0EA5E9?style=flat)](https://drillr.ai/docs/api)
[![Docs](https://img.shields.io/badge/🌐_Docs-8B5CF6?style=flat)](https://drillr.ai/docs)
[![Issues](https://img.shields.io/badge/💬_Issues-EC4899?style=flat)](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues)

</div>

Browser sign-in, no API key to copy. One hosted Streamable HTTP endpoint, ten tools: turn a name or a description into tickers, list and search a company's filings (structured as-reported facts and the filed text), run read-only SQL over the financial tables, and pull research signals from earnings calls and news.

> ⭐ **If drillr helps your agent, star us — that's how we know to keep building this in the open.**

## Quick Start

1. Sign up at [drillr.ai](https://drillr.ai) (80 free credits, no card)
2. Add `https://gateway.drillr.ai/mcp/data` to an OAuth-capable MCP client as `drillr-data`
3. Approve **drillr Data** in the browser. Restart the client — MCP tools load at startup.

Per-client instructions, kept current on the docs site: <https://drillr.ai/developer/mcp-install.md>. Or paste this into any coding agent with a shell:

```text
Please install the drillr MCP server for me and walk me through the sign-in: server drillr-data, url https://gateway.drillr.ai/mcp/data, auth browser OAuth. Use my client's own MCP command to add it. Per-client details: https://drillr.ai/developer/mcp-install.md
```

### Claude Code

```bash
claude mcp add --scope user --transport http drillr-data https://gateway.drillr.ai/mcp/data
```

Restart, run `/mcp`, pick `drillr-data` → **Authenticate**.

### Codex CLI

```bash
codex mcp add drillr-data --url https://gateway.drillr.ai/mcp/data
```

Codex starts the browser sign-in from this command. `codex mcp login drillr-data` re-runs it later.

### Cursor / VS Code / Claude Desktop / other OAuth hosts

```jsonc
{ "mcpServers": { "drillr-data": { "type": "http", "url": "https://gateway.drillr.ai/mcp/data" } } }
```

Cursor reads `.cursor/mcp.json`; VS Code reads `.vscode/mcp.json` with a top-level `servers` key; Claude Desktop takes the URL under Settings → Connectors → Add custom connector.

[![Install in Cursor](https://img.shields.io/badge/Install_in-Cursor-171717?style=for-the-badge&logo=cursor&logoColor=white)](https://cursor.com/en/install-mcp?name=drillr-data&config=eyJ1cmwiOiJodHRwczovL2dhdGV3YXkuZHJpbGxyLmFpL21jcC9kYXRhIn0=) [![Install in VS Code](https://img.shields.io/badge/Install_in-VS_Code-0078D4?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=drillr-data&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fgateway.drillr.ai%2Fmcp%2Fdata%22%7D)

### API-key fallback

For hosts without browser OAuth (Hermes, OpenClaw, self-built clients) or for REST: create an `external` key at [drillr.ai/account/api-keys](https://drillr.ai/account/api-keys), keep it in a secret store, and add it as a bearer header on the same URL. Never configure OAuth and a static header on the same server entry.

```jsonc
"headers": { "Authorization": "Bearer <YOUR_DRILLR_API_KEY>" }
```

```yaml
# Hermes Agent
mcp_servers:
  drillr-data:
    url: 'https://gateway.drillr.ai/mcp/data'
    headers: { Authorization: 'Bearer <YOUR_DRILLR_API_KEY>' }
```

### Agent skill

The companion skill teaches an agent how to onboard a user and use drillr well over REST or MCP, with a generated endpoint reference:

```bash
npx skills add Little-Grebe-Inc/drillr-skill
```

<https://github.com/Little-Grebe-Inc/drillr-skill>

### Smithery / Claude Code plugin fallbacks

```bash
npx -y @smithery/cli install drillr/drillr --client claude   # Smithery: leave the key empty on OAuth clients
```

```text
/plugin marketplace add Little-Grebe-Inc/drillr-mcp-server      # Claude Code plugin marketplace
/plugin install drillr
```

## Hello World

> _"What does NVDA's latest 10-K say about supply commitments, and how did data-center revenue move over the last four quarters?"_

1. The agent calls `ticker_lookup` if it only has a name, `filing_list` to see what is indexed, then `filing_search` — which returns **as-reported facts** (value, period, XBRL concept, accession number) and the **passages** they sit in, in one call.
2. For the quarterly series it calls `run_sql` on `financial_statements`.
3. You get an answer whose every number cites its filing. Typical round-trip 8–15 s.

## The ten tools

Each tool has a page at `https://drillr.ai/docs/mcp/<tool>` with parameters, limits and error shapes; the server's `tools/list` carries the same descriptions. The set stays small on purpose: the agent composes them.

| Group | Tool | What it does | Cost |
|---|---|---|---|
| **Company** | [`ticker_lookup`](https://drillr.ai/docs/mcp/ticker_lookup) | Name, brand or ticker substring → canonical symbols, historical names included | free |
| | [`company_search`](https://drillr.ai/docs/mcp/company_search) | Natural-language description → companies with a match reason; US / CN / JP / HK / KR; all matches returned | 3–5 cr |
| **Filings** | [`filing_list`](https://drillr.ai/docs/mcp/filing_list) | Which filings are indexed for a ticker: fiscal period, type, filing date | 0.1 cr |
| | [`filing_search`](https://drillr.ai/docs/mcp/filing_search) | Search one company's filings; returns structured as-reported facts and the filing text together | 0.1 cr |
| **Datasets** | [`list_tables`](https://drillr.ai/docs/mcp/list_tables) | Alt-data category index, or the tables under up to five categories | free |
| | [`get_table_schema`](https://drillr.ai/docs/mcp/get_table_schema) | One table's columns, types and usage note — required filters, coverage, gotchas | free |
| | [`run_sql`](https://drillr.ai/docs/mcp/run_sql) | One read-only PostgreSQL SELECT over the financial, market and alt-data tables | 0.1 cr |
| **Signal** | [`industry_inflections`](https://drillr.ai/docs/mcp/industry_inflections) | Industry changes synthesized from many US earnings calls: mechanism, scope, degree, per-company impact | 1 cr |
| | [`ai_adoption`](https://drillr.ai/docs/mcp/ai_adoption) | Concrete enterprise AI applications disclosed on US earnings calls: workflow, stage, value, evidence | 1 cr |
| | [`news_search`](https://drillr.ai/docs/mcp/news_search) | Semantic search over company and market news: storylines, events, attributed claims | 0.2 cr |

## What the data is

Built by drillr from primary sources — filings taken from each market's official channel (SEC EDGAR, cninfo, EDINET) and parsed in house, what companies publish on the web (IR pages, earnings calls, news), and drillr's own estimates. No data vendor in between. Every reported figure links back to the passage it was filed in. Provenance per dataset: <https://drillr.ai/docs/provenance>.

| Dataset | What is in it | How to reach it | Coverage |
|---|---|---|---|
| **Company** | Ticker resolution from any name, code, ISIN, CIK or CUSIP; discovery by description; profile (exchange, industry, listing date, website, headcount) | `ticker_lookup`, `company_search`; `company_snapshot` via SQL | US CN JP (discovery + HK KR) |
| **Filings** | Filing index with form type, date and official link; full-text search in EN / ZH / JA; as-reported fact records with value, unit, period, filing and position, linked in a knowledge graph so the right version of a figure wins | `filing_list`, `filing_search` | US CN JP |
| **Financials** | Income statement, balance sheet, cash flow as reported under each market's standard, annual and quarterly; precomputed valuation, margin, return, growth, leverage and liquidity metrics | `run_sql` on `financial_statements`, `company_snapshot` | US JP HK CN KR |
| **Prices** | Daily / weekly / monthly OHLCV, indices, pre- and after-market quotes | `run_sql` on `price_volume_history`, `index_price`, `equity_extended_rt` | equities US JP HK CN KR; indices, FX, crypto, commodities |
| **Earnings** | Calendar with estimates and actuals; structured call summaries (highlights, guidance, risks, segments, Q&A) | `run_sql` on `earning_call_calendar`, `earning_call_summary` | US JP |
| **Ownership** | Insider holdings and transactions (Forms 3 / 4 / 5); institutional quarterly positions (13F-HR) | `run_sql` on `insider_and_institution_activities` | US |
| **Events** | 8-K corporate events: executive changes, deals, debt issuance, securities offerings; 13D / G stake changes | `run_sql` on `executive_change`, `company_deal_events`, `debt_issuance`, `securities_offering` | US |
| **Executives** | Roster with status; annual compensation from DEF 14A | `run_sql` on `executive_profile`, `executive_compensation` | US |
| **Analyst** | Individual rating and target changes; consensus distribution and targets | `run_sql` on `analyst_ratings`, `analyst_ratings_consensus` | US |
| **Signal** | Conclusions drawn by drillr with the evidence quoted: industry inflections, enterprise AI adoption, cross-source news storylines and attributed claims | `industry_inflections`, `ai_adoption`, `news_search` | US (news: US CN JP + macro) |
| Alt-data (secondary) | 9 categories, 65 tables around the AI supply chain and macro: energy & power, data centers, semiconductors, compute pricing, model development, inference economics, macro & trade, prediction markets, critical minerals | `list_tables` → `get_table_schema` → `run_sql` | global |

Ticker forms: US bare (`AAPL`), A-shares `.SH` / `.SZ` (`600519.SH`), Japan `.T` (`6758.T`), Hong Kong `.HK` (`00700.HK`), Korea `.KS` / `.KQ` (`005930.KS`); indices `^GSPC`; quote symbols with `.` or `^` in SQL. Freshness: filings, ownership and events within minutes of the filing; calls and news within minutes to hours; statements and analyst data daily.

Benchmark: the drillr MCP-powered model scores 94.06 % on Vals Finance Agent, and drillr publishes its own restatement-aware benchmark — <https://drillr.ai/drillr-benchmark>.

## REST API

The same data as 29 typed REST endpoints for scripts and pipelines — one `X-API-KEY` header, JSON out, public OpenAPI 3.1 contract:

```bash
curl -H "X-API-KEY: $DRILLR_API_KEY" \
  "https://gateway.drillr.ai/api/v2/income-statements?ticker=AAPL&period=FY&limit=4"
```

Reference: <https://drillr.ai/docs/api> · Contract: <https://gateway.drillr.ai/api/v2/openapi.json>. The older `/api/v1/data/*` tool mirrors are documented in [`docs/rest-api.md`](./docs/rest-api.md).

## Pricing and limits

One credit wallet across MCP and REST. Free: 80 credits on sign-up. Plus $29 / 300 cr per month, Ultra $99 / 1,500 cr, Enterprise for redistribution and SLA. `run_sql` returns 100 rows (500 on Ultra), 10 s per statement, 5 in flight; `company_search` is capped per day (20 / 100 / 500). Failed calls are not billed. <https://drillr.ai/pricing>

## Out of scope

Private or unlisted companies · on-chain crypto metrics (CEX prices only) · options chains, order book, tick data · placing orders · drillr does not produce its own price forecasts.

## Community

<table>
  <tr>
    <td align="center" width="50%"><a href="https://discord.gg/YAh96nw5Vh"><b>Discord</b></a></td>
    <td align="center" width="50%"><b>WeChat</b></td>
  </tr>
  <tr>
    <td align="center"><img src="https://gateway.drillr.ai/qr/discord.svg" width="160" alt="drillr Discord QR" /></td>
    <td align="center"><img src="https://gateway.drillr.ai/qr/wechat.svg" width="160" alt="drillr WeChat group QR" /></td>
  </tr>
  <tr>
    <td align="center">Devs building agentic research products — office hours, debugging help, early access.</td>
    <td align="center">Chinese-speaking dev community — fastest product feedback.</td>
  </tr>
</table>

## License

MIT — see [`LICENSE`](./LICENSE).
