---
name: stock-deep-analysis
description: >-
  美股数据获取：akshare的stock_us_daily(symbol)获取OHLCV日线数据（国内可达），含当前价、52周区间、成交量。
  财务数据用 stock_financial_us_analysis_indicator_em(symbol)。
  次选：新浪财经网页browser抓取新闻+补充字段。
  A股数据源：akshare → Tushare → 腾讯/新浪API回退。
version: 3.0.0
tags:
  - finance
  - stock-analysis
  - akshare
  - investment
  - financial-report
  - security-analysis
  - data-fallback
dependency:
  python:
    - akshare>=1.18.64
    - tushare>=1.4.0
environment:
  TUSHARE_TOKEN: Tushare token（可选的备用数据源）
  HTTPS_PROXY: HTTP 代理地址（可选）
  HTTP_PROXY: HTTP 代理地址（可选）
triggers:
  - 分析股票
  - 深度报告
  - 股票分析
  - 持仓分析
  - 美股分析
  - 港股分析
  - A股分析
  - 财务报告
---

# Stock Deep Analysis（股票深度分析）

> ⭐ **给其它 AI 模型的说明**：本 skill 包含完整的数据获取 + 报告生成逻辑。如果你可以运行 Python，请直接用下方的命令行调用脚本。否则请完全按照第五节的「报告框架」格式输出——那是本 skill 的标准输出格式。

---

## 一、功能概述

输入股票代码，从多数据源自动拉取数据（主源限流则自动切到备用源），生成以下七大章节的结构化 Markdown 报告：

1. **公司基本情况** — 名称、行业、板块、业务简介
2. **估值分析** — PE/PB/PS、52周高低、分析师目标价、Beta、市值
3. **盈利能力** — 营收/毛利/净利、EPS历史走势、ROE/ROA、利润率
4. **资产负债质量** — 总资产/负债/权益、债务结构、流动比率
5. **自由现金流** — 经营现金流、FCF、资本支出、FCF/OCF比率
6. **持仓状态**（如传入成本）— 买入成本、盈亏、距52周高低距离
7. **综合结论** — 利好因素、风险因素、操作建议

---

## 二、前置准备

### 2.1 依赖安装

```bash
# 使用 hermes-agent venv
/c/Users/Administrator/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe -m pip install 'akshare>=1.18.64' 'tushare>=1.4.0'

# 验证
/c/Users/Administrator/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe -c "import akshare; print('akshare OK:', akshare.__version__); import tushare; print('tushare OK')"
```

> **🐞 Windows 用户注意：** 以下所有命令中的 `~/.hermes/hermes-agent/venv/bin/python3` 在 Windows 上应为 `~/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe`。本 skill 所有命令均以 Linux 路径编写，Windows 使用时请自行替换 Python 路径。推荐在 Windows 上将 Python 路径写入变量：
> ```bash
> PY=~/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe
> ```

### 2.2 Tushare Token 配置

Tushare 是 A 股的备用数据源（当 akshare 网络异常时回退）。美/港股不使用 Tushare。

```bash
# 方式一：临时传
TUSHARE_TOKEN=your_token python generate_report.py --ticker PDD --market US

# 方式二：写入环境变量
export TUSHARE_TOKEN="your_token"
```

> 免费 Tushare 账号主要接口限 1次/小时，日常使用注意间隔。

### 2.3 代理配置（无代理时不需设置）

```bash
export HTTPS_PROXY=http://your_proxy_host:port
export HTTP_PROXY=http://your_proxy_host:port
```

> ⚠️ Python HTTP 库只认 `http://` 前缀，不要用 `socks5://`。

### 2.4 货币符号映射

| 市场 | currency 参数 | 符号 |
|------|-------------|------|
| 美股 USD | `--currency USD` | `$` |
| 港股 HKD | `--currency HKD` | `HK$` |
| A股 CNY | `--currency CNY` | `¥` |

---

## 三、命令行用法

### 3.1 A 股

