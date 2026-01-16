## ⚠️ WARNING

This is a **deprecated experimental project**.

It does not survive real-world ML training requirements and should not be used for production, research, or strategy development.

Fixed time horizons proved fundamentally incompatible with robust ML-based trading systems. Event-driven pipelines operating on raw ticks are significantly more practical.

This repository is kept for reference only and is **effectively useless** for serious applications.



# moonStreamProcess Library

## Overview
The moonStreamProcess Library is a powerhouse designed to seamlessly process and aggregate real-time financial data streams. It ingests critical market data like order books, trades, liquidations, open interest, and positions, transforming them into actionable insights and a unified view of the market.

## Purpouse
While moonStreamProcess does not directly generate financial dashboards, its core functionality revolves around aggregating data from various exchanges and providing meticulous one-minute summaries in the form of heatmaps. These heatmaps offer a detailed visualization of market dynamics. moonStreamProcess's role is pivotal in efficiently aggregating and synthesizing this heatmap data, making it an indispensable tool for traders and analysts seeking real-time insights into market trends. Additionally, it proves invaluable for storing data in non-relational databases, supporting the development of machine learning algorithms, and facilitating the execution of complex analyses

## Core Functionality: Stream Processing Redefined
moonStreamProcess excels at real-time financial data processing. Here's how it breaks down:
* Modular Design: The library utilizes specialized flow modules as building blocks. Each module meticulously handles a specific data type within the financial stream.
* Stream Aggregation & Synthesis: After the flow modules refine the data, synthesis modules take over. These modules not only consolidate the processed information but also synthesize it into actionable insights, providing a comprehensive market picture.

## Supported Instruments

moonStreamProcess currently supports only Bitcoin. However, creating processors for ETH or any other altcoin shouldn't take much time. The crucial steps are only to change lookups for the data based on the websocket or API you are using and remove unneeded instruments. To get started, follow the instructions provided in the section on making your own streams.

## Supported Exchanges

moonStreamProcess currently supports the following exchanges for spot/perpetual instruments:

- Binance
- OKX
- Bybit
- KuCoin
- Deribit
- MXC
- Gate.io
- BingX
- Bitget
- Coinbase
- HTX

Additionally, for option instruments, moonStreamProcess supports:

- Bybit
- Deribit
- OKX

## Important Notes

### Deribit Open Interest

**Caution:** Deribit's Open Interest (OI) values of BTC-PERP may exhibit unusual and abrupt jumps. This behavior could potentially be attributed to API errors. Please use TradeStreamEngine with Deribit on your own risk, and exercise caution when interpreting OI data.

### MEXC API Support

**Attention:** MEXC's API support is reported to have challenges. If you intend to use MEXC data, it is advisable to verify the accuracy of the information. Ensure that the data is not manipulated, as reported Open Interest (OI) values fetched via the API may differ significantly from those reported by external sources such as CoinMarketCap.

**Additional Note on MEXC Perpetual BTCUSDT Books**

**Caution:** The order books for MEXC perpetual BTCUSDT contracts may display unusually high values. This anomaly could be a result of the unregulated nature of the market. Exercise discretion and be aware of potential discrepancies when analyzing MEXC perpetual BTCUSDT books.


# Table of Contents

