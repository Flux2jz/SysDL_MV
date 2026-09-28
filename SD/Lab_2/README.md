<!--
markdownlint-configure-file
{"MD013": {"tables": false, "code_blocks": false}}
-->

# Lab 2: Quantify the Dashboard Reads

---

## 1. Quality requirements

Every requirement has the form measure + target + operating condition.

### 1.1 Read latency

**REQ-LAT-1.** At the steady and market-open workloads in section 3, for each
scale, the client-observed latency of each of the six reads must be p95 <= 2,000
ms.

- Start: the Client sends the read request.
- End: the User can see a correct result. That is either data with its provider
  time and delay label, or an explicit "unavailable" result where the rules
  below require it.
- Percentile: the client says "most" reads. I read "most" as 95%
  **(assumption)**, so 95 of 100 correct reads finish within 2 seconds.
- At the 2-second limit: a read that is still running at 2 s counts as a latency
  miss and the Client shows a loading indicator. The 5% that may miss the limit
  are still capped. At 5,000 ms the read stops and the User sees "unavailable,
  try again" **(assumption)**. A result that arrives after 2 s is not counted as
  an acceptable completed read.

**REQ-LAT-2.** The client wants prices to "feel immediate". Under the same
conditions the Stock price read must be p50 <= 300 ms and p95 <= 1,000 ms,
client-observed **(assumption: I made the most used read stricter)**.

A fast "unavailable" result can meet REQ-LAT-1 and still fail the availability
and throughput requirements below.

### 1.2 Uptime-style availability

The Dashboard is usable in a given minute when the Overview, Stock price and
Watchlist reads each return an acceptable result, meaning a correct result
within 2 s. An "unavailable" price does not count as usable. I measure this with
one synthetic probe of the three reads every 60 seconds **(assumption)**.
Downtime caused by the Market Data Provider is counted and tagged with its
cause, because the User sees the same failure either way.

The market-hours window is the regular US session, 09:30 to 16:00
America/New_York on trading days. That is 6.5 h = 390 min per trading day
**(assumption)**. Everything else is the rest-of-day window.

For the 30-day window, 30 days = 43,200 min. Trading days in 30 calendar days:
30 x 252 / 365 = 20.7, so I use 21 **(assumption)**.

| Window | Measured minutes in 30 days | Uptime target | Downtime budget in 30 days |
| ------ | --------------------------- | ------------- | -------------------------- |
| Market hours | 21 x 390 = 8,190 | 99.9% (three nines) | 8,190 x 0.001 = 8.19 min = 8 min 11.4 s |
| Rest of the day | 43,200 - 8,190 = 35,010 | 99% (two nines) | 35,010 x 0.01 = 350.1 min = 5 h 50 min 6 s |

Why the two targets are different:

- Users check prices mostly during market hours, when numbers move and a delayed
  price is most noticeable. An outage then costs the most.
- Outside market hours prices don't change, so the last close stays correct. An
  outage mostly blocks history and the Watchlist.
- The rest-of-day window leaves room for planned maintenance and the nightly
  sync.
- One extra nine divides the budget by 10, so 99.99% in market hours would leave
  only about 49 s per 30 days. That is too costly for an informational Dashboard
  with 15-minute-delayed data, so I chose 99.9% instead of the highest target.

The two windows are measured and reported separately for every 30-day period.

### 1.3 Consistency

**REQ-CON-1 (Stock price, bounded staleness).** During market hours, a Stock
price read returns the last accepted price only if its provider time is at most
20 minutes before the read time. That is the expected 15-minute provider delay
plus 5 minutes for synchronization **(assumption)**. The result always shows the
provider time and a "delayed" label. If there is no accepted price, or the age
is over 20 minutes, the read returns "unavailable". It never shows zero and
never says an old price is current.

- From market open until open + 20 min, the last close is shown with the label
  "previous close".
- Outside market hours, the last official close is shown as "market closed,
  close of the last trading day".
- Monotonic reads: once a read shows a price with provider time T for a Stock,
  later reads of that Stock never show a price with an earlier provider time.