```bash
PYTHON=/c/Users/Administrator/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe

$PYTHON ~/AppData/Local/hermes/skills/stock-deep-analysis/scripts/generate_report.py \
  --ticker 600426.SH --market CN --cost 23.89 --currency CNY
```

**数据源：** akshare（同花顺/新浪） → Tushare → 腾讯/新浪网页API回退

### 3.2 美股

**akshare 自动获取（推荐）：** `stock_us_daily(symbol)` 获取 OHLCV 日线数据——含当前价、52周高低、成交量。脚本已将此集成到 `fetch_stock_data()`。

```bash
PYTHON=/c/Users/Administrator/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe
$PYTHON scripts/generate_report.py --ticker PDD --market US --cost 100 --currency USD
```

> ⚠️ `stock_us_daily` 只能获取价格和成交量，**无法获取 PE、市值、EPS、盘前数据、分析师评级**。需要这些补充字段时，配合新浪财经页面（浏览器）获取。

**新浪财经页面（补充字段）：**
```
https://stock.finance.sina.com.cn/usstock/quotes/{ticker}.html
```
可获取：PE、EPS、市值、盘前数据、公司新闻等。详见 §3.5。

### 3.3 港股

> 🔴 港股可通过新浪财经港股页面获取，akshare 暂无可用的港股日线接口。

**⚠️ A股报告的常见问题（三阶回退触发点）：** Tushare 免费账号 `fina_indicator`/`stock_basic` 限 1次/小时。限流时脚本输出可能非常稀疏——仅含 ROE、毛利率、EPS、资产负债率等基础字段，**缺少 PE、PB、52周范围、市值、分析师目标价**等关键估值数据。这种情况下，不要依赖脚本输出。应执行三阶回退中的 Tier 3（见 §四），使用 `references/a-share-fallback-apis.md` 中记录的腾讯/新浪/东方财富公开 API 获取完整数据后，按第五节报告格式手写。

### 3.4 Python 调用

```python
import sys
sys.path.insert(0, '~/.hermes/skills/stock-deep-analysis/scripts')
from generate_report import generate_stock_report

report = generate_stock_report(
    ticker='PDD',
    market='US',       # 'US' | 'HK' | 'CN'
    cost=100.0,        # 持仓成本，不传则不计算盈亏
    currency='USD'     # 'USD' | 'HKD' | 'CNY'
)
print(report)
```

### 3.5 美股数据获取：akshare + 新浪财经混合方案

**步骤1：akshare 自动获取基础行情**
```python
import akshare as ak
df = ak.stock_us_daily(symbol='PDD')        # 日线OHLCV
latest = df.iloc[-1]                         # 当前价
recent = df.tail(365)
high52 = recent['high'].max()                # 52周最高
low52 = recent['low'].min()                  # 52周最低
```

**步骤2：新浪财经页面补充 PE/市值/EPS/盘前/新闻**
```
https://stock.finance.sina.com.cn/usstock/quotes/{ticker}.html
```
从页面提取（browser 抓取）：
- PE：`市盈率：8.97`
- EPS：`每股收益：9.73`
- 市值：`市值：1241.77亿`
- 盘前：`盘前 : 89.13 1.89(2.17%)`
- 公司新闻：点击"公司新闻"tab

**步骤3：构造 data 字典并生成报告**
```python
data = {
    'info': {
        'regularMarketPrice': 87.24,           # ← 从 akshare
        'fiftyTwoWeekHigh': 139.41,            # ← 从 akshare
        'fiftyTwoWeekLow': 81.56,              # ← 从 akshare
        'marketCap': 124177000000,             # ← 从新浪
        'trailingPE': 8.97,                    # ← 从新浪
        'priceToBook': 1.92,
        'shortName': '拼多多',
        'industry': '互联网零售',
        'sector': '电子商务',
        'targetMeanPrice': 118.04,
        'numberOfAnalystOpinions': 12,
        'recommendationKey': 'buy',
        # 更多字段见第六节字段映射表
    },
    'hist': None,                              # 不需要
    'income_stmt': None,
    'balance_sheet': None,
    'cashflow': None,
}
from generate_report import analyze_us_stocks
report = analyze_us_stocks(data, cost=100.0, currency='USD')
print(report)
```

