# global-stock-data 调研报告：数据源清单与 Apify Actor 接入方案

> **调研对象：** [simonlin1212/global-stock-data](https://github.com/simonlin1212/global-stock-data)  
> **姊妹项目：** [simonlin1212/a-stock-data](https://github.com/simonlin1212/a-stock-data)（A 股）  
> **调研日期：** 2026-07-26  
> **仓库版本：** v2.0.3（2026-07-26）  
> **目的：** 完整评估产品与功能，沉淀可调用数据源清单，并给出可落地的 Actor 拆分与接入方案。

---

## 1. 执行摘要

`global-stock-data` 不是传统 Python 数据库，而是面向 AI 编程助手的**自包含 Skill**（`SKILL.md` + 内嵌 Python）。它把美股/港股分散在 11 个公开数据源上的接口，整理成 13 层、30+ 端点的可执行工具集。

| 维度 | 结论 |
|------|------|
| 产品形态 | Claude Code / Codex / OpenClaw Skill；仅依赖 `requests` |
| 核心差异化 | **官方源优先** + **合规分级（S/B/C）**；CBOE Greeks/0DTE、FINRA 空头量、EDGAR 事件流与 frames screener |
| Actor 化价值最高 | **Tier S：SEC EDGAR 全家桶 + Treasury + CFTC** |
| 不建议 Store 商用旗舰 | Tier C 行情/Yahoo/CBOE（需授权或 personal use only） |
| 产品边界 | **不要做成一个 mega Actor**；按合规层级 × 调用形态拆成 4–6 个 SKU |

**一句话：** 最值得 Actor 化的不是新浪/东财行情，而是 **SEC EDGAR + 宏观日历**；行情与 CBOE 留作内部/研究或完成授权后再上架。

---

## 2. 仓库与产品概况

### 2.1 元数据

| 项 | 值 |
|----|-----|
| 仓库 | https://github.com/simonlin1212/global-stock-data |
| License | Apache-2.0 |
| 创建 | 2026-05-20 |
| 最新版 | v2.0.3 — 腾讯行情 / EDGAR Frames / CBOE 三项数据正确性修复 |
| Stars / Forks | ~1,283 / ~195（调研时点） |
| 依赖 | `pip install requests` |
| 主体文件 | `SKILL.md`（~2221 行，约 54 个函数，35 个 Python 代码块 / ~1600 LOC） |

### 2.2 产品定位

- **目标用户：** AI Agent（自然语言触发取数）与个人研究员
- **覆盖市场：** 美股 + 港股（期权 / 做空 / EDGAR 仅美股）
- **交付方式：** 下载 `SKILL.md` 到 `~/.claude/skills/global-stock-data/`
- **明确声明：** 只分发**代码**，不分发、不转售市场数据；商用请只依赖 S 级源

### 2.3 与 a-stock-data 的关系

| | a-stock-data | global-stock-data |
|--|--------------|-------------------|
| 市场 | 中国 A 股 | 美股 + 港股 |
| 版本（调研时） | v3.5.1 | v2.0.3 |
| 层数 / 端点 / 源 | 10 / 44 / 15 | 13 / 30+ / 11 |
| 合规标注 | 较少体系化 | **S/B/C 分级 + 条款原文** |
| 关键依赖 | mootdx + requests + pandas | 仅 requests |
| 姊妹关系 | 可并存安装，不冲突 | 同左 |

---

## 3. 架构（13 层）

```
美股/港股全栈 · V2.0 · 官方源优先
│
├── Layer 1  行情层       新浪 + 腾讯 + 东财 push2
├── Layer 2  K线层        新浪(美股至1984) + Yahoo chart
├── Layer 3  技术指标     MA/EMA/MACD/RSI/KDJ/布林（本地计算）
├── Layer 4  基本面       东财三表+指标 + Yahoo 23模块 + SEC XBRL
├── Layer 5  资金面       东财 push2his 日级四档资金流
├── Layer 6  期权层       CBOE 官方(Greeks/IV/0DTE) + Yahoo 后备
├── Layer 7  SEC Filing   EDGAR submissions + companyfacts
├── Layer 8  工具层       搜索 / 全市场列表 / 新闻 / ticker↔CIK
├── Layer 9  做空层 ⭐     FINRA Reg SHO 全市场日空头量
├── Layer 10 申报事件流 ⭐ EDGAR 每日索引 + 全文检索(FTS)
├── Layer 11 横截面 ⭐     EDGAR frames 全市场 screener
└── Layer 12 宏观/日历 ⭐  Treasury + CFTC + Nasdaq 财报日历
```

⭐ = V2.0 新增能力，多数开源方案（含 yfinance）不具备或不全。

---

## 4. 合规分级（Actor 上架硬约束）

> 结论来自仓库作者于 **2026-07-24** 逐家实读条款；引号为原文摘录。  
> **「官方」≠「可自由商用」。**

| 级别 | 可商用 | 可再分发 | 数据源 | 依据摘要 |
|------|--------|----------|--------|----------|
| **S** | ✅ | ✅ | SEC EDGAR / US Treasury / CFTC | EDGAR：*"for free"*、*"allow scripted access"*；硬限 **10 req/s** + 声明 UA |
| **B** | ⚠️ 自行确认 | ❌ | FINRA | 文件主动发布；条款禁 *"data mining, scraping…robots"*，并写 *"non-commercial use"* |
| **C** | ❌ 需授权 | ❌ | CBOE / Nasdaq / Yahoo / 东财 / 新浪 / 腾讯 | Cboe 需事先批准+许可协议；Yahoo personal use only |
| **⛔ 排除** | — | — | HKEX CCASS | 明文禁止 robot/scraper，不论是否营利 → **禁止做成 Actor** |

### 给接入方的三条硬规则

1. **Store 商用旗舰只做 S 级。** B 级需法务确认；C 级须授权或标为研究/私有。
2. **Actor 分发的是访问工具，不是数据包。** 勿把 B/C 级结果二次转售分发。
3. **限速不可绕过。** SEC 10 req/s 为官方硬顶；Skill 内置约 8 req/s。

---

## 5. 数据源总表（可直接对接）

### 5.1 Tier S — 优先 Actor 化

| 源 | Base / Endpoint | Skill 函数 | 能力 | 市场 | 鉴权 / 限速 |
|----|-----------------|------------|------|------|-------------|
| SEC EDGAR | `data.sec.gov/submissions/CIK{cik}.json` | `sec_filings` | 10-K / 10-Q / 8-K 列表 | US | UA=`SEC_CONTACT`；≤8–10/s |
| SEC EDGAR | `data.sec.gov/api/xbrl/companyfacts/CIK{cik}.json` | `sec_xbrl_facts` | ~503 个 GAAP 指标 | US | 同上 |
| SEC EDGAR | `www.sec.gov/Archives/edgar/daily-index/` | `daily_filings` | 当日 Form 4 / 8-K / 13F / 144 | US | 同上 |
| SEC EDGAR | `efts.sec.gov/LATEST/search-index` | `fulltext_search` | 2001 起全文检索 | US | 同上 |
| SEC EDGAR | `data.sec.gov/api/xbrl/frames/us-gaap/{tag}/...` | `market_frame` / `frame_ranking` / `frame_screen` | 全市场横截面 screener（实测可覆盖数千家） | US | 同上 |
| SEC EDGAR | `www.sec.gov/files/company_tickers.json` | `ticker_to_cik` | ticker ↔ CIK | US | 同上 |
| US Treasury | `home.treasury.gov/resource-center/data-chart-center/interest-rates/` | `treasury_yield_curve` | 收益率曲线 1M–30Y | Macro | 无 Key |
| CFTC | `publicreporting.cftc.gov/resource/6dca-aqww.json` | `cftc_cot` | COT 持仓报告 | Macro | 无 Key |

### 5.2 Tier B — 谨慎接入

| 源 | Endpoint | Skill 函数 | 能力 | 风险与用法建议 |
|----|----------|------------|------|----------------|
| FINRA Reg SHO | `cdn.finra.org/equity/regsho/daily/` | `short_volume_all` / `short_volume_symbol` / `short_volume_ranking` | 全市场每日空头成交量（约 12,112 标的）+ 个股时序 + 排行 | 按「下载已发布日文件」实现，勿爬站点页面；商用前向 FINRA 确认；建议私有 Actor 或显式非商用 |

### 5.3 Tier C — 研究 / 私有 / 授权后再商用

| 源 | Endpoint | Skill 函数 | 能力 | 备注 |
|----|----------|------------|------|------|
| CBOE | `cdn.cboe.com/api/global/delayed_quotes/options/{ticker}.json` | `options_chain_cboe`, `filter_expiry`, `unusual_activity`, `chain_summary`, `cboe_quote` | 全链 + IV + delta/gamma/vega/theta/rho + 0DTE + 异动 flow | 商用需 Cboe 事先授权 |
| CBOE | `cdn.cboe.com/api/global/delayed_quotes/quotes/{ticker}.json` | `cboe_quote` | 延时报价 | 同上 |
| Nasdaq | `api.nasdaq.com/api/calendar/earnings` | `earnings_calendar` | 财报日历（盘前盘后 + EPS 预期） | 条款未核实 |
| Yahoo Finance | `query2.finance.yahoo.com/v8/finance/chart/{symbol}` | `stock_kline_yahoo` | 美/港 K 线（日→分钟） | cookie + crumb；personal use |
| Yahoo Finance | `query2.finance.yahoo.com/v10/finance/quoteSummary/{symbol}` | `yahoo_quote_summary` 及包装函数 | 23 模块：财务 / 分析师 / 机构持仓等 | 同上 |
| Yahoo Finance | `query2.finance.yahoo.com/v7/finance/options/{symbol}` | `options_chain` | 期权链（无 Greeks） | CBOE 后备 |
| Yahoo Finance | `query2.finance.yahoo.com/v1/finance/search` | （新闻/搜索） | 按代码新闻等 | 同上 |
| 东财 push2 | `push2.eastmoney.com/api/qt/stock/get` | `stock_quote_eastmoney` | 美/港实时行情（含中文名等） | secid 前缀见 §6 |
| 东财 push2 | `push2.eastmoney.com/api/qt/clist/get` | `market_stock_list` | 全市场列表（美股 5925+ / 港股 18000+） | 有风控封 IP 风险 |
| 东财 push2his | `push2his.eastmoney.com/api/qt/stock/fflow/daykline/get` | `fund_flow_daily` | 日级主力/大/中/小单资金流 | 同上 |
| 东财 datacenter | `datacenter-web.eastmoney.com/api/data/v1/get` | `financial_statements_eastmoney`, `key_indicators_eastmoney` | 三表 + GMAININDICATOR | 美 49 / 港 75 字段级指标 |
| 东财 search | `searchapi.eastmoney.com/api/suggest/get` | `stock_search` | 中英文搜索 + secid/`MktNum` 映射 | 工具层关键 |
| 新浪 | `hq.sinajs.cn/list=...` | `us_stock_quote_sina`, `hk_stock_quote_sina` | 美股 36 字段 / 港股 25 字段 | 零鉴权 |
| 新浪 | `stock.finance.sina.com.cn/.../US_MinKService.getDailyK` | `us_stock_kline_sina` | 美股日 K（回溯至 1984） | 仅美股日 K |
| 腾讯 | `qt.gtimg.cn/q=...` | `us_stock_quote_tencent`, `hk_stock_quote_tencent` | 美股 ~71 / 港股 ~78 字段 | 字段下标需实测校准 |

### 5.4 本地计算层（无外部源）

| 函数 | 说明 |
|------|------|
| `calc_ma` / `_ema` / `calc_macd` / `calc_rsi` / `calc_kdj` / `calc_boll` | 基于 OHLCV 的纯 Python 指标，可挂在任意 K 线 Actor 后处理 |

### 5.5 禁止接入

| 源 | 原因 |
|----|------|
| HKEX CCASS（港股席位持股） | ToS 禁止 robot/bot/spider/scraper，不论是否营利；原作者已删除实现 |

---

## 6. 调用约定（封装必读）

### 6.1 SEC

```text
User-Agent = "Your Name you@example.com"   # 对应 Skill 中 SEC_CONTACT，必改
速率      ≤ 8–10 req/s（官方硬顶 10）
未声明 UA → Access Denied / Undeclared Automated Tool
```

Skill 统一出口：`official_get()` —— 负责按源限流、SEC UA、友好错误（区分「无数据」与「配置/网络错误」）。

### 6.2 东财 secid 前缀

| 前缀 | 市场 | 示例 |
|------|------|------|
| 105 | NASDAQ | `105.AAPL` |
| 106 | NYSE | `106.BABA` |
| 107 | 美股 ETF/其他 | `107.CRSH` |
| 116 | 港股 | `116.00700` |

判断 105/106/107：调用 `stock_search()` 读 `MktNum`。

### 6.3 代码格式对照

| 体系 | 美股 | 港股 |
|------|------|------|
| Yahoo | `AAPL` | `0700.HK` |
| 东财 SECUCODE（datacenter） | `AAPL.O` / `BABA.N` | `00700.HK` |
| 东财 secid | `105.AAPL` | `116.00700` |
| 新浪美股 | `gb_aapl` 类 | `rt_hk00700` 类 |
| 腾讯 | `usAAPL` | `r_hk00700` |

### 6.4 Yahoo crumb

```text
Session GET https://fc.yahoo.com          → cookie
GET https://query2.finance.yahoo.com/v1/test/getcrumb → crumb
后续 v7/v10 请求携带 crumb
```

Skill 封装：`get_yahoo_session()` / `yahoo_quote_summary()`。

### 6.5 建议统一输出字段

```json
{
  "symbol": "AAPL",
  "market": "US",
  "source": "sec-edgar",
  "tier": "S",
  "operation": "daily_filings",
  "fetchedAt": "2026-07-26T08:00:00Z",
  "asOf": "2026-07-25",
  "...": "业务字段",
  "raw": {}
}
```

---

## 7. 函数 ↔ 层 ↔ 源 速查

| 层 | 主力函数 | 源 | Tier |
|----|----------|-----|------|
| L1 行情 | `us_stock_quote_sina/tencent`, `hk_stock_quote_*`, `stock_quote_eastmoney` | 新浪/腾讯/东财 | C |
| L2 K线 | `us_stock_kline_sina`, `stock_kline_yahoo` | 新浪/Yahoo | C |
| L3 指标 | `calc_ma/macd/rsi/kdj/boll` | 本地 | — |
| L4 基本面 | `financial_statements_*`, `key_indicators_*`, `key_statistics`, `analyst_estimates`, `institutional_holders`, `sec_xbrl_facts` | 东财/Yahoo/SEC | C/S |
| L5 资金 | `fund_flow_daily` | 东财 | C |
| L6 期权 | `options_chain_cboe`, `unusual_activity`, `options_chain` | CBOE/Yahoo | C |
| L7 Filing | `sec_filings`, `sec_xbrl_facts` | SEC | S |
| L8 工具 | `stock_search`, `stock_news`, `ticker_to_cik`, `market_stock_list` | 东财/Yahoo/SEC | C/S |
| L9 做空 | `short_volume_*` | FINRA | B |
| L10 事件流 | `daily_filings`, `fulltext_search` | SEC | S |
| L11 Screener | `market_frame`, `frame_ranking`, `frame_screen` | SEC | S |
| L12 宏观 | `treasury_yield_curve`, `cftc_cot`, `earnings_calendar` | Treasury/CFTC/Nasdaq | S/C |

---

## 8. Apify Actor 拆分方案

### 8.1 原则

- **禁止**一个「global-stock-data everything」Store Actor（输入爆炸、单源故障拖垮质量分、合规风险不一致）。
- **禁止**一 endpoint 一 Actor（Store 噪音、维护成本高）。
- **推荐：** 共享 adapter 私有库 + **4–6 个对外 SKU**，边界按 **合规 × 延迟 × 计费单位** 切。

### 8.2 建议 SKU

| ID | 建议 Actor 名 | 覆盖 | 运行模式 | PPE 计费单位建议 | Store 优先级 |
|----|---------------|------|----------|------------------|--------------|
| **A1** | `us-sec-edgar` | filings + XBRL + daily index + FTS + frames + CIK | Batch 为主；单票 lookup 可 Standby | `filing-item` / `frame-row` / `search-hit` | **P0 上架** |
| **A2** | `us-macro-calendar` | Treasury + CFTC（+ 可选 Nasdaq） | Batch / 轻量 Standby | `macro-snapshot` / `cot-row` | **P0** |
| **A3** | `us-finra-short-volume` | FINRA Reg SHO 日文件 | 按日 Batch | `short-volume-row` | **P1 私有或非商用** |
| **A4** | `us-hk-quotes-klines` | 新浪/腾讯/东财/Yahoo | 短 Batch；慎 Standby | `quote-result` / `kline-bar-batch` | **P2 研究/私有** |
| **A5** | `us-fundamentals` | 东财三表 + Yahoo modules（SEC XBRL 可并入 A1） | Batch | `fundamentals-result` | P2 |
| **A6** | `us-options-flow` | CBOE 主 + Yahoo 备 | Batch / 严限流 Standby | `options-chain` / `unusual-flow` | **P3 授权后再商用** |

### 8.3 输入 / 输出约定（Batch Actor）

**Input（建议字段）**

| 字段 | 类型 | 说明 |
|------|------|------|
| `symbols` | `string[]` | 必填，设上限 |
| `market` | `US` \| `HK` | 市场 |
| `operation` | enum | 如 `quote` / `kline` / `filings` / `frame_screen` …；**单次 run 一种主操作** |
| `interval` / `from` / `to` / `limit` | — | 时间与条数 |
| `maxItems` | int | 硬封顶 |
| `secContact` | secret | A1 必填（SEC UA） |
| `proxyConfiguration` | Apify proxy | Tier C 强烈建议 |

**Output**

- 一行 = 一个业务结果（symbol×operation，或一根 K，或一条 filing）
- 必含：`symbol`, `market`, `source`, `tier`, `fetchedAt`, `asOf`
- 原始响应可放可选字段 `raw`
- 提供 `dataset_schema.json` 视图（Quotes / Klines / Filings）与 `output_schema.json`

### 8.4 Standby / OpenAPI

| 适合 Standby | 不适合 Standby |
|--------------|----------------|
| A1 单票 CIK/filing 快查 | 全市场 frames screener |
| A2 收益率曲线当日快照 | 多年 K 线批量 |
| 已授权且有代理池的报价（谨慎） | 未授权 CBOE/Yahoo 高频报价 |

说明：Cursor/Apify 侧对 Standby 建议配 `usesStandbyMode` + OpenAPI `webServerSchema`；PPE-only Standby 仍需控制 idle 超时。

### 8.5 PPE / 稳定性

- 按**成功落盘的业务单元**计费，另保留 `apify-actor-start`。
- 单价需覆盖：重试、节流等待、代理流量（尤其 Tier C）。
- 实现：源级 RPS、退避、熔断；部分失败返回错误行，避免整 run 失败。
- 勿承诺「免费上游上的无限实时行情」。

### 8.6 Store 文案与合规

- 名称勿冒充 SEC / CBOE / Yahoo / Nasdaq 官方产品。
- README 显著位置：**非官方、非附属、非投资建议、AS IS**。
- 产品文案与 Skill 分级对齐：仅 S 级做商用旗舰；B/C 级限制声明或私有。
- SEC：强制用户提供联系用 UA；文档写明 10 req/s。
- 保持与原仓一致的纪律：**不做 HKEX CCASS Actor**。

---

## 9. 落地路线图

### Phase 1（Store 可推）

1. **A1 `us-sec-edgar`**
   - operations：`filings` / `xbrl_facts` / `daily_filings` / `fulltext_search` / `frame_screen` / `ticker_to_cik`
   - 必填 Secret：`secContact`
   - PPE：`filing-item`、`frame-row`、`search-hit`
2. **A2 `us-macro-calendar`**
   - operations：`treasury_yield_curve` / `cftc_cot`（Nasdaq 日历可选且单独 disclaimer）

### Phase 2

3. **A3 FINRA** — 日文件下载解析；默认非商用文案或私有
4. 共享 `adapters` 包抽取（`official_get`、secid、Yahoo session）

### Phase 3（视合规）

5. **A4/A5** 行情与基本面 — 代理 + 严限流 + 研究用途声明  
6. **A6 期权** — 取得 Cboe 授权前不上 Store 商用

---

## 10. 能力亮点与风险

### 亮点

- 合规分级写进产品，利于 Store 选型与法务沟通
- EDGAR frames = 免费全市场基本面 screener（差异化强）
- Form 4 / 8-K / 13F 同日事件流 + 全文检索（投研场景完整）
- CBOE 官方 Greeks + 0DTE 异动（能力强，但合规为 C）
- 极简依赖，适配器可直接移植为 Actor 代码

### 风险

- Tier C 源易变、易封 IP；共享出口 IP 上 Standby 风险高
- Yahoo crumb 机制可能随时变更
- FINRA / CBOE 商用边界需额外确认
- Skill 无 CI/包版本；移植时需自建契约测试与源级 smoke
- 大陆网络访问 Yahoo/SEC 可能不稳定，需代理策略

---

## 11. 附录

### 11.1 仓库内关键文件

| 文件 | 说明 |
|------|------|
| `SKILL.md` | 全部端点与可运行代码 |
| `README.md` / `README_zh.md` | 中英文产品说明 |
| `CHANGELOG.md` | 版本演进 |
| `LICENSE` | Apache-2.0 |

### 11.2 完整 HTTP 基址列表（调研提取）

```
https://api.nasdaq.com/api/calendar/earnings
https://cdn.cboe.com/api/global/delayed_quotes
https://cdn.finra.org/equity/regsho/daily/
https://data.sec.gov/api/xbrl/companyfacts/CIK
https://data.sec.gov/api/xbrl/frames/us-gaap/
https://data.sec.gov/submissions/CIK
https://datacenter-web.eastmoney.com/api/data/v1/get
https://efts.sec.gov/LATEST/search-index
https://fc.yahoo.com
https://home.treasury.gov/resource-center/data-chart-center/interest-rates/
https://hq.sinajs.cn/list
https://publicreporting.cftc.gov/resource/6dca-aqww.json
https://push2.eastmoney.com/api/qt/clist/get
https://push2.eastmoney.com/api/qt/stock/get
https://push2his.eastmoney.com/api/qt/stock/fflow/daykline/get
https://qt.gtimg.cn/q
https://query2.finance.yahoo.com/v1/finance/search
https://query2.finance.yahoo.com/v1/test/getcrumb
https://query2.finance.yahoo.com/v10/finance/quoteSummary/
https://query2.finance.yahoo.com/v7/finance/options/
https://query2.finance.yahoo.com/v8/finance/chart/
https://searchapi.eastmoney.com/api/suggest/get
https://stock.finance.sina.com.cn/usstock/api/jsonp.php/var/US_MinKService.getDailyK
https://www.sec.gov/Archives/edgar/daily-index/
https://www.sec.gov/files/company_tickers.json
```

### 11.3 免责声明（建议写入 Actor README）

本调研与后续 Actor 仅提供**数据访问工具与接口封装参考**，不构成投资建议。上游条款以各数据源最新 ToS 为准；商用前请自行完成合规确认。股市有风险，投资需谨慎。

---

## 12. 下一步建议

1. 确认先做 **A1** 还是 **A1+A2** 并行。  
2. 提供 Apify Actor 目标仓库 / `lentic_clockss` 命名规范后，可直接脚手架：`input_schema` + `dataset_schema` + `official_get` 移植。  
3. Store 上架文案与 PPE 事件名按本文 §8 固化，避免后期改计费口径。

---

*文档生成自对 upstream Skill / README / CHANGELOG 的静态分析与 Apify 产品化调研；未在本环境对全部端点做全量实测。SEC / 腾讯等部分接口受网络环境影响，上线前需在目标出口 IP 上做 smoke test。*
