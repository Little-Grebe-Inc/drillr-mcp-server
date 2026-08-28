<div align="center">

🌐 [English](./README.md) · **中文**

# Drillr · 给 Agent 的金融研究数据底座

给 AI agent 的金融 MCP。扫市场。建判断。追每一个信号。引每一条出处。

[![License](https://img.shields.io/badge/License-MIT-0969DA?style=flat)](./LICENSE)
[![MCP](https://img.shields.io/badge/MCP-Streamable_HTTP-F97316?style=flat)](https://modelcontextprotocol.io)
[![工具参考](https://img.shields.io/badge/🛠_工具参考-2EA44F?style=flat)](./docs/tools.md)
[![REST API](https://img.shields.io/badge/🔧_REST_API-0EA5E9?style=flat)](./docs/rest-api.md)
[![开发者文档](https://img.shields.io/badge/🌐_开发者文档-8B5CF6?style=flat)](https://drillr.ai/developer/docs)
[![反馈](https://img.shields.io/badge/💬_反馈-EC4899?style=flat)](https://github.com/Little-Grebe-Inc/drillr-mcp-server/issues)

</div>

浏览器登录，不用复制 API key。9 个工具覆盖 Agent 研究：标准化财务数据、公司发现、新闻与事件语义检索、带段落引用的公司披露检索，以及另类数据。

> ⭐ **如果 Drillr 帮到了你的 agent，给我们点个 Star——这是我们持续 in the open 迭代的信号。**

## 快速接入

1. 在 [drillr.ai](https://drillr.ai) 注册
2. 把 `https://gateway.drillr.ai/mcp/data` 加到支持 OAuth 的 MCP 客户端
3. 在浏览器登录并允许对应客户端。全程不会显示或要求复制密钥。

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

Codex 会在添加时自动打开浏览器登录；如果已经配置过，运行 `codex mcp login drillr`。

### Claude Desktop / 支持 OAuth 的 Host

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

保存后重启 Host，按提示在浏览器允许 Drillr。

### Cursor / VS Code

点一下即可写入配置，编辑器随后会引导你在浏览器登录。

[![Install in Cursor](https://img.shields.io/badge/Install_in-Cursor-171717?style=for-the-badge&logo=cursor&logoColor=white)](https://cursor.com/en/install-mcp?name=drillr&config=eyJ1cmwiOiJodHRwczovL2dhdGV3YXkuZHJpbGxyLmFpL21jcC9kYXRhIn0=) [![Install in VS Code](https://img.shields.io/badge/Install_in-VS_Code-0078D4?style=for-the-badge&logo=visualstudiocode&logoColor=white)](https://insiders.vscode.dev/redirect/mcp/install?name=drillr&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Fgateway.drillr.ai%2Fmcp%2Fdata%22%7D)

### API Key 兜底

仅 REST 或客户端确实不支持浏览器 OAuth 时使用。在 [drillr.ai/developer/keys](https://drillr.ai/developer/keys) 创建 `external` key，放进密钥管理器，再在 server 配置中加入：

```jsonc
"headers": { "Authorization": "Bearer <YOUR_DRILLR_API_KEY>" }
```

#### 扣子（字节跳动） / 千帆（百度智能云） / 火山方舟（字节跳动）兜底

按各平台 MCP server 添加规范填入:transport `http`、URL `https://gateway.drillr.ai/mcp/data`、Authorization header 写 `Bearer <YOUR_DRILLR_API_KEY>`(把尖括号占位换成你真实的 `drl_*` key)。

#### Hermes Agent 兜底

```yaml
mcp_servers:
  drillr:
    url: 'https://gateway.drillr.ai/mcp/data'
    headers: { Authorization: 'Bearer <YOUR_DRILLR_API_KEY>' }
```

#### 其他 host

任何支持 MCP 的 Host 都使用同一个 Streamable HTTP Endpoint。支持 OAuth 时删掉 `headers`，走浏览器登录；不支持时再用上面的 API Key 兜底。同一个 server entry 不要同时配置 OAuth 和静态 Bearer header。

### Smithery 兜底

```bash
npx -y @smithery/cli install drillr/drillr --client claude
```

Smithery 目前仍会在首次安装时提示输入 `drl_*` API key，并写入客户端配置。它是兼容兜底，不是支持 OAuth 客户端的默认方案。

Listing：https://smithery.ai/servers/drillr/drillr

### Claude Code 插件兜底

本仓库自带 Claude Code single-plugin marketplace。在 Claude Code 里跑：

```
/plugin marketplace add Little-Grebe-Inc/drillr-mcp-server
/plugin install drillr
```

插件安装时不需要 key。重启 Claude Code 后运行 `/mcp`，选中 `drillr`，再选 `Authenticate` 完成浏览器授权。

## Hello World

配好后，对你的 agent 这样问：

> _"拉一下 NVDA 最近一季的毛利率，对比 AMD 同期的——找出业务结构差异在哪。"_

幕后发生什么：
1. Host 把问题路由给 `drillr` MCP server
2. Agent 自己挑工具——通常是 `sec_report_search`（拿 10-Q 内容）+ `run_sql`（查 financial_statements 拿毛利率）+ `company_search`（拿业务分部定义）
3. 你拿回带引用的 markdown 回答，一般 8-15 秒
4. 在 [drillr.ai/developer/keys](https://drillr.ai/developer/keys) 查看 credit 余额;REST 调用额外在每次 2xx 响应里内联 `{ "data": ..., "_credits": ... }` envelope(详见 [REST API › Response Envelope](./docs/rest-api.md#response-envelope)) —— MCP 响应走标准 JSON-RPC,不在响应里携带 per-call 计费字段

## 一个 Toolkit，9 个工具

Drillr 用一个 MCP endpoint 暴露 9 个工具——你的 agent 按需组合：

| 工具 | 用途 |
|---|---|
| `run_sql` | 90+ 张表的标准化财务数据——三大表、比率、业绩、内部交易、股东结构、行情、另类数据 |
| `sec_report_search` | 检索美股、日股、港股和 A 股公司披露文件中的段落 |
| `sec_report_list` | 按 ticker / 文件类型列出已收录的公司披露 |
| `company_search` | 在四个市场中按业务模式、供应链、可比公司或主题找公司 |
| `news_search` | 语义检索新闻、市场事件和带归属的观点，并按 storyline 聚合 |
| `ticker_lookup` | 把公司名、品牌或 ticker 片段解析成 ticker 历史 |
| `list_tables` | 按类目列出可用的另类数据 SQL 表 |
| `get_table_schema` | 看任意 SQL 表的列和类型 |
| `fiscal_utility` | 财年 / 财季工具（处理非自然年公司的 FY/FQ 解析） |

完整 tool 参考：[`docs/tools.md`](./docs/tools.md)。

## 数据覆盖

- **核心股票覆盖**：美股、日股、港股和 A 股；ticker 格式分别为 `AAPL`、`6758.T`、`00700.HK`、`600519.SH` / `300750.SZ`
- **基本面**：`financial_statements`、`company_snapshot`、`price_volume_history` 覆盖四个核心市场，财报历史回溯到 1980 年代
- **公司披露**：覆盖 SEC EDGAR、日本 EDINET、港交所和 A 股报告，支持段落级语义检索
- **业绩**：电话会 transcript、AI 结构化摘要和 estimate vs actuals；这类专业数据目前覆盖美股 + 日股
- **行情**：股票、ETF、指数（含 Nikkei 225 / TOPIX）、外汇、加密货币、大宗商品
- **美股专业数据**：分析师评级、持仓、管理层、8-K 事件和盘前盘后行情
- **新闻 + 事件**：覆盖四个股票市场和跨资产内容，支持 storyline 聚合与观点归属
- **AI 价值链另类数据**：24 个类目，跨能源 / 芯片 / 算力定价 / LLM token 经济 / 模型 benchmark / AI 公司财务 / app 使用 / 网站流量 / 专利 / 学术论文 / 政府合同 / 贸易流 / 金融 KOL（Twitter / Reddit / Substack / YouTube）

完整数据字典：[`docs/tools.md`](./docs/tools.md)。

## REST API

每个 MCP tool 都有 1:1 对应的 REST endpoint。同一把 `drl_*` key、同样的数据。详见 [`docs/rest-api.md`](./docs/rest-api.md)。

```bash
curl -X POST https://gateway.drillr.ai/api/v1/data/run_sql \
  -H "Authorization: Bearer $DRILLR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sql":"SELECT ticker, close FROM price_volume_history WHERE ticker='\''AAPL'\'' AND time_frame='\''daily'\'' ORDER BY period_end DESC LIMIT 5"}'
```

## 不在覆盖范围内

提前讲清楚边界，避免 agent 浪费研究循环：

- 非上市 / 私募公司（只覆盖公开上市公司）
- Crypto 链上指标——我们有 CEX 价格（BTCUSD / ETHUSD / SOLUSD 等），但没有 TVL / 持币地址 / 钱包数据
- 期权链、实时盘口、逐笔
- 零售经纪动作（下单 / 管仓）
- Drillr 不出自家价格预测——只 surface 分析师 consensus

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
