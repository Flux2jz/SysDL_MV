# Lab 3: Draw the Dashboard Boundary

## Starting assumptions

| Item | Value from Lab 2 |
| ---- | ---------------- |
| Scales | 300, 3,000 and 30,000 concurrent Users |
| Market-open load targets | 103, 1,030 and 10,296 read RPS across six reads: Overview, Filter, Stock price, History, Watchlist, Search |
| Largest steady reads at 30,000 Users | Stock price 6,000 RPS (75.8% of traffic), Filter 750, Overview 700 (1,600 at open), Watchlist 300 (840 at open) |
| Latency | Every read p95 <= 2,000 ms. Stock price p50 <= 300 ms and p95 <= 1,000 ms. At 5,000 ms the Web App stops waiting and shows "unavailable, try again" |
| Availability | 99.9% in market hours (8.19 min downtime per 30 days), 99% outside |
| Price freshness rule | A price is accepted if its provider time is at most 20 minutes old (15 min provider delay + 5 min sync). Outside market hours, the last close is shown labelled "market closed". From open until open + 20 min, the last close is shown labelled "previous close". Otherwise the read returns "unavailable". A price is never shown as zero |
| Monotonic reads | A later read of a Stock never shows an earlier provider time than a read before it |
| Watchlist | Read-your-writes for the same User in any session. A User can never read or change another User's Watchlist |
| Sync schedule | Latest prices every 60 s in market hours, intraday points every 5 min, daily bars once after close, reference data once per day |
| Data size | 5,172 active Stocks, 5,690 reference records. Latest prices about 0.74 MiB, reference data about 1.10 MiB, all market data about 1.05 GiB initially |
| Not in Lab 2 | Provider request quota (assumed enough for one latest-price scan per 60 s, to be verified). Identity: the User holds a session token, and the Dashboard Application verifies it on every request |

---

## 1. System Context

```mermaid
flowchart LR
    user["User<br/>Authenticated investor"]
    provider["Market Data Provider<br/>Delayed prices and history"]

    dashboard["Stock Dashboard System"]

    user -->|"Browses and filters Stocks, 
    reads prices and history, 
    manages private Watchlist"| dashboard
    dashboard -->|"Requests delayed prices and 
    history"| provider
    provider -->|"Returns data, or invalid 
    data, rate limit, outage"| dashboard
```

The provider is a separately controlled system. Its failures are shown at the boundary, and its internals are not.

---

## 2. Container

Five parts are the smallest set that meets the Lab 2 targets:

| Container | Responsibility |
| --------- | -------------- |
| Web App | Shows the Dashboard to the User in a browser |
| Dashboard Application | Checks identity, input and permissions, and serves the six reads and Watchlist changes. Runs as several identical stateless instances |
| Market Data Collector | Fetches, validates and saves provider data in the background. Holds the provider credentials |
| Market Data Cache | Holds the latest price of every active Stock in memory, so price reads skip the Database |
| Dashboard Database | Retains Stocks, accepted prices and history, sync progress, and Watchlists. It is the source of truth |

```mermaid
flowchart LR
    user["User"]
    provider["Market Data Provider"]

    subgraph dashboard["Stock Dashboard System"]
        web["Web App"]
        app["Dashboard Application"]
        collector["Market Data Collector"]
        cache[("Market Data Cache<br/>Latest prices in memory")]
        db[("Dashboard Database<br/>Stocks, accepted market data,<br/>sync progress, Watchlists")]

        web -->|"Sends authenticated 
        requests over HTTPS"| app
        app -->|"Reads latest prices, 
        refills an entry only if 
        newer"| cache
        app -->|"Reads reference data, 
        history and prices on cache miss, 
        reads and writes Watchlists"| db
        collector -->|"Saves accepted data and 
        progress atomically, reads 
        progress"| db
        collector -->|"Updates latest prices only if 
        provider time is newer"| cache
    end

    user -->|"Uses Dashboard"| web
    collector -->|"Requests delayed data with 
    server-side credentials"| provider
```

Why this shape:

- **Separate Collector.** Provider quota, retries and outages stay off the User read path. User reads never call the provider.
- **Market Data Cache (new compared with a minimal design).** Stock price reads are 6,000 of 7,920 steady RPS at 30,000 Users and have the tightest target (p95 <= 1,000 ms). The data they need is tiny (0.74 MiB for all latest prices), so keeping it in memory is cheap. At 300 Users (103 RPS) the Database alone would probably be enough, so this part is justified by the largest scale. Verify it with a load test.
- **Several Dashboard Application instances.** One instance is unlikely to serve 10,296 RPS, and one instance would also be a single point of failure against the 8.19 min monthly budget. Instances keep no per-User state, so any instance can serve any request.
- **Reference data is not cached in a separate container.** Filter and Search use only reference fields (symbol, name, exchange, sector, industry, cap band). That is 1.10 MiB and changes once per day, so each instance keeps an in-memory copy inside StockCatalogService.
- **One Database.** History (about 1 GiB) and Watchlists (about 43 MiB at 30,000 Users) do not need a second store. Split it only if measurements require it.

