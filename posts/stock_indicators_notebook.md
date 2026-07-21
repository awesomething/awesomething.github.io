---
title: Anomaly Stock ML
date: 2024-12-22
---

---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: "0.13"
    jupytext_version: 1.16.4
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# Stock Indicators and EMA Alert Engine

This notebook calculates common stock-market indicators and shows how the same
logic can be adapted to merchant metrics such as net revenue, ticket count, and
average ticket.

## Objectives

By the end of this notebook, you will be able to:

- Calculate simple and exponential moving averages.
- Calculate MACD, RSI, and Bollinger Bands.
- Measure deviation from an EMA baseline.
- Create persistent anomaly alerts.
- Visualize price and indicator trends.
- Reuse the approach for business-performance metrics.

## 1. Setup

```{code-cell} ipython3
from __future__ import annotations

import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

pd.set_option("display.max_columns", 50)
pd.set_option("display.width", 120)
```

## 2. Create or load price data

The next cell creates sample daily price data so the notebook can run without
an external file.

Replace this cell with the CSV-loading example below when real data is
available.

```{code-cell} ipython3
rng = np.random.default_rng(seed=42)

dates = pd.date_range("2025-01-01", periods=260, freq="B")
daily_returns = rng.normal(loc=0.0004, scale=0.015, size=len(dates))

close = 100 * np.cumprod(1 + daily_returns)
high = close * (1 + rng.uniform(0.001, 0.02, len(dates)))
low = close * (1 - rng.uniform(0.001, 0.02, len(dates)))
volume = rng.integers(500_000, 3_000_000, len(dates))

prices = pd.DataFrame(
    {
        "Close": close,
        "High": high,
        "Low": low,
        "Volume": volume,
    },
    index=dates,
)

prices.index.name = "Date"
prices.head()
```

### Optional: load a CSV instead

The CSV should contain at least `Date` and `Close` columns.

```{code-cell} ipython3
# Uncomment to load real data.
#
# prices = pd.read_csv(
#     "stock_prices.csv",
#     parse_dates=["Date"],
#     index_col="Date",
# )
#
# prices = prices.sort_index()
# prices.head()
```

## 3. Indicator function

```{code-cell} ipython3
def add_stock_indicators(
    data: pd.DataFrame,
    price_column: str = "Close",
) -> pd.DataFrame:
    """
    Add common technical indicators to stock-price data.

    Required column:
        Close, or the column supplied through price_column.

    Optional columns:
        High, Low, Volume.

    Parameters
    ----------
    data:
        Input DataFrame indexed by date or time.
    price_column:
        Name of the price column used for calculations.

    Returns
    -------
    pd.DataFrame
        A copy of the original data with indicator columns added.
    """
    if price_column not in data.columns:
        raise ValueError(f"Missing required column: {price_column}")

    df = data.copy()
    close = pd.to_numeric(df[price_column], errors="coerce")

    if close.isna().all():
        raise ValueError(f"Column {price_column!r} contains no numeric values.")

    # Simple moving averages
    df["SMA_20"] = close.rolling(window=20, min_periods=20).mean()
    df["SMA_50"] = close.rolling(window=50, min_periods=50).mean()
    df["SMA_200"] = close.rolling(window=200, min_periods=200).mean()

    # Exponential moving averages
    df["EMA_12"] = close.ewm(span=12, adjust=False).mean()
    df["EMA_20"] = close.ewm(span=20, adjust=False).mean()
    df["EMA_26"] = close.ewm(span=26, adjust=False).mean()
    df["EMA_50"] = close.ewm(span=50, adjust=False).mean()

    # MACD
    df["MACD"] = df["EMA_12"] - df["EMA_26"]
    df["MACD_SIGNAL"] = df["MACD"].ewm(span=9, adjust=False).mean()
    df["MACD_HISTOGRAM"] = df["MACD"] - df["MACD_SIGNAL"]

    # RSI using Wilder-style exponential smoothing
    price_change = close.diff()
    gains = price_change.clip(lower=0)
    losses = -price_change.clip(upper=0)

    average_gain = gains.ewm(
        alpha=1 / 14,
        adjust=False,
        min_periods=14,
    ).mean()

    average_loss = losses.ewm(
        alpha=1 / 14,
        adjust=False,
        min_periods=14,
    ).mean()

    relative_strength = average_gain / average_loss.replace(0, np.nan)
    df["RSI_14"] = 100 - (100 / (1 + relative_strength))

    # Handle periods with gains but no losses.
    df.loc[(average_loss == 0) & (average_gain > 0), "RSI_14"] = 100

    # Bollinger Bands
    rolling_mean = close.rolling(window=20, min_periods=20).mean()
    rolling_std = close.rolling(window=20, min_periods=20).std()

    df["BOLLINGER_MIDDLE"] = rolling_mean
    df["BOLLINGER_UPPER"] = rolling_mean + (2 * rolling_std)
    df["BOLLINGER_LOWER"] = rolling_mean - (2 * rolling_std)

    # Percentage deviation from the EMA baseline
    df["EMA_20_DEVIATION_PCT"] = (
        (close - df["EMA_20"]) / df["EMA_20"] * 100
    )

    # Example alert: price is at least 15% below its EMA baseline
    df["EMA_DROP_ALERT"] = df["EMA_20_DEVIATION_PCT"] <= -15

    return df
```