---

## 四、支持的市场及数据源策略

**数据获取优先级（已嵌入为永久规则）：**
- **A股：** akshare(同花顺/新浪) → Tushare → 腾讯/新浪网页API回退
- **美股：** akshare stock_us_daily(symbol) → 新浪财经网页补充PE/市值/EPS → analyze_us_stocks()
- **港股：** 新浪财经港股页面(browser)

> ⚠️ **Yahoo Finance 已完全弃用** — 国内 IP 永久 429 限流，不再尝试。

### 数据获取表

| market | 主源 | 备用 | 说明 |
|--------|------|------|------|
| `CN` | **akshare**（同花顺/新浪） | Tushare → 腾讯/新浪API | akshare网络问题 → Tushare限流 → 网页API |
| `US` | **akshare**(stock_us_daily) | 新浪财经网页补充PE/市值/EPS |
| `HK` | **新浪财经港股**(browser) | — | 同新浪方法 |

### 美股数据获取标准流程

```python
# Step 1: akshare 自动获取行情
df = ak.stock_us_daily(symbol='PDD')
close = df.iloc[-1]['close']        # 当前价
high52 = df.tail(365)['high'].max()  # 52周高
low52 = df.tail(365)['low'].min()    # 52周低

# Step 2 (可选): 新浪页面补充PE/市值/盘前/新闻
browser_navigate("https://stock.finance.sina.com.cn/usstock/quotes/PDD.html")

# Step 3: 构造data → analyze_us_stocks(data, cost, currency)
```

> 无需安装 yfinance。无需 Tushare Token（仅A股回退需要）。

---

## 五、报告输出标准格式

**无论是否运行脚本，生成的报告必须严格遵循此格式。**

```markdown
# {公司名}（{ticker}）深度分析报告

**报告日期：** {YYYY-MM-DD} | **成本价：** {sym}{cost} → **当前价：** {sym}{price} | **盈亏：** {sym}{pnl} ({pnl_pct}%)

---

## 一、公司基本情况

| 项目 | 内容 |
|------|------|
| 名称 | {name} |
| 行业 | {industry} |
| 板块 | {sector} |
| 业务简介 | {summary 前300字} |

## 二、估值分析

| 指标 | 数值 | 评级 | 说明 |
|------|------|------|------|
| 当前价 | {sym}{price} | | 距52周高点 {-((high52-price)/high52*100):.0f}% |
| 52周最高 | {sym}{high52} | | |
| 52周最低 | {sym}{low52} | | 距当前 {+((price-low52)/low52*100):.1f}% |
| 52周位置 | {range_pct:.0f}% 分位 | {低位/中位/高位} | {描述} |
| Trailing PE | {pe} | {评级} {⭐} | {说明或N/A} |
| Forward PE | {fwd_pe} | {评级} | 预期2026 EPS {sym}{eps_fwd} |
| Price/Book | {pb} | {评级} | |
| Beta | {beta} | {描述} | 与大盘相关性{极低/低/高} |
| 市值 | {cap} | | |
| 分析师目标价 | {sym}{target} | ✅ 高于现价 {upside:.0f}% | {N} 位分析师覆盖 |
| 分析师评级 | {rec_key} | | {推荐分布} |

## 三、盈利能力

**年度稀释EPS（{货币}）：**
  {年份1}: {sym}{eps1} ({+eps变化})
  {年份2}: {sym}{eps2} ({+eps变化})
  ...

| 指标 | 数值 | 评级 |
|------|------|------|
| 总营收 | {营收格式化} | |
| 毛利润 | {毛利润格式化} | |
| 净利润 | {净利润格式化} | |
| 营收增速（YoY） | {rev_growth*100:.1f}% | {评级} |
| 季度盈利增速 | {earn_qg*100:.1f}% | {描述} |
| 营业利润率 | {op_margin*100:.1f}% | {描述} |
| 净利润率 | {profit_margin*100:.1f}% | {描述} |
| ROE | {roe*100:.1f}% | {描述} |
| ROA | {roa*100:.1f}% | {描述} |

## 四、资产负债质量

| 指标 | 数值 | 评级 | 说明 |
|------|------|------|------|
| 总资产 | {格式化} | | |
| 股东权益 | {格式化} | | |
| 总债务 | {格式化} | | |
| 债务/权益 | {debt_eq:.1f}% | {评级} | {描述} |
| 流动比率 | {current_ratio:.2f} | {评级} | {描述} |
| 现金及等价物 | {现金格式化} | ✅ | |

## 五、自由现金流

| 指标 | 数值 |
|------|------|
| 经营现金流 | {格式化} |
| 自由现金流 | {格式化} |
| 资本支出 | {格式化} |
| FCF/经营现金流 | {ratio:.1f}% |

## 六、持仓状态

| 项目 | 数值 |
|------|------|
| 买入成本 | {sym}{cost} |
| 当前价格 | {sym}{price} |
| 浮动盈亏 | {sym}{pnl}（{pnl_pct:.1f}%）|
| 距52周低点 | {sym}{low52}（{+'%.1f'%}%）|
| 距52周高点 | {sym}{high52}（{hi_ratio:.1f}%）|
| 距分析师目标 | {sym}{target}（+{upside:.0f}%）|

## 七、综合结论

### ✅ 利好因素
- {利好1}
- {利好2}
- ...

### 🔴 风险因素
- {风险1}
- {风险2}
- ...

### 📊 操作建议
{根据盈亏给出具体建议}

---

> ⚠️ _本报告仅供参考，不构成投资建议。数据来源：{数据源}，报告生成时间：{YYYY-MM-DD}_
```

