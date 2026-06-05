# 4 档 Tier 公开版定价

> 统一 tier 体系：一套 tier、一套 cr 配额,跨 MCP 和 REST API 通用(CLI 即将推出)。同一 tool 在任何 surface 调用收同样 cr、享同样配额。

## Tier 总览

| Tier | 月费 | 含 cr | 加购单价 | 受众 |
|---|---|---|---|---|
| Free | $0 | 80 cr（一次性） | $0.15 / cr | AI agent 社区 / 试用 |
| Plus | $99 / 月 | 500 cr / 月 | $0.12 / cr（8 折） | Indie Quant |
| Ultra | $499 / 月 | 3,000 cr / 月 | $0.10 / cr（67 折） | 研报分析师 / 买方机构 |
| Enterprise | $2K+ / 月（定制） | 不限 | 谈（~$0.07 / cr） | 应用开发者 / 嵌入 drillr 的 SaaS |

## Tier 间功能差异化

| 维度 | Free | Plus | Ultra | Enterprise |
|---|---|---|---|---|
| API key 数量 | 1 | 3 | 20 | 不限 |
| Redistribution（转售 / 嵌入 SaaS） | ❌ | ❌ | ❌（单席位） | ✅ |
| `run_sql` 单查询行限制 | 500 | 500 | 500 | 500 |
| `run_sql` 超时 | 30s | 30s | 30s | 30s |
| `sec_report_search` `top_k` | 30 | 30 | 30 | 30 |
| Support | 无 | community | email | 专属 + SLA |

> 一期所有 tier 的 limit 统一为 500（行数 / top_k），后续按真实滥用案例再分层。

## Tool 单价矩阵

| Tool | 计价方式 | 单价 | 资产类别 | 备注 |
|---|---|---|---|---|
| `run_sql` | call-base | 1 cr | SQL | 90+ 表统一单价（alt-data 与 public 暂不区分） |
| `signal_list` | call-base | 2 cr | Signal | 跨资产信号管线 |
| `sec_report_search` / `sec_report_list` | call-base | 1 cr | SEC | 公开数据 |
| `list_tables` / `get_table_schema` / `fiscal_utility` / `ticker_resolve` | free | 0 cr | Meta | 元数据探查 + ticker 解析,永久免费 |
| `search`（NL agent）/ `company_search` NL 模式 | LLM-cost | `max(2, ceil(api_cost_usd / 0.034))` | LLM | 按真实 LLM 成本动态计价 |

## Referral 计划（概要）

注册即得 **300 cr / 7 天有效**；好友订阅付费后双方均拿 cr 奖励（Plus 档：分享者 +200 cr / 好友 +400 cr；Ultra 档：分享者 +1,000 cr / 好友 +2,000 cr）。奖励 cr 永久有效，14 天延迟兑现。完整规则见 `/referral` 页面。
