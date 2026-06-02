# 新浪财经美股行情页面字段提取指南

## 页面 URL

```
https://stock.finance.sina.com.cn/usstock/quotes/{TICKER}.html
```

## 页面 → data.info 字典字段映射

| 新浪页面上文字 | 提取方法 | data.info 字段名 | 示例值(PDD) |
|---------------|---------|-----------------|------------|
| 标题下方大号数字 | 页面第一个 StaticText 数字 | `regularMarketPrice` | 87.24 |
| 旁边小号 "+2.80(+3.32%)" | 第二个 StaticText | `regularMarketChange`, `regularMarketChangePercent` | 2.80, 3.32% |
| 前收盘： | `前收盘：` 后面的 cell | `previousClose` | 84.44 |
| 开盘： | `开盘：` 后面的 cell | `open` | 83.83 |
| 市盈率： | `市盈率：` 后面的 cell | `trailingPE` | 8.97 |
| 市值： | `市值：` 后面的 cell（亿＝100M） | `marketCap` | 1241.77亿→$124.18B |
| 每股收益： | `每股收益：` 后面的 cell | `epsCurrentYear` | 9.73 |
| 股本： | `股本：` 后面的 cell（亿） | `sharesOutstanding` | 14.23亿 |
| 52周区间： | `52周区间：` 后面的 cell `低-高` | `fiftyTwoWeekLow`, `fiftyTwoWeekHigh` | 81.56-139.41 |
| 52周区间： | 同上，提取低值 | `fiftyTwoWeekLow` | 81.56 |
| 52周区间： | 同上，提取高值 | `fiftyTwoWeekHigh` | 139.41 |
| 贝塔系数： | `贝塔系数：` 后面的 cell | `beta` | 可能为"--" |
| 股息/收益率： | `股息/收益率：` 后面的 cell | `dividendYield` | 通常为"--/--" |
| 盘前数字 | `盘前 :` 后面的数字 | 无对应字段，需单独处理 | 89.13 |
| 盘前涨跌幅 | `盘前 :` 后面的括号内容 | 无对应字段 | +1.89(2.17%) |
| 名称 | h1 heading 文字 | `shortName` | 拼多多公司 |
| 行业 | 从业务简介推断 | `industry`, `sector` | 互联网零售/电子商务 |

## 市值单位转换

新浪的市值显示为"亿"（即 1亿 = 100,000,000），但美股股价是美元。

```python
# 新浪显示 "1241.77亿" → 需要转为 1241.77 * 100,000,000 = 124,177,000,000
def parse_market_cap(sina_text: str) -> float:
    \"\"\"解析新浪市值到美元\"\"\"
    if not sina_text: return 0
    val = float(sina_text.replace('亿', '').replace('$', '').strip())
    return val * 100_000_000  # 亿→美元
```

## 盘前数据处理

盘前数据在 snapshot 中位于价格行下方：

```
盘前 : 89.13 1.89(2.12%)
    成交量：192,336
```

提取后可直接用于判断当日方向，但不写入 `data['info']`（analyze_us_stocks() 不消费该字段）。

## 步进式提取流程

```python
def extract_from_sina_snapshot(snapshot_text: str, ticker: str) -> dict:
    \"\"\"从 browser_snapshot 输出的文本中提取字段\"\"\"
    import re
    info = {}
    
    # 当前价 - 通常在 h1 标题下面的 StaticText
    price_match = re.search(r'StaticText "(\d+\.\d{2})"', snapshot_text)
    if price_match:
        info['regularMarketPrice'] = float(price_match.group(1))
    
    # 52周区间
    range_match = re.search(r'52周区间：(\d+\.\d+)-(\d+\.\d+)', snapshot_text)
    if range_match:
        info['fiftyTwoWeekLow'] = float(range_match.group(1))
        info['fiftyTwoWeekHigh'] = float(range_match.group(2))
    
    # 市盈率
    pe_match = re.search(r'市盈率：([\d.]+)', snapshot_text)
    if pe_match:
        info['trailingPE'] = float(pe_match.group(1))
    
    # 每股收益
    eps_match = re.search(r'每股收益：([-\d.]+)', snapshot_text)
    if eps_match:
        info['epsCurrentYear'] = float(eps_match.group(1))
    
    # 市值
    mcap_match = re.search(r'市值：([\d.]+亿)', snapshot_text)
    if mcap_match:
        val = float(mcap_match.group(1).replace('亿', ''))
        info['marketCap'] = val * 100_000_000
    
    return info
```

## 注意事项

1. snapshot 文本中字段顺序可能变化，使用正则而非位置提取
2. 市盈率为"--"时表示该股亏损（EPS 为负），trailingPE 设为 None 或 0
3. 市值带"亿"后缀时需要乘以 1e8 转换为美元
4. 贝塔系数很多美股显示"--"，此时 beta 不可用
5. 页面在美股交易时段外显示前收盘价的静态数据
6. 盘前数据在美东时间 04:00-09:30 之间可用