---

## 六、关键数据字段名映射（data.info 字典 → Python 变量）

| info 字典字段（驼峰） | Python 变量名 | 说明 |
|----------------------|--------------|------|
| `profitMargins` | `profit_margin` | 净利润率（不是 `net_margin`）|
| `debtToEquity` | `debt_eq` | 债务/权益比（不是 `debt_to_equity`）|
| `returnOnAssets` | `return_on_assets` | ROA |
| `returnOnEquity` | `return_on_equity` | ROE |
| `currentRatio` | `current_ratio` | 流动比率 |
| `freeCashflow` | `fcf` | 自由现金流 |
| `operatingCashflow` | `op_cashflow` | 经营现金流 |
| `totalRevenue` | `total_revenue` | 总营收 |
| `grossProfit` | `gross_profit` | 毛利润 |
| `netIncome` | `net_income` | 净利润 |
| `totalAssets` | `total_assets` | 总资产 |
| `totalDebt` | `total_debt` | 总债务 |
| `regularMarketPrice` | `price` | 当前价 |
| `fiftyTwoWeekHigh` | `high52` | 52周最高 |
| `fiftyTwoWeekLow` | `low52` | 52周最低 |
| `marketCap` | `mktcap` | 市值 |
| `trailingPE` | `trailing_pe` | TTM 市盈率 |
| `forwardPE` | `forward_pe` | 前向市盈率 |
| `priceToBook` | `pb` | 市净率 |
| `beta` | `beta` | Beta 系数 |
| `targetMeanPrice` | `target` | 分析师目标价均值 |
| `recommendationKey` | `rec_key` | 综合评级 |
| `numberOfAnalystOpinions` | `num_analysts` | 覆盖分析师数量 |
| `revenueGrowth` | `rev_growth` | 营收增速（YoY，小数）|
| `earningsQuarterlyGrowth` | `earnings_qg` | 季度盈利增速（YoY，小数）|
| `epsCurrentYear` | `eps_ttm` | TTM EPS |
| `epsForward` | `eps_fwd` | 前向 EPS |
| `shortName` | `name` | 短名称 |
| `longName` | `name` | 全称（备用）|
| `longBusinessSummary` | `summary` | 业务简介 |