---

## 3. Component (Dashboard Application only)

| Component | Responsibility |
| --------- | -------------- |
| AuthGuard | Verifies the User identity on every request |
| InputValidator | Checks request parameters (Stock symbol, filters, ranges, Watchlist items) |
| StockCatalogService | Serves Search and Filter from an in-memory copy of reference data, reloaded daily |
| MarketDataQueryService | Serves Overview, Stock price and History. Applies the 20-minute rule using provider time and market-state labels |
| WatchlistService | Checks that the Watchlist belongs to the identified User, changes it, and returns it with current prices |

```mermaid
flowchart LR
    web["Web App"]
    cache[("Market Data Cache")]
    db[("Dashboard Database")]

    subgraph app["Dashboard Application"]
        guard["AuthGuard"]
        validator["InputValidator"]
        catalog["StockCatalogService"]
        query["MarketDataQueryService"]
        watchlist["WatchlistService"]

        guard -->|"Passes identified requests"| validator
        validator -->|"Routes valid Search and 
        Filter requests"| catalog
        validator -->|"Routes valid Overview, 
        price and History requests"| query
        validator -->|"Routes valid Watchlist 
        requests with identity"| watchlist
        watchlist -->|"Requests current prices for 
        entries"| query
    end

    web -->|"Sends all requests"| guard
    catalog -->|"Reloads reference data 
    daily"| db
    query -->|"Reads latest prices, refills 
    an entry only if newer"| cache
    query -->|"Reads history, and prices 
    on cache miss"| db
    watchlist -->|"Reads and writes the User's 
    Watchlist"| db
```

The Collector and provider are outside this boundary. They never talk to these components directly.

---

## 4. Sequence diagrams

### A. Sync market data

```mermaid
sequenceDiagram
    participant Collector as Market Data Collector
    participant Provider as Market Data Provider
    participant Database as Dashboard Database
    participant Cache as Market Data Cache

    Note over Collector: Timer fires, no sync is active for this scope
    Collector->>Database: Load saved progress
    Database-->>Collector: Saved or initial position
    Collector->>Provider: Request next batch with server-side credentials
    alt Provider rate-limits or times out
        Provider-->>Collector: Rate limit with optional Retry-After, or no response
        Note over Collector: Stop run, set next attempt using backoff and Retry-After
        Note over Database,Cache: Previously accepted data and progress unchanged
    else Provider returns a batch
        Provider-->>Collector: Records with provider times
        Collector->>Collector: Validate fields, numbers, units, provider time
        alt Batch is valid
            Collector->>Database: Commit records and next position atomically
            Database-->>Collector: Commit confirmed
            Note over Database: Accepted data is now available to later reads
            Collector->>Cache: Update latest prices only if provider time is newer
            alt Cache updated
                Cache-->>Collector: Updated
            else Cache update fails
                Note over Collector: Data stays accepted in Database, reads fall back to Database
            end
        else Batch is invalid
            Note over Collector: Reject batch, record failure, do not advance progress
            Note over Database,Cache: Previously accepted data and progress unchanged
        end
    end
```

Starting assumptions:

- The only trigger is the schedule: latest prices every 60 s in market hours, intraday points every 5 minutes, daily bars after close, and reference data daily. No User request starts a sync. Only latest prices go to the Cache.
- Provider credentials exist only in the Collector's server-side configuration. They never reach the Web App or the Dashboard Application.
- **When sync fails:** previously accepted data stays in the Database and Cache, unchanged. It keeps being served until its provider time passes the 20-minute limit. After that, reads return "unavailable" (or the labelled last close where the rule allows it). Provider outages therefore count as downtime after about 5 minutes beyond the provider delay, which the design cannot hide.
- Data becomes available to reads only after the Database confirms the atomic commit. The Cache is updated after that, never before. A missing price is never stored as zero.
- Writes to the Database and Cache accept only a newer provider time for the same Stock. A repeated or older batch cannot overwrite newer data, which supports monotonic reads and safe retries.

### B. Read a Stock price

```mermaid
sequenceDiagram
    actor User
    participant Web as Web App
    participant Guard as AuthGuard
    participant Validator as InputValidator
    participant Query as MarketDataQueryService
    participant Cache as Market Data Cache
    participant Database as Dashboard Database

    User->>Web: Open price for a Stock
    Web->>Guard: Request price with session token
    alt Identity invalid or missing
        Guard-->>Web: Reject as unauthenticated
        Web-->>User: Ask User to sign in
    else Identity valid
        Guard->>Validator: Pass request with identity
        alt Input invalid
            Validator-->>Web: Reject with validation error
            Web-->>User: Show input error
        else Input valid
            Validator->>Query: Route price request
            Query->>Cache: Read latest price and provider time
            alt Cache entry meets the 20-minute and market-state rule
                Cache-->>Query: Price with provider time
                Query-->>Web: Price with provider time and delay label
                Web-->>User: Show delayed price, provider time and label
            else Cache entry missing, too old, or Cache unavailable
                Query->>Database: Read latest accepted price and provider time
                alt Storage failure
                    Database-->>Query: Error
                    Query-->>Web: Unavailable
                    Web-->>User: Show "unavailable, try again", no price
                else No accepted price stored
                    Database-->>Query: No record
                    Query-->>Web: Unavailable
                    Web-->>User: Show "unavailable", never zero
                else Price meets the rule
                    Database-->>Query: Price with provider time
                    Query->>Cache: Refill entry only if newer
                    Query-->>Web: Price with provider time and delay label
                    Web-->>User: Show delayed price, provider time and label
                else Price too old and no last-close label applies
                    Database-->>Query: Price with old provider time
                    Query-->>Web: Unavailable
                    Web-->>User: Show "unavailable", never an old price as current
                end
            end
        end
    end
```