- [Installation](#installation)
- [Usage](#usage)
- [Data](#data)
- [Interpretation](#interpretation)
- [Making Streams](#modules)
  - [Flow Modules](#flow-modules)
  - [Lookups Modules](#lookups-modules)
  - [Synthesis Modules](#synthesis-modules)
  - [Assembler Modules](#assembler-modules)
- [Examples](#examples)
- [Contacts](#contacts)
- [Licence and Contributing](#license-and-contributing)
- [Support Development](#support-development)

# Installation

To use the moonStreamProcess and its modules, follow these installation steps:

1. Clone the repository to your virtual environment library directory:
   ```bash
   git clone https://github.com/badcoder-cloud/moonStreamProcess
2. Navigate to the project directory:
   ```bash
   cd .../moonStreamProcess
3. Install the required dependencies:
   ```bash
   pip install -r requirements.txt

# Usage

To use the Synthesis Hub with Bitcoin data, follow these steps:
For Python code:

```python
import numpy as np
from moonStreamProcess.StreamEngine.synthHub import btcSynth

level_size = 20
prage = np.array([0.0, 1.0, 2.0, 5.0, 10.0])
expiration_ranges = np.array([0.0, 1.0, 2.0, 5.0, 10.0])
exchanges_spot_perp = ["binance", "okx", "bybit", "bitget", "bingx", "kucoin", "deribit", "coinbase", "htx", "gateio", "mexc"]
exchanges_option = ["bybit", "okx", "deribit"]
abs_path = "" # Absolut path to the directory containing json files. Read method docs

btcDataProcessor = btcSynth(level_size, prage, expiration_ranges, exchanges_spot_perp=exchanges_spot_perp, exchanges_option=exchanges_option)
btcDataProcessor.input_from_json(abs_path) # For testing purpouses to load all of the data at once.
                                           # You may want to fork the code and input the data from unrelational database
btcDataProcessor.merge()
btcDataProcessor.display_full_data()

data = ""
btcDataProcessor.add_okx_perp_btcusd_oi(data) # Will input okx oi. Check for all methods
```

**Level Size** 

Imagine you're sorting coins. Instead of having a giant pile, this setting groups similar prices together. The "level size" determines the size of these groups. If it's set to 20, then order book and trade, ois and liquidations data will be grouped into buckets that represent price ranges of $20 each.

**Price Ranges for Options (pranges)**

This setting focuses on option contracts, which give you the right to buy or sell Bitcoin at a specific price by a certain date (expiration). pranges helps categorize options based on their strike price, which is the price at which you can exercise the option to buy or sell.

The example np.array([0.0, 1.0, 2.0, 5.0, 10.0]) creates buckets for options relative to the current Bitcoin price:

10.0: This bucket holds options with a strike price that matches the strikes 10% above the current Bitcoin price (or below if the value is negative). For example if the current price is $10,000, the bucket will encomapass options with strikes above $11,000
0.0:1.0, 1.0:2.0, and so on: These buckets hold options with strike prices that fall within the specified range relative to the current price. For example, if the current price is $10,000, a 1.0:2.0 bucket might encompass options with strike prices between $10,200 and $10,400. It's important to note that negative values, such as -5 and -10, represent options with strike prices below the current price.

**Expiration Time Ranges (expiration_ranges)**

Similar to price ranges, this setting groups options based on how much time is left until they expire. This helps analyze how option prices change as the expiration date approaches. (Doesn't contain negative ranges)

**In Short**

These settings organize complex Bitcoin market data into simpler buckets, making it easier to understand price trends, option behavior based on strike price and expiration time, and ultimately gain valuable insights into the market.

# Data
This btcSynth contains dynamically generated data that serves various purposes within the project. It can be accessed with following methods:

``` Python
btcDataProcessor.data.get("spot_books") # to get spot books data
btcDataProcessor.data                   # to access full data
```
The data can be categorized into the Spot, Perpetual and Option.

### Spot Data
| key | clarification |
| -------- | -------- |
| <span style="color:green">spot_books</span> | the dictionary of absolute amount of books by aggregated level. |
| <span style="color:blue">timestamp</span> | Timestamp of the data. |
| spot_buyVol | the total amount of market buys at the current timestamp. |
| spot_sellVol | the total amount of market sells at the current timestamp. |
| spot_open | the price at the beginning of the 1-minute interval. |
| spot_close | the price at the end of the 1-minute interval. |
| spot_low | the lowest price within a 1-minute interval. |
| spot_high | the highest price within a 1-minute interval. |
| spot_Vola | the volatility of the price within a 1-minute interval, calculated using standard deviation. |
| spot_VolProfile | the dictionary of market trades (sell + buy) by aggregated level. |
| spot_buyVolProfile | the dictionary of market buys by aggregated level. |
| spot_sellVolProfile | the dictionary of market sells by aggregated level. |
| spot_numberBuyTrades | number of buy trades at the current timestamp. |
| spot_numberSellTrades | number of sell trades at the current timestamp. |
| <span style="color:yellow">spot_orderedBuyTrades</span> | the list of ordered buy trades. (may be usefull for entropy) |
| spot_orderedSellTrades | the list of ordered sell trades. |
| spot_voids | the dictionary of canceled books by aggregated level. (sums all canceled amounts) |
| spot_reinforces | the dictionary of reinforced books by aggregated level. (sums all reinforced amounts) |
| spot_totalVoids | the total amount of canceled orders at the current timestamp. |
| spot_totalReinforces | the total amount of reinforced orders at the current timestamp. |
| spot_voidsDuration | the dictionary of duration of canceled books by aggregated level in seconds. If someone is engaging in spoofing, you can determine the duration for which they place an order before canceling it by measuring the time interval between order placement and cancellation. |
| spot_voidsDurationVola | the dictionary of volatility of spot_voidsDuration. |
| spot_reinforcesDuration | the dictionary of duration of reinforced books by aggregated level in seconds.|
| spot_reinforcesDurationVola | the dictionary of volatility of spot_reinforcesDuration. |

### Perpetual Data
| key | clarification |
| -------- | -------- |
| perp_books | the dictionary of absolute amount of books by aggregated level |
| perp_buyVol | the total amount of market buys at the current timestamp |
| perp_sellVol | the total amount of market sells at the current timestamp |
| perp_open | the price at the beginning of the 1-minute interval |
| perp_close | the price at the end of the 1-minute interval |
| perp_low | the lowest price within a 1-minute interval |
| perp_high | the highest price within a 1-minute interval |
| perp_Vola | the volatility of the price within a 1-minute interval, calculated using standard deviation |
| perp_VolProfile | the dictionary of market trades (sell + buy) by aggregated level |
| perp_buyVolProfile | the dictionary of market buys by aggregated level |
| perp_sellVolProfile | the dictionary of market sells by aggregated level |
| perp_numberBuyTrades | number of buy trades at the current timestamp |
| perp_numberSellTrades | number of sell trades at the current timestamp |
| perp_orderedBuyTrades | the list of ordered buy trades. (may be usefull for entropy) |
| perp_orderedSellTrades | the list of ordered sell trades |
| perp_voids | the dictionary of canceled books by aggregated level. (sums all canceled amounts) |
| perp_reinforces | the dictionary of reinforced books by aggregated level. (sums all reinforced amounts) |
| perp_totalVoids | the total amount of canceled orders at the current timestamp |
| perp_totalReinforces | the total amount of reinforced orders at the current timestamp. |
| perp_voidsDuration | the dictionary of duration of canceled books by aggregated level in seconds. If someone is engaging in spoofing, you can determine the duration for which they place an order before canceling it by measuring the time interval between order placement and cancellation. |
| perp_voidsDurationVola | the dictionary of volatility of spot_voidsDuration. |
| perp_reinforcesDuration | the dictionary of duration of reinforced books by aggregated level in seconds. |
| perp_reinforcesDurationVola | the dictionary of volatility of spot_reinforcesDuration. |
| perp_weighted_funding | weighted funding rate at the current timestamp. |
| perp_total_oi | the total open interest at the current timestamp. |
| perp_oi_increases | the dictionary of increases of open interest by aggregated level. |
| perp_oi_increases_Vola | the dictionary of volatilities of increases of open interest by aggregated level. |
| perp_oi_decreases | the dictionary of decreases of open interest by aggregated level.  |
| perp_oi_decreases_Vola | the dictionary of volatilities of decreases of open interest by aggregated level. |
| perp_oi_total | the dictionary of oi changes (buys-sells) by aggregated level. |
| perp_oi_total_Vola | the dictionary of volatilities of perp_oi_total. | 
| perp_oi_change | the change of open inerest at the current timestamp. |
| perp_oi_Vola | the volatility of open interest at the current timestamp. |
| perp_orderedOIChanges | the list of ordered OI changes. |
| perp_OIs_per_instrument | the dictionary of open interest per instrument at the current timestamp. |
| perp_fundings_per_instrument | the dictionary of funding rates per instrument at the current timestamp. |
| perp_liquidations_longsTotal | the total amount of long liquidations at the current timestamp. |
| perp_liquidations_longs | the dictionary of liquidations by aggregated level, longs. |
| perp_liquidations_shortsTotal | the total amount of short liquidations at the current timestamp. |
| perp_liquidations_shorts | the dictionary of liquidations by aggregated level, shorts. |
| perp_TTA_ratio | weighted Long/Short ration of Top Traders Accounts |
| perp_TTP_ratio | weighted Long/Short ration of Top Traders Positions |
| perp_GTA_ratio | weighted Long/Short ration of Global Traders Positions |


### Option Data (Keys depend on the pranges and expiration_windows)
| key | clarification |
| -------- | -------- |
| oi_option_puts_0 | the dictionary of open interests of put options that expire today |
| oi_option_puts_0_5 | the dictionary of open interests of put options that expire in 5 days |
| oi_option_puts_5_10 | the dictionary of open interests of put options that expire between 5 to 10 days |
| oi_option_calls_0 | the dictionary of open interests of call options that expire today |
| oi_option_calls_0_5 | the dictionary of open interests of call options that expire in 5 days |
| oi_option_calls_5_10 | the dictionary of open interests of call options that expire between 5 to 10 days |


# Interpretation

Methematical interpretation of the highlighted features:

#### Books - absolute quantity of order books for certain price levels at certain timestamp.
- $P_i$: The price level at index $i$.
- $Q_i(t)$: The absolute quantity of limit orders at price level \( P_i \) at timestamp \( t \).
- $B_t$ = ${(P_1, Q_1(t)), (P_2, Q_2(t)), ... , (P_n, Q_n(t))}$ : Books at different levels
#### Trades - absolute quantity of market trades for certain price levels at certain timestamp.
- $P_i$: The price level at index $i$.
- $Q_i(t)$: The absolute quantity of market orders at price level \( P_i \) at timestamp \( t \).
- $T_t$ = ${(P_1, Q_1(t)), (P_2, Q_2(t)), ... , (P_n, Q_n(t))}$ : Books at different levels
#### Canceled limit orders - estimated quantity of canceled limit orders for certain price levels at certain timestamp.
- $D_t$ = ${ (B_t1 + T_t1 ) - ( B_t0 + T_t0 ) }$   -Difference of total placed prders between 2 consegutive timestamps
- $CB_t$ = ${CB_t | CB_t = (D_t - D_t-1) ⋅ [D_t - D_t-1 > 0], ∀_i, 0 ≤ i < n}$ - Total closed orders over a single timestamp 
#### Reforced limit orders - estimated quantity of reforced limit orders for certain price levels at certain timestamp.
- $D_t$ = ${ (B_t + T_t ) - ( B_t-1 + T_t-1 ) }$   -Difference of total placed prders between 2 consegutive timestamps
- $CB_t$ = ${CB_t | CB_t = (D_t - D_t-1) ⋅ [D_t - D_t-1 < 0], ∀_i, 0 ≤ i < n}$ - Total closed orders over a single timestamp 

The same logic applies to open interest and liquidations.
All volatility metrics were calculated using standard deviation.

# Making Streams

StreamEngineBase has a set of modules to cater to your streaming needs, making the process straightforward. 

## Lookups Modules

The lookups module is crafted to convert various data into the required format and units. Depending on the API or websocket you're utilizing, you'll have to adjust the lookup according to your specific use case. Exercise caution, as different instruments use varying units of measure. For instance, while Binance measures order books in Bitcoin, Deribit uses dollars. In the provided lookups, note that the 'bingx btcusdt perp books' is intended for API calls, serving the same purpose as 'gateio btcusdt perp' for both books and trades. This is due to technical issues with the API. Also, look closely for the formats to input into flow objects.
``` Python
from lookups import btc as btc_lookups_btc
from lookups import unit_conversion_btc
lookups_btc = btc_lookups_btc(unit_conversion_btc)

# or

unit_conversion_btc = {
    "binance_perp_btcusd" : lambda value, btc_price: value * 100 / btc_price,  
    "bybit_perp_btcusd" :   lambda value, btc_price: value  / btc_price,      
    "bybit_perp_btcusdt" :   lambda value, btc_price: value  / btc_price,
    "bingx_perp_btcusdt_depth" : lambda size: size / 10000,  
    "bingx_perp_btcusdt_OI" : lambda openInterest, price : openInterest / price,    
    "deribit_perp_btcusd" : lambda size, price : size / price,
    # ...
}
```
Most of the lookups will be compatible with what you need. Be cautious with altcoins USD-C margined contracts, as these have different units for BTC and altcoins.

## Flow Modules

Flow modules are designed to handle books, trades, open interest, liquidations and positions of spot/perpetual instruments and open interest of option instruments.

An example of using flow module:
```Python
from TradeStreamBase import flow

books = flow.booksflow('binance', 'btc_usdt', 'perpetual', level_size, lookups_btc.binance_depth_lookup, book_ceil_thresh)
trades = flow.tradesflow('binance', 'btc_usdt', 'perpetual', level_size, lookups_btc.binance_trades_lookup, book_ceil_thresh)
oi = flow.oiFundingflow('binance', 'btc_usdt', 'perpetual', level_size, lookups_btc.binance_OI_lookup, book_ceil_thresh)
liquidations = flow.booksflow('binance', 'btc_usdt', 'perpetual', level_size, lookups_btc.binance_depth_lookup, book_ceil_thresh)
positions = flow.booksflow('binance', 'btc_usdt', 'perpetual', level_size, lookups_btc.binance_liquidations_lookup, book_ceil_thresh)
oi_option = flow.indicatorflow('binance', 'btc_usdt', 'perpetual', "TTA", lookups_btc.binance_GTA_TTA_TTP_lookup)

data = "some data"
books.update_books(data)
trades.input_trades(data)
oi.input_oi(data)
oi.input_funding(data)
liquidations.input_liquidations(data)
oi_option.input_binance_gta_tta_ttp(data) # this is different depending on exchange
oi_option.input_oi(data)
```

## Synthesis Modules

Synthesis module play a crucial role in the aggregation of data from various exchanges. These modules facilitate the consolidation of information, allowing for a unified and comprehensive view of data sourced from diverse trading platforms. By combining data from multiple exchanges, synthesis module contribute to a more holistic understanding of market trends, pricing, and other relevant factors. Not only it aggregates the data of liquidations, books, trades, open interest and positions, but it creates 2 new features, voided books and reinforced books.
An example of using synthesis module:
``` Python
axis_spot = {
    "binanceusdt" : bbt, # these are flow objects
    "binancefdusd" : bbf,
    "okxusdt" : obt,
    "bybitusdt" : bybt,
    "coinbaseusd" : cu,
}

spotAggDepth = booksmerger("btcusd", "spot", axis_spot)
spotAggDepth.merge_snapshots()

adjustments = synthesis.booksadjustments("btc", "spot", spotAggDepth, spotAggTrades)

# spotAggTrades not specified but follows the same logic
# Inspect the module for more details
```

## Assembler Modules

Kindly examine the logic within the moonStreamProcess carefully and follow the flow. You won't need to fork the code entirely; instead, kindly replicate it and omit any unnecessary components. Should you encounter any bugs or difficulties, please feel free to reach out to me on Telegram. I would be more than happy to assist you in resolving them and guide you in creating your own streams.

Follow this logic:

* Create the spot-perp coin object with your needed exchanges.
* Create the option object if needed.
* Create a new class in the Synth Hub, or create a new module.

# Examples
Some examples can be found in the "examples" folder. Please make sure to examine the charts and look for any discrepancies. If you find outliers in the data, then you may have made a mistake with the lookups.


# Contacts
Feel free to reach out if you have any questions, feedback, or just want to say hello! We value your input and are here to help.

- **Email:**
  - Address: pvtoa@iscte-iul.pt

- **GitHub Issues:**
  - Address: https://github.com/badcoder-cloud/moonStreamProcess/issues

- **Telegram:**
  - Address: https://t.me/+OeAAF_1FpU01Mjg8

# Licence and Contributing

This project is open-source and free to fork. Feel free to use, modify, and distribute the code as you see fit. If you found it helpful, consider including my GitHub username (badcoder-cloud) in the forking process. I am extremely open to contributions, whether it be in the form of a new feature, improved infrastructure, or better documentation.

# Support Development

If you find this project helpful and would like to support its development, consider making a donation. Your contribution is greatly appreciated!

- **Bitcoin (BTC):**
  - Address (Bitcoin Network): 31tRbakC3ebHsqWsgbEi3VHj9M98Eesmm1

- **Tether (USDT):**
  - Address (Polygon Network): 0xc2a77f3f3b0015f5f785b220451853c39a886894 

- **Ethereum (ETH):** 
  - Address (Etherium Network): 0xb15d751c19ceacccfb8d6e6c9a06217e0b692856

*Exciting Updates Ahead!*

Stay tuned for streaming wrappers, machine learning clustering pipelines and trading bots backed by reinforcement learning are on the horizon, bringing new possibilities and advancements to the project.