Tushare 返回的中文字段名直接在 `generate_report.py` 的 `analyze_cn_stock()` 中使用。详见 `references/indicators-guide.md`。

---

## 七、评级逻辑规则

### 7.1 估值指标（越低越好 → `reverse=True`）

```python
_rating(pe, (15, 25), reverse=True)      # PE < 15 极低，15-25 偏低，> 25 偏高
_rating(pb, (3, 5), reverse=True)        # PB < 3 低估，3-5 合理，> 5 偏高
_rating(debt_eq, (50, 100), reverse=True) # 债务/权益，< 50 低杠杆，> 100 高杠杆
```

**评级文字映射：**
- 极低/低估 → `✅ 极低/低估 ⭐`
- 偏低/合理 → `✅ 低估` 或 `⚠️ 合理`
- 偏高 → `❌ 偏高`

### 7.2 盈利/质量指标（越高越好 → `reverse=False`）

```python
_rating(roe, (0.15, 0.20))       # ROE > 20% 优秀，15-20% 良好，< 8% 差
_rating(current_ratio, (1.5, 2.0)) # > 2 优秀，1.5-2 良好，< 1 较差
```

### 7.3 营收增速

```python
_rating(rev_growth * 100, (10, 30))  # > 30% 优秀，10-30% 良好，< 10% 一般
```

---

## 八、已知陷阱（Pitfalls）

### 🔴 数据格式陷阱（from_dict index 格式）

generate_report.py 中的 `_get_latest()` 函数处理的是 `{date: value}` 字典格式。当通过新浪财经页面手动构造 data 字典时，`hist`/`income_stmt`/`balance_sheet`/`cashflow` 均可传 `None`（不需要）。

### 🔴 fcf/op_cf/capex 未定义（现金流数据缺失）

**问题：** `cf` 为空时 `fcf`/`op_cf`/`capex` 从未赋值，综合结论中 `if fcf and fcf < 0` 抛 `UnboundLocalError`。

**修复（generate_report.py）：** 在 `if cf:` 块前显式初始化三个变量为 `None`。

### 🔴 目标价除零

**问题：** `target` 不为空但 `price=0` 时，`(target-price)/price` 除零崩溃。

**修复：** 除零前加 `and price` 检查。

### 🔴 操作建议文案不准确

**问题：** `+6.5%` 微盈被显示为"当前亏损"。

**修复：** 拆分为四档——盈利>10%（分批止盈）、盈利0-10%（微盈持有观察）、亏损0-10%（小幅亏损持有观察）、亏损>10%（结合基本面判断）。

### 🔴 硬编码代理双重同步

`generate_report.py` 和 `fetch_stock_data.py` 都有代理设置区域。修改时必须双文件同步清理。当前 `fetch_stock_data.py` 已改为纯环境变量读取；`generate_report.py` 第 21 行仍有 `os.environ.get('HTTP_PROXY', 'http://172.30.192.1:7890')` 硬编码回退，应一起修复。

### 🔴 akshare 美股财务指标为人民币计价

`stock_financial_us_analysis_indicator_em(symbol='PDD')` 返回的营收/净利润/EPS等字段均以**人民币（元）**计价，非美元。`CURRENCY` 列显示"人民币"。不要将这些数据与美股美元价格直接混算。仅供趋势参考（同比增速等），绝对值不适用。

### 🔴 手工构造 data 时的字段名

当从新浪财经页面提取数据后构造 `data['info']` 字典，必须使用第六节映射表中**右侧**的驼峰字段名（如 `regularMarketPrice`、`fiftyTwoWeekHigh`），不用中文名。错误的关键字段名会导致 `analyze_us_stocks()` 读到 0 值，抛出除零错误。

### 🔴 A股脚本输出稀疏（Tushare 免费账号限流）

**问题：** Tushare 免费账号 `fina_indicator`/`stock_basic` 限 1次/小时。限流时脚本只输出 ROE、毛利率、EPS、资产负债率等少数字段，**缺少 PE/PB/52周范围/市值/分析师目标价**等关键估值数据。用户收到半份报告，无法做判断。