**REQ-CON-2 (Watchlist, read-your-writes).** After the Dashboard confirms a
Watchlist add or remove for a User, that same User's next Watchlist read, in any
of their sessions, shows the change. If the change can't be confirmed, the User
sees a failure and the Watchlist stays unchanged. A User can't read or change
another User's Watchlist. That attempt returns "unauthorized" and changes
nothing.

### 1.4 Throughput

**REQ-THR-1.** At each scale, the Dashboard completes at least the market-open
target from section 3 (103, 1,030 and 10,296 read RPS) acceptable reads per
second, while REQ-LAT-1, REQ-LAT-2, REQ-CON-1 and REQ-CON-2 still hold. A late,
wrong or unavailable result is not an acceptable completed read. A Watchlist
change is a write and would need its own target later. This lab only counts the
six reads.

---

## 2. Steady RPS estimates

The formula is RPS = concurrent Users x participating share x actions per User /
seconds. Every action creates one Dashboard request. I don't model
Dashboard-to-provider requests here.

| Read | Formula | 300 Users | 3,000 Users | 30,000 Users |
| ---- | ------- | --------- | ----------- | ------------ |
| Overview | U x 0.70 x 1 / 30 | 300 x 0.70 / 30 = 7 | 3,000 x 0.70 / 30 = 70 | 30,000 x 0.70 / 30 = 700 |
| Filter | U x 0.50 x 3 / 60 | 300 x 0.50 x 3 / 60 = 7.5 | 3,000 x 0.50 x 3 / 60 = 75 | 30,000 x 0.50 x 3 / 60 = 750 |
| Stock price | U x 0.20 x 1 / 1 | 300 x 0.20 = 60 | 3,000 x 0.20 = 600 | 30,000 x 0.20 = 6,000 |
| History | U x 0.20 x 1 / 300 | 300 x 0.20 / 300 = 0.2 | 3,000 x 0.20 / 300 = 2 | 30,000 x 0.20 / 300 = 20 |
| Watchlist | U x 0.60 x 1 / 60 | 300 x 0.60 / 60 = 3 | 3,000 x 0.60 / 60 = 30 | 30,000 x 0.60 / 60 = 300 |
| Search | U x 0.10 x 3 / 60 | 300 x 0.10 x 3 / 60 = 1.5 | 3,000 x 0.10 x 3 / 60 = 15 | 30,000 x 0.10 x 3 / 60 = 150 |
| **Steady total** | sum of rows | 7 + 7.5 + 60 + 0.2 + 3 + 1.5 = **79.2** | 70 + 75 + 600 + 2 + 30 + 15 = **792** | 700 + 750 + 6,000 + 20 + 300 + 150 = **7,920** |

The unrounded steady totals are 79.2, 792 and 7,920 RPS. Each User adds 0.264
RPS, so the total grows in a straight line with the number of Users. Stock price
reads make up 0.20 / 0.264 = 75.8% of steady traffic at every scale.

---

## 3. Market-open RPS estimates

At market open, 30% of Users refresh the Overview once during 10 seconds, and
60% of that group also refresh the Watchlist. I add these two flows on top of
the steady traffic. The steady Overview and Watchlist traffic is counted once,
in the steady row.

| Market-open calculation | 300 Users | 3,000 Users | 30,000 Users |
| ----------------------- | --------- | ----------- | ------------ |
| Steady read traffic | 79.2 | 792 | 7,920 |
| Additional Overview refresh flow (U x 0.30 / 10) | 300 x 0.30 / 10 = 9 | 3,000 x 0.30 / 10 = 90 | 30,000 x 0.30 / 10 = 900 |
| Additional Watchlist refresh flow (U x 0.30 x 0.60 / 10) | 300 x 0.30 x 0.60 / 10 = 5.4 | 3,000 x 0.30 x 0.60 / 10 = 54 | 30,000 x 0.30 x 0.60 / 10 = 540 |
| **Market-open subtotal** | 79.2 + 9 + 5.4 = **93.6** | 792 + 90 + 54 = **936** | 7,920 + 900 + 540 = **9,360** |
| 10% capacity margin | 93.6 x 1.10 = 102.96 | 936 x 1.10 = 1,029.6 | 9,360 x 1.10 = 10,296 |
| **Rounded-up market-open target** | **103 RPS** | **1,030 RPS** | **10,296 RPS** |

