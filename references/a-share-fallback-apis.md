# A 股数据回退 API 链（当 Tushare 限流 + Yahoo 429 时）

## 问题场景

- Tushare 免费账号 `fina_indicator`/`stock_basic` 限 1次/小时
- Yahoo Finance 对部分 IP 长期 429（数小时无法恢复）
- 脚本 `generate_report.py` 在以上情况下输出仅含 ROE/毛利率等基础字段，**缺少 PE、PB、52周高低、市值、分析师目标价**等关键估值指标

## 回退 API 链（从易到难）

### 1️⃣ 实时行情 + 核心估值（腾讯财经 API）⭐ 首选

最稳定、信息最全的单来源替代。一次调用即可获取股价、PE、PB、市值、EPS：

```python
import urllib.request
url = 'https://qt.gtimg.cn/q=sh600426'  # 沪深A股: sh{symbol}, sz{symbol}
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
resp = urllib.request.urlopen(req, timeout=10).read().decode('gbk')
```

返回格式为 `v_sh600426="..."` 以 `~` 分隔的字符串，关键字段索引：

| 索引 | 字段 | 示例 |
|------|------|------|
| [3] | 当前价 | 29.80 |
| [31] | 涨跌额 | 0.37 |
| [32] | 涨跌幅% | 1.26 |
| [33] | 今日最高 | 30.42 |
| [34] | 今日最低 | 29.02 |
| [37] | 成交额(万) | 98559 |
| [38] | EPS(每股收益) | 1.56 |
| [39] | PE(TTM/动态) | 16.92 |
| [43] | 振幅% | 4.76 |
| [44] | 流通市值(亿) | 630.29 |
| [45] | 总市值(亿) | 630.37 |
| [46] | PB(市净率) | 1.86 |
| [47] | 涨停价 | 32.37 |
| [48] | 跌停价 | 26.49 |
| [52] | PE(静态) | 14.11 |

**特点：** 无 API Key、无频率限制、数据实时、包含 tushare 和 yahoo 的几乎所有估值字段。

### 2️⃣ 52周高低 + 历史 K 线（新浪财经）⭐ 次选

获取近一年的日K线数据，从中提取52周最高/最低价：

```python
import urllib.request, json
url = 'https://money.finance.sina.com.cn/quotes_service/api/json_v2.php/CN_MarketData.getKLineData?symbol=sh600426&scale=240&ma=no&datalen=250'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0', 'Referer': 'https://finance.sina.com.cn'})
resp = urllib.request.urlopen(req, timeout=15).read().decode('gbk')
data = json.loads(resp)

# 提取52周高/低
high52 = max(float(d['high']) for d in data)
low52 = min(float(d['low']) for d in data)
close = float(data[-1]['close'])

# 计算52周分位
position_pct = (close - low52) / (high52 - low52) * 100
print(f'52周最高: {high52}, 52周最低: {low52}, 当前: {close}, 分位: {position_pct:.0f}%')

# 打印最近几个交易日
for d in data[-5:]:
    print(f'{d[\"day\"]} O={d[\"open\"]} H={d[\"high\"]} L={d[\"low\"]} C={d[\"close\"]} V={d[\"volume\"]}')
```

**特点：** ~250个交易日数据，覆盖一整年。无频率限制。返回包含 `day, open, high, low, close, volume, amount`。

### 3️⃣ 实时报价（新浪实时行情）

```python
import urllib.request
url = 'https://hq.sinajs.cn/list=sh600426'  # 逗号分隔可查多只
req = urllib.request.Request(url, headers={'Referer': 'https://finance.sina.com.cn'})
resp = urllib.request.urlopen(req, timeout=10).read().decode('gbk')
# 返回: var hq_str_sh600426="华鲁恒升,昨收,今开,当前,最高,最低,..."
```

### 4️⃣ 财务报告数据（东方财富数据中心）

```python
import urllib.request, json
url = 'https://datacenter.eastmoney.com/securities/api/data/v1/get?reportName=RPT_LICO_FN_CPD&columns=ALL&filter=(SECUCODE=%22600426.SH%22)&pageNumber=1&pageSize=2&sortTypes=-1&sortColumns=REPORT_DATE'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
resp = urllib.request.urlopen(req, timeout=10).read().decode('utf-8')
data = json.loads(resp)
```

**特点：** 需要 `SECUCODE` 格式（含 `.SH`/`.SZ` 后缀）。返回包含 EPS、ROE、营收、净利润、毛利率、每股净资产、每股经营现金流等核心财务指标。注意这个端点可能因 IP 频率触发连接中断（`Remote end closed connection`），建议间隔 5 秒以上重试。

### 5️⃣ 备用：东方财富行情 API（不稳定）

```
https://push2.eastmoney.com/api/qt/stock/get?secid=1.600426&fields=f43,f44,f45,f46,f47,f48,f57,f58,f116,f117
```

**注意：** 此端点已被 CDN/WAF 限制，目前基本不可用（`Remote end closed connection`），不再推荐。

## 推荐的完整数据获取流程（当脚本不可用/输出不足时）

```
1. 腾讯 API → 当前价、PE(TTM)、PB、总市值、EPS、振幅
2. 新浪 K 线 → 52周最高/最低、近期走势
3. 东方财富数据中心 → 最新 ROE、营收、净利润、毛利率、每股净资产
4. 手动计算:
   - 52周分位 = (当前价 - 低点) / (高点 - 低点) * 100%
   - TTM PE = 腾讯数据直接获取，无需计算
   - PE(静态) = 价格 / 最近年报 EPS
```

## 已知风险

- 东方财富数据中心 API 偶发连接中断（频率敏感）— 重试 1-2 次即可
- 新浪 K 线返回 GBK 编码，需 `decode('gbk')` 
- 腾讯返回包含非标准空字段，split 后注意长度检查
- 以上均为网页公开 API，无官方 SLA，仅供脚本回退使用
