<div align="center">

🌐 [English](./README.md) · **中文**

# Drillr · 给 Agent 的金融研究数据底座

给 AI agent 的金融数据与研究 MCP：披露文件、三大报表、业绩、股东结构、公司事件、高管、分析师、公司发现与研究信号，覆盖美股、A 股和日股，每个数字都能追溯到出处。

[![License](https://img.shields.io/badge/License-MIT-0969DA?style=flat)](./LICENSE)
[![MCP](https://img.shields.io/badge/MCP-Streamable_HTTP-F97316?style=flat)](https://modelcontextprotocol.io)
[![工具](https://img.shields.io/badge/🛠_10_个工具-2EA44F?style=flat)](https://drillr.ai/docs/mcp)
[![REST API](https://img.shields.io/badge/🔧_REST_API_v2-0EA5E9?style=flat)](https://drillr.ai/docs/api)
[![开发者文档](https://img.shields.io/badge/🌐_开发者文档-8B5CF6?style=flat)](https://drillr.ai/docs)
[![反馈](https://img.shields.io/badge/💬_反馈-EC4899?style=flat)](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues)

</div>

浏览器登录，不用复制 API key。一个托管的 Streamable HTTP 端点、10 个工具：把公司名或一段描述解析成 ticker；列出并检索一家公司的披露文件（结构化的 as-reported 数据点 + 原文段落）；对财务表跑只读 SQL；从电话会和新闻里取研究信号。覆盖美股、A 股和日股，每个数字都能追溯到出处文件。

> ⭐ **如果 Drillr 帮到了你的 agent，给我们点个 Star——这是我们持续 in the open 迭代的信号。**

## 安装 drillr MCP

安装或连接 drillr MCP，请阅读[官方安装指南](https://drillr.ai/developer/mcp-install.md)。
各客户端的安装、浏览器 OAuth 登录和连接验证步骤统一在这份指南里维护。

把下面这段提示粘贴到你要使用 drillr 的应用里：

```text
Please connect drillr MCP in this app using this installation guide: https://drillr.ai/developer/mcp-install.md. Walk me through sign-in and verify the connection.
```

公开服务器名称为 `drillr-data`，地址为 `https://gateway.drillr.ai/mcp/data`，使用 Streamable HTTP。
默认通过浏览器 OAuth 登录。客户端的特殊要求及不支持 OAuth 时的 API key 备选方案，请查阅安装指南。

手动设置和工具参考见 [drillr.ai/docs/mcp](https://drillr.ai/docs/mcp)。

### Agent Skill

[drillr skill](https://github.com/Little-Grebe-Inc/drillr-skill) 提供 MCP 和 REST 的研究工作流。连接 MCP 不要求先安装 Skill：

```bash
npx skills add Little-Grebe-Inc/drillr-skill
```

### 分发入口

- [Smithery 页面](https://smithery.ai/servers/drillr/drillr)
- 本仓库也提供 Claude Code 插件市场：

```text
/plugin marketplace add Little-Grebe-Inc/drillr-mcp-server
/plugin install drillr
```

安装插件后，按官方指南完成当前客户端的登录和连接验证。

## Hello World

> _"NVDA 最新 10-K 里关于供应承诺（supply commitments）说了什么？过去四个季度数据中心收入怎么变？"_

1. 只有公司名时先 `ticker_lookup`，再 `filing_list` 看已收录哪些文件，然后 `filing_search`——一次调用同时返回 **as-reported 数据点**（数值、期间、XBRL 概念、accession number）和它们所在的**原文段落**。
2. 季度序列用 `run_sql` 查 `financial_statements`。
3. 回答里每个数字都带出处文件。通常 8–15 秒。

## 十个工具

每个工具在 `https://drillr.ai/docs/mcp/<tool>` 有单独页面（参数、限制、错误形态）；服务器的 `tools/list` 带同样的描述。工具集刻意保持小，由 agent 自行组合。

| 分组 | 工具 | 作用 | 计费 |
|---|---|---|---|
| **Company** | [`ticker_lookup`](https://drillr.ai/docs/mcp/ticker_lookup) | 公司名、品牌或 ticker 片段 → 规范代码，含历史名称 | 免费 |
| | [`company_search`](https://drillr.ai/docs/mcp/company_search) | 自然语言描述 → 公司列表及匹配理由；US / CN / JP / HK / KR | 3–5 cr |
| **Filings** | [`filing_list`](https://drillr.ai/docs/mcp/filing_list) | 某 ticker 已收录的披露文件：财期、类型、申报日 | 0.1 cr |
| | [`filing_search`](https://drillr.ai/docs/mcp/filing_search) | 检索一家公司的披露文件，同时返回结构化 as-reported 数据点和原文段落 | 0.1 cr |
| **Datasets** | [`list_tables`](https://drillr.ai/docs/mcp/list_tables) | 另类数据类目索引，或最多五个类目下的表 | 免费 |
| | [`get_table_schema`](https://drillr.ai/docs/mcp/get_table_schema) | 一张表的列、类型和使用说明（必填过滤、覆盖范围、坑） | 免费 |
| | [`run_sql`](https://drillr.ai/docs/mcp/run_sql) | 对财务、行情和另类数据表跑一条只读 PostgreSQL SELECT | 0.1 cr |
| **Signal** | [`industry_inflections`](https://drillr.ai/docs/mcp/industry_inflections) | 从大量美股电话会中综合出的行业拐点：机制、范围、程度、逐公司影响 | 1 cr |
| | [`ai_adoption`](https://drillr.ai/docs/mcp/ai_adoption) | 美股公司在电话会上披露的具体企业 AI 应用：流程、阶段、价值、证据 | 1 cr |
| | [`news_search`](https://drillr.ai/docs/mcp/news_search) | 对公司与市场新闻做语义检索：故事线、事件、带归属的观点 | 0.2 cr |

## 数据是什么

由 drillr 从一手来源构建：各市场官方披露渠道（SEC EDGAR、巨潮 cninfo、EDINET）的文件自行解析，公司在网上发布的内容（IR 页面、业绩电话会、新闻），以及 drillr 自己的预估值。中间没有数据供应商。每个已报告的数字都能追溯到它被申报时所在的段落。各数据集来源见 <https://drillr.ai/docs/provenance>。

| 数据集 | 内容 | 入口 | 覆盖 |
|---|---|---|---|
| **公司** | 用任意名称、代码、ISIN、CIK、CUSIP 解析 ticker；按描述发现公司；公司档案（交易所、行业、上市日、官网、员工数） | `ticker_lookup`、`company_search`；SQL 查 `company_snapshot` | US CN JP（发现另含 HK KR） |
| **披露文件** | 文件索引（类型、日期、官方链接）；中英日全文检索；as-reported 数据点带数值、单位、期间、文件和位置，用知识图谱关联，保证取到正确版本 | `filing_list`、`filing_search` | US CN JP |
| **财务** | 按各市场准则原样呈报的利润表、资产负债表、现金流量表，年度与季度；预计算的估值、利润率、回报、增长、杠杆、流动性指标 | `run_sql` 查 `financial_statements`、`company_snapshot` | US JP HK CN KR |
| **行情** | 日/周/月 OHLCV、指数、盘前盘后报价 | `run_sql` 查 `price_volume_history`、`index_price`、`equity_extended_rt` | 股票 US JP HK CN KR；指数、外汇、加密、商品 |
| **业绩** | 业绩日历（预期与实际）；结构化电话会摘要（要点、指引、风险、分部、问答） | `run_sql` 查 `earning_call_calendar`、`earning_call_summary` | US JP |
| **股东结构** | 内部人持股与交易（Form 3/4/5）；机构季度持仓（13F-HR） | `run_sql` 查 `insider_and_institution_activities` | US |
| **公司事件** | 8-K 事件：高管变动、交易、发债、证券发行；13D/G 持股变动 | `run_sql` 查 `executive_change`、`company_deal_events`、`debt_issuance`、`securities_offering` | US |
| **高管** | 高管名单与在任状态；DEF 14A 年度薪酬 | `run_sql` 查 `executive_profile`、`executive_compensation` | US |
| **分析师** | 逐条评级与目标价变动；一致预期分布与目标价 | `run_sql` 查 `analyst_ratings`、`analyst_ratings_consensus` | US |
| **信号** | drillr 得出的结论并附证据：行业拐点、企业 AI 应用、跨来源新闻故事线与带归属观点 | `industry_inflections`、`ai_adoption`、`news_search` | US（新闻：US CN JP + 宏观） |
| 另类数据（次要） | 9 个类目 65 张表，围绕 AI 供应链与宏观：能源电力、数据中心、半导体、算力价格、模型发展、推理经济、宏观贸易、预测市场、关键矿产 | `list_tables` → `get_table_schema` → `run_sql` | 全球 |

Ticker 形式：美股裸代码（`AAPL`）、A 股 `.SH` / `.SZ`（`600519.SH`）、日股 `.T`（`6758.T`）、港股 `.HK`（`00700.HK`）、韩股 `.KS` / `.KQ`（`005930.KS`）；指数 `^GSPC`；SQL 里带 `.` 或 `^` 的代码要加引号。更新频率：披露、股东、事件在申报后数分钟内；电话会与新闻数分钟到数小时；报表与分析师数据每日。

基准：drillr MCP 驱动的模型在 Vals Finance Agent 上得分 94.06%，并发布了自己的、关注重述的基准——<https://drillr.ai/drillr-benchmark>。

## REST API

同一份数据也以 29 个类型化 REST 端点提供，给脚本和流水线：一个 `X-API-KEY` 头，JSON 输出，公开的 OpenAPI 3.1 契约：

```bash
curl -H "X-API-KEY: $DRILLR_API_KEY" \
  "https://gateway.drillr.ai/api/v2/income-statements?ticker=AAPL&period=FY&limit=4"
```

参考：<https://drillr.ai/docs/api> · 契约：<https://gateway.drillr.ai/api/v2/openapi.json>。旧的 `/api/v1/data/*` 工具镜像见 [`docs/rest-api.md`](./docs/rest-api.md)。

## 计费与限制

MCP 与 REST 共用一个 credit 钱包。注册送 80 credits。Plus 每月 $29 / 300 cr，Ultra $99 / 1,500 cr，Enterprise 提供再分发授权与 SLA。`run_sql` 返回 100 行（Ultra 500），单条 10 秒，并发 5；`company_search` 每日有上限（20 / 100 / 500）。失败调用不计费。<https://drillr.ai/pricing>

## 不在覆盖范围内

未上市公司 · 链上加密指标（只有 CEX 价格） · 期权链、盘口、tick 数据 · 下单交易 · drillr 不做自己的价格预测。

## 社群

用 drillr 搭东西、遇到坑、或者想第一时间拿到新功能?扫码进群,或点标题链接。

<table>
  <tr>
    <td align="center" width="50%"><b>微信中文社群</b></td>
    <td align="center" width="50%"><a href="https://discord.gg/YAh96nw5Vh"><b>Discord</b></a></td>
  </tr>
  <tr>
    <td align="center"><img src="https://gateway.drillr.ai/qr/wechat.svg" width="160" alt="Drillr 微信群二维码" /></td>
    <td align="center"><img src="https://gateway.drillr.ai/qr/discord.svg" width="160" alt="Drillr Discord QR" /></td>
  </tr>
  <tr>
    <td align="center">中文开发者社群,产品反馈最快响应。</td>
    <td align="center">和一线 agent 开发者交流,office hours、调试支持、抢先体验。</td>
  </tr>
</table>

## License

MIT —— 详见 [`LICENSE`](./LICENSE)。
