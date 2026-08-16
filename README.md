# Crypto Market Data Aggregation Lab

> [!IMPORTANT]
> This repository is both a **reusable market feature-engineering toolkit** and an **engineering postmortem**.
>
> The code still has practical value. Its exchange adapters, flow processors, price-level aggregation, cross-exchange synthesis, and generated feature dictionaries can be imported into another Python project as a local plugin.
>
> The part that failed was the original assumption that fixed one-minute bars should be the main representation for machine-learning training.
>
> **Fixed-time bars contain radically different amounts of information.**
>
> A quiet minute and an extremely active minute both become one training row. This makes samples statistically inconsistent and encourages models to overfit to volatility regimes, activity regimes, and arbitrary clock boundaries.
>
> The code is therefore useful for:
>
> - feature prototyping;
> - dashboards and market summaries;
> - exploratory research;
> - cross-exchange normalization;
> - local feature generation;
> - integration into another data pipeline.
>
> It should not be treated as a production market-data collector or a lossless ML dataset generator without additional infrastructure.

---

## Quick Navigation

- [What This Project Is](#what-this-project-is)
- [Why the Code Is Still Useful](#why-the-code-is-still-useful)
- [Using It as a Local Plugin](#using-it-as-a-local-plugin)
- [Architecture](#architecture)
- [Technology Used](#technology-used)
- [Feature Engineering](#feature-engineering)
- [Generated Feature Groups](#generated-feature-groups)
- [How the Processing Works](#how-the-processing-works)
- [Using Individual Flow Modules](#using-individual-flow-modules)
- [Using the Combined Processor](#using-the-combined-processor)
- [What Worked](#what-worked)
- [The Main Problem With Fixed-Time Bars](#the-main-problem-with-fixed-time-bars)
- [Problems With Some Derived Features](#problems-with-some-derived-features)
- [The pandas Performance Limitation](#the-pandas-performance-limitation)
- [Why the Original Architecture Failed](#why-the-original-architecture-failed)
- [What I Learned](#what-i-learned)
- [Better Architecture](#better-architecture)
- [Successor Project](#successor-project)
- [Repository Structure](#repository-structure)
- [Current Status](#current-status)

---

## What This Project Is

This project is an experimental Python library for processing and combining crypto market data from multiple exchanges.

It accepts recorded JSON payloads or payloads supplied by an external API/WebSocket collector, converts exchange-specific messages into a common internal format, processes them through stateful market-data flows, and generates unified feature dictionaries.

The basic idea is:

```text
many exchanges
    ↓
different native payloads
    ↓
exchange-specific normalization
    ↓
stateful market-data processors
    ↓
price-level and time-window aggregation
    ↓
cross-exchange feature synthesis
```

The system processes:

- order books;
- trades;
- open interest;
- funding rates;
- liquidations;
- trader positioning ratios;
- options open interest;
- experimental order-book adjustment features.

The target instrument was mainly Bitcoin across:

- spot markets;
- linear perpetuals;
- inverse perpetuals;
- options markets.

Adapters were created for exchanges including:

- Binance;
- OKX;
- Bybit;
- Bitget;
- BingX;
- KuCoin;
- Deribit;
- Coinbase;
- HTX;
- Gate.io;
- MEXC.

Support and source reliability vary between exchanges and data types.

The project is not a trading execution engine and does not place orders.

---

## Why the Code Is Still Useful

The project failed as a complete production and ML-data architecture, but the code itself is not useless.

The feature-engineering modules can still be useful inside another project.

### Reusable components

The repository contains reusable logic for:

- parsing exchange-specific payloads;
- converting contract quantities into comparable units;
- grouping prices into configurable levels;
- maintaining local order-book state;
- processing trades by side and price;
- calculating one-minute OHLC summaries;
- generating volume profiles;
- tracking open-interest changes;
- combining funding rates;
- grouping liquidations by price;
- aggregating options by strike distance and expiration;
- merging data from multiple exchanges;
- flattening features into one dictionary.

A developer can use:

```text
the complete btcSynth processor
```

or only selected components such as:

```text
booksflow
tradesflow
oiFundingflow
liquidationsflow
oiflowOption
booksmerger
tradesmerger
oiomnifier
```

This makes the repository useful as a local feature plugin, even though it is not packaged as a clean PyPI library.

---

### Good use cases

The code is still suitable for:

- offline research with recorded payloads;
- exploratory notebooks;
- generating order-book heatmaps;
- building market dashboards;
- testing feature ideas;
- normalizing multiple exchange formats;
- creating market-state summaries;
- feeding features into another application;
- using individual processors inside a custom collector;
- studying cross-exchange market microstructure.

---

### What it does not provide

The repository does not provide a complete production networking layer.

It does not fully handle:

- WebSocket connection management;
- sequence validation;
- reconnect recovery;
- raw event persistence;
- deterministic replay;
- queue backpressure;
- distributed processing;
- complete schema validation;
- production monitoring.

The host application must supply the exchange payloads.

---

## Using It as a Local Plugin

The repository is not packaged as a standard Python package.

The simplest way to use it inside another project is to clone it locally and expose the internal module directories through `sys.path`.

### Clone and install

```bash
git clone https://github.com/flowdrivenml/fixedbar-failure.git
cd fixedbar-failure

python -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
```

The existing `requirements.txt` contains the original development environment and is much larger than the strict runtime requirement.

That file is preserved because the code is intentionally not being rewritten.

---

### Add the repository to another project

```python
from pathlib import Path
import sys

REPO_ROOT = Path("/absolute/path/to/fixedbar-failure").resolve()

sys.path.insert(0, str(REPO_ROOT / "StreamEngineBase"))
sys.path.insert(0, str(REPO_ROOT / "StreamEngine"))
```

After those paths are available, the main processor can be imported:

```python
from synthHub import btcSynth
```

Or individual processors can be imported:

```python
from flow import (
    booksflow,
    tradesflow,
    oiFundingflow,
    liquidationsflow,
    oiflowOption,
    indicatorflow,
)
```

The exchange adapters can be imported from:

```python
from lookups import btc as BTCLookups
from lookups import unit_conversion_btc
```

The synthesis components can be imported from:

```python
from synthesis import (
    booksmerger,
    tradesmerger,
    oiomnifier,
)
```

> [!NOTE]
> Some internal imports still assume the original development directory layout.
>
> Adding both `StreamEngine/` and `StreamEngineBase/` to `sys.path` allows the current code to be used without rewriting those imports.

---

## Architecture

The original processing architecture looks like this:

```text
Recorded JSON or external collector
                 │
                 ▼
         Exchange lookup adapters
   parsing · timestamps · unit conversion
                 │
                 ▼
         Stateful flow processors
 books · trades · OI · funding · liquidations
                 │
                 ▼
       Per-second pandas structures
                 │
                 ▼
        Price-level aggregation
                 │
                 ▼
        One-minute flow snapshots
                 │
                 ▼
       Cross-exchange synthesis
                 │
                 ▼
       Flattened feature dictionary
```

The repository separates the system into three major layers:

```text
lookups
    → understand exchange payloads

flows
    → process individual market streams

synthesis
    → combine streams and exchanges
```

This separation is one of the stronger parts of the design.

---

## Technology Used

### Python

Python handles:

- object-oriented flow processors;
- exchange adapters;
- orchestration;
- feature construction;
- cross-exchange aggregation;
- replaying recorded JSON;
- exploratory testing.

Python made it possible to develop and test a large number of feature ideas quickly.

---

### NumPy

NumPy is used for:

- price-level calculation;
- bucket assignment;
- unique-level detection;
- grouped sums;
- array alignment;
- numerical transformations;
- option strike ranges;
- expiration windows.

A central price aggregation pattern is:

```python
unique_levels, inverse_indices = np.unique(
    levels,
    return_inverse=True
)

group_sums = np.bincount(
    inverse_indices,
    weights=amounts
)
```

This groups many individual observations into configurable price levels without manually iterating over every output bucket.

---

### pandas

pandas handles:

- 60-row second-level matrices;
- dynamic price-level columns;
- snapshot creation;
- DataFrame joins;
- cross-exchange merging;
- forward filling;
- backward filling;
- OHLC calculations;
- standard deviation;
- volume profiles;
- open-interest profiles;
- order-book heatmaps.

pandas was excellent for feature experimentation.

It became problematic when the intended workload grew into continuous high-volume streaming.

---

### JSON

The flow processors accept JSON payloads.

Recorded exchange responses can therefore be replayed through the same feature processors used for live messages.

This is useful for:

- adapter development;
- offline debugging;
- manual validation;
- notebook experiments;
- reproducing specific exchange responses.

---

### Jupyter and IPython

Jupyter was used to inspect:

- generated DataFrames;
- price-level heatmaps;
- feature dictionaries;
- exchange normalization;
- unusual API values;
- empty snapshots;
- suspicious outliers.

The example notebook is located at:

```text
examples/btcSynth.ipynb
```

---

### Matplotlib

Matplotlib was used for exploratory visualization of:

- order-book depth;
- volume profiles;
- price-level activity;
- feature distributions;
- generated heatmaps.

---

## Feature Engineering

Feature engineering is the strongest and most reusable part of this repository.

The project does much more than create OHLC bars.

It attempts to transform different forms of market microstructure data into a unified feature space.

---

### Price-level aggregation

Order books, trades, liquidations, and open-interest changes are grouped into configurable price levels.

For example:

```python
level_size = 20
```

Conceptually creates buckets such as:

```text
60000–60020
60020–60040
60040–60060
60060–60080
```

This reduces a high-dimensional market stream into a smaller structured representation.

Instead of modelling every exact price, another project can work with approximate liquidity and activity zones.

---

### Configuring the level size

A smaller level size creates more detailed features:

```python
level_size = 5
```

A larger level size creates a more compressed representation:

```python
level_size = 100
```

The correct value depends on:

- instrument price;
- volatility;
- expected holding period;
- model design;
- required feature resolution;
- available compute.

---

### Order-book features

The order-book processors maintain local bid and ask state.

They produce features such as:

- displayed liquidity by price level;
- aggregated bid and ask depth;
- current approximate midpoint;
- one-minute order-book snapshots;
- cross-exchange liquidity profiles;
- experimental removed-liquidity estimates;
- experimental reinforced-liquidity estimates.

Conceptually:

```text
Price Level    Displayed Liquidity
60000          35.4 BTC
60020          19.8 BTC
60040          42.1 BTC
```

These features can be used for:

- heatmaps;
- liquidity concentration;
- local support/resistance research;
- imbalance features;
- cross-exchange depth comparison.

---

### Trade features

Trades are separated by side and aggregated by second and price level.

The processor tracks:

- total buy volume;
- total sell volume;
- number of buy trades;
- number of sell trades;
- buy volume profile;
- sell volume profile;
- total volume profile;
- ordered buy trade sizes;
- ordered sell trade sizes.

Conceptually:

```text
Price Level    Buy Volume    Sell Volume
60000          12.4          8.1
60020          20.2          18.6
60040          7.4           22.5
```

These features make it possible to compare:

```text
where liquidity was displayed
```

against:

```text
where trades actually occurred
```

---

### OHLC and volatility features

The trade snapshots also produce standard market summaries:

- open;
- high;
- low;
- close;
- standard deviation of price;
- buy and sell volume;
- trade counts.

These are useful for:

- dashboards;
- monitoring;
- simple indicators;
- baseline ML features;
- compatibility with traditional bar-based analysis.

The problem was not that these features are useless.

The problem was treating fixed one-minute summaries as the only canonical representation of the market.

---

### Volume profiles

The system generates:

```text
total volume by price level
buy volume by price level
sell volume by price level
```

Example access:

```python
features = processor.ratrive_data()

volume_profile = features.get("spot_VolProfile")
buy_profile = features.get("spot_buyVolProfile")
sell_profile = features.get("spot_sellVolProfile")
```

These profiles can be used to build:

- volume heatmaps;
- buy/sell imbalance;
- high-volume nodes;
- low-volume zones;
- cross-exchange activity comparisons.

---

### Open-interest features

The perpetual processor tracks:

- current open interest;
- total cross-exchange open interest;
- positive OI changes;
- negative OI changes;
- total OI change;
- OI volatility;
- OI change history;
- OI values per instrument;
- OI changes grouped by price level.

Example feature names include:

```text
perp_total_oi
perp_oi_increases
perp_oi_decreases
perp_oi_total
perp_oi_change
perp_oi_Vola
perp_orderedOIChanges
perp_OIs_per_instrument
```

These features are useful for describing changes in market participation.

The price-level interpretation must still be treated carefully because open-interest data does not expose the entry price of individual positions.

---

### Funding features

Funding rates from perpetual markets are normalized and combined.

Generated features include:

```text
perp_weighted_funding
perp_fundings_per_instrument
```

These can be used for:

- market positioning summaries;
- cross-exchange funding divergence;
- identifying unusually expensive long or short exposure;
- dashboard indicators;
- research features.

---

### Liquidation features

Liquidations are divided into long and short liquidations.

The processor generates:

- total long liquidation volume;
- total short liquidation volume;
- long liquidations by price level;
- short liquidations by price level;
- liquidation event counts;
- liquidation event histories.

Example feature names include:

```text
perp_liquidations_longsTotal
perp_liquidations_longs
perp_liquidations_shortsTotal
perp_liquidations_shorts
```

Liquidations are among the cleaner feature groups because they are directly reported market events.

They are still dependent on exchange coverage and payload quality.

---

### Trader positioning ratios

Where exchanges expose trader positioning data, the system processes:

- top trader account ratios;
- top trader position ratios;
- global trader ratios.

Example feature names include:

```text
perp_TTA_ratio
perp_TTP_ratio
perp_GTA_ratio
```

These features can be combined with:

- funding;
- OI;
- liquidations;
- price movement;
- trade imbalance.

---

### Options open-interest features

Options are grouped by:

- call or put;
- strike distance from current price;
- time to expiration.

Example configuration:

```python
pranges = np.array([
    0.0,
    1.0,
    2.0,
    5.0,
    10.0,
])

expiry_windows = np.array([
    0.0,
    1.0,
    3.0,
    7.0,
])
```

This can create conceptual strike groups such as:

```text
puts:
    0–1% below price
    1–2% below price
    2–5% below price
    5–10% below price

calls:
    0–1% above price
    1–2% above price
    2–5% above price
    5–10% above price
```

The same groups are separated by expiration windows.

Dynamic output keys can look like:

```text
oi_option_puts_0
oi_option_puts_0_5
oi_option_puts_5_10

oi_option_calls_0
oi_option_calls_0_5
oi_option_calls_5_10
```

The exact keys depend on the configured price ranges and expiration windows.

---

### Experimental order-book adjustment features

The system also attempted to estimate:

- removed liquidity;
- reinforced liquidity;
- duration of removed liquidity;
- duration of reinforced liquidity;
- volatility of these changes.

Example output names include:

```text
spot_voids
spot_reinforces
spot_totalVoids
spot_totalReinforces
spot_voidsDuration
spot_reinforcesDuration

perp_voids
perp_reinforces
perp_totalVoids
perp_totalReinforces
perp_voidsDuration
perp_reinforcesDuration
```

These are experimental proxy features.

They can still be useful for visualization or hypothesis generation, but they should not be interpreted as exact cancellation data.

---

## Generated Feature Groups

### Spot feature examples

```text
spot_books
spot_buyVol
spot_sellVol
spot_open
spot_close
spot_low
spot_high
spot_Vola
spot_VolProfile
spot_buyVolProfile
spot_sellVolProfile
spot_numberBuyTrades
spot_numberSellTrades
spot_orderedBuyTrades
spot_orderedSellTrades
spot_voids
spot_reinforces
spot_totalVoids
spot_totalReinforces
spot_voidsDuration
spot_voidsDurationVola
spot_reinforcesDuration
spot_reinforcesDurationVola
```

---

### Perpetual feature examples

```text
perp_books
perp_buyVol
perp_sellVol
perp_open
perp_close
perp_low
perp_high
perp_Vola
perp_VolProfile
perp_buyVolProfile
perp_sellVolProfile
perp_numberBuyTrades
perp_numberSellTrades
perp_orderedBuyTrades
perp_orderedSellTrades
perp_weighted_funding
perp_total_oi
perp_oi_increases
perp_oi_increases_Vola
perp_oi_decreases
perp_oi_decreases_Vola
perp_oi_total
perp_oi_total_Vola
perp_oi_change
perp_oi_Vola
perp_orderedOIChanges
perp_OIs_per_instrument
perp_fundings_per_instrument
perp_liquidations_longsTotal
perp_liquidations_longs
perp_liquidations_shortsTotal
perp_liquidations_shorts
perp_TTA_ratio
perp_TTP_ratio
perp_GTA_ratio
```

---

### Reading the feature dictionary

After the complete flow snapshots have been merged:

```python
processor.merge()

features = processor.ratrive_data()
```

> [!NOTE]
> `ratrive_data` is the current method spelling in the existing code.
>
> It is intentionally documented as-is because the implementation is not being changed.

Individual features can then be accessed with:

```python
spot_books = features.get("spot_books")
spot_buy_volume = features.get("spot_buyVol")
perp_open_interest = features.get("perp_total_oi")
long_liquidations = features.get("perp_liquidations_longs")
funding = features.get("perp_weighted_funding")
```

The complete dictionary can also be inspected directly:

```python
for feature_name, value in features.items():
    print(feature_name, value)
```

---

## How the Processing Works

### Exchange normalization

Exchange-specific parsing lives mainly in:

```text
StreamEngineBase/lookups.py
```

Different exchanges use different:

- JSON structures;
- field names;
- timestamps;
- contract units;
- instrument naming conventions;
- order-book formats;
- open-interest units;
- option identifiers.

The lookup layer converts those responses into structures expected by the flow processors.

For example, trade lookup methods generally return:

```text
[
    side,
    price,
    amount,
    timestamp
]
```

Order-book lookup methods generally return:

```text
[
    [price, amount],
    [price, amount],
    ...
]
```

plus a normalized timestamp.

---

### Unit conversion

Crypto derivatives are not directly comparable across exchanges.

Quantity may be expressed in:

```text
BTC
USDT
USD
contracts
inverse contract units
linear contract units
```

The repository includes a conversion map:

```python
from lookups import unit_conversion_btc
```

This provides exchange-specific conversion functions used by the adapters.

These conversion rules must still be independently validated before serious use.

Some were based on exchange specifications, while others had to be inferred from observed market values.

---

### Stateful flow processing

The main stream processors live in:

```text
StreamEngineBase/flow.py
```

Each processor maintains local state.

For example, `booksflow` stores:

```text
current bids
current asks
current timestamp
current midpoint
per-second aggregated depth
completed snapshot
```

The trade flow stores:

```text
buy volume
sell volume
trade counts
ordered trade sizes
per-second price-level activity
```

The OI flow stores:

```text
current OI
previous OI
OI change
funding rate
price
per-second values
```

---

### Snapshot creation

The processors use the second component of the timestamp:

```text
second 0
second 1
second 2
...
second 59
```

When the second rolls from the end of one minute back to the start of the next minute, the previous structure becomes a completed snapshot.

That snapshot is then available to the synthesis layer.

---

### Cross-exchange synthesis

The synthesis code lives mainly in:

```text
StreamEngineBase/synthesis.py
```

It combines individual exchange snapshots.

Conceptually:

```text
Binance snapshot
OKX snapshot
Bybit snapshot
Deribit snapshot
        │
        ▼
normalize columns
        │
        ▼
merge price levels
        │
        ▼
sum or combine features
        │
        ▼
global BTC market representation
```

The synthesis layer produces:

- unified order-book levels;
- combined trade profiles;
- OHLC summaries;
- combined OI;
- weighted funding;
- liquidation profiles;
- options profiles;
- trader positioning summaries.

---

## Using Individual Flow Modules

A project does not need to use the complete synthesis engine.

Individual flow classes can be imported and used separately.

### Create exchange lookups

```python
from lookups import btc as BTCLookups
from lookups import unit_conversion_btc

lookups = BTCLookups(unit_conversion_btc)
```

---

### Create an order-book processor

```python
from flow import booksflow

books = booksflow(
    exchange="binance",
    symbol="btc_usdt",
    insType="perpetual",
    level_size=20,
    lookup=lookups.binance_depth_lookup,
    book_ceil_thresh=5,
)
```

Feed a JSON payload into it:

```python
books.update_books(depth_payload_json)
```

After a completed window is available:

```python
snapshot = books.snapshot
```

---

### Create a trade processor

```python
from flow import tradesflow

trades = tradesflow(
    exchange="binance",
    symbol="btc_usdt",
    insType="perpetual",
    level_size=20,
    lookup=lookups.binance_trades_lookup,
)
```

Feed a trade payload:

```python
trades.input_trades(trade_payload_json)
```

Completed outputs include:

```python
buy_snapshot = trades.snapshot_buys
sell_snapshot = trades.snapshot_sells
total_snapshot = trades.snapshot_total
```

---

### Create an open-interest and funding processor

```python
from flow import oiFundingflow

oi_funding = oiFundingflow(
    exchange="binance",
    symbol="btc_usdt",
    insType="perpetual",
    level_size=20,
    lookup_oi=lookups.binance_OI_lookup,
    lookup_funding=lookups.binance_funding_lookup,
)
```

Feed the streams independently:

```python
oi_funding.input_funding(funding_payload_json)
oi_funding.input_oi(open_interest_payload_json)
```

The completed DataFrame is available through:

```python
snapshot = oi_funding.snapshot
```

---

### Create a liquidation processor

```python
from flow import liquidationsflow

liquidations = liquidationsflow(
    exchange="binance",
    symbol="btc_usdt",
    insType="perpetual",
    level_size=20,
    lookup=lookups.binance_liquidations_lookup,
)
```

Feed liquidation messages:

```python
liquidations.input_liquidations(liquidation_payload_json)
```

Completed outputs include:

```python
longs = liquidations.snapshot_longs
shorts = liquidations.snapshot_shorts
total = liquidations.snapshot_total
```

---

## Using the Combined Processor

The higher-level `btcSynth` class creates and combines multiple exchange flows.

### Create the processor

```python
from pathlib import Path
import sys
import numpy as np

REPO_ROOT = Path("/absolute/path/to/fixedbar-failure").resolve()

sys.path.insert(0, str(REPO_ROOT / "StreamEngineBase"))
sys.path.insert(0, str(REPO_ROOT / "StreamEngine"))

from synthHub import btcSynth

processor = btcSynth(
    level_size=20,
    pranges=np.array([
        0.0,
        1.0,
        2.0,
        5.0,
        10.0,
    ]),
    expiry_windows=np.array([
        0.0,
        1.0,
        3.0,
        7.0,
    ]),
    exchanges_spot_perp=[
        "binance",
        "okx",
        "bybit",
    ],
    exchanges_option=[
        "bybit",
        "okx",
        "deribit",
    ],
)
```

---

### Route exchange messages

The processor exposes stream-specific methods.

For example:

```python
processor.add_binance_spot_btcusdt_depth(
    binance_spot_depth_json
)

processor.add_binance_spot_btcusdt_trades(
    binance_spot_trade_json
)
```

The same pattern is used for:

- other exchanges;
- perpetual order books;
- perpetual trades;
- funding;
- open interest;
- liquidations;
- positioning ratios;
- options open interest.

---

### Merge completed snapshots

After the relevant flows have completed a time window:

```python
processor.merge()
```

Retrieve the complete feature dictionary:

```python
features = processor.ratrive_data()
```

Read individual features:

```python
spot_books = features.get("spot_books")
perp_books = features.get("perp_books")
spot_volume = features.get("spot_VolProfile")
perp_open_interest = features.get("perp_total_oi")
long_liquidations = features.get("perp_liquidations_longs")
```

This is the main plugin-style integration point for another Python application.

---

## What Worked

### The feature engine was genuinely useful

The project generated a broad set of market features from multiple data types.

It was not only an OHLC bar generator.

It combined:

```text
books
trades
open interest
funding
liquidations
positioning ratios
options
```

into one market representation.

That feature-engineering logic can still be useful in other projects.

---

### The adapter abstraction was correct

Separating:

```text
exchange parsing
```

from:

```text
market feature logic
```

was a good design decision.

The feature processors do not need to understand every native exchange schema.

They consume normalized values produced by the lookup layer.

---

### Individual components can be reused

A developer can use only:

```text
booksflow
```

or:

```text
tradesflow
```

without using the full system.

The same applies to:

- OI processing;
- liquidations;
- options;
- synthesis classes.

This makes the repository more useful as a feature library than as a complete application.

---

### It was a good feature-research environment

Python, NumPy, pandas, and Jupyter made it easy to experiment with:

- different price levels;
- new volume profiles;
- cross-exchange features;
- options groupings;
- OI transformations;
- heatmaps;
- new combinations of existing features.

This was the correct environment for discovering what should exist.

It was not the correct final architecture for processing high-volume live data.

---

## The Main Problem With Fixed-Time Bars

The most important research conclusion was that fixed-time bars are a weak default representation for market-data ML.

### Bars contain different amounts of information

Consider two one-minute windows.

```text
Minute A
--------
200 trades
small order-book changes
almost no liquidations
low volatility
```

```text
Minute B
--------
25,000 trades
massive order-book movement
large liquidations
open-interest shock
extreme volatility
```

Both become:

```text
1 training row
```

The model therefore receives samples with completely different information content.

A quiet bar may represent almost no meaningful market activity.

An active bar may compress thousands of related events.

---

### This encourages overfitting

The model can easily learn:

```text
activity regime
volatility regime
session regime
liquidation regime
```

rather than a robust relationship between market structure and future outcomes.

A model may perform well when validation data resembles the same regime.

It may then fail when:

- volatility changes;
- liquidity changes;
- the active trading session changes;
- the market moves from trending to mean-reverting;
- exchange behavior changes;
- a new liquidation regime appears.

---

### Clock boundaries are arbitrary

The market does not care that time changed from:

```text
12:34:59
```

to:

```text
12:35:00
```

A coherent event may cross that boundary:

```text
12:34:58    liquidation begins
12:34:59    book starts collapsing
12:35:00    aggressive selling continues
12:35:02    OI drops
12:35:04    price rebounds
```

A one-minute system splits this into two observations even though it is one market event.

---

### Event order disappears

A sequence such as:

```text
liquidation
    ↓
order-book collapse
    ↓
large market sell
    ↓
open-interest reduction
    ↓
price rebound
```

may become:

```text
sell volume = X
liquidations = Y
OI change = Z
close = P
```

The totals remain, but the event ordering is gone.

For some models, the sequence may contain more information than the aggregate values.

---

### Better sampling alternatives

Depending on the research problem, more suitable representations may include:

- tick bars;
- volume bars;
- dollar bars;
- imbalance bars;
- volatility-triggered windows;
- event-triggered samples;
- custom market-state transitions.

The better mental model is:

```text
clock time != information time
```

Fixed-time bars remain useful for:

- dashboards;
- reports;
- monitoring;
- simple indicators;
- compatibility with standard charting.

They should not automatically be treated as a lossless ML representation.

---

## Problems With Some Derived Features

The feature engineering is useful, but not every generated feature represents ground truth.

### Order-book removals are ambiguous

Suppose displayed liquidity changes from:

```text
100 BTC
```

to:

```text
60 BTC
```

It is tempting to interpret this as:

```text
40 BTC canceled
```

But the change may represent:

- canceled orders;
- executed orders;
- missed updates;
- reconnect behavior;
- a new snapshot;
- exchange-specific update semantics.

Without sequence-aware reconstruction, the code cannot always distinguish these events.

The removal and reinforcement features should therefore be treated as proxies.

---

### Open-interest changes do not reveal entry price

Suppose:

```text
BTC price = $60,000
OI change = +500 BTC
```

The system can associate the change with the current price region.

It cannot prove that the new positions were opened at exactly `$60,000`.

Open interest is an aggregate market value.

The feature means:

```text
open interest changed while price was around this level
```

not:

```text
all new positions entered at this level
```

---

### Backward filling can introduce leakage

The prototype sometimes uses:

```python
series.ffill().bfill()
```

Forward filling uses previously available information.

Backward filling can use a later observation to fill an earlier missing value.

Example:

```text
12:00:01    missing
12:00:02    missing
12:00:03    value = 100
```

Backward filling can produce:

```text
12:00:01    100
12:00:02    100
12:00:03    100
```

The value was not known at `12:00:01`.

This may be acceptable for a visual heatmap.

It is dangerous when the resulting frame is used directly for ML training.

---

### Cross-exchange numbers require validation

A unified number can look correct while combining incompatible inputs.

Every instrument should ideally include metadata such as:

```text
exchange
symbol
instrument type
base asset
quote asset
contract multiplier
linear or inverse
quantity unit
price unit
```

The conversion logic in this repository is useful, but it should not be accepted blindly.

---

## The pandas Performance Limitation

At the time I built this project, I did not fully understand that most of the pandas-heavy operations used by this pipeline are effectively single-core.

pandas does not automatically distribute a DataFrame workload across all CPU cores.

Some underlying NumPy, BLAS, compression, or native-library operations may use additional threads, but common operations used heavily here remain mostly single-threaded from the application's perspective.

These include:

- row-by-row DataFrame mutation;
- dynamic column creation;
- repeated DataFrame copying;
- many joins and merges;
- sorting columns;
- Python-level orchestration;
- repeated object allocation.

The pipeline performs operations such as:

```text
receive event
    ↓
update Python object
    ↓
mutate DataFrame row
    ↓
possibly create a new column
    ↓
sort columns
    ↓
copy snapshot
    ↓
merge with other DataFrames
```

That works for:

- experimentation;
- recorded examples;
- small symbol sets;
- lower-frequency streams.

It scales poorly for:

```text
many exchanges
×
many symbols
×
many stream types
×
continuous high-frequency data
```

---

### Python was not the mistake

Python was the right tool for discovering the feature model.

It allowed rapid experimentation with:

- market representations;
- price buckets;
- new features;
- exchange adapters;
- cross-exchange aggregation.

The mistake was assuming that the same pandas architecture should become the high-throughput ingestion system.

A better division is:

```text
Rust
    → ingestion
    → concurrency
    → normalization
    → persistence
    → runtime services

Python
    → research
    → feature engineering
    → statistics
    → notebooks
    → machine learning
```

---

## Why the Original Architecture Failed

The project did not fail because the features were uninteresting.

It failed because the system aggregated data before preserving the original events.

The original design effectively did:

```text
raw market event
        ↓
feature aggregation
        ↓
original event discarded
```

That creates several problems:

- feature bugs cannot be corrected from the original data;
- alternative bars cannot be generated;
- event ordering is lost;
- source timestamps may be lost;
- exchange sequence data may be lost;
- exact replay becomes impossible;
- new research requires recollecting data.

The better design is:

```text
raw market event
        ↓
normalized persistent event
        ↓
replayable feature pipeline
        ↓
derived feature versions
```

---

### Features should not be canonical data

A feature is a transformation.

It is not the original dataset.

The correct relationship is:

```text
raw events
    ↓
feature version 1

raw events
    ↓
feature version 2

raw events
    ↓
feature version 3
```

All feature versions should be reproducible from the same underlying event history.

---

### The networking layer was incomplete

The repository processes messages, but it does not fully guarantee:

- message completeness;
- correct event sequence;
- reconnection recovery;
- snapshot recovery;
- queue limits;
- event persistence;
- reliable long-running operation.

That limits the reliability of any resulting historical dataset.

---

### Errors could disappear silently

The code contains broad patterns such as:

```python
try:
    ...
except:
    return
```

This was convenient while handling inconsistent experimental payloads.

In a production service, parsing failures should generate:

- structured logs;
- error metrics;
- rejected-message counters;
- payload samples;
- exchange and symbol metadata.

Silent errors make market-data quality difficult to prove.

---

## What I Learned

### The feature layer was worth keeping

The project produced useful abstractions and feature ideas.

The correct conclusion was not:

```text
delete all feature logic
```

It was:

```text
move feature logic behind a reliable raw-data layer
```

---

### Raw events should be canonical

A stored event should ideally contain:

```text
exchange
symbol
stream type
exchange event time
receive time
processing time
sequence number
normalized fields
raw payload
```

Everything else should be derived from that event history.

---

### Features should be replayable

The same raw history should support:

- fixed-time bars;
- tick bars;
- volume bars;
- imbalance bars;
- alternative price buckets;
- corrected formulas;
- new feature versions;
- different ML datasets.

---

### Ingestion and research should be separate

The cleaner architecture is:

```text
Exchange
    ↓
streaming ingestion service
    ↓
normalized raw events
    ↓
persistent database
    ↓
replayable research pipeline
    ↓
feature generation
    ↓
ML model
```

The ingestion service should not need to know which exact features a future model will use.

---

### Event time matters

A reliable event should distinguish:

```text
exchange_event_time
receive_time
processing_time
```

These timestamps describe different parts of the data path.

They should not be treated as interchangeable.

---

### Sequence validation matters

An order book can look valid while already being wrong.

Without sequence validation:

```text
order book contains data
```

does not guarantee:

```text
order book represents the exchange correctly
```

---

### Backpressure matters

If:

```text
input rate > processing rate
```

then:

```text
queue size grows continuously
```

unless queues are bounded and the service has an explicit backpressure policy.

This became one of the reasons I started studying distributed and concurrent systems.

---

### Observability is part of correctness

A process being online does not mean it is keeping up.

A proper market-data service should expose:

```text
message rate
processing latency
queue size
database latency
dropped events
parse failures
reconnect count
active streams
last event timestamp
```

That is why Prometheus and Grafana became part of the successor project.

---

## Better Architecture

The improved architecture became:

```text
Exchange WebSocket / HTTP
            │
            ▼
       ingestion worker
            │
            ▼
      normalized event
            │
            ▼
 PostgreSQL / TimescaleDB
            │
            ▼
      replayable pipeline
            │
            ▼
      feature generation
            │
            ▼
        ML research
```

The key rule is:

```text
store first
aggregate later
```

not:

```text
aggregate first
lose the original information
```

The feature-engineering code from this project can still sit in the later part of that architecture.

For example:

```text
TimescaleDB events
        ↓
Python replay process
        ↓
fixedbar-failure flow modules
        ↓
feature dictionary
        ↓
research notebook or model
```

This is a much better use of the existing code.

---

## Successor Project

The practical successor is:

```text
mini-fintickstreams
```

That project is written in Rust and focuses on:

```text
collect
normalize
persist
observe
replay
```

Its architecture includes work around:

- Rust;
- Tokio;
- asynchronous exchange workers;
- typed market events;
- PostgreSQL;
- TimescaleDB;
- batching;
- backpressure;
- rate limiting;
- reconnect handling;
- Prometheus;
- Grafana;
- Redis Streams;
- Kubernetes.

Conceptually:

```text
Exchange
    ↓
Rust / Tokio
    ↓
typed normalized event
    ↓
TimescaleDB
    ↓
Python research and feature generation
```

This Python project was the experiment that exposed the need for that infrastructure.

---

## Repository Structure

### Exchange normalization

```text
StreamEngineBase/lookups.py
```

Contains:

- exchange-specific payload parsing;
- timestamp normalization;
- trade-side normalization;
- contract unit conversion;
- open-interest conversion;
- order-book conversion;
- option expiration handling.

---

### Stream processors

```text
StreamEngineBase/flow.py
```

Contains:

- `booksflow`;
- `tradesflow`;
- `oiFundingflow`;
- `liquidationsflow`;
- `oiflowOption`;
- `indicatorflow`.

These are the most directly reusable plugin components.

---

### Cross-exchange synthesis

```text
StreamEngineBase/synthesis.py
```

Contains logic for combining:

- order-book snapshots;
- trade snapshots;
- OI and funding;
- liquidations;
- positioning indicators;
- experimental order-book adjustments.

---

### Bitcoin spot and perpetual configuration

```text
StreamEngine/spotperp/btc.py
```

Defines exchange-specific Bitcoin flows and routes payloads into the correct processors.

---

### Bitcoin options configuration

```text
StreamEngine/option/btc.py
```

Defines the options-processing setup.

---

### Main combined processor

```text
StreamEngine/synthHub.py
```

Creates the exchange flows, performs synthesis, and exposes the flattened feature dictionary.

---

### Example notebook

```text
examples/btcSynth.ipynb
```

Used for:

- loading sample data;
- processing recorded payloads;
- inspecting DataFrames;
- checking generated features;
- manually validating output.

---

### Recorded sample data

```text
examples/data
```

Contains recorded payloads used during development.

These examples are useful for understanding the input structure expected by the adapters.

---

## Current Status

The code is intentionally left unchanged.

I do not plan to rewrite this implementation into a production market-data service because that would require replacing the underlying architecture.

The repository remains useful for two reasons.

### It contains reusable feature-engineering code

The processors can still be imported into another project for:

- market summaries;
- heatmaps;
- feature experiments;
- cross-exchange normalization;
- offline research;
- replay processing;
- dashboard data.

---

### It documents an important engineering failure

The project showed that:

- fixed-time bars contain inconsistent information;
- this can cause serious ML overfitting;
- some market features are proxies rather than ground truth;
- pandas-heavy streaming is effectively limited by single-core execution in many critical paths;
- raw data must be preserved before aggregation;
- feature generation should be replayable;
- ingestion and research should be separate systems.

The project failed at its original goal of becoming the final market-data and ML pipeline.

The feature-engineering work itself did not fail.

It became a reusable research layer that belongs after a more reliable ingestion and storage system.