Starting assumptions:

- **A User read never makes a provider request.** It reads accepted data from the Cache or Database, so User traffic does not consume provider quota.
- "Meets the rule" means the provider time is at most 20 minutes old in market hours. Otherwise the last close is shown with its label ("market closed" outside market hours, "previous close" from open until open + 20 min).
- Age is judged from provider time, never from fetch time.
- Targets: p50 <= 300 ms and p95 <= 1,000 ms for this read, and the Web App stops waiting at 5,000 ms. A cache hit is the fast path. A Database fallback is the slow path and must stay within the p95 target.
- A Database fallback refills the Cache only with a newer provider time. If that refill fails, a later read could in theory show an older value than a read before it. This is a known gap to test for monotonic reads.

### C. Change a private Watchlist

```mermaid
sequenceDiagram
    actor User
    participant Web as Web App
    participant Guard as AuthGuard
    participant Validator as InputValidator
    participant Watchlist as WatchlistService
    participant Database as Dashboard Database

    User->>Web: Add Stock to my Watchlist
    Web->>Guard: Submit change with session token
    alt Identity invalid or missing
        Guard-->>Web: Reject as unauthenticated
        Web-->>User: Ask User to sign in
    else Identity valid
        Guard->>Validator: Pass request with identity
        alt Input invalid, for example unknown Stock
            Validator-->>Web: Reject with validation error
            Web-->>User: Show input error
        else Input valid
            Validator->>Watchlist: Route change with identity
            Watchlist->>Watchlist: Check Watchlist belongs to identified User
            alt Not permitted, Watchlist belongs to another User
                Watchlist-->>Web: Reject as unauthorized, nothing changed
                Web-->>User: Show rejection
            else Permitted
                Watchlist->>Database: Store Watchlist change
                alt Storage failure
                    Database-->>Watchlist: Error or unconfirmed
                    Watchlist-->>Web: Change failed
                    Web-->>User: Show failure, Watchlist not reported as changed
                else Change committed
                    Database-->>Watchlist: Commit confirmed
                    Note over Watchlist: Success can be reported only from here
                    Watchlist-->>Web: Change accepted
                    Web-->>User: Show success
                end
            end
        end
    end
```

Starting assumptions:

- The Watchlist owner is taken from the verified identity, never from a User-supplied ID. A valid identity alone does not permit access to another User's Watchlist.
- Success is reported only after the Database confirms the commit.
- **Next read:** the same User's next Watchlist read, in any session, must show the change (REQ-CON-2). Watchlist reads therefore always come from the Database and never from a Cache or copy that could lag. Prices for its entries come from the Cache as in diagram B. After a rejection or storage failure, the Watchlist must be unchanged.
- Other Users can never see this Watchlist.

---

## Check against Lab 2 targets

| Target | How the design addresses it | Still to verify |
| ------ | --------------------------- | --------------- |
| 10,296 RPS at 30,000 Users | Stateless Dashboard Application instances, latest prices in memory, reference data in memory | Load test of the mixed workload, including the 900 RPS Overview burst at open |
| Price p95 <= 1,000 ms | Cache hit is the normal path, no provider call | p50 and p95 at 6,000 RPS, and how often reads fall back to the Database |
| Other reads p95 <= 2,000 ms | Filter and Search use in-memory reference data. History uses the Database | History query time over 1,260 daily bars and 1,638 intraday points per Stock |
| 99.9% in market hours (8.19 min) | Several app instances, Cache falls back to Database | Cache or Database failover time. Provider outages of about 13 minutes use the whole budget and cannot be masked |
| 20-minute freshness, monotonic reads | Rule applied from provider time, writes accept only newer provider time | Sync lag distribution, cache-refill gap noted in diagram B |
| Read-your-writes for Watchlist | Watchlist reads always go to the Database after a confirmed commit | Test that a read after a confirmed change never misses it |
| Provider quota | Collector limits requests and backs off, User reads never use quota | Real provider limits against one latest-price scan per 60 s plus history |

---

## Checklist

- [x] I included one System Context, one Container, and one Component diagram.
- [x] I included the three required sequence diagrams.
- [x] All six Mermaid blocks render without an error.
- [x] The Component diagram opens only the Dashboard Application.
- [x] My flows show normal results, rejected actions, and failure results.
- [x] The design covers normal reads, private data, provider ingestion, and failures.