**解决方案（按优先级）：**

1. 使用腾讯 API（`qt.gtimg.cn`）直接获取 PE/PB/市值/振幅 → 详见 `references/a-share-fallback-apis.md`
2. 使用新浪 K 线 API 获取52周高/低 → 同上
3. 手动按第五节「报告输出标准格式」补全分析章节，不要依赖脚本的残缺输出

### 🟡 最新财报期提取

- 年报按年份排序后取最新，必须 `sorted(years, reverse=True)`（降序）
- 默认 `sorted()` 升序会取到最老的年份

### 🟡 结论数据残留（多股票分析）

- `_conclusion()` 中所有字段必须从传入的 `info`/`analysis` 参数读取
- 严禁硬编码上一只股票的数据（如分析师目标价 `$143`、成本 `-18%` 等）

---

## 九、财务指标评价参考

详见 `references/indicators-guide.md`。核心阈值：

| 指标 | 优秀 | 良好 | 一般 | 差 |
|------|------|------|------|-----|
| ROE | > 20% | 15-20% | 8-15% | < 8% |
| ROA | > 8% | 4-8% | 1-4% | < 1% |
| 净利润率 | > 20% | 10-20% | 5-10% | < 5% |
| 流动比率 | > 2 | 1.5-2 | 1-1.5 | < 1 |
| 债务/权益 | < 30% | 30-60% | 60-100% | > 100% |
| 营收增速 | > 50% | 20-50% | 10-20% | < 10% |
| PE（反向）| < 10 | 10-15 | 15-25 | > 25 |
| PB（反向）| < 1 | 1-3 | 3-5 | > 5 |

---

## 十、文件结构

```
stock-deep-analysis/
├── SKILL.md                              ← 本文件
├── scripts/
│   ├── generate_report.py                ← 主脚本（A股报告生成 + analyze_us_stocks() 函数）
│   └── fetch_stock_data.py               ← 数据拉取模块（仅 A 股: akshare→Tushare）
│       ├── _fetch_akshare_cn()           → A股：同花顺/新浪接口
│       ├── _fetch_akshare_us()           → 美股：akshare（备选）
│       ├── _fetch_tushare()              → Tushare 备用
│       └── fetch_stock_data()            → 统一入口
└── references/
    ├── indicators-guide.md               ← 财务指标评价参考
    ├── akshare-endpoints-tested.md        ← akshare 各端点实测记录
    ├── akshare-us-endpoints-confirmed.md   ← akshare 美股端点实测（国内可用/不可用清单）
    ├── a-share-fallback-apis.md           ← A股回退 API（腾讯/新浪/东方财富）
    └── sina-finance-us-stock-page-guide.md ← 新浪财经美股页字段提取指南
```
```

---

## 十一、持仓数据参考

| 股票 | ticker | 成本 | 市场 |
|------|--------|------|------|
| 拼多多 | PDD | $100 | US（NASDAQ）|
| 金山云 | KC | $13 | US（NASDAQ）|

---

## 十二、快速验证

```bash
PYTHON=/c/Users/Administrator/AppData/Local/hermes/hermes-agent/venv/Scripts/python.exe

# A股（akshare 主源）
$PYTHON ~/AppData/Local/hermes/skills/stock-deep-analysis/scripts/generate_report.py --ticker 600426.SH --market CN --currency CNY

# 美股（需手动从新浪财经抓取 data 后调用 analyze_us_stocks()）
# 详见 §3.5

# 快速验证
$PYTHON -c "
import akshare as ak
# 美股 - 验证 stock_us_daily 可用
df = ak.stock_us_daily(symbol='PDD')
latest = df.iloc[-1]
print(f'✅ akshare 美股 OK: PDD = {latest[\"close\"]} (vol={latest[\"volume\"]})')

# 验证财务指标
fin = ak.stock_financial_us_analysis_indicator_em(symbol='PDD')
print(f'✅ akshare 美股财务 OK: {len(fin)} periods')
"
```
