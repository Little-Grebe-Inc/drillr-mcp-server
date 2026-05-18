# Tool Reference

> drillr 对外的 8 个 MCP tool 完整参考手册。每个 tool 一段：用途、参数、调用示例、返回样本、常见陷阱、sibling tool 指引。
>
> **MCP 与 REST 等价**：MCP tools/call 与 REST endpoint 走同一份 `drl_*` API key、同样的数据、同样的计费。下面示例用 REST（最易在终端跑）；通过 MCP 调时把参数包成 `tools/call` 的 `arguments` 字段即可。
>
> **CLI(`drillr` 命令行)即将推出。**

## 目录

URL: `https://gateway.drillr.ai/mcp/data` · 8 tools · 一把 `drl_*` key 全部走通。

| Tool | 单价 | REST endpoint |
|---|---|---|
| [`run_sql`](#run_sql) | 1 cr / call | `POST /api/v1/data/run_sql` |
| [`sec_report_search`](#sec_report_search) | 1 cr / call | `POST /api/v1/data/sec_report_search` |
| [`sec_report_list`](#sec_report_list) | 1 cr / call | `GET /api/v1/data/sec_report_list` |
| [`company_search`](#company_search) | LLM-cost(`max(2, ceil(usd/0.034))`) | `POST /api/v1/data/company_search` |
| [`signal_list`](#signal_list) | 2 cr / call | `GET /api/v1/data/signal_list` |
| [`list_tables`](#list_tables) | Free | `GET /api/v1/data/list_tables` |
| [`get_table_schema`](#get_table_schema) | Free | `GET /api/v1/data/get_table_schema?table_name=:table` |
| [`fiscal_utility`](#fiscal_utility) | Free | `GET /api/v1/data/fiscal_utility` |

**Recommended workflow**:

1. `fiscal_utility` / `get_table_schema` / `list_tables` — call freely to explore
2. `signal_list` for recent news + market events
3. `run_sql` for structured data (financial / market / alt-data, 90+ tables)
4. `sec_report_search` for narrative (10-K / 10-Q risk factors, segment, guidance)
5. `company_search` for qualitative discovery (industry, supply chain, competitors)

---

### `run_sql`

Read-only PostgreSQL SELECT against 90+ structured tables. The workhorse of the toolkit.

**When to use**:

- Need structured rows (financials, market data, alt-data)
- Custom filter / JOIN / aggregation that no specialized tool covers

**When NOT to use**:

- SEC filing narrative content → use [`sec_report_search`](#sec_report_search) instead
- Qualitative company discovery → use [`company_search`](#company_search) instead
- Recent news + market events → use [`signal_list`](#signal_list) instead

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `sql` | string | Yes | PostgreSQL SELECT statement |

**SQL constraints**:

- No CTE (`WITH ... AS`) — use subqueries instead
- Date columns are TEXT — use plain string comparison (`period_end >= '2024-01'`). Never `::date` cast or `INTERVAL` arithmetic
- No `ROUND(float8, int)` — use `CAST(value AS DECIMAL(10,2))` if rounding is needed
- Structured-data queries must filter by ticker (`WHERE ticker IN ('AAPL','MSFT')`). Alt-data is macro / industry / patent — no ticker filter required

**Pricing**: 1 cr / call, flat across all 90+ tables regardless of how many tables a query JOINs.

**Example call** (price_volume_history):

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/run_sql \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sql":"SELECT ticker, period_end, close FROM price_volume_history WHERE ticker='\''AAPL'\'' AND time_frame='\''daily'\'' ORDER BY period_end DESC LIMIT 5"}'
```

**Response**:

```json
{
  "data": {
    "columns": ["ticker", "period_end", "close"],
    "rows": [
      ["AAPL", "2026-05-08", 232.55],
      ["AAPL", "2026-05-07", 230.12]
    ],
    "rowCount": 5
  },
  "_credits": { "charged": 1, "method": "per_call", "balance_after": 505 }
}
```

> Response envelope:REST 响应统一为 `{data, _credits}` 形式(见 [REST API § Response Envelope](./rest-api.md#response-envelope))。

**Ticker conventions** (for `price_volume_history` and other market tables):

| Class | Format | Examples |
|---|---|---|
| US stock / ETF | bare 1-5 letters | `AAPL`, `MSFT`, `SPY`, `QQQ` |
| US index | leading `^` | `^GSPC` (S&P 500), `^DJI`, `^IXIC`, `^NDX`, `^RUT`, `^VIX` |
| Foreign listing | exchange suffix | `1557.T` (JP), `310960.KS` (KR) — Hong Kong / A-shares / Korea native coming soon |
| Foreign index | leading `^` | `^N225` (Nikkei 225), `^TPX` (TOPIX), `^FTSE`, `^GDAXI` |
| Commodity | code + USD/USX | `CLUSD` (WTI futures), `GCUSD` (gold), `ZCUSX` (corn in cents) |
| Forex | base+quote, no separator | `EURUSD`, `USDJPY`, `GBPUSD` |
| Crypto | token + USD | `BTCUSD`, `ETHUSD`, `SOLUSD` |

**Common pitfalls**:

- Same asset, different tickers: NASDAQ 100 index `^NDX` (~26,000) vs ETF `QQQ` (~640). Pick the one matching user intent.
- WTI spot ≠ futures. `CLUSD` is NYMEX futures, not spot.
- Tickers with `.` or `^` MUST be quoted in SQL: `WHERE ticker = '^NDX'`.

---

### `sec_report_search`

Paragraph-level semantic search across SEC filings.

**When to use**:

- Narrative content: risk factors, MD&A, guidance language, segment commentary, related-party transactions, accounting policies
- Search for specific phrases or concepts within filings

**When NOT to use**:

- Consolidated financial numbers (revenue, margins, EPS) — use [`run_sql`](#run_sql) on `financial_statements` instead
- Institutional holdings (13F, 13D, 13G) — use `run_sql` on `insider_and_institution_activities` with `source='institution'`
- Listing what filings exist for a ticker — use [`sec_report_list`](#sec_report_list) first

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `ticker` | string | Yes | Stock ticker, e.g. `NVDA`, `AAPL`. Foreign companies via ADR (e.g. `TSM` for TSMC) |
| `query` | string | Yes | Natural-language search phrase, e.g. "supply chain concentration", "share dilution" |
| `top_k` | integer | No | Max paragraphs returned (default 10, max 30) |
| `period_start` | string | No | YYYY-MM, filter filings from this period |
| `period_end` | string | No | YYYY-MM, filter filings up to this period |
| `filing_types` | string[] | No | e.g. `["10-K","10-Q"]`. Default covers main reports (10-K / 10-Q / 20-F / 6-K / S-1 / F-1) |

**Filing types covered**:

- 10-K (US annual), 10-Q (US quarterly), 8-K (US current/material events)
- 20-F (foreign annual), 6-K (foreign current)
- S-1 (US IPO registration), F-1 (foreign IPO registration)

**Pricing**: 1 cr / call.

**Example call**:

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/sec_report_search \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "ticker": "NVDA",
    "query": "data center revenue concentration",
    "top_k": 5,
    "period_start": "2024-01"
  }'
```

**Search strategy**:

- **Time-specific queries** ("recent", "last year", "Q4 2025"): apply a time range filter upfront to narrow results
- **Broad queries**: do 1-2 minimal-filter searches first to discover what filings contain — then add filters

**High-value search scenarios** (use proactively when relevant):

- Share dilution / SBC / buyback → search `"Shareholders' Equity"` or `"Capital Stock"`
- Facilities / properties → search `"Properties"` for owned vs leased, lease terms, land use rights
- Interest rate sensitivity → search `"Market Risk"` or `"Interest Rate Risk"`
- Segment revenue breakdown → search `"Segment Information"` or `"Revenue Disaggregation"`
- Risk factors → search `"Risk Factors"` for specific risks
- Management guidance → search `"Outlook"` or `"Guidance"` in MD&A
- Accounting policies → search `"Critical Accounting Policies"`
- Stock split history → search `"Stock Split"` for split ratios and dates

---

### `sec_report_list`

List indexed SEC filings for a ticker, with a summary header.

**When to use**:

- Discover which filings exist for a ticker before searching content
- Audit filing coverage for a company

**When NOT to use**:

- Filing content → use [`sec_report_search`](#sec_report_search) instead

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `ticker` | string | Yes | Stock ticker |
| `filing_types` | string[] | No | Filter by type. Omit for default (periodic reports + IPO/shelf + proxy: 10-K, 10-Q, 20-F, S-1, F-1, S-3, F-3, DEF 14A and /A amendments; excludes 8-K/6-K). Pass `[]` for all indexed types. Pass explicit allowlist like `["8-K"]` to override |

**Pricing**: 1 cr / call.

**Example call**:

```bash
curl -G https://gateway.drillr.ai/api/v1/data/sec_report_list \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "ticker=AAPL"
```

**Example response** (truncated):

```
AAPL — 21 filings (28 total before filing_types filter)
Period coverage: 2020-09-26 → 2026-03-28
Types: 10-K×5, 10-Q×14, DEF 14A×2

| FY | Q | Type | Filed | Period |
|---|---|---|---|---|
| 2026 | Q2 | 10-Q | 2026-05-01 | 2025-12-29 ~ 2026-03-28 |
| 2025 | FY | 10-K | 2025-10-31 | 2024-09-29 ~ 2025-09-27 |
...
```

---

### `company_search`

Qualitative company discovery — industry classification, business model, supply chain, competitors, management background.

> **Natural language only.** As of v2.0, `company_search` accepts only natural-language descriptions. The v1.x SQL mode is retired — for numerical screening or schema-based filtering, use [`run_sql`](#run_sql) on `company_snapshot` instead.

**When to use**:

- Qualitative discovery ("EV battery suppliers to Tesla", "Japanese semiconductor equipment makers", "AI inference chip startups")
- Understanding business model, segments, competitive landscape

**When NOT to use**:

- Numerical screening (revenue, margins, ratios, growth rates) — use [`run_sql`](#run_sql) on `company_snapshot` instead
- Specific known company's financials — use `run_sql` directly
- Writing raw SQL against a company knowledge base table — that's a v1.x pattern; use the NL query, or use `run_sql` if you really need column-level filtering

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `query` | string | Yes | Natural-language description of the companies you're looking for (e.g. "lithium-ion battery cell makers supplying European OEMs"). Do not pass SQL. |

**Pricing**: LLM-cost based — `max(2, ceil(api_cost_usd / 0.034))` cr / call.

**Example call**:

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/company_search \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "AI inference chip startups"}'
```

**Coverage**:

- Industry classification, product offerings, business model
- Segment structure, competitive landscape, supply chain
- Management background, customer profile

Returns a structured list of matching companies with context snippets.

---

### `signal_list`

Recent news + market events filtered by ticker / sector / time range. Each row is one signal: headline, summary, suggested_tickers, sector, created_at.

**When to use**:

- Recent news on specific tickers or sectors
- Polling for new market-moving events since last check
- Building news-driven alerting workflows

**When NOT to use**:

- SEC filing narrative → use [`sec_report_search`](#sec_report_search)
- Historical archive older than ~30 days → use [`run_sql`](#run_sql) on the underlying `content_signals` table

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `tickers` | string[] | No | Return signals whose `suggested_tickers` overlap any of these |
| `sector` | string[] | No | Return signals whose sector overlaps any of these |
| `since` | string (ISO 8601) | No | Return signals created at or after this timestamp |
| `limit` | integer | No | Default 20, max 100 |
| `offset` | integer | No | Pagination offset, default 0 |

**Pricing**: 2 cr / call.

**Coverage**:

- ~6,900 tickers across US + ADRs of global companies
- Cross-asset: equities, macro, geopolitics, commodities, crypto
- Continuously updating, typically <1 hour lag from source
- Newest first by `created_at`

**Example call**:

```bash
curl -G https://gateway.drillr.ai/api/v1/data/signal_list \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "tickers=NVDA,AMD" \
  --data-urlencode "limit=5"
```

**Example response**:

```json
{
  "data": {
    "items": [
      {
        "headline": "Greg Abel ends Berkshire's Buffett-era share selling streak",
        "summary": "...",
        "suggested_tickers": ["BRK.B", "OXY"],
        "sector": ["Financial Services"],
        "created_at": "2026-05-11T04:34:19Z"
      }
    ]
  },
  "_credits": { "charged": 2, "method": "per_call", "balance_after": 498 }
}
```

---

### `list_tables`

List alternative-data tables under given categories. Returns each table's name, one-line purpose, and column names.

**When to use**:

- BEFORE [`run_sql`](#run_sql) when exploring alt-data — `run_sql` alone won't tell you which tables exist
- Discovering what's in a category you don't know yet

**Pricing**: Free; call freely.

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `categories` | string[] | Yes | 1-5 alt-data category names |

**Available categories** (24 total):

| Category | Contents |
|---|---|
| Energy & Power | US power plants, electricity prices |
| Data Centers | Facilities, GPU clusters, cooling |
| Chip Specs | AI chip hardware specifications |
| Chip Sales | Chip sales by chip and by designer |
| Chip Ownership | Chip owner timelines |
| Foundry & Trade | TSMC revenue, Taiwan customs |
| Compute Pricing | GPU rental, cloud VM spot / on-demand |
| AI Models & Benchmarks | Model specs, training compute, benchmark scores |
| AI Companies | Revenue, staff, funding, compute spend |
| AI Polling | Public opinion polls on AI |
| LLM Token Pricing | LLM API pricing across providers |
| OpenRouter Usage | Model/app-level usage on OpenRouter |
| App Store Rankings | iOS / Android app rankings |
| Web Traffic | Cloudflare domain rankings |
| Developer Skills | Job-market skill demand signals |
| Twitter | Tweets, users (curated finance accounts) |
| Reddit | Posts, comments (finance subreddits) |
| Substack | Posts, authors |
| YouTube | Videos with transcripts |
| Government Contracts | US federal contracts |
| Trade Flows | US / global imports / exports |
| Macro Indicators | Macro data (US-focused) |
| Patents | USPTO granted patents |
| Papers | Academic papers with citations and topics |

**Example call**:

```bash
curl -G https://gateway.drillr.ai/api/v1/data/list_tables \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "categories=LLM Token Pricing,Compute Pricing"
```

---

### `get_table_schema`

Look up column definitions (name, type, description) for a specific table.

**When to use**:

- BEFORE [`run_sql`](#run_sql) when you're unsure which columns a table has
- Verifying column types before constructing a JOIN

**Pricing**: Free; call freely.

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `table_name` | enum | Yes | One of the 90+ table names (enum enforced) |

**Example call**:

```bash
curl https://gateway.drillr.ai/api/v1/data/get_table_schema?table_name=financial_statements \
  -H "Authorization: Bearer $DRILLR_API_KEY"
```

**Example response** (truncated):

```
Table: financial_statements

| Column | Type | Description |
|---|---|---|
| ticker | text | Stock ticker |
| period_end | text | yyyy-mm |
| fiscal_period | text | 'FY' annual or 'Q1'..'Q4' quarterly |
| revenue | numeric | Total revenue, reporting currency |
...
```

---

### `fiscal_utility`

Bidirectional fiscal year ↔ calendar month conversion. Different companies have different fiscal year starts (Apple ends in September, Nvidia ends in January) — use this before filtering on `period_end` columns.

**When to use**:

- Converting "Nvidia Q3 FY2026" → calendar months (Aug 2025 - Oct 2025)
- Converting "what fiscal quarter is 2025-08 for Nvidia" → FY2026 Q3

**Pricing**: Free; call freely.

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `ticker` | string | Yes | Stock ticker |
| `fiscal_year` | integer | Forward only | Fiscal year (required for forward conversion) |
| `fiscal_quarter` | integer (0-4) | Forward only | Quarter 1-4, or 0 for full fiscal year |
| `yyyy_mm` | string | Reverse only | Calendar month yyyy-mm (required for reverse conversion) |

**Example call** (forward: NVDA FY2026 Q3 → calendar):

```bash
curl -G https://gateway.drillr.ai/api/v1/data/fiscal_utility \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "ticker=NVDA" \
  --data-urlencode "fiscal_year=2026" \
  --data-urlencode "fiscal_quarter=3"
```

**Response**:

```json
{
  "ticker": "NVDA",
  "fiscal_year": 2026,
  "fiscal_quarter": 3,
  "period_start": "2025-08",
  "period_end": "2025-10"
}
```

---

## Data Coverage Reference

For the full list of structured tables, categories, ticker conventions, and what's currently in coverage vs coming soon, see:

- [Developer docs / data coverage](https://drillr.ai/developer/docs/coverage)
- [REST API reference](./rest-api.md) — endpoint-by-endpoint, plus the `{ data, _credits }` envelope contract
- [Pricing](./pricing.md) — Tier 总览 / 差异 / 单价矩阵 / Referral
