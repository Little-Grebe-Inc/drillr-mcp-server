# Tool Reference

> drillr 对外的 9 个 MCP tool 完整参考手册。每个 tool 一段：用途、参数、调用示例、返回样本、常见陷阱、sibling tool 指引。
>
> **MCP 与 REST 等价**：MCP tools/call 与 REST endpoint 走同一份 `drl_*` API key、同样的数据。下面示例用 REST（最易在终端跑）；通过 MCP 调时把参数包成 `tools/call` 的 `arguments` 字段即可。
>
> **CLI(`drillr` 命令行)即将推出。**

## 目录

URL: `https://gateway.drillr.ai/mcp/data` · 9 tools · 一把 `drl_*` key 全部走通。

| Tool | REST endpoint |
|---|---|
| [`run_sql`](#run_sql) | `POST /api/v1/data/run_sql` |
| [`sec_report_search`](#sec_report_search) | `POST /api/v1/data/sec_report_search` |
| [`sec_report_list`](#sec_report_list) | `GET /api/v1/data/sec_report_list` |
| [`company_search`](#company_search) | `POST /api/v1/data/company_search` |
| [`ticker_lookup`](#ticker_lookup) | `POST /api/v1/data/ticker_lookup` |
| [`news_search`](#news_search) | `POST /api/v1/data/news_search` |
| [`list_tables`](#list_tables) | `GET /api/v1/data/list_tables` |
| [`get_table_schema`](#get_table_schema) | `GET /api/v1/data/get_table_schema?table_name=:table` |
| [`fiscal_utility`](#fiscal_utility) | `GET /api/v1/data/fiscal_utility` |

**Recommended workflow**:

1. `fiscal_utility` / `get_table_schema` / `list_tables` — call freely to explore
2. `news_search` for news, market events, and attributed claims
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
- News, market events, and attributed claims → use [`news_search`](#news_search) instead

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `sql` | string | Yes | PostgreSQL SELECT statement |

**SQL constraints**:

- No CTE (`WITH ... AS`) — use subqueries instead
- Date columns are TEXT — use plain string comparison (`period_end >= '2024-01'`). Never `::date` cast or `INTERVAL` arithmetic
- No `ROUND(float8, int)` — use `CAST(value AS DECIMAL(10,2))` if rounding is needed
- Structured-data queries must filter by ticker (`WHERE ticker IN ('AAPL','MSFT')`). Alt-data is macro / industry / patent — no ticker filter required


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
  "_credits": { "charged": "0.1", "method": "per_call", "balance_after": "505.0" }
}
```

> Response envelope:REST 响应统一为 `{data, _credits}` 形式(见 [REST API § Response Envelope](./rest-api.md#response-envelope))。

**Ticker conventions** (for `price_volume_history` and other market tables):

| Class | Format | Examples |
|---|---|---|
| US stock / ETF | bare 1-5 letters | `AAPL`, `MSFT`, `SPY`, `QQQ` |
| US index | leading `^` | `^GSPC` (S&P 500), `^DJI`, `^IXIC`, `^NDX`, `^RUT`, `^VIX` |
| Japan listing | `.T` suffix | `6758.T`, `7203.T` |
| Hong Kong listing | five digits + `.HK` | `00700.HK`, `09988.HK` |
| China A-share | `.SH` / `.SZ` suffix | `600519.SH`, `300750.SZ` |
| Other foreign listing | exchange suffix | `310960.KS` (KR), `FTSEMIB.MI` (Italy) |
| Foreign index | leading `^` | `^N225` (Nikkei 225), `^TPX` (TOPIX), `^FTSE`, `^GDAXI` |
| Commodity | code + USD/USX | `CLUSD` (WTI futures), `GCUSD` (gold), `ZCUSX` (corn in cents) |
| Forex | base+quote, no separator | `EURUSD`, `USDJPY`, `GBPUSD` |
| Crypto | token + USD | `BTCUSD`, `ETHUSD`, `SOLUSD` |

**Common pitfalls**:

- Same asset, different tickers: NASDAQ 100 index `^NDX` (~26,000) vs ETF `QQQ` (~640). Pick the one matching user intent.
- WTI spot ≠ futures. `CLUSD` is NYMEX futures, not spot.
- Tickers with `.` or `^` MUST be quoted in SQL: `WHERE ticker = '^NDX'`.

**Equity coverage**:

- `financial_statements`, `company_snapshot`, and `price_volume_history` cover US, Japan, Hong Kong, and China A-shares.
- `price_volume_history` additionally carries other global listings, indices, FX, commodities, and crypto.
- Specialized tables can be narrower: earnings calls/calendar are US + Japan; analyst, ownership, executive, 8-K event, and extended-hours datasets are US-only.
- Call `get_table_schema` before interpreting zero rows as a factual absence.

---

### `sec_report_search`

Paragraph-level semantic search across company-filed reports in the US, Japan, Hong Kong, and China A-shares.

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
| `ticker` | string | Yes | US bare (`NVDA`), Japan `.T` (`6758.T`), Hong Kong `.HK` (`00700.HK`), or A-share `.SH` / `.SZ` (`600519.SH`) |
| `query` | string | Yes | Natural-language search phrase, e.g. "supply chain concentration", "share dilution" |
| `top_k` | integer | No | Max paragraphs returned (default 10, max 30) |
| `period_start` | string | No | YYYY-MM, filter filings from this period |
| `period_end` | string | No | YYYY-MM, filter filings up to this period |
| `filing_types` | string[] | No | Filing-type allowlist. Omit to search all types. Values differ by market; use `sec_report_list` to discover available values. |

**Filing types covered**:

- 10-K (US annual), 10-Q (US quarterly), 8-K (US current/material events)
- 20-F (foreign annual), 6-K (foreign current)
- S-1 (US IPO registration), F-1 (foreign IPO registration)
- Japan EDINET numeric codes such as `120` (annual), `140` (quarterly), and `160` (semi-annual)
- Hong Kong / A-share names such as `annual_report`, `quarterly_report`, and A-share quarter-specific values such as `q1_report`

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

List indexed company filings for a ticker, with a summary header.

**When to use**:

- Discover which filings exist for a ticker before searching content
- Audit filing coverage for a company

**When NOT to use**:

- Filing content → use [`sec_report_search`](#sec_report_search) instead

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `ticker` | string | Yes | `NVDA`, `6758.T`, `00700.HK`, `600519.SH`, etc. |
| `filing_types` | string[] | No | Omit for the market-appropriate main reports; pass `[]` for all indexed types; or pass an explicit allowlist using values returned by a prior call |


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
| `market` | string or string[] | No | One value or a list from `"us"` / `"jp"` / `"hk"` / `"cn"`. Omit or pass `[]` to search all four markets. List order does not set priority. |

**Example call**:

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/company_search \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"Hong Kong and China EV battery suppliers","market":["hk","cn"]}'
```

**Coverage**:

- US, Japan, Hong Kong, and China A-shares
- Industry classification, product offerings, business model
- Segment structure, competitive landscape, supply chain
- Management background, customer profile

Returns a structured list of matching companies with context snippets.

---

### `ticker_lookup`

Resolve a company name, brand, or ticker substring to canonical ticker(s). Use this FIRST when the user mentions a company by name / brand / nickname before running any ticker-keyed tool.

**When to use**:

- User mentions a company by name (e.g. "Apple", "苹果", "OpenAI", "Tesla") and you need the canonical ticker before calling [`sec_report_search`](#sec_report_search) / [`sec_report_list`](#sec_report_list) / [`run_sql`](#run_sql)
- Mapping historical / former names to current ticker
- Disambiguating between similar-sounding companies — top 5 ranked matches help the agent pick

**When NOT to use**:

- User already gave you the ticker directly — skip this, go straight to the keyed tool
- Looking for companies *by description* (industry, business model, supply chain) — use [`company_search`](#company_search) instead

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `query` | string | Yes | Company name, brand, or ticker substring. Matching is case-insensitive and includes ticker history. |
| `market` | string | No | Optional filter: `"us"` \| `"jp"` \| `"hk"` \| `"cn"`. Omit to search all four markets. |


**Example call**:

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/ticker_lookup \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"Tencent","market":"hk"}'
```

Returns up to 5 matches ranked by prefix-hit first, then name length.

Current ticker formats are US bare (`AAPL`), Japan `.T` (`6758.T`), Hong Kong five digits + `.HK` (`00700.HK`), and A-shares `.SH` / `.SZ` (`600519.SH`). Results include ticker history, so older/raw entries may occasionally appear without the current suffix. Localized-name recall varies by company; retry with the official English name or ticker substring if a local-language name misses.

The old REST path `POST /api/v1/data/ticker_resolve` is a temporary deprecated alias. The MCP tool itself has no old-name alias.

---

### `news_search`

Semantic search over news, market events, and attributed claims, grouped into storylines.

**When to use**:

- News or developments about a company, theme, or market
- Market-moving events within a time window
- Attributed statements such as analyst actions, company guidance, or central-bank remarks

**When NOT to use**:

- SEC filing narrative → use [`sec_report_search`](#sec_report_search)
- Company discovery by business description → use [`company_search`](#company_search)

**Parameters**:

| Name | Type | Required | Description |
|---|---|---|---|
| `query` | string | Conditional | English semantic query |
| `theme` | string | Conditional | Theme resolved to the nearest canonical theme; not valid by itself with `search_type="claims"` |
| `ticker` | string or string[] | Conditional | One exact ticker, an array, or a comma-separated string. Multiple tickers act as an OR filter. |
| `since` | ISO 8601 string | Conditional | Include events with `time_event >= since` |
| `until` | ISO 8601 string | Conditional | Include events with `time_event < until` |
| `search_type` | enum | No | `"all"` (default), `"events"`, or `"claims"` |
| `order_by` | enum | No | `"relevance"` (default), `"event_time"`, or `"create_time"` |
| `top_k` | integer | No | Story count; default 10, max 50 |

At least one of `query`, `theme`, `ticker`, `since`, or `until` is required.

**Coverage**:

- US, Japan, Hong Kong, and China A-shares
- Cross-asset: equities, macro, geopolitics, commodities, crypto
- Ticker formats: `AAPL`, `7203.T`, `00700.HK`, `600519.SH`

**Example call**:

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/news_search \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"ticker":"00700.HK","search_type":"events","order_by":"event_time","top_k":3}'
```

MCP renders a compact Markdown response: a numbered `Stories` section followed by flat `Events` and `Claims` tables. REST returns structured JSON containing story-grouped events and a parallel claims collection.

`signal_list` and `GET /api/v1/data/signal_list` are retired with no alias.

---

### `list_tables`

List alternative-data tables under given categories. Returns each table's name, one-line purpose, and column names.

**When to use**:

- BEFORE [`run_sql`](#run_sql) when exploring alt-data — `run_sql` alone won't tell you which tables exist
- Discovering what's in a category you don't know yet


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

Coverage is primarily US, with sparse Japan / Hong Kong configuration and no verified China A-share configuration. Do not infer support from the broader four-market coverage of the core equity tables.

**When to use**:

- Converting "Nvidia Q3 FY2026" → calendar months (Aug 2025 - Oct 2025)
- Converting "what fiscal quarter is 2025-08 for Nvidia" → FY2026 Q3


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

For the full list of structured tables, categories, ticker conventions, and current coverage boundaries, see:

- [Developer docs / data coverage](https://drillr.ai/developer/docs/coverage)
- [REST API reference](./rest-api.md) — endpoint-by-endpoint, plus the `{ data, _credits }` envelope contract
