# akshare 美股数据端点实测（国内可达性）

> 验证时间：2026-06-02 | 环境：Windows 10, 国内网络无代理
> 结论：Yahoo Finance 永久弃用。akshare + 新浪财经网页为美股标准化获取方案。

## ✅ 可用端点

### stock_us_daily(symbol) — 核心接口

```python
import akshare as ak

df = ak.stock_us_daily(symbol='PDD')
# 返回：date, open, high, low, close, volume
# 最新一条即最新交易日数据
latest = df.iloc[-1]
price = latest['close']          # 当前价 87.24
volume = latest['volume']        # 成交量

# 52周区间
recent = df.tail(365)
high52 = recent['high'].max()    # 52周最高
low52 = recent['low'].min()      # 52周最低
```

**特点：** 从新浪源获取，国内稳定可达。返回完整日线历史（IPO至今）。
**限制：** 仅含 OHLCV，不含 PE/市值/EPS/盘前/分析师等基本面字段。

### stock_financial_us_analysis_indicator_em(symbol) — 财务指标

```python
fin = ak.stock_financial_us_analysis_indicator_em(symbol='PDD')
# 返回多个报告期的财务数据
# 重要！所有货币字段为**人民币**，非美元
latest_fin = fin.iloc[0]
roe = latest_fin.get('ROE_AVG')          # ROE(%)
eps = latest_fin.get('BASIC_EPS')         # 基本每股收益（人民币）
gross_margin = latest_fin.get('GROSS_PROFIT_RATIO')  # 毛利率(%)
operate_income = latest_fin.get('OPERATE_INCOME')    # 营收（人民币）
```

**特点：** 东方财富数据源，国内可达。
**限制：** 所有财务数据以人民币计价。适用于趋势分析（同比增速、利润率百分比）。

## ❌ 不可用端点

| 函数 | 失败原因 |
|------|---------|
| `stock_us_spot_em()` | 超时——一次性下载全部872只美股，国内网络不可行 |
| `stock_us_hist(symbol)` | 连接被拒绝——东方财富push API被墙 |
| `stock_us_valuation_baidu(symbol)` | JSON解析失败——百度接口格式变更 |
| `stock_individual_basic_info_us_xq(symbol)` | 雪球API需要登录cookies |

## 推荐数据获取流程

```
美股分析需求
    │
    ├─ 基础行情（价格/52周/量）
    │   └─ akshare.stock_us_daily(symbol)  ✅
    │
    ├─ 基本面（PE/市值/EPS/分红）
    │   └─ 新浪财经网页 browser 抓取  ✅
    │       https://stock.finance.sina.com.cn/usstock/quotes/{ticker}.html
    │
    ├─ 财务指标（ROE/毛利率/EPS趋势）
    │   └─ akshare.stock_financial_us_analysis_indicator_em(symbol)  ✅（人民币）
    │
    └─ 公司新闻/分析师评级
        └─ 新浪财经→公司新闻tab  ✅
```

## 补充说明

- KC（金山云）的52周区间在不同数据源有差异：
  - akshare stock_us_daily: $8.06 - $22.26
  - 新浪财经页面: $10.29 - $18.52
  差异原因是数据覆盖的日期范围不同（akshare 365天精确计算 vs 新浪可能使用不同起止日）。
  建议以新浪页面为准，页面51周区间是标准化显示。
