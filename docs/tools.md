# Tool index

The authoritative reference for each MCP tool lives on the docs site and is what the
server itself describes in `tools/list`. This file is an index so it cannot go stale.

Endpoint `https://gateway.drillr.ai/mcp/data` · Streamable HTTP · browser OAuth (API key for
non-OAuth hosts) · overview and error shapes: <https://drillr.ai/docs/mcp>

| Group | Tool | Page |
|---|---|---|
| Company | `ticker_lookup` | https://drillr.ai/docs/mcp/ticker_lookup |
| Company | `company_search` | https://drillr.ai/docs/mcp/company_search |
| Filings | `filing_list` | https://drillr.ai/docs/mcp/filing_list |
| Filings | `filing_search` | https://drillr.ai/docs/mcp/filing_search |
| Datasets | `list_tables` | https://drillr.ai/docs/mcp/list_tables |
| Datasets | `get_table_schema` | https://drillr.ai/docs/mcp/get_table_schema |
| Datasets | `run_sql` | https://drillr.ai/docs/mcp/run_sql |
| Signal | `industry_inflections` | https://drillr.ai/docs/mcp/industry_inflections |
| Signal | `ai_adoption` | https://drillr.ai/docs/mcp/ai_adoption |
| Signal | `news_search` | https://drillr.ai/docs/mcp/news_search |

Renamed or removed since the 2026-05 reference: `sec_report_search` → `filing_search`,
`sec_report_list` → `filing_list`; `fiscal_utility` removed (fiscal calendars are in
`financial_statements` and `fiscal_year_config`); `industry_inflections` and `ai_adoption` added.

`run_sql` core tables, SQL rules and the alt-data catalog are summarised in the companion skill:
<https://github.com/Little-Grebe-Inc/drillr-skill/blob/main/reference/mcp-supplement.md>.