## 4. Calculate indicators

```{code-cell} ipython3
results = add_stock_indicators(prices)

results[
    [
        "Close",
        "SMA_20",
        "EMA_20",
        "EMA_20_DEVIATION_PCT",
        "RSI_14",
        "MACD",
        "MACD_SIGNAL",
        "EMA_DROP_ALERT",
    ]
].tail(10)
```

## 5. Visualize price and moving averages

```{code-cell} ipython3
fig, ax = plt.subplots(figsize=(12, 6))

ax.plot(results.index, results["Close"], label="Close")
ax.plot(results.index, results["SMA_20"], label="SMA 20")
ax.plot(results.index, results["EMA_20"], label="EMA 20")
ax.plot(results.index, results["EMA_50"], label="EMA 50")

ax.set_title("Closing Price with Moving Averages")
ax.set_xlabel("Date")
ax.set_ylabel("Price")
ax.legend()
ax.grid(True, alpha=0.3)

plt.show()
```

## 6. Visualize MACD

```{code-cell} ipython3
fig, ax = plt.subplots(figsize=(12, 5))

ax.plot(results.index, results["MACD"], label="MACD")
ax.plot(results.index, results["MACD_SIGNAL"], label="Signal")
ax.bar(
    results.index,
    results["MACD_HISTOGRAM"],
    label="Histogram",
    width=1.0,
)

ax.axhline(0, linewidth=1)
ax.set_title("MACD")
ax.set_xlabel("Date")
ax.set_ylabel("MACD Value")
ax.legend()
ax.grid(True, alpha=0.3)

plt.show()
```

## 7. Visualize RSI

```{code-cell} ipython3
fig, ax = plt.subplots(figsize=(12, 4))

ax.plot(results.index, results["RSI_14"], label="RSI 14")
ax.axhline(70, linestyle="--", label="Overbought threshold")
ax.axhline(30, linestyle="--", label="Oversold threshold")

ax.set_ylim(0, 100)
ax.set_title("Relative Strength Index")
ax.set_xlabel("Date")
ax.set_ylabel("RSI")
ax.legend()
ax.grid(True, alpha=0.3)

plt.show()
```

## 8. Visualize Bollinger Bands

```{code-cell} ipython3
fig, ax = plt.subplots(figsize=(12, 6))

ax.plot(results.index, results["Close"], label="Close")
ax.plot(
    results.index,
    results["BOLLINGER_MIDDLE"],
    label="20-period mean",
)
ax.plot(
    results.index,
    results["BOLLINGER_UPPER"],
    label="Upper band",
)
ax.plot(
    results.index,
    results["BOLLINGER_LOWER"],
    label="Lower band",
)

ax.fill_between(
    results.index,
    results["BOLLINGER_LOWER"],
    results["BOLLINGER_UPPER"],
    alpha=0.15,
)

ax.set_title("Bollinger Bands")
ax.set_xlabel("Date")
ax.set_ylabel("Price")
ax.legend()
ax.grid(True, alpha=0.3)

plt.show()
```

## 9. Build a persistent alert

A single observation can be noisy. This example requires the condition to be
true for three consecutive periods.

```{code-cell} ipython3
results["PERSISTENT_DROP_ALERT"] = (
    results["EMA_DROP_ALERT"]
    .rolling(window=3, min_periods=3)
    .sum()
    .eq(3)
)

results.loc[
    results["PERSISTENT_DROP_ALERT"],
    [
        "Close",
        "EMA_20",
        "EMA_20_DEVIATION_PCT",
        "EMA_DROP_ALERT",
        "PERSISTENT_DROP_ALERT",
    ],
].tail()
```

## 10. Add absolute-impact and volume conditions

For merchant alerts, a percentage threshold alone is usually insufficient.
A useful alert may require:

- A large percentage deviation.
- A meaningful absolute financial impact.
- Enough transaction volume.
- Persistence over multiple periods.

The example below demonstrates the structure using price and volume data.

