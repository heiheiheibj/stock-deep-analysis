# stock-deep-analysis

> 股票深度财务分析报告生成工具 — Hermes Agent SKILL

A股 + 美股的深度分析，支持七大章节的结构化 Markdown 报告。

## 数据源

| 市场 | 主源 | 备用 |
|------|------|------|
| **A股** | akshare（同花顺/新浪） | Tushare → 腾讯/新浪网页API |
| **美股** | akshare `stock_us_daily()` | 新浪财经网页补充PE/市值/EPS/新闻 |

> ❌ Yahoo Finance 已弃用（国内 IP 永久 429 限流）

## 快速使用

### A股
```bash
python scripts/generate_report.py --ticker 600426.SH --market CN --cost 23.89 --currency CNY
```

### 美股 (akshare 自动获取行情)
```bash
python scripts/generate_report.py --ticker PDD --market US --cost 100 --currency USD
```

### Python 调用
```python
from generate_report import analyze_us_stocks

data = {
    'info': {
        'regularMarketPrice': 87.24,
        'fiftyTwoWeekHigh': 139.41,
        'fiftyTwoWeekLow': 81.56,
        'marketCap': 124177000000,
        'trailingPE': 8.97,
        'priceToBook': 1.92,
        'shortName': '拼多多',
        'industry': '互联网零售',
        'sector': '电子商务',
        'targetMeanPrice': 118.04,
        'numberOfAnalystOpinions': 12,
        'recommendationKey': 'buy',
    },
}
report = analyze_us_stocks(data, cost=100.0, currency='USD')
print(report)
```

## 报告章节

1. **公司基本情况** — 名称、行业、板块、业务简介
2. **估值分析** — PE/PB/PS、52周高低、分析师目标价、Beta、市值
3. **盈利能力** — 营收/毛利/净利、EPS历史走势、ROE/ROA、利润率
4. **资产负债质量** — 总资产/负债/权益、债务结构、流动比率
5. **自由现金流** — 经营现金流、FCF、资本支出
6. **持仓状态** — 成本、盈亏、距52周高低距离
7. **综合结论** — 利好因素、风险因素、操作建议

## 依赖

```bash
pip install akshare>=1.18.64 tushare>=1.4.0
```

> 需要 Tushare Token 作为 A 股备用数据源，注册: https://tushare.pro

## 数据获取说明

- **美股价格数据**: `akshare.stock_us_daily(symbol)` 从东方财富获取日线OHLCV
- **美股财务数据**: `akshare.stock_financial_us_analysis_indicator_em(symbol)` 获取ROE/EPS等
- **美股补充字段**: 新浪财经页面 (PE、市值、盘前数据、新闻)
- **A股数据**: akshare 同花顺/新浪接口 → Tushare 备用

## License

MIT
