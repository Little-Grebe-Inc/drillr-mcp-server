# REST API Reference

> MCP 与 REST 等价：用同一份 `drl_*` API key、同样的数据、同样的计费。本文档对应 REST 路径。CLI(`drillr` 命令行)即将推出。
>
> Each MCP tool maps 1:1 to a REST endpoint — see [`tools.md`](./tools.md) for tool semantics, params, and what-to-use-when; this doc focuses on HTTP-specific concerns (auth, schemas, rate limits, errors).

---

## Base URL

```
https://gateway.drillr.ai
```

---

## Authentication

API key generated at [drillr.ai/developer/keys](https://drillr.ai/developer/keys) (`external` scope). Key format: `drl_xxxxxxxx_xxx...` (45 chars). **Shown only once at creation — store in a secrets manager immediately.**

Two equivalent ways to pass it:

```http
# Recommended
Authorization: Bearer drl_xxxxxxxx_xxx...

# Also supported
X-API-Key: drl_xxxxxxxx_xxx...
```

> Legacy `dgr_live_*` keys issued before 2026-04 are permanently compatible — no forced rotation. New keys use the `drl_` prefix.

> **OAuth not supported on `/api/v1/*` REST endpoints.** OAuth 2.1 is available only on the v1.x MCP endpoint (`/mcp/private` — internal use); the v2.0 external MCP endpoint (`/mcp/data`) and all REST endpoints are Bearer API key only.

---

## Endpoint Map

| Method | Path | Tool | Cr / call |
|---|---|---|---|
| `POST` | `/api/v1/search` | [`search`](#post-apiv1search) | LLM-cost (`max(2, ceil(usd/0.034))`) |
| `POST` | `/api/v1/data/run_sql` | [`run_sql`](#post-apiv1datarun_sql) | 1 |
| `GET` | `/api/v1/data/list_tables` | [`list_tables`](#get-apiv1datalist_tables) | Free |
| `GET` | `/api/v1/data/get_table_schema?table_name=:table_name` | [`get_table_schema`](#get-apiv1dataget_table_schema) | Free |
| `GET` | `/api/v1/data/sec_report_list` | [`sec_report_list`](#get-apiv1datasec_report_list) | 1 |
| `POST` | `/api/v1/data/sec_report_search` | [`sec_report_search`](#post-apiv1datasec_report_search) | 1 |
| `POST` | `/api/v1/data/company_search` | [`company_search`](#post-apiv1datacompany_search) | LLM-cost (`max(2, ceil(usd/0.034))`) |
| `GET` | `/api/v1/data/signal_list` | [`signal_list`](#get-apiv1datasignal_list) | 2 |
| `GET` | `/api/v1/data/fiscal_utility` | [`fiscal_utility`](#get-apiv1datafiscal_utility) | Free |

---

## Response Envelope

Every `2xx` response from `/api/v1/*` REST endpoints wraps the payload in a uniform envelope so clients can use a single parser across all endpoints:

```json
{
  "data": { /* endpoint-specific payload — object or array */ },
  "_credits": {
    "charged": 1,
    "method": "per_call",
    "balance_after": 503
  }
}
```

> **REST only.** MCP responses (over `/mcp/data`) follow standard JSON-RPC and do **not** carry a `_credits` field — to track usage and balance from an MCP-only client, query [drillr.ai/developer/keys](https://drillr.ai/developer/keys) or use any REST endpoint to peek at `_credits.balance_after`.

### `_credits` field

| Field | Type | Description |
|---|---|---|
| `charged` | integer | Credits deducted for this call. `0` for free tools. |
| `method` | string | Billing model: `per_call` (fixed per-request) / `usage_based` (LLM-driven, scales with upstream cost) / `free` |
| `balance_after` | integer | Credit balance after this call (`monthly_remaining + purchased_remaining`; may be negative when overdrafted) |

See [`pricing.md`](./pricing.md) for the Tier × Tool cost matrix.

---

## Rate Limit

| Limit | Value | Applied to |
|---|---|---|
| Request rate | 30 req / min | Per API key, all endpoints |

On rate limit (`429`), the error envelope includes `retry_after_seconds`:

```json
{
  "error": {
    "code": "rate_limited",
    "message": "Per-key rate limit exceeded",
    "retry_after_seconds": 8
  }
}
```

---

## Error Handling

### Error Response Shape

All errors follow the same envelope (mirroring the success envelope so SDKs can branch on `res.data` vs `res.error`):

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Missing required query param: ticker"
  }
}
```

The `error` object may also include extra fields specific to the error class (e.g. `retry_after_seconds` on `rate_limited`).

### HTTP Status Codes

| Status | Meaning |
|---|---|
| 200 | Success |
| 400 | Bad request — param missing / malformed / invalid body (see `error.code`) |
| 401 | Credentials missing, invalid, revoked, or expired |
| 402 | Insufficient credits |
| 403 | API key doesn't have permission for this endpoint |
| 429 | Rate / concurrency limit hit |
| 500 | Internal error — please report |
| 502 / 504 | Upstream data source unavailable / timed out |

### Error Codes

**Stable codes you can branch on:**

| Code | HTTP | When |
|---|---|---|
| `invalid_request` | 400 | Missing / malformed param, invalid body, SQL rejected, row-limit exceeded, etc. (the `message` carries specifics) |
| `unauthenticated` | 401 | No `Authorization` header or invalid JWT |
| `key_invalid` | 401 | API key prefix matched but hash didn't |
| `key_revoked` | 401 | API key was revoked |
| `key_expired` | 401 | API key past `expires_at` |
| `forbidden` | 403 | API key scope doesn't allow this endpoint |
| `insufficient_credits` | 402 | Credit balance < required for this call |
| `rate_limited` | 429 | Per-key rate limit exceeded |
| `concurrency_limit_exceeded` | 429 | Too many in-flight requests for this key |
| `timeout` / `gateway_timeout` / `agent_unavailable` | 502 / 504 | Upstream issues |
| `internal_error` | 500 | Unexpected gateway-side error |

> **New codes may be added; treat unknown codes as opaque.** Always branch on `error.code` exact match for known cases and fall back gracefully for unknown ones. The `error.message` text is human-readable and may change without notice — log but do not parse.

---

## Endpoints

### `POST /api/v1/search`

**Tool**: `search` (REST-only; the v2.0.2 `/mcp/search` MCP server was retired on 2026-05-12) · **Cost**: LLM-cost based — `max(2, ceil(api_cost_usd / 0.034))` cr / call

Multi-step research with auto-citation. Drillr orchestrates the agent loop internally and returns a thesis-quality answer with sources cited.

**Request body**:

```json
{
  "question": "Pull NVDA latest 10-Q gross margin and compare to AMD same quarter.",
  "session_id": "ses_abc123",
  "context": "Building Q4 2025 semiconductor comp sheet.",
  "stream": false
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `question` | string | Yes | Full natural-language research question |
| `session_id` | string | No | Pass from prior call to continue thread (~3× faster on follow-ups) |
| `context` | string | No | Background prose for ground-truth context |
| `stream` | boolean | No | `true` (default) for SSE streaming response; `false` for one-shot JSON |

**Non-stream response** (`stream=false`):

```json
{
  "data": {
    "text": "## NVDA vs AMD Gross Margin (Q4 FY2025)\n\n| Company | GM | YoY | ...",
    "session_id": "ses_abc123",
    "sources": [
      {"tool": "sec_report_search", "ticker": "NVDA"},
      {"tool": "run_sql", "table": "financial_statements"}
    ],
    "duration_ms": 9420
  },
  "_credits": { "charged": 12, "method": "usage_based", "balance_after": 488 }
}
```

**Stream response** (default, `Accept: text/event-stream`):

```
event: status
data: {"session_id": "ses_abc123"}

event: tool_call
data: {"label": "Looking up NVDA filings"}

event: text_delta
data: {"text": "## NVDA vs AMD Gross Margin..."}

event: done
data: {"session_id": "ses_abc123", "sources": [...], "duration_ms": 9420}
```

| Event | Payload |
|---|---|
| `status` | Connection established, `session_id` |
| `tool_call` | Drillr's internal agent is calling a sub-tool (friendly label) |
| `text_delta` | Incremental text |
| `done` | Final state: `session_id`, `sources`, `duration_ms` |
| `error` | If something fails mid-stream |

**Curl example** (stream):

```bash
curl -X POST https://gateway.drillr.ai/api/v1/search \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -d '{"question": "NVDA latest gross margin?"}'
```

---

### `POST /api/v1/data/run_sql`

**Tool**: `run_sql` · **Server**: drillr-data · **Cost**: 1 cr / call

Read-only PostgreSQL SELECT against 90+ structured tables.

**Request body**:

```json
{
  "sql": "SELECT ticker, period_end, close FROM price_volume_history WHERE ticker='AAPL' AND time_frame='daily' ORDER BY period_end DESC LIMIT 5"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `sql` | string | Yes | PostgreSQL SELECT statement |

**SQL constraints** (enforced server-side, violations return `sql_invalid`):

- SELECT only — no INSERT / UPDATE / DELETE / DDL
- No CTE (`WITH ... AS`) — use subqueries
- No `pg_*` / `information_schema` access
- No `pg_sleep`, `COPY`, `SELECT INTO`
- Date columns are TEXT — use string comparison (`period_end >= '2024-01'`), not `::date` cast or `INTERVAL` arithmetic
- Structured tables require ticker filter; alt-data tables don't

**Response**:

```json
{
  "data": {
    "columns": [
      "ticker",
      "period_end",
      "close"
    ],
    "rows": [
      [
        "AAPL",
        "2026-05-14",
        298.21
      ],
      [
        "AAPL",
        "2026-05-13",
        298.87
      ]
    ],
    "rowCount": 2
  },
  "_credits": {
    "charged": 1,
    "method": "per_call",
    "balance_after": 505
  }
}
```

**Curl example**:

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/run_sql \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sql":"SELECT ticker, period_end, close FROM price_volume_history WHERE ticker='\''AAPL'\'' AND time_frame='\''daily'\'' ORDER BY period_end DESC LIMIT 5"}'
```

---

### `GET /api/v1/data/list_tables`

**Tool**: `list_tables` · **Server**: drillr-data · **Cost**: Free

List alternative-data tables under given categories. Returns each table's name, one-line purpose, column names.

**Query params**:

| Name | Type | Required | Description |
|---|---|---|---|
| `categories` | CSV string | Yes | 1-5 alt-data category names (URL-encode commas as `,`) |

**Available categories**: see [`tools.md`](./tools.md#list_tables) for the full 24-category list.

**Response**:

```json
{
  "data": [
    {
      "category": "LLM Token Pricing",
      "tables": [
        {
          "name": "litellm_price_history",
          "summary": "Historical LLM API pricing (from LiteLLM).",
          "columns": [
            "date",
            "model_name",
            "event",
            "provider",
            "mode"
          ]
        }
      ]
    }
  ],
  "_credits": {
    "charged": 0,
    "method": "free",
    "balance_after": 509
  }
}
```

**Curl example**:

```bash
curl -G https://gateway.drillr.ai/api/v1/data/list_tables \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "categories=LLM Token Pricing,Compute Pricing"
```

---

### `GET /api/v1/data/get_table_schema?table_name=:table_name`

**Tool**: `get_table_schema` · **Server**: drillr-data · **Cost**: Free

Look up column definitions for a specific table.

**Path params**:

| Name | Type | Description |
|---|---|---|
| `:table_name` | string | One of the 90+ enumerated table names — see [`tools.md`](./tools.md#list_tables) |

**Response**:

```json
{
  "data": {
    "table": "financial_statements",
    "columns": [
      {
        "column_name": "id",
        "data_type": "text",
        "comment": null
      },
      {
        "column_name": "ticker",
        "data_type": "text",
        "comment": null
      },
      {
        "column_name": "period_end",
        "data_type": "text",
        "comment": "End date of the fiscal period (TEXT, format: yyyy-mm). Primary time filter — use string comparison: period_end >= ''2024-01''."
      }
    ]
  },
  "_credits": {
    "charged": 0,
    "method": "free",
    "balance_after": 507
  }
}
```

**Curl example**:

```bash
curl https://gateway.drillr.ai/api/v1/data/get_table_schema?table_name=financial_statements \
  -H "Authorization: Bearer $DRILLR_API_KEY"
```

---

### `GET /api/v1/data/sec_report_list`

**Tool**: `sec_report_list` · **Server**: drillr-data · **Cost**: 1 cr / call

List indexed SEC filings for a ticker, with a summary header.

**Query params**:

| Name | Type | Required | Description |
|---|---|---|---|
| `ticker` | string | Yes | Stock ticker |
| `filing_types` | CSV string | No | Omit for default (periodic reports + IPO/shelf + proxy). Pass empty for all. Pass `10-K,10-Q` to override |

**Response**:

```json
{
  "data": {
    "ticker": "AAPL",
    "filings": [
      {
        "fiscal_year": 2026,
        "fiscal_quarter": "Q2",
        "filing_type": "10-Q",
        "filing_date": "2026-05-01",
        "period_start": "2025-12-29",
        "period_end": "2026-03-28"
      },
      {
        "fiscal_year": 2026,
        "fiscal_quarter": "Q2",
        "filing_type": "DEF 14A",
        "filing_date": "2026-01-08",
        "period_start": null,
        "period_end": "2026-01-08"
      }
    ],
    "total_before_filter": 28,
    "filter": {
      "mode": "default",
      "types": [
        "10-K",
        "10-K/A",
        "10-Q",
        "10-Q/A",
        "..."
      ]
    }
  },
  "_credits": {
    "charged": 1,
    "method": "per_call",
    "balance_after": 507
  }
}
```

**Curl example**:

```bash
curl -G https://gateway.drillr.ai/api/v1/data/sec_report_list \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "ticker=AAPL"
```

---

### `POST /api/v1/data/sec_report_search`

**Tool**: `sec_report_search` · **Server**: drillr-data · **Cost**: 1 cr / call

Paragraph-level semantic search across SEC filings.

**Request body**:

```json
{
  "ticker": "NVDA",
  "query": "data center revenue concentration",
  "top_k": 5,
  "period_start": "2024-01",
  "period_end": "2026-12",
  "filing_types": ["10-K", "10-Q"]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `ticker` | string | Yes | Stock ticker (foreign companies via ADR, e.g. `TSM`) |
| `query` | string | Yes | Search phrase |
| `top_k` | integer | No | Default 10, max 30 |
| `period_start` | string | No | yyyy-mm |
| `period_end` | string | No | yyyy-mm |
| `filing_types` | string[] | No | e.g. `["10-K", "10-Q"]`; default covers main reports |

**Response**:

```json
{
  "data": {
    "ticker": "NVDA",
    "total": 1,
    "results": [
      {
        "score": 0.0745,
        "node_id": "100242_2026-05-12_DEF 14A_0001045810-26-000036_node_0036",
        "title": "BUSINESS OVERVIEW - Table",
        "content": "BUSINESS OVERVIEW - Table[TABLE: Data Center Gaming Professional Visualization Automotive $193.7 billion revenue $16.0 b...",
        "metadata": {
          "ticker": "NVDA",
          "market": "us",
          "filing_type": "DEF 14A",
          "filing_date": "2026-05-12",
          "fiscal_year": 2027,
          "fiscal_quarter": 2,
          "period_start": null,
          "period_end": "2026-05-12"
        },
        "sources": [
          "bm25"
        ]
      }
    ]
  },
  "_credits": {
    "charged": 1,
    "method": "per_call",
    "balance_after": 503
  }
}
```

---

### `POST /api/v1/data/company_search`

**Tool**: `company_search` · **Server**: drillr-data · **Cost**: LLM-cost based — `max(2, ceil(api_cost_usd / 0.034))` cr / call

Qualitative company discovery. **Natural-language input only** — the v1.x SQL mode is retired in v2.0; for column-level filtering use [`POST /api/v1/data/run_sql`](#post-apiv1datarun_sql) on `company_snapshot`.

**Request body**:

```json
{ "query": "AI inference chip startups" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `query` | string | Yes | Natural-language description (e.g. "lithium-ion battery cell makers supplying European OEMs"). Do not pass SQL — use `run_sql` for that |

**Response**:

```json
{
  "data": {
    "query": "AI inference chip startups",
    "results": [
      {
        "ticker": "GROQ",
        "company_name": "Groq Inc",
        "match_reason": "Develops LPU inference chips purpose-built for low-latency LLM serving; pure-play inference accelerator vendor."
      }
    ]
  },
  "_credits": { "charged": 5, "method": "usage_based", "balance_after": 498 }
}
```

---

### `GET /api/v1/data/signal_list`

**Tool**: `signal_list` · **Server**: drillr-data · **Cost**: 2 cr / call

Recent news + market events feed.

**Query params**:

| Name | Type | Required | Description |
|---|---|---|---|
| `tickers` | CSV string | No | Filter to signals whose `suggested_tickers` overlaps any of these |
| `sector` | CSV string | No | Filter by sector overlap |
| `since` | ISO 8601 string | No | Signals created at or after this timestamp |
| `limit` | integer | No | Default 20, max 100 |
| `offset` | integer | No | Pagination, default 0 |

**Response**:

```json
{
  "data": {
    "items": [
      {
        "id": 35847,
        "headline": "$NVDA  - TRUMP: CHINA HASN’T BOUGHT NVIDIA H200 CHIPS  President Donald Trump said China has not pur...",
        "summary": "$NVDA  - TRUMP: CHINA HASN’T BOUGHT NVIDIA H200 CHIPS\n\nPresident Donald Trump said China has not purchased Nvidia’s H200...",
        "suggested_tickers": [
          "NVDA",
          "AAOI",
          "ACLS",
          "ACMR",
          "ADI",
          "..."
        ],
        "sector": [
          "Information Technology"
        ],
        "score": 3.1,
        "trigger_sources": [
          {
            "source_url": "https://twitter.com/DeItaone/status/2055255449507598362",
            "source_name": "Twitter @DeItaone"
          }
        ],
        "earliest_trigger_event_time": "2026-05-15T11:54:18.000Z",
        "created_at": "2026-05-15T12:31:25.407Z",
        "tags": {
          "themes": [
            "semiconductors",
            "ai"
          ],
          "regions": [
            "us",
            "china"
          ],
          "version": 2,
          "event_types": [
            "general_news"
          ],
          "sectors_gics": [
            "information_technology"
          ],
          "primary_class": "sector_event",
          "affected_industries": [
            "Semiconductors"
          ]
        }
      }
    ]
  },
  "_credits": {
    "charged": 2,
    "method": "per_call",
    "balance_after": 498
  }
}
```

**Curl example**:

```bash
curl -G https://gateway.drillr.ai/api/v1/data/signal_list \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "tickers=NVDA,AMD" \
  --data-urlencode "limit=5"
```

---

### `GET /api/v1/data/fiscal_utility`

**Tool**: `fiscal_utility` · **Server**: drillr-data · **Cost**: Free

Bidirectional fiscal year ↔ calendar month conversion.

**Query params**:

| Name | Type | Required (when) | Description |
|---|---|---|---|
| `ticker` | string | Always | Stock ticker |
| `fiscal_year` | integer | Forward only | Fiscal year |
| `fiscal_quarter` | integer 0-4 | Forward only | Quarter 1-4, or 0 for full fiscal year |
| `yyyy_mm` | string | Reverse only | Calendar month yyyy-mm |

**Forward example** (NVDA FY2026 Q3 → calendar):

```bash
curl -G https://gateway.drillr.ai/api/v1/data/fiscal_utility \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "ticker=NVDA" \
  --data-urlencode "fiscal_year=2026" \
  --data-urlencode "fiscal_quarter=3"
```

```json
{
  "data": {
    "ticker": "NVDA",
    "fiscal_year": 2026,
    "fiscal_quarter": 3,
    "fiscal_period": "Q3",
    "period_start": "2025-08",
    "period_end": "2025-10",
    "fiscal_year_end_month": 1
  },
  "_credits": { "charged": 0, "method": "free", "balance_after": 507 }
}
```

**Reverse example** (calendar 2025-08 → NVDA fiscal):

```bash
curl -G https://gateway.drillr.ai/api/v1/data/fiscal_utility \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  --data-urlencode "ticker=NVDA" \
  --data-urlencode "yyyy_mm=2025-08"
```

---

## OpenAPI Spec

A machine-readable OpenAPI 3.1 spec is available at:

```
https://gateway.drillr.ai/openapi.json
```

Use it to generate clients in your language (`openapi-generator` / `openapi-typescript` / etc.).

---

## See Also

- 🛠 [Tool Reference](./tools.md) — tool semantics, when-to-use, parameter details, sibling-tool routing
- 💰 [Pricing](./pricing.md) — Tier 总览 / 差异 / 单价矩阵 / Referral
- 🌐 [Developer Portal](https://drillr.ai/developer/docs) — Web docs with interactive try-it-out
- 💬 [Issues & feedback](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues)