```{code-cell} ipython3
results["EXPECTED_VALUE"] = results["EMA_20"]
results["ABSOLUTE_GAP"] = results["Close"] - results["EXPECTED_VALUE"]

minimum_absolute_gap = 5
minimum_volume = 750_000
minimum_drop_pct = -10

results["QUALIFIED_ALERT"] = (
    (results["EMA_20_DEVIATION_PCT"] <= minimum_drop_pct)
    & (results["ABSOLUTE_GAP"].abs() >= minimum_absolute_gap)
    & (results["Volume"] >= minimum_volume)
)

results["PERSISTENT_QUALIFIED_ALERT"] = (
    results["QUALIFIED_ALERT"]
    .rolling(window=3, min_periods=3)
    .sum()
    .eq(3)
)

results[
    [
        "Close",
        "EMA_20",
        "EMA_20_DEVIATION_PCT",
        "ABSOLUTE_GAP",
        "Volume",
        "PERSISTENT_QUALIFIED_ALERT",
    ]
].tail(10)
```

## 11. Fast and slow EMA crossover

A fast EMA reacts quickly to recent changes. A slow EMA represents a more
stable long-term baseline.

```{code-cell} ipython3
results["FAST_EMA"] = results["Close"].ewm(span=10, adjust=False).mean()
results["SLOW_EMA"] = results["Close"].ewm(span=40, adjust=False).mean()

results["FAST_BELOW_SLOW"] = results["FAST_EMA"] < results["SLOW_EMA"]

results["BEARISH_CROSSOVER"] = (
    results["FAST_BELOW_SLOW"]
    & ~results["FAST_BELOW_SLOW"].shift(1, fill_value=False)
)

results.loc[
    results["BEARISH_CROSSOVER"],
    ["Close", "FAST_EMA", "SLOW_EMA", "BEARISH_CROSSOVER"],
].tail()
```

```{code-cell} ipython3
fig, ax = plt.subplots(figsize=(12, 6))

ax.plot(results.index, results["Close"], label="Close")
ax.plot(results.index, results["FAST_EMA"], label="Fast EMA")
ax.plot(results.index, results["SLOW_EMA"], label="Slow EMA")

crossovers = results[results["BEARISH_CROSSOVER"]]

ax.scatter(
    crossovers.index,
    crossovers["Close"],
    marker="v",
    s=80,
    label="Bearish crossover",
)

ax.set_title("Fast and Slow EMA Crossover")
ax.set_xlabel("Date")
ax.set_ylabel("Price")
ax.legend()
ax.grid(True, alpha=0.3)

plt.show()
```

## 12. Adapt the logic to merchant metrics

The same function can be used with a merchant metric by setting
`price_column` to the relevant column name.

```{code-cell} ipython3
merchant_dates = pd.date_range("2025-01-01", periods=120, freq="D")
merchant_rng = np.random.default_rng(seed=7)

merchant_data = pd.DataFrame(
    {
        "NetRevenue": (
            10_000
            + 1_200 * np.sin(np.arange(120) * 2 * np.pi / 7)
            + merchant_rng.normal(0, 500, 120)
        ),
        "TicketCount": (
            600
            + 60 * np.sin(np.arange(120) * 2 * np.pi / 7)
            + merchant_rng.normal(0, 25, 120)
        ),
    },
    index=merchant_dates,
)

merchant_data.index.name = "Date"

# Introduce a temporary revenue decline.
merchant_data.loc[
    merchant_data.index[-8:],
    "NetRevenue",
] *= 0.78

merchant_results = add_stock_indicators(
    merchant_data,
    price_column="NetRevenue",
)

merchant_results[
    [
        "NetRevenue",
        "EMA_20",
        "EMA_20_DEVIATION_PCT",
        "EMA_DROP_ALERT",
    ]
].tail(12)
```

## 13. Recommended next step for restaurant or merchant data

A restaurant baseline should account for seasonality. Instead of comparing
every day with one global EMA, calculate separate baselines for combinations
such as:

- Merchant or location
- Weekday
- Daypart
- Channel
- Metric

For example, compare Tuesday lunch delivery revenue with earlier Tuesday lunch
delivery periods.

A production alert engine should also track:

- Expected value
- Actual value
- Percentage deviation
- Absolute impact
- Duration
- Confidence
- Likely contributing dimensions
- Alert severity
- Merchant feedback

## 14. Export results

```{code-cell} ipython3
results.to_csv("stock_indicators_output.csv")
merchant_results.to_csv("merchant_metric_indicators_output.csv")

print("Export complete.")
```

## Running this Markdown notebook

Install Jupytext:

```bash
pip install jupytext notebook pandas numpy matplotlib
```

Convert the Markdown file into an `.ipynb` notebook:

```bash
jupytext --to notebook stock_indicators_notebook.md
jupyter notebook stock_indicators_notebook.ipynb
```

You can also pair Markdown and notebook files:

```bash
jupytext --set-formats ipynb,md:myst stock_indicators_notebook.ipynb
```
