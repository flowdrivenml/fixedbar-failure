# Crypto Market Data Aggregation Lab

> [!IMPORTANT]
> This project is intentionally preserved as an **engineering experiment and postmortem**.
>
> The main conclusion was simple:
>
> **Fixed-time bars are a poor default representation for machine-learning models on market data.**
>
> A quiet one-minute bar and an extremely active one-minute bar both become exactly one training sample, even though they contain completely different amounts of information.
>
> That inconsistency makes the statistical meaning of each sample unstable and encourages models to overfit to market regimes, volatility conditions, and arbitrary clock boundaries instead of learning robust relationships.
>
> The project still turned out to be useful because it forced me to understand market-data normalization, feature engineering, streaming architecture, data leakage, replayability, and eventually why the ingestion layer should not have been built around pandas and fixed aggregations in the first place.

---

## Quick Navigation

- [What This Project Was](#what-this-project-was)
- [The Main Problem With Fixed-Time Bars](#the-main-problem-with-fixed-time-bars)
- [What the System Produced](#what-the-system-produced)
- [Architecture](#architecture)
- [Technology Used](#technology-used)
- [Feature Engineering](#feature-engineering)
- [How the Processing Worked](#how-the-processing-worked)
- [What Worked](#what-worked)
- [Why the Project Failed](#why-the-project-failed)
- [Problems With the Features](#problems-with-the-features)
- [Why the ML Representation Was Weak](#why-the-ml-representation-was-weak)
- [Why Python Became the Wrong Tool for the Hot Path](#why-python-became-the-wrong-tool-for-the-hot-path)
- [What I Learned](#what-i-learned)
- [Better Architecture](#better-architecture)
- [Successor Project](#successor-project)
- [Repository Structure](#repository-structure)
- [Current Status](#current-status)
---

## What This Project Was

This project was an experimental Python system for combining crypto market data from multiple exchanges into one normalized representation.

The basic idea was:

```text
many exchanges
    ↓
different native payloads
    ↓
normalize everything
    ↓
aggregate market activity
    ↓
build one unified feature set
```

The system processed data such as:

- order books;
- trades;
- liquidations;
- open interest;
- funding rates;
- trader positioning ratios;
- options open interest.

The target was mainly Bitcoin spot, perpetuals, and options.

Adapters were created for exchanges including:

- Binance
- OKX
- Bybit
- Bitget
- BingX
- KuCoin
- Deribit
- Coinbase
- HTX
- Gate.io
- MEXC

The project was not a trading bot and did not place orders.

It was essentially a **market-data normalization and feature-engineering engine**.

---

## The Main Problem With Fixed-Time Bars

This became the most important finding from the project.

The system aggregated market activity into fixed one-minute windows.

That sounds reasonable at first.

The problem is that **time does not correspond to a fixed amount of market information**.

Consider two different minutes:

```text
Minute A
-------
200 trades
small order-book changes
almost no liquidations
low volatility

Minute B
-------
25,000 trades
massive order-book movement
large liquidations
open-interest shock
extreme volatility
```

Both become:

```text
1 row
```

from the perspective of a machine-learning dataset.

That is a major statistical problem.

### Different rows represent different amounts of information

In a normal tabular ML dataset, there is usually an implicit assumption that observations are at least somewhat comparable.

With fixed-time market bars, that assumption becomes weak.

One row may summarize almost nothing.

Another row may compress thousands of economically meaningful events.

The model is therefore trained on samples with radically different information content.

---

### Fixed-time bars encourage regime overfitting

Market activity changes constantly.

A one-minute representation behaves differently during:

- low volatility;
- high volatility;
- news events;
- liquidations;
- weekends;
- US market hours;
- Asian market hours;
- trending markets;
- mean-reverting markets.

The same features can have completely different distributions across these regimes.

A model can easily learn something like:

```text
high activity + certain volatility structure
    ↓
usually happened during regime X
    ↓
predict regime X behavior
```

instead of learning something genuinely stable about the market.

This creates strong **regime dependency**.

The model performs well while the regime resembles the training data and then collapses when the market changes.

---

### Clock boundaries are arbitrary

Markets do not care that the clock changed from:

```text
12:34:59
```

to:

```text
12:35:00
```

But a fixed-time bar system treats that as a hard information boundary.

A large market event can easily be split across two bars:

```text
event begins
12:34:58

event continues
12:35:00

event ends
12:35:04
```

The feature engine then represents one coherent event as two unrelated training samples.

That makes the representation dependent on an arbitrary clock boundary.

---

### Information density changes constantly

The better mental model is:

```text
clock time != information time
```

For machine learning, it can be much more useful to sample based on actual market activity.

Examples include:

- tick bars;
- volume bars;
- dollar bars;
- imbalance bars;
- volatility-triggered events;
- custom event-driven windows.

These approaches attempt to create samples containing more comparable amounts of market activity.

They are not automatically better for every problem, but they avoid one of the major weaknesses of purely fixed-time sampling.

---

## What the System Produced

The project generated unified market features across spot, perpetual, and options markets.

### Spot features

The spot pipeline produced features such as:

- aggregated order-book depth;
- buy volume;
- sell volume;
- open;
- close;
- high;
- low;
- realized volatility;
- buy trade count;
- sell trade count;
- volume profile;
- buy volume profile;
- sell volume profile.

---

### Perpetual features

The perpetual pipeline included:

- order-book depth;
- trades;
- open interest;
- funding rates;
- liquidations;
- trader long/short ratios;
- open-interest changes;
- open-interest volatility;
- liquidation profiles;
- aggregated cross-exchange positioning.

---

### Options features

Options open interest was grouped by:

- strike distance from current BTC price;
- option side;
- time to expiration.

For example:

```text
puts
    0–1% below price
    1–2% below price
    2–5% below price

calls
    0–1% above price
    1–2% above price
    2–5% above price
```

and then additionally separated by expiration windows.

This produced a rough representation of where option interest was concentrated around the current market price.

---

## Architecture

The original processing pipeline looked roughly like this:

```text
Recorded JSON / external API collector
                    │
                    ▼
            Exchange adapters
     parsing · timestamps · units
                    │
                    ▼
             Flow processors
 books · trades · OI · funding · liquidations
                    │
                    ▼
       per-second temporary DataFrames
                    │
                    ▼
       fixed one-minute aggregation
                    │
                    ▼
         cross-exchange synthesis
                    │
                    ▼
         flattened feature dictionary
```

The architecture was modular, but the main architectural mistake was performing irreversible aggregation too early.

---

## Technology Used

### Python

Python handled the overall system structure.

It was used for:

- exchange adapters;
- flow processors;
- feature generation;
- orchestration;
- aggregation;
- testing with recorded data.

For a research prototype, Python was extremely productive.

---

### NumPy

NumPy was used heavily for:

- price-level calculation;
- bucket assignment;
- unique level detection;
- grouped sums;
- array transformations;
- fast numerical operations.

Example conceptually:

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

This made price-bucket aggregation relatively straightforward.

---

### pandas

pandas handled most temporary time-series structures.

It was used for:

- 60-row second-level matrices;
- merging exchanges;
- forward filling;
- backward filling;
- volatility calculations;
- order-book heatmaps;
- trade profiles;
- open-interest aggregation.

It made experimentation easy, but eventually became one of the main performance problems.

---

### JSON

Recorded exchange responses were stored as JSON and replayed into the processors.

This allowed feature logic to be tested without constantly reconnecting to live exchange APIs.

---

### Jupyter

Jupyter notebooks were used to inspect:

- DataFrames;
- generated features;
- heatmaps;
- strange exchange values;
- normalization errors.

---

## Feature Engineering

A major part of the project was experimenting with derived market features.

The idea was not only to store raw exchange values but to generate representations that might be useful for later statistical modelling.

---

### Price-level aggregation

Order books and trades were grouped into configurable price buckets.

For example:

```text
level_size = 20
```

could create buckets conceptually like:

```text
60000–60020
60020–60040
60040–60060
...
```

This converts a very high-dimensional order book into a smaller structured representation.

Instead of tracking every individual price level, the system could reason about liquidity around approximate price regions.

---

### Volume profiles

Trades were aggregated by price level.

This produced features such as:

```text
buy volume by level
sell volume by level
total volume by level
```

Conceptually:

```text
Price Level    Buy Volume    Sell Volume
60000          12.4          8.1
60020          20.2          18.6
60040          7.4           22.5
```

These profiles were designed to describe where actual market activity occurred.

---

### Order-book heatmaps

The system also aggregated displayed liquidity by price bucket.

This created a simplified representation of:

```text
where bids exist
where asks exist
how much liquidity exists
how liquidity changes
```

The feature could then be compared across exchanges.

---

### Open-interest features

Open-interest data was processed into:

- total OI;
- OI increase;
- OI decrease;
- OI change;
- OI volatility;
- OI changes by price level.

Some of these features later turned out to be conceptually weak, especially the mapping of open-interest changes to observed market price.

More on that in [[#Problems With the Features]].

---

### Funding features

Funding data from multiple perpetual exchanges was normalized and combined.

The idea was to produce an approximate cross-exchange view of perpetual positioning pressure.

---

### Liquidation features

Liquidations were grouped into:

- long liquidations;
- short liquidations;
- total liquidations;
- liquidation volume by price level.

This was one of the cleaner feature groups because liquidation events are directly observable events from exchanges.

---

### Positioning ratios

Where exchanges exposed positioning metrics, the system also collected things like:

- top trader account ratios;
- top trader position ratios;
- global long/short ratios.

These were later merged into broader market features.

---

### Experimental order-book features

The system attempted to estimate:

- removed liquidity;
- reinforced liquidity;
- duration of liquidity;
- volatility of those changes.

The intuition was interesting:

```text
liquidity appears
    ↓
stays briefly
    ↓
disappears
```

might contain information about order-book behavior.

However, the implementation could not reliably distinguish:

```text
cancellation
vs
trade execution
vs
missed message
vs
snapshot replacement
```

so these features were only proxies.

---

## How the Processing Worked

### Exchange normalization

The exchange-specific normalization logic lived mainly in:

```text
StreamEngineBase/lookups.py
```

Different exchanges represent the same concepts differently.

For example, quantity may be expressed in:

```text
BTC
USDT
USD
contracts
inverse contracts
linear contracts
```

The adapters attempted to convert those values into a more comparable representation.

They also handled:

- timestamps;
- trade side;
- options expiration;
- order-book format;
- funding;
- open interest.

---

### Stateful flow processing

The main flow logic lived in:

```text
StreamEngineBase/flow.py
```

The system maintained state separately for:

- books;
- trades;
- OI;
- funding;
- liquidations;
- options;
- positioning indicators.

Each flow accumulated activity over a minute.

A simplified representation looked like:

```text
second 0
second 1
second 2
...
second 59
```

When the second rolled back to zero, the previous minute became a completed snapshot.

---

### Cross-exchange synthesis

The synthesis layer lived mainly in:

```text
StreamEngineBase/synthesis.py
```

It combined data from multiple exchanges.

Conceptually:

```text
Binance
OKX
Bybit
Deribit
...
    ↓
normalize
    ↓
merge
    ↓
global BTC market representation
```

This produced aggregated:

- order books;
- trades;
- OI;
- funding;
- liquidations;
- positioning features.

---

## What Worked

The project was not useless technically.

Several parts were genuinely valuable.

### Exchange adapters were the right abstraction

Separating:

```text
exchange parsing
```

from:

```text
feature logic
```

was a good decision.

It prevented every feature processor from needing to understand Binance, OKX, Bybit, Deribit, and every other exchange independently.

---

### It was a good research environment

Python + NumPy + pandas made it extremely easy to test ideas.

I could quickly try:

```text
different price buckets
different feature formulas
different exchange combinations
different market representations
```

without building a large infrastructure system first.

---

### It exposed the actual hard problems

Initially the interesting part seemed to be feature engineering.

The project eventually showed that the difficult problems were actually:

```text
data correctness
event ordering
timestamp alignment
replayability
unit normalization
backpressure
persistence
monitoring
failure recovery
```

Those became the foundation for the next project.

---

## Why the Project Failed

The biggest architectural mistake was:

> **aggregating before preserving the raw event stream.**

The system treated one-minute features as the main dataset.

That was backwards.

The correct order should have been:

```text
raw event
    ↓
persistent normalized event
    ↓
derived features
```

Instead, the project effectively did:

```text
raw event
    ↓
feature aggregation
    ↓
raw information lost
```

That made the system difficult to validate and impossible to fully replay.

---

## Problems With the Features

Some generated features looked mathematically interesting but did not represent what I originally assumed they represented.

### Order-book removals are ambiguous

Suppose the displayed order book changes from:

```text
100 BTC
```

to:

```text
60 BTC
```

It is tempting to interpret:

```text
40 BTC removed
```

as:

```text
40 BTC canceled
```

But that is not necessarily true.

Possible explanations include:

- cancellation;
- market execution;
- missed WebSocket update;
- reconnect;
- snapshot reset;
- exchange-specific book semantics.

Without exact sequence-aware order-book reconstruction, the system cannot know the true reason.

---

### Open-interest changes do not reveal entry price

Imagine:

```text
BTC price = $60,000
OI change = +500 BTC
```

It is tempting to map:

```text
+500 BTC
```

to the `$60,000` price bucket.

But that does not mean those positions were opened at exactly `$60,000`.

Open interest is an aggregate market value.

The system knows:

```text
OI changed around this time
```

but not:

```text
these positions entered at this price
```

So these features were useful visually but dangerous if interpreted literally.

---

### Forward and backward filling can distort market state

The prototype sometimes used:

```python
series.ffill().bfill()
```

for incomplete frames.

Forward filling uses previously known information.

Backward filling uses a future observation to fill an earlier missing value.

For charts this can be convenient.

For ML training, this can create look-ahead leakage.

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

But the value at `12:00:03` was not actually known at `12:00:01`.

That is exactly the kind of subtle leakage that can make a model look much better in backtests than it really is.

---

## Why the ML Representation Was Weak

This was the core reason the project stopped making sense as a serious ML data source.

### Samples were not statistically comparable

One bar may represent:

```text
50 trades
```

while another represents:

```text
50,000 trades
```

Yet the model receives both as:

```text
one observation
```

The amount of underlying information varies dramatically.

---

### Market regimes dominate the feature distribution

Volatility and activity strongly affect almost every generated feature.

For example:

```text
trade volume
book movement
OI changes
liquidations
volatility
number of events
```

all expand massively during active regimes.

A model may therefore learn:

```text
what volatility regime am I currently in?
```

rather than:

```text
what market structure predicts future behavior?
```

That can create excellent-looking validation results when train and test periods contain similar regimes.

It then performs poorly when the market changes.

---

### Fixed-time bars hide event structure

Suppose the market experiences:

```text
liquidation
    ↓
order-book collapse
    ↓
large market sell
    ↓
open-interest drop
    ↓
price rebound
```

That sequence matters.

A one-minute aggregation may reduce it to:

```text
sell volume = X
liquidations = Y
OI change = Z
close = P
```

The event ordering disappears.

For many predictive problems, the order of events may be more informative than their total amount.

---

### Aggregation destroys optionality

If raw events are stored, later research can generate:

```text
1-second bars
10-second bars
1-minute bars
tick bars
volume bars
imbalance bars
custom events
```

If only one-minute features are stored, none of those alternatives can be reconstructed properly.

That was one of the most important architectural lessons.

---

## Why Python Became the Wrong Tool for the Hot Path

Python itself was not the problem.

Python was excellent for developing the idea.

The problem appeared when the scope became:

```text
many exchanges
×
many symbols
×
many stream types
×
continuous ingestion
```

The system used pandas in ways that are expensive for high-frequency processing:

- creating columns dynamically;
- mutating individual rows;
- copying DataFrames;
- merging frames repeatedly;
- sorting columns;
- allocating many Python objects.

This is fine for exploratory research.

It is not ideal for a high-throughput market-data service.

---

### The problem shifted from analytics to systems engineering

Eventually the important questions became:

```text
How do I process thousands of concurrent streams?

How do I avoid unbounded queues?

How do I detect dropped messages?

How do I replay data after failure?

How do I batch database writes?

How do I monitor processing lag?

How do I reconnect reliably?
```

Those are distributed-systems and streaming-infrastructure questions.

That pushed me toward learning more about:

- asynchronous programming;
- backpressure;
- queue design;
- stream partitioning;
- database batching;
- observability;
- distributed systems.

Eventually that led me to Rust.

---

## What I Learned

The project produced several architectural rules that I now consider much more important than the original feature set.

### Raw data should be canonical

Store the event first.

For example:

```text
exchange
symbol
stream type
exchange timestamp
receive timestamp
sequence number
normalized fields
raw payload
```

Everything else should be derived.

---

### Features should be disposable

A feature is not the dataset.

It is a transformation of the dataset.

That means:

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

should all be possible without recollecting the market.

---

### Research and ingestion should be separate

A cleaner design is:

```text
Exchange
    ↓
Streaming service
    ↓
Normalized raw events
    ↓
Database
    ↓
Research pipeline
    ↓
Features
    ↓
ML model
```

The ingestion system should not care what features the model needs.

---

### Event time matters

Each event ideally needs multiple timestamps:

```text
exchange_event_time
receive_time
processing_time
```

These are not interchangeable.

For high-frequency market data, this difference matters.

---

### Sequence numbers matter

For order books especially, you need to know whether messages are missing.

Without sequence validation:

```text
order book looks valid
```

does not necessarily mean:

```text
order book is correct
```

---

### Backpressure is mandatory

If input rate is greater than processing rate:

```text
input > output
```

then eventually:

```text
queue size → infinity
```

unless the system has bounded queues and explicit policies.

A reliable streaming service needs to control this.

---

### Observability is part of correctness

A process being alive does not mean it is healthy.

A proper service needs metrics around:

```text
processing latency
queue size
database latency
dropped events
reconnect count
message rate
error rate
active streams
```

That is why Prometheus and Grafana became part of the successor architecture.

---

## Better Architecture

The architecture I eventually wanted became:

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

The important difference is:

```text
store first
aggregate later
```

not:

```text
aggregate first
lose the original data
```

---

### Language split

The eventual division of responsibilities became clearer:

```text
Rust
    → ingestion
    → normalization
    → concurrency
    → persistence
    → runtime services

Python
    → analysis
    → feature engineering
    → statistics
    → notebooks
    → machine learning
```

Python did not become useless.

Its role simply moved to the part of the system where it is strongest.

---

## Successor Project

The practical successor is:

`mini-fintickstreams`

That project is written in Rust and follows a very different design philosophy.

Instead of immediately converting market data into fixed bars, it focuses on:

```text
collect
normalize
persist
observe
replay
```

The newer architecture includes work around:

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
research dataset
```

This project was basically the experiment that made that architecture necessary.

---

## Repository Structure

### Exchange normalization

```text
StreamEngineBase/lookups.py
```

Contains exchange-specific parsing, unit conversion, timestamp conversion, and normalization logic.

---

### Stream processing

```text
StreamEngineBase/flow.py
```

Contains the stateful processors for:

- order books;
- trades;
- open interest;
- funding;
- liquidations;
- options;
- market indicators.

---

### Cross-exchange synthesis

```text
StreamEngineBase/synthesis.py
```

Combines data coming from individual exchange flows into unified market representations.

---

### Bitcoin spot and perpetual configuration

```text
StreamEngine/spotperp/btc.py
```

Defines the exchange-specific Bitcoin flows.

---

### Bitcoin options configuration

```text
StreamEngine/option/btc.py
```

Contains the options-specific processing setup.

---

### Main processor

```text
StreamEngine/synthHub.py
```

Connects the individual processors and exposes the flattened combined feature dictionary.

---

### Example notebook

```text
examples/btcSynth.ipynb
```

Used for manually testing and inspecting the generated market features.

---

### Recorded sample data

```text
examples/data
```

Contains example exchange payloads used during development.

---

## Current Status

The code is intentionally left unchanged.

I do not plan to rewrite this implementation into production-quality infrastructure because that would effectively mean rebuilding the whole system around a different architecture.

The repository is more useful as a record of:

- what I originally tried;
- how the feature system worked;
- which assumptions failed;
- why fixed-time bars were problematic for ML;
- where data leakage could appear;
- why raw event storage matters;
- why high-throughput ingestion needs different infrastructure;
- how the project eventually led into distributed computing and Rust.

The project failed at its original goal.

That failure was useful.

It changed the problem from:

```text
How do I create more market features?
```

into:

```text
How do I build a reliable market-data system where features can be regenerated correctly?
```

That turned out to be the much more important question.
