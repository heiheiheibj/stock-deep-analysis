# akshare 端点实测记录

测试日期：2026-06-01 | Python: 3.x | akshare 1.18.64

## A 股可用端点

| 端点函数 | 数据 | 状态 | 备注 |
|---------|------|------|------|
| `stock_zh_a_spot_em()` | 全市场实时行情 | ✅ 可用 | 返回 DataFrame |
| `stock_zh_a_daily(symbol='sh600426', adjust='qfq')` | 日K历史行情 | ✅ 可用 | 需要 pandas |
| `stock_zh_a_hist(symbol='600426', period='daily', ...)` | 日K历史行情 | ✅ 可用 | 东方财富源 |
| `stock_financial_analysis_indicator(symbol='600426')` | 财务指标 | ✅ 可用 | 同花顺源 |
| `stock_profit_sheet_by_report_em(symbol='600426')` | 利润表 | ✅ 可用 | 东方财富源 |
| `stock_balance_sheet_by_report_em(symbol='600426')` | 资产负债表 | ✅ 可用 | 东方财富源 |
| `stock_cash_flow_sheet_by_report_em(symbol='600426')` | 现金流量表 | ✅ 可用 | 东方财富源 |

## A 股不可用/有限端点

| 端点函数 | 状态 | 备注 |
|---------|------|------|
| `stock_zh_a_spot_em()` 中部分字段 | ⚠️ | 不同版本返回字段可能不同 |

## 美股可用端点

| 端点函数 | 数据 | 状态 | 备注 |
|---------|------|------|------|
| `stock_zh_market_spot_em()` | 全球行情 | ✅ 可用 | 含美股 |
| `stock_zh_us_hist(symbol='AAPL', period='daily', ...)` | 美股历史行情 | ✅ 可用 | 东方财富美股源 |

## 港股可用端点

| 端点函数 | 数据 | 状态 | 备注 |
|---------|------|------|------|
| `stock_zh_hk_daily(symbol='00700', period='daily', ...)` | 港股历史行情 | ✅ 可用 | 东方财富港股源 |
| `stock_zh_hk_spot_em()` | 港股实时行情 | ✅ 可用 | 全市场 |

## 注意事项

1. **akshare 依赖网络** — 国内网络直接可用，境外可能需要代理
2. **东方财富接口有频率限制** — 高频请求可能被封 IP，建议间隔 > 1 秒
3. **不同 akshare 版本的函数名可能变化** — 注意版本兼容
4. **`stock_zh_a_daily` 的 `adjust` 参数** — `'qfq'` 前复权, `'hfq'` 后复权, `''` 不复权
5. **symbol 格式** — 历史行情用纯数字（如 `'600426'`），实时行情用交易所后缀（如 `'sh600426'`）
