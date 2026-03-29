# Price Tracking Service

CamelCamelCamel is a price tracking service that monitors Amazon product prices over time and alerts users when prices drop below their specified thresholds. It also has a popular Chrome extension with 1 million active users that displays price history directly on Amazon product pages, allowing for one-click subscription to price drop notifications without needing to leave the Amazon product page.

---

## Table of Contents

- [Functional Requirements](#functional-requirements)
- [Non-Functional Requirements](#non-functional-requirements)
- [Core Entities](#core-entities)
- [API](#api)
- [Data Flow](#data-flow)
- [High-Level Design](#high-level-design)
- [Deep Dives](#deep-dives)
  - [How do we efficiently discover and track 500 million Amazon products?](#1-how-do-we-efficiently-discover-and-track-500-million-amazon-products)
  - [How do we handle potentially malicious price updates from Chrome extension users?](#2-how-do-we-handle-potentially-malicious-price-updates-from-chrome-extension-users)
  - [How do we efficiently process price changes and notify subscribed users?](#3-how-do-we-efficiently-process-price-changes-and-notify-subscribed-users)
  - [How do we serve fast price history queries for chart generation?](#4-how-do-we-serve-fast-price-history-queries-for-chart-generation)
- [WhiteBoarding](https://excalidraw.com/#json=yXjVfXefjQhTs3q1Hg04K,9Qoz3Cd5OuCoRZ417eEi-Q)
---

## Functional Requirements

### In scope

1. Users should be able to view price history for Amazon products (via website or Chrome extension)
2. Users should be able to subscribe to price drop notifications with thresholds (via website or Chrome extension)

### Out of scope

Below the line (out of scope):

- Search and discover products on the platform
- Price comparison across multiple retailers
- Product reviews and ratings integration

---

## Non-Functional Requirements

### In scope

1. The system should prioritize availability over consistency (eventual consistency acceptable)
2. The system should handle 500 million Amazon products at scale
3. The system should provide price history queries with < 500ms latency
4. The system should deliver price drop notifications within 1 hour of price change

### Out of scope

Below the line (out of scope):

- Strong consistency for price data
- Real-time price updates (sub-minute)

---

## Core Entities

- Product
- User
- Subscription
- Price

---

## API

### 1. Get price history

```http
GET /api/v1/product/{id}/price?period=30d&granualrity=daily -> PriceHistory[]
```

---

### 2. Create subscription

```http
POST /api/v1/subscription
```

**Request body:**

```text
{
    productId,
    threshold,
    notificationType
} -> 200 OK
```

---

## Data Flow

1. Web crawlers and Chrome extension collect current prices from Amazon product pages
2. Price data is validated and stored in our price database
3. Price changes trigger events for notification processing
4. User receives email notification when price drops below threshold

---

## High-Level Design

Note: Amazon is not friendly to price tracking services. They don't provide an API, actively discourage scraping, and implement rate limiting (typically 1 request per second per IP address). We must design our system to work within these constraints while remaining respectful of Amazon's terms of service.

### 1. Users should be able to view price history for Amazon products (via website or Chrome extension)

**Client:** Our client is either the website or the Chrome extension. In either case, they're how the user views price history graphs.

**API Gateway:** Handles authentication, rate limiting, and routes requests to appropriate services

**Price History Service:** Manages price data retrieval and coordinates with the database

**Web Crawler Service:** Scrapes Amazon product pages to collect current prices

**Price Database:** Stores both current product information and historical price data

**Primary Database:** Stores all other tables aside from price like Users, Products, and Subscriptions.

As for the databases, we'll make a strategic separation based on scale and access patterns. Our Primary Database will store Users, Products, and Subscriptions together since these have similar operational characteristics - they're relatively small and have traditional CRUD access patterns. However, we'll put our Price Database completely separate because price history data has fundamentally different requirements: it will grow to billions of rows, is append-only, requires time-series optimizations, and can tolerate eventual consistency. This separation allows us to optimize each database independently and scale them based on their specific needs.

When a user requests price history through our website or Chrome extension:

1. Client sends GET request to `/products/{product_id}/price?period=30d`
2. API Gateway authenticates the request and routes to Price History Service
3. Price History Service queries the Price table filtered by product_id and time range
4. Historical prices are returned as JSON, ready for chart rendering
5. Chrome extension or website displays the price chart to the user

### 2. Users should be able to subscribe to price drop notifications with thresholds (via website or Chrome extension)

Our enhanced architecture includes:

1. **Subscription Service:** Manages user subscriptions and price threshold settings
2. **Notification Cron Job:** Periodically checks for price changes and sends email notifications

For our initial notification system, we have a straightforward batch processing approach:

1. Cron job runs every 2 hours to check for notifications
2. Job queries Price table for any price changes in the last 2 hours
3. For each price change, query Subscriptions table from the Primary Database to find users with thresholds >= new_price
4. Send email notifications for any triggered subscriptions
5. Mark notifications as sent to avoid duplicates

This polling approach works but has obvious limitations. Users might wait up to 2 hours for price drop notifications, and we're doing expensive database scans every few hours. We'll explore much better real-time approaches in our deep dives that provide sub-minute notification delivery.

---

## Deep Dives

### 1. How do we efficiently discover and track 500 million Amazon products?

We're solving two distinct but related problems here.

1. The first challenge is product discovery: we need to find all 500 million existing products when we launch, plus discover the approximately 3,000 new products Amazon adds daily.
2. The second challenge is price monitoring: we need to efficiently update prices for millions of known products while prioritizing which ones to check most frequently.

Both challenges face the same fundamental constraint of Amazon's rate limiting, but understanding them separately helps us design more effective solutions.

#### Bas Solution: Naive Web Crawling Approach

The most straightforward approach involves deploying traditional web crawlers that systematically browse Amazon's website. We start from seed URLs like Amazon's homepage and category pages, extract product links from each page, and recursively follow these links to discover the entire product catalog. Each crawler runs through categories methodically, extracting product IDs, prices, and basic metadata before moving to the next page.

This treats Amazon like any other website to be indexed. We implement a frontier queue containing URLs to visit, a visited URL tracker to avoid cycles, and crawlers that process URLs sequentially. When a crawler encounters product recommendation links or "customers also viewed" sections, it adds these new URLs to the frontier queue for future processing. The system maintains a simple database of discovered products and updates prices whenever a product page is revisited.

**Challenges**

The fundamental problem is scale arithmetic. With 500 million products to track and Amazon's rate limit of approximately 1 request per second per IP address, a single crawler would require over 15 years just to visit each product page once. Even if we deploy 1,000 servers with different IP addresses, we're still looking at 5+ days for a complete catalog refresh, during which millions of price changes will go undetected.

500*10^6 / 10^5 = 5000days / 300/yr = 15 years

With 1000 servers = 5 + days

This also creates a discovery lag problem. New products added to Amazon might not be found for weeks or months, depending on where they appear in our crawling sequence. Popular products that change prices multiple times per day will have stale data for extended periods, making our service unreliable for time-sensitive deals. The massive infrastructure cost of running thousands of crawling servers makes this economically unfeasible for most companies.

#### Good Solution: Priortized Crawling Based on User Interest

The key realization here is that we have fundamentally limited crawling resources, so we better use them wisely. Product popularity follows a Pareto distribution - a small percentage of Amazon's products get the vast majority of user attention, while there's a long tail of products that people rarely care about. Instead of treating all 500 million products equally, we can tier our crawling based on actual user interest and dramatically improve our service quality where it matters most.

We can implement a priority scoring system where products get higher scores based on active user engagement. Products with many price drop subscriptions receive the highest priority since users are actively waiting for updates on these items. Products that users search for frequently on our website also get elevated priority, indicating current interest even if subscriptions haven't been created yet.

The crawling system maintains priority queues where high-interest products might be checked every few hours, medium-interest products get daily updates, and low-interest products are refreshed weekly or even less frequently. We also implement feedback loops where successful price drop notifications (ones that lead to user clicks or purchases) boost a product's priority score, creating a self-reinforcing system that focuses on the most valuable updates.

This allows us to achieve excellent coverage for the products that matter most to our users while using a fraction of the infrastructure required for comprehensive crawling. Popular products stay current while niche products still receive periodic updates.

**Challenges**

The fundamental limitation is coverage gaps for new or trending products that haven't yet built up user interest signals. A hot new product release might not get discovered quickly if no users are searching for it yet, potentially missing early adoption opportunities when prices are most volatile.

#### Great Solution: Chrome Extension + Selective Crawling

Remember that Chrome extension with 1 million users we mentioned earlier? It turns out to be more than just a convenience feature, it's actually our secret weapon for data collection.

The most elegant solution leverages our Chrome extension's 1 million users as a distributed data collection network. When users browse Amazon with our extension installed, it automatically captures product IDs, current prices, and page metadata, then reports this information to our backend services. This crowdsourced approach transforms user browsing behavior into our primary data collection mechanism.

The extension operates transparently, extracting structured data from Amazon's product pages using DOM parsing and sending updates to our price reporting API. We receive real-time price data for products that users are actively viewing, which naturally prioritizes popular and trending items. This user-generated data covers the products people actually care about without requiring extensive crawler infrastructure.

Our traditional crawlers now just need to handle products that haven't been viewed by extension users recently. The system also uses extension data to discover new products; when users visit previously unknown product pages, we add them to our Product table. Brilliant!

**Challenges**

This introduces dependency on user adoption and browsing patterns. Products in niche categories with low user interest might receive infrequent updates, creating coverage gaps. But most importantly, we must carefully handle the data validation challenge since user-generated price reports could be manipulated or incorrect, requiring verification systems we'll explore in our next deep dive. We'll talk about this next.

---

### 2. How do we handle potentially malicious price updates from Chrome extension users?

#### Great Solution: Trust-But-Verify with Priority Verification

The best approach is simple: trust the extension data immediately, but verify it quickly with our crawlers when something looks suspicious.

Here's how it could work. When our extension reports a price change, we accept it right away and can send notifications immediately. But if the change seems fishy - like a huge price drop, or it's from a user who's been wrong before, or it's a popular product with lots of subscribers - we automatically queue up a verification crawl job to check Amazon directly within a few minutes.

We use our existing crawler infrastructure but give these verification jobs high priority. Instead of waiting hours or days for our regular crawling schedule, suspicious price updates get checked within 1-5 minutes. If our crawler finds out the extension data was wrong, we can send correction notifications and mark that user as less trustworthy.

For really important products, we can wait for multiple users to report the same price change before we fully trust it. If we see conflicting reports from different users, we immediately send a crawler to figure out who's right.

The nice thing is that most extension data gets processed immediately (fast notifications), but we catch the bad stuff quickly enough that it doesn't cause real damage. Users get fast alerts for legitimate deals, and we maintain trust by correcting the occasional mistake.

**Challenges**

This does add significant complexity to our crawling infrastructure, requiring priority queue management and rapid response capabilities that consume additional server resources. The verification crawling creates more load on Amazon's servers, potentially increasing our risk of rate limiting or IP blocking.

---

### 3. How do we efficiently process price changes and notify subscribed users?

We have two solid ways to implement this, each with different trade-offs.

1. The first option is database change data capture (CDC), where we configure our database to automatically publish events whenever price data changes. When our crawlers or extension data processors insert new price records, database triggers fire and send events to Kafka (or similar) containing the product ID, old price, and new price. Our notification service subscribes to these events and immediately queries the subscriptions table to find affected users.

2. The second option is dual writes, where our price collection services write to both the database and publish events to Kafka simultaneously. When crawler data or extension updates come in, the same service that writes to the price database also publishes structured events to our notification stream.

The dual-write approach lets us be smarter about which changes trigger notifications. We can filter out tiny price fluctuations that wouldn't interest users or batch multiple rapid changes before publishing events. We also avoid the overhead of database triggers on every single price update.

---

### 4. How do we serve fast price history queries for chart generation?

Looking at our current data storage approach, we're keeping price history in a straightforward PostgreSQL table with basic indexing on (product_id, timestamp). This works for initial functionality, but fails to meet our < 500ms latency requirement when serving price charts for popular products with extensive history data.

Consider the scale challenge that comes from aggregating price data for popular products. Popular products might have price data points every few hours for several years, resulting in many thousands of records per product. When users request a 2-year price chart, we must aggregate this data into appropriate time buckets (daily or weekly averages) and return it quickly enough for smooth user experience. Raw database queries struggle with this aggregation workload.

#### Good Solution: Scheduled Pre-Aggregation with Cron Jobs

The straightforward solution is pre-computing price aggregations at different time granularities using scheduled batch jobs. Every night, we run a job that calculates daily, weekly, and monthly price summaries for all products, storing these aggregations in a simple table optimized for fast chart queries.

Our batch job processes the previous day's price data and computes relevant statistics: average price, minimum price, maximum price, opening price, and closing price for each time period. We store all this in a single price_aggregations table with a granularity column that indicates whether each row represents daily, weekly, or monthly data.

When users request price charts, our API queries this aggregation table instead of the raw price data. A 30-day chart queries for daily aggregations, while a 2-year chart uses monthly aggregations. With proper indexing on (product_id, granularity, date), these queries return dozens of pre-computed records in milliseconds instead of aggregating thousands of raw price points.

The core idea is simple. price_aggregations is a summary table that stores one row per product, per time bucket, per granularity, so your chart API reads small precomputed rows instead of scanning raw price events.

A clean schema would look like this.

```sql
CREATE TABLE price_aggregations (
  product_id BIGINT NOT NULL,
  granularity VARCHAR(16) NOT NULL,
  bucket_start DATE NOT NULL,
  avg_price NUMERIC(10,2) NOT NULL,
  min_price NUMERIC(10,2) NOT NULL,
  max_price NUMERIC(10,2) NOT NULL,
  open_price NUMERIC(10,2) NOT NULL,
  close_price NUMERIC(10,2) NOT NULL,
  sample_count INT NOT NULL,
  currency CHAR(3) NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  PRIMARY KEY (product_id, granularity, bucket_start)
);

CREATE INDEX idx_price_aggs_lookup
ON price_aggregations (product_id, granularity, bucket_start);
```

What each row means. If product 123 has a daily row for 2026-03-28, that row summarizes all raw price changes for that product during that day. open_price is the first seen price in that bucket. close_price is the last seen price. avg_price, min_price, and max_price are computed across all price samples in that bucket. sample_count tells you how many raw points contributed.

A few example rows might look like this.

```json
[
  {
    "product_id": 123,
    "granularity": "daily",
    "bucket_start": "2026-03-26",
    "avg_price": 95.50,
    "min_price": 90.00,
    "max_price": 100.00,
    "open_price": 100.00,
    "close_price": 90.00,
    "sample_count": 4,
    "currency": "USD"
  },
  {
    "product_id": 123,
    "granularity": "monthly",
    "bucket_start": "2026-03-01",
    "avg_price": 92.10,
    "min_price": 85.00,
    "max_price": 104.00,
    "open_price": 101.00,
    "close_price": 89.00,
    "sample_count": 73,
    "currency": "USD"
  }
]
```

Your raw table is still separate. It would usually look like prices(product_id, price, observed_at, source, currency). The aggregation job reads raw rows from prices, groups them into buckets, computes the summary fields, and upserts into price_aggregations.

For a 30 day chart, the API query is simple.

```sql
SELECT bucket_start, avg_price, min_price, max_price, open_price, close_price
FROM price_aggregations
WHERE product_id = 123
  AND granularity = 'daily'
  AND bucket_start >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY bucket_start;
```

One important detail is weekly and monthly rows. You usually build daily first, then derive weekly and monthly from daily if approximation is okay. If you need exact open_price and close_price, it is safer to compute each granularity directly from raw price events.

#### Great Solution: TimescaleDB for Real-Time Price Analytics

Rather than running these cron jobs, we can use TimescaleDB, a time-series extension for PostgreSQL that's purpose-built for this type of workload. Since we're already using PostgreSQL for our operational data (users, products, subscriptions), TimescaleDB lets us handle price history analytics within the same database ecosystem while getting specialized time-series performance.

With TimescaleDB, we can perform real-time aggregations directly without any pre-computation. When a user requests a 6-month price chart, we run a query like `SELECT time_bucket('1 day', timestamp), avg(price) FROM prices WHERE product_id = ? GROUP BY 1` and get results in milliseconds even with billions of price records. TimescaleDB's automatic partitioning and compression make these aggregations incredibly fast while maintaining PostgreSQL's familiar interface.

This keeps our architecture simple. We don't need complex pre-aggregation jobs or multiple database systems. Users can request any time range or granularity, and TimescaleDB computes the results on demand. The system handles our scale requirements naturally since TimescaleDB is designed for exactly this type of time-series workload, while we maintain all the operational benefits of staying within the PostgreSQL ecosystem.

---
