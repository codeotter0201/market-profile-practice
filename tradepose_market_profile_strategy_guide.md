# TradePose Market Profile Strategy Guide

這份 guide 對應 notebook：

- `tradepose_market_profile_strategy_guide.ipynb`

目標是讓新使用者或 LLM 快速理解：

1. 用 `BatchTester` 查詢 instrument metadata。
2. 建立一個極簡 `StrategyConfig`。
3. 用 `submit_ohlcv()` 下載 OHLCV 與 Market Profile 指標。
4. 查看 Market Profile struct 與 `tpo_distribution` 內部值。
5. 用 `submit()` 產生 trades，供後續 MAE / MFE / PnL 分析使用。

## 1. 環境與連線

Colab 先安裝 SDK：

```python
!pip install tradepose-client --ignore-requires-python --quiet
```

本機先同步 uv 環境：

```bash
uv sync
```

VS Code 開啟 `.ipynb` 後，選擇這個專案的 `.venv` Python kernel。專案已透過 `uv add ipykernel python-dotenv` 加入 notebook 與 `.env` 載入需要的套件。

建立 tester：

```python
import os
from getpass import getpass

from dotenv import load_dotenv
from tradepose_client import BatchTester

load_dotenv()

API_KEY = os.getenv("TRADEPOSE_API_KEY") or getpass("TradePose API key: ")
SERVER_URL = os.getenv("TRADEPOSE_SERVER_URL", "https://api.tradepose.com")

tester = BatchTester(api_key=API_KEY, server_url=SERVER_URL, poll_interval=2.0)
```

本機使用時在 repo root 建立 `.env`：

```bash
TRADEPOSE_API_KEY=sk_xxx
```

`.env` 已由 `.gitignore` 排除，不要把實際 API key 寫進 notebook 或 markdown。

## 2. 查詢 Instrument

先用 server 上的 instrument metadata 確認可用商品代碼與 tick size：

```python
instruments = tester.list_instruments(symbol="XAUUSD", limit=20)

for inst in instruments.instruments:
    print(inst)
```

策略中的 `instrument` 必須使用 server 可辨識的 instrument ID，例如：

```python
INSTRUMENT = "PEPPERSTONE:spot:XAUUSD"
```

Market Profile 的 `tick_size` 會影響 POC / VAH / VAL 計算粒度。XAUUSD 通常先用 `1.0`；實際值請依 instrument metadata 或商品特性調整。

## 3. 極簡 StrategyConfig

如果目標只是下載 OHLCV 並看 Market Profile，策略邏輯不用寫得很複雜。本範例只做：

- 指標：Daily Market Profile、Initial Balance Market Profile、ATR。
- 進場：`open > vah`。
- 出場：`open < poc`。

`indicators` 內部可以指定不同商品，例如主策略交易 XAUUSD，但某個 indicator 使用 NAS100；後端會依 instrument / freq 自動載入並 join 到計算資料中。

`volatility_indicator` 在 trades 分析中用來正規化 MAE / MFE，例如 `mae / ATR`、`mfe / ATR`。本範例使用 ATR。

`volatility_level` 用來判斷市場 regime，例如低波動 / 中波動 / 高波動。它不是必要設定；有設定時，trades 裡會出現相關 level 欄位，方便後續依 regime 分組分析。

```python
strategy = SimpleMarketProfileParams(
    instrument="PEPPERSTONE:spot:XAUUSD",
    base_freq=Freq.MIN_15,
    trade_direction=TradeDirection.LONG,
    tick_size=1.0,
    atr_freq=Freq.HOUR_1,
    atr_period=120,
).create_strategy()
```

`print(strategy.name)` 會看到 `sp:v1z:...` 開頭的壓縮名稱。這是 SDK 用 zlib + base64url 產生的 machine-readable reference；如果要看可讀參數，使用 params class 的 decode helper：

```python
print(strategy.name)
print(SimpleMarketProfileParams.decode_label(strategy.name))
```

## 4. 取回指標計算結果

如果只是想檢查 OHLCV 加上指標欄位，使用 `submit_ohlcv()`：

```python
period = Period(start="2025-01-01", end="2027-02-01")

ohlcv_result = tester.submit_ohlcv(strategy=strategy, period=period, timeout=300)

import time
time.sleep(5)

ohlcv_df = ohlcv_result.data
ohlcv_df
```

這會從 `StrategyConfig` 抽出 indicators，提交 on-demand OHLCV task。背景輪詢下載完成後可從 `ohlcv_result.data` 或 `ohlcv_result.df` 讀到包含 Market Profile 欄位的 DataFrame。`OHLCVPeriodResult` 本身不提供 `wait()` method。

Market Profile 欄位是 struct。先找出 daily MP 與 IB MP 欄位，再取其中一筆有 `tpo_distribution` 的 row：

```python
market_profile_cols = [
    spec.display_name()
    for spec in strategy.indicators
    if spec.indicator.type == "MarketProfile"
]
daily_mp_col, ib_mp_col = market_profile_cols

mb = pl.col(daily_mp_col).struct.field("tpo_distribution")
ib = pl.col(ib_mp_col).struct.field("tpo_distribution")

df = ohlcv_result.df.filter(mb.is_not_null() | ib.is_not_null())

row = df.head(1).to_dicts()[0]
mp_data = row[daily_mp_col].pop("tpo_distribution")[0]

row
```

把 `tpo_distribution` 轉成表格後，依價格由高到低檢查 Market Profile 內部值：

```python
pl.Config.set_fmt_str_lengths(10_000)
pl.Config.set_fmt_table_cell_list_len(100)
pl.Config.set_tbl_rows(100)

mp_df = pl.DataFrame(mp_data)
mp_df.sort("price")[::-1]
```

## 5. 產生 Trades

`tester.submit()` 會依照策略的進出場規則產生 trades。這裡只是帶到流程；產生後的 trades 會包含 MAE、MFE、PnL 等欄位，可供後續分析、篩選、分組與視覺化使用。

```python
batch = tester.submit(strategies=[strategy], periods=[period])
batch.wait(timeout=600)

summary = batch.summary()
trades = batch.all_trades()

print(summary)
trades.tail(20)
```

## 6. 常見失敗點

- 找不到 instrument：先查 `tester.list_instruments()`，不要手猜 instrument ID。
- Market Profile 欄位大多為 null：確認 period 有足夠資料、`tick_size` 合理、Market Profile 的 mode 時段符合交易時段。
- `ohlcv_result.data` 是 `None`：背景下載尚未完成，稍後重新讀 `ohlcv_result.data` 或增加等待時間。