Only the last row is rounded. The two market-open flows add about 18% to the
steady traffic at every scale (14.4 / 79.2 = 0.1818), so the open spike is small
next to the steady Stock price load.

10x sensitivity: going from 3,000 to 30,000 Users multiplies the subtotal by
exactly 10 (936 to 9,360). The target goes from 1,030 to 10,296, which is 9.996x
because of rounding. This only holds if User behavior stays the same. It doesn't
prove that behavior, Stock popularity or provider limits stay the same at that
size.

---

## 4. Storage estimates

### 4.1 Research and define a Stock

What "Stock" can mean: a stock is a share of ownership in a corporation. Common
stock gives ownership units with rights such as voting and dividends, and other
classes are usually called preferred stock, which generally has no voting rights
and pays a fixed dividend ([New York State Attorney General, "Stock Investments"](https://ag.ny.gov/resources/individuals/investing-finance/stocks), 
observed 28 Sep 2026). Market-data products also show ETFs, funds, indexes and
currencies next to stocks. For example, Apple Stocks groups "stocks, indexes,
mutual funds, ETFs, currencies" in watchlists ([App Store listing](https://apps.apple.com/us/app/stocks/id1069512882), observed 28 Sep 2026).

My decisions for the first version:

| Decision | Choice | Reason |
| -------- | ------ | ------ |
| Markets | Nasdaq, NYSE and NYSE American (US) | One country and one currency (USD) keeps the first version small. |
| Instrument types | Included: common stock of operating companies, including foreign-incorporated companies whose common shares list directly on these exchanges. Excluded: preferred stock, ADRs, ETFs, funds, REITs and other types. | Fits the Investor's basic need to follow companies. The count source below also excludes REITs, closed-end funds and ETFs, so the scope and the count agree. |
| Inactive or delisted | Kept as inactive reference records with a "no longer trading" state. No new prices and no history kept. | A Watchlist entry must still resolve after a delisting. |
| Count | 5,172 active Stocks (3,657 domestic + 1,515 foreign), plus a 10% inactive allowance: 5,172 x 1.10 = 5,689.2, rounded up to 5,690 reference records | See below. |

Count source (observed 28 Sep 2026): Jay R. Ritter, University of Florida,
[Number of operating companies listed on Nasdaq and the NYSE](https://site.warrington.ufl.edu/ritter/files/number-of-listed-firms-on-US-exchanges.pdf).
It uses CRSP data and includes the NYSE MKT segment (the former American Stock
Exchange). At the end of 2025 it reports 3,657 domestic operating companies and
1,515 foreign firms.

Cross-checks that I did not use for the estimate:

- World Federation of Exchanges data via [Statbase](https://statbase.org/datasets/business-and-investments/listed-companies/)
  (observed 28 Sep 2026): Nasdaq 2,325 and NYSE 1,588 at the end of 2025, so
  3,913. That is lower than 5,172. The counting rules differ (for example NYSE
  American and share classes), so I take it as a low bound.
- [StockVS](https://stockvs.com/en/exchanges/nyse-nasdaq) (observed 28 Sep 2026, prices as of 26 Aug 2026  ): 7,485 "companies" 
  on NYSE and Nasdaq. Its definition is unclear and it seems to include other instrument types, 
  so I take it as a high bound and don't use it.

Limits of this count: it assumes one tradable line per company (companies with
several share classes would raise the count), and that the foreign-firm count
contains no ADRs (not verified). The 10% inactive allowance has no source. The
count is a year-end 2025 value and will change.

### 4.2 Synchronized data

| Data set | Stored fields | Product need | Retention and interval |
| -------- | ------------- | ------------ | ---------------------- |
| Stock reference data | Symbol, name, exchange, currency, country, sector, industry, market-cap band, status | Identity and Search (symbol, name), Filter (exchange, sector, industry, cap band), Stock detail | All 5,690 records, refreshed daily |
| Latest prices | Price, previous close, change, change %, volume, provider time, delay state | Overview, Stock price, Watchlist, delay labels (REQ-CON-1) | One record per active Stock, overwritten in place |
| Price history, daily | Date, open, high, low, close, volume | History read for long ranges | 1 point per Stock per trading day, 5 years = 5 x 252 = 1,260 trading days |
| Price history, intraday | Time, price, volume | History read for day, week and month ranges | 5-minute points during the session: 390 / 5 = 78 per day, for 21 trading days (about 30 calendar days) = 1,638 points per Stock |
| Other selected data | Latest value and daily history for 3 market indexes (S&P 500, Nasdaq Composite, Dow Jones Industrial Average) | The Overview shows the overall market direction | Latest overwritten, daily history 1,260 days |

The time ranges follow common products. Apple Stocks offers interactive charts
for day, week, month and multi-month periods (App Store listing, observed 28 Sep
2026).

Synchronization frequency **(assumption)**: latest prices every 60 s during
market hours, intraday points every 5 minutes, daily records once after close,
and reference data once per day. I don't estimate provider request traffic here.

Not counted as market data: the Watchlist belongs to the User and isn't
synchronized from the provider. As a side estimate, 30,000 Users x 25 entries x
60 bytes = 45,000,000 bytes, about 42.9 MiB.

### 4.3 Bytes per record

I wrote representative records as compact UTF-8 text and measured them. The text
form is only a sizing sample and not a decision about storage format.

| Data set | Sample records (bytes each) | Average | Used (rounded up) |
| -------- | --------------------------- | ------- | ----------------- |
| Reference (5 Stocks: AAPL, JPM, CRWD, GME, AMS) | 188, 202, 210, 191, 221 | 202.4 | 203 |
| Latest price (3 samples) | 153, 147, 151 | 150.3 | 151 |
| Daily history (3 samples) | 102, 92, 102 | 98.7 | 99 |
| Intraday point (3 samples) | 59, 53, 56 | 56.0 | 56 |
| Index latest | 139 | 139 | 139 |
| Index daily history | 88 | 88 | 88 |

Example reference record: `{"id":1,"symbol":"AAPL","name":"Apple
Inc.","exchange":"NASDAQ","currency":"USD","country":"US","sector":"Technology","industry":"Consumer
Electronics","cap_band":"mega","status":"active"}`

### 4.4 Calculation

The formulas are raw storage = record count x average bytes per record, and
history record count = supported Stocks x history points per Stock per day x
retained days.

| Data set | Product decision and retention | Record-count calculation | Bytes per record | Raw storage |
| -------- | ------------------------------ | ------------------------ | ---------------- | ----------- |
| Stock reference data | 5,690 records, all kept | 5,172 x 1.10 = 5,689.2, rounded up to 5,690 | 203 | 1,155,070 B = 1.10 MiB |
| Latest prices | Active Stocks only, overwritten | 5,172 x 1 | 151 | 780,972 B = 0.74 MiB |
| Price history, daily | 1 point/day, 1,260 trading days | 5,172 x 1 x 1,260 = 6,516,720 | 99 | 645,155,280 B = 615.27 MiB |
| Price history, intraday | 78 points/day, 21 trading days | 5,172 x 78 x 21 = 8,471,736 | 56 | 474,417,216 B = 452.44 MiB |
| Other selected data | 3 indexes: latest + 1,260 daily | 3 + 3 x 1,260 = 3,783 | 139 (3 records), 88 (3,780 records) | 333,057 B = 0.32 MiB |
| **Total (initial)** | Full retention backfilled | 15,003,101 records | | **1,121,841,595 B = 1,069.87 MiB = 1.045 GiB** |

Initial, daily and one-year storage:

| Measure | Calculation | Result |
| ------- | ----------- | ------ |
| Initial (after backfill) | total above | 1,121,841,595 B = 1.045 GiB |
| Daily, gross ingest per trading day | daily bars 5,172 x 99 + 3 x 88 = 512,292 B, plus intraday 5,172 x 78 x 56 = 22,591,296 B | 23,103,588 B = 22.03 MiB per trading day |
| Daily, net growth per trading day (steady state) | intraday data expires as fast as it arrives, so only daily bars grow | 512,292 B = 0.49 MiB per trading day |
| One year after launch | initial + 252 trading days x 512,292 B = 1,121,841,595 + 129,097,584 | 1,250,939,179 B = 1.165 GiB (growth 123.1 MiB) |

The one-year figure has no daily-history expiry yet, because retention is 5
years. It leaves out replicas, backups, logs and any provider fields I didn't
list. It is raw data and not a purchase plan.

Sensitivity: daily history is 58% and intraday history 42% of the initial total,
and reference and latest data are under 0.2%. If the Stock count were 10x
bigger, storage would grow about 10x, to roughly 10.4 GiB.

---

## 5. Potential bottlenecks

A high RPS number is a reason to investigate a path, not proof that it is a
bottleneck. Each row below is a hypothesis.

| Quality | Potential bottleneck | Evidence from this lab | Possible effect | What to measure next |
| ------- | -------------------- | ---------------------- | --------------- | -------------------- |
| Latency | Stock price read path (reading the latest price and its provider time) | 75.8% of steady traffic at every scale, which is 6,000 of 7,920 RPS at 30,000 Users. It also has the tightest target (p95 <= 1,000 ms) and has to add the provider time and delay label. | Repeated work for many Users could push p95 over 1,000 ms and slow the whole client experience. | Stock price p50 and p95 under a constant arrival rate at 60, 600 and 6,000 RPS. How requests are spread across Stocks (a few hot Stocks or many). |
| Consistency | Freshness of the latest price between the Market Data Provider and what the User reads | The age limit is 20 min and the expected provider delay is about 15 min, which leaves only 5 min for synchronization. Monotonic reads (REQ-CON-1) and read-your-writes for the Watchlist (REQ-CON-2) also cover state that many reads and writes touch. | A slow or missed sync turns a delayed price into "unavailable". One read could show an older provider time than another. A Watchlist read could miss a change that was just confirmed. | Distribution of provider delay and sync lag (provider time vs time received). Number of reads with age over 20 min. A test that a Watchlist read after a confirmed change never misses it, and that provider times never go backwards. |
| Throughput | Overview and Filter reads over the full Stock set | At market open with 30,000 Users, Overview is 700 steady + 900 burst = 1,600 RPS and Filter is 750 RPS, both working across 5,172 active Stocks. The total target is 10,296 RPS. | Each Overview or Filter read may touch many records, so acceptable throughput could fall below the RPS target and the open burst could overload it. | Acceptable completed reads per second for Overview and Filter at the 900 RPS burst. Work per request (records touched). The mixed-traffic point where p95 or availability first fails. |
| Availability | Market Data Provider (the single external source of all prices) | All prices depend on it. A provider outage of X minutes causes about X - 5 minutes of unavailable prices (20-min age limit minus 15-min delay), and the market-hours budget is only 8.19 min per 30 days. About 13 minutes of provider outage would use the whole budget. | The Dashboard could miss its 99.9% market-hours target because of an external failure, even when its own parts work. | How often and how long the provider is down (MTBF, MTTR). Minutes of downtime by cause (provider or Dashboard) from the probes. How long prices stay acceptable after the provider stops. |

My conclusions in the lecture's form:

- The Stock price path is a potential latency pressure point because it carries
  75.8% of the modelled traffic under a p95 <= 1,000 ms target. The next design
  must serve this path without repeating all the work for every User. We still
  need Stock popularity data and measured p95 under load before choosing a
  mechanism.
- The Market Data Provider is a potential availability pressure point because
  about 13 minutes of outage would use the market-hours budget. The next design
  must keep useful, labelled prices for the allowed age while the provider is
  down. We still need measured provider reliability before choosing a mechanism.

---

## Checklist

- [x] I wrote measurable requirements for all core qualities (latency, availability,
 consistency, throughput).
- [x] I chose and justified uptime targets for both parts of the day.
- [x] I showed the steady and market-open RPS calculations for all three scales.
- [x] I researched and defined the supported Stock scope.
- [x] I estimated initial, daily and one-year raw market-data storage.
- [x] I stated assumptions, units, windows and final rounding.
- [x] I analyzed one potential bottleneck for each core quality.
