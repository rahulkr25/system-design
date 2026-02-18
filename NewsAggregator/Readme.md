# News Aggregator (Google News)

Google News is a digital service that aggregates and displays news articles from thousands of publishers worldwide in a scrollable interface for users to stay updated on current events.

---

## Table of Contents

1. [Functional Requirements](#functional-requirements)
2. [Non-Functional Requirements](#non-functional-requirements)
3. [Core Entities](#core-entities)
4. [API Design](#api-design)
5. [High-Level Design](#high-level-design)
6. [Deep Dives](#deep-dives)
   - [Pagination Consistency & Efficiency](#1-how-can-we-improve-pagination-consistency-and-efficiency)
   - [Low Latency Feed Requests](#2-how-do-we-achieve-low-latency--200ms-feed-requests)
   - [Article Freshness (30-minute SLA)](#3-how-do-we-ensure-articles-appear-in-feeds-within-30-minutes-of-publication)
   - [Media Content Handling](#4-how-do-we-handle-media-content-imagesvideos-efficiently)
   - [Traffic Spikes During Breaking News](#5-how-do-we-handle-traffic-spikes-during-breaking-news)
   - [Category-Based Feeds](#6-how-can-we-support-category-based-news-feeds-sports-politics-tech-etc)
   - [Personalized Feeds](#7-how-do-we-generate-personalized-feeds-based-on-user-reading-behavior-and-preferences)
7. WhiteBoarding - https://excalidraw.com/#json=2H3Sisd72kgqxORLGBBuv,1nYHyHboFlKn3OiWZA1gsw

---

## Functional Requirements

### In Scope

1. Users should be able to view an aggregated feed of news articles from thousands of source publishers all over the world.
2. Users should be able to scroll through the feed "infinitely."
3. Users should be able to click on articles and be redirected to the publisher's website to read the full content.

### Out of Scope

- Users should be able to customize their feed based on interests.
- Users should be able to save articles for later reading.
- Users should be able to share articles on social media platforms.

---

## Non-Functional Requirements

### In Scope

1. The system should prioritize **availability over consistency** (CAP theorem).
2. The system should be scalable to handle **100 million daily active users** with spikes up to 500 million.
3. The system should have **low latency** feed load times (< 200ms).

### Out of Scope

- The system should protect user data and privacy.
- The system should handle traffic spikes during breaking news events.
- The system should have appropriate monitoring and observability.
- The system should be resilient against publisher API failures.

---

## Core Entities

| Entity | Description |
|--------|-------------|
| **Article** | Core content entity. Attributes: `id`, `title`, `summary`, `thumbnail_url`, `publish_date`, `publisher_id`, `region`, `media_urls`. |
| **Publisher** | Origin of content. Attributes: `id`, `name`, `url`, `feed_url`, `region`. |
| **User** | System users. Attributes: `id`, `region` (inferred from IP or explicitly set). Supports anonymous users. |

---

## API Design

```
GET /api/v1/feed?region={region}&cursorId={cursorId} → Article[]
```

---

## High-Level Design

### Requirement 1 — Aggregated feed from thousands of publishers

**Data Collection Service** polls publisher RSS feeds and APIs every 3–6 hours based on each publisher's update frequency.

Key components:
- **Publishers** — Thousands of news sources worldwide providing content via RSS feeds or APIs.
- **Database** — Stores collected articles, publishers, and metadata.
- **Object Storage** — Stores thumbnails for articles.

**Workflow:**
1. Data Collection Service queries the database for the list of publishers and their RSS feed URLs.
2. Queries each publisher feed sequentially.
3. Extracts article content, metadata, and downloads media files for thumbnails.
4. Stores thumbnail files in Object Storage and saves article data with media URLs to the database.

**Feed Service** handles user feed requests by querying for relevant articles based on the user's region and formatting the response. It is intentionally separated from the Data Collection Service because:
- Different scaling requirements (read-heavy vs. write-heavy).
- Different update frequencies (real-time vs. batch).
- Different operational concerns (user-facing vs. background processing).

---

### Requirement 2 — Infinite scrolling

**Initial load:**
1. Client sends `GET /feed?region=US&limit=20&page=1`.
2. Feed Service queries for the first 20 articles in the user's region, ordered by publish date.
3. Response includes articles plus pagination metadata (`total_pages`, `current_page`).
4. Client stores the current page number for subsequent requests.

**Subsequent pages (as user scrolls):**
1. Client sends `GET /feed?region=US&limit=20&page=2`.
2. Feed Service calculates `OFFSET = (page - 1) * limit` and fetches the next 20 articles.
3. Process repeats as the user continues scrolling.

> This provides a simple foundation for infinite scrolling. Performance and consistency limitations are addressed in the deep dives below.

---

### Requirement 3 — Redirect to publisher on article click

This is handled entirely by the browser. When users click an article, the browser redirects to the article URL stored in the database, taking them directly to the publisher's website.

> **Note:** In real Google News, article links would point to a tracking endpoint like `GET /article/{article_id}` which logs the click event and returns a `302` redirect to the publisher's site. This is considered out of scope here.

---

## Deep Dives

### 1. How can we improve pagination consistency and efficiency?

**Problem:** Offset-based pagination breaks when new articles are published mid-session. If a user is on page 2 and new articles are added to the top of the feed, content shifts and the user may see duplicates or miss articles entirely.

---

#### Good Solution: Timestamp-Based Cursor

Instead of page numbers, return a cursor representing the timestamp of the last article. Subsequent requests query:

```sql
WHERE published_at < cursor_timestamp
ORDER BY published_at DESC
LIMIT 20
```

##### Challenges
When multiple articles share the same timestamp (common in batch imports or automated publishing), this approach may silently skip articles with identical timestamps as the cursor.

---

#### Great Solution: Composite Cursor (Timestamp + Article ID)

Combine timestamp and article ID into a unique composite cursor (e.g., `"2024-01-15T10:30:00Z_article123"`). The database query becomes:

```sql
WHERE (published_at, article_id) < (cursor_timestamp, cursor_id)
ORDER BY published_at DESC, article_id DESC
LIMIT 20
```

Create a composite index on `(published_at, article_id)` for efficient lookups.

##### Challenges
- Slightly more complex cursor encoding/decoding logic in the application layer.
- Marginally more complex queries (though databases handle tuple comparisons efficiently).
- Composite index adds minor storage overhead — worthwhile given the consistency guarantees.

---

#### Great Solution: Monotonically Increasing Article IDs

Design article IDs to be monotonically increasing from the start using time-ordered identifiers like **ULIDs** or database auto-increment IDs. Since articles are collected chronologically, newer articles will always have higher IDs. Pagination becomes:

```sql
WHERE article_id < cursor_id
ORDER BY article_id DESC
LIMIT 20
```

The cursor is a single ID — no timestamps, no composite keys, no tuple comparisons.

##### Challenges
- Requires planning the ID strategy upfront — migrating from random UUIDs is non-trivial.
- Distributed ID generation requires coordination across instances (ULID generation or a centralized ID service handles this).

---

### 2. How do we achieve low latency (< 200ms) feed requests?

**Problem:** Querying the database directly for each request doesn't scale. With 100M DAU refreshing feeds 5–10 times per day, we face 500M–1B feed requests daily. Direct DB queries will far exceed the 200ms target.

---

#### Good Solution: Redis Cache with TTL

Cache recent articles by region in Redis sorted sets (e.g., `feed:US`, `feed:UK`) ordered by timestamp. On feed requests, read from Redis first; fall back to the database only on cache misses.

Cursor-based pagination still works via `ZREVRANGEBYSCORE` using the cursor value as the score bound.

##### Challenges
- Cache misses still trigger expensive DB queries that may violate latency requirements.
- TTL expiration causes **thundering herd**: all users in a region hit the database simultaneously when the cache expires, causing periodic degradation lasting several minutes.
- User experience becomes inconsistent — fast on cache hits, slow on misses.

---

#### Great Solution: Real-Time Cached Feeds with CDC

Pre-compute and cache feeds per region using **Change Data Capture (CDC)** for immediate updates. No TTL — caches stay fresh in real time.

**How it works:**
1. Data Collection Service writes new articles to the database.
2. CDC events are consumed by Feed Generation Workers.
3. Workers determine which regional feeds to update based on article region/relevance.
4. Workers add new articles to the appropriate Redis sorted sets using `ZADD` with timestamp as the score.
5. `ZREMRANGEBYRANK` with negative indices prunes old articles beyond the per-region limit (typically 1,000–2,000 articles).

Feed reads use `ZREVRANGE`, completing in under 5ms, ensuring consistent sub-200ms responses with immediate content freshness.

##### Challenges
- Requires maintaining CDC infrastructure, message queues, and worker processes.
- Increased operational complexity (monitoring the full pipeline, handling worker failures, cache rebuilding).
- Slightly higher storage costs from duplicating article data across regional caches — manageable with size limits.

---

### 3. How do we ensure articles appear in feeds within 30 minutes of publication?

---

#### Good Solution: Increased RSS Polling Frequency

Implement a **tiered polling system** based on publisher priority:

| Priority | Examples | Polling Frequency |
|----------|----------|-------------------|
| High | CNN, BBC, Reuters | Every 5–10 minutes |
| Medium | Mid-tier outlets | Every 30 minutes |
| Low | Weekly/niche publications | Every 2–3 hours |

Workers track `Last-Modified` headers and `ETags` to skip unchanged feeds and reduce unnecessary processing.

##### Challenges
- With 10,000+ publishers, high-priority polling generates 100,000+ HTTP requests per hour — significant infrastructure cost and risk of getting IP-blocked by smaller publishers.
- Still reactive rather than proactive; breaking news can still lag by up to 5 minutes plus processing time.
- Not all publishers have RSS feeds.

---

#### Good Solution: Web Scraping as Fallback

For publishers without RSS feeds, programmatically scrape news websites using CSS selectors to extract article links, titles, and publication timestamps. A fingerprint database of previously seen articles (URL hashes or content checksums) identifies new content. Extracted content is normalized into the standard article format and fed through the same ingestion pipeline.

Combine with tiered scheduling: high-traffic sites scraped every 10–15 minutes, smaller sites hourly.

##### Challenges
- Significant maintenance overhead — websites frequently change HTML structure, breaking extraction logic.
- Slower and less reliable than RSS; legal concerns on sites that prohibit scraping.
- Best used as a strategic fallback for publishers without RSS, not as a primary method.

---

#### Great Solution: Publisher Webhooks with Fallback Polling

Flip the model from **pull-based to push-based**. Implement a webhook endpoint:

```
POST /webhooks/article-published
```

Publishers call this endpoint the moment they publish new content, providing article metadata or full content payload. The endpoint:
- Validates and authenticates the payload (shared secrets or API keys).
- Queues the content for processing through the standard ingestion pipeline.
- Triggers immediate cache updates for relevant regional feeds.

Content appears in user feeds within **30 seconds of publication** for webhook-enabled publishers.

Keep RSS polling and web scraping as fallbacks for non-webhook publishers, creating a hybrid system.

##### Challenges
- Requires coordination and buy-in from publishers — cannot be implemented unilaterally.
- Smaller publishers may lack the technical resources to implement webhook integrations.

---

### 4. How do we handle media content (images/videos) efficiently?

---

#### Bad Solution: Database Blob Storage

Store thumbnail images directly in the database as binary data alongside article metadata.

##### Why this is bad
Even small thumbnails (20–50KB) cause severe database performance issues at scale. Queries slow down, backups balloon, and database memory is consumed by binary data instead of indexed queries. **Databases store structured data — binary blobs belong in object storage.**

---

#### Good Solution: S3 with Direct Links

Store thumbnails in Amazon S3 and reference their URLs in article metadata. During ingestion, download the original image, generate a 300×200 thumbnail, upload to S3, and store the URL. Thumbnails load directly from S3 to the browser, reducing application server load.

##### Challenges
- High latency for globally distributed users loading thumbnails from a distant S3 region.
- S3 egress costs accumulate with millions of thumbnail views daily.
- No support for different screen densities (retina vs. standard) or slow network connections.

---

#### Great Solution: S3 + CloudFront CDN with Multiple Sizes

Store thumbnails in S3 and serve through **CloudFront CDN** for global distribution. Generate multiple thumbnail variants at ingestion time:

| Variant | Dimensions | Use Case |
|---------|-----------|----------|
| Small | 150×100 | Mobile |
| Standard | 300×200 | Desktop |
| Retina | 600×400 | High-DPI displays |

Use HTML `srcset` or client-side logic to request the appropriate variant. CloudFront edge caches thumbnails globally, delivering sub-200ms load times worldwide.

##### Challenges
- Higher storage costs for multiple variants — minimal compared to performance gains.
- CDN caching reduces S3 requests by over 90%, significantly lowering overall cost.

---

### 5. How do we handle traffic spikes during breaking news?

**Context:** Normal traffic of 100M DAU can spike to 10M concurrent users within minutes during major events. Google News has a natural advantage: news consumption is **inherently regional**, so infrastructure can be deployed per-region, with each deployment handling only its local traffic.

---

#### Feed Service (Application Layer)

Feed Services are **stateless**, making horizontal scaling straightforward. New instances can be spun up in seconds and torn down when traffic subsides.

---

#### Database Layer

With the CDC-based cache in place, virtually all read traffic is served from Redis — the database is shielded from read spikes during breaking news.

---

#### Cache Layer (Redis)

A single Redis instance handles ~100K requests/second. At 10M concurrent users, a single instance is insufficient.

**Solution: Read Replicas**

Each regional Redis master gets multiple read replicas. Since each regional feed holds only ~2,000 articles, a single master easily stores all regional content — scaling is purely a read throughput problem.

**Scaling math:**
- 10M concurrent users ÷ 100K req/s per instance = **100 Redis instances** needed at peak.
- In practice, deploy per-region with elasticity — scale up during local spikes without affecting other regions.

**Write path:** All writes (new articles, cache updates) go to the master.  
**Read path:** Load-balanced across replicas via round-robin or least-connections.  
**Failover:** Redis Sentinel promotes a replica to master automatically if the master fails. Replication lag is typically under 200ms — acceptable for news feeds.

**Benefits of the regional approach:**
- Sub-50ms cache response times from the nearest cluster.
- Traffic spikes in one region don't impact others.
- Independent scaling per region based on local usage patterns.

---

### 6. How can we support category-based news feeds (Sports, Politics, Tech, etc.)?

**Context:** With 25+ categories and 100M DAU, peak category traffic (e.g., Sports during game season) can reach 10M requests for a single category. Our current regional cache structure doesn't handle granular category filtering efficiently.

---

#### Bad Solution: Database Query Filtering with Category Column

Add a `category` column to the `Article` table and filter in real-time:

```sql
WHERE region = 'US' AND category = 'sports'
ORDER BY published_at DESC
LIMIT 20
```

Use a composite index on `(region, category, published_at)`.

##### Challenges
- Every category request hits the database — 50M peak requests = 50M DB operations.
- 250 different cache keys (25 categories × 10 regions) with independent invalidation — complex and cache-miss prone.

---

#### Good Solution: Pre-Computed Category Feeds in Redis

Pre-compute separate Redis sorted sets per category-region combination: `feed:sports:US`, `feed:politics:UK`, `feed:technology:CA`. During ingestion, CDC workers categorize each article and add it to both the regional feed and the relevant category feed simultaneously.

Feed request:
```
ZREVRANGE feed:sports:US 0 19  → completes in < 5ms
```

##### Challenges
- 250+ sorted sets instead of 10 — significant memory overhead.
- Articles must be removed from multiple sorted sets on expiry — more complex cache invalidation.

---

#### Great Solution: In-Memory Filtering on Regional Cache

Store complete article metadata (including `category`) as JSON in each regional sorted set. On a category-specific request, retrieve the full regional cache and filter in application memory.

Example cached entry:

```json
{
  "id": "123",
  "title": "NBA Finals Game 7 Results",
  "description": "Warriors defeat Celtics in thrilling finale",
  "url": "https://espn.com/nba/finals/game7",
  "category": "sports",
  "region": "US",
  "published_at": "2024-06-21T22:30:00Z"
}
```

For `/feed?region=US&category=sports`:
1. `ZREVRANGE feed:US 0 999` — fetch the most recent 1,000 regional articles (~10ms).
2. Filter in-memory for `category === "sports"` (~1–2ms).
3. Paginate and return.

**Advantages:**
- No article duplication across multiple caches.
- Minimal changes to the existing architecture — only the cached data format and Feed Service filtering logic need updating.
- Memory usage stays flat — each article lives in exactly one regional cache.

> This is a case where the best solution is the most straightforward one.

---

### 7. How do we generate personalized feeds based on user reading behavior and preferences?

**Context:** Users expect feeds personalized to their topics of interest, preferred publishers, and reading history. The actual ranking/scoring function is typically a machine learning model, abstracted away here.

---

#### Bad Solution: Real-Time Recommendation Scoring

Score articles against user preferences in real-time during each feed request, using reading history, topic preferences, and collaborative filtering signals.

##### Challenges
- Scoring thousands of articles against 100M user profiles at request time = billions of computations per hour.
- Response times balloon to several seconds — far beyond the 200ms target.
- Performance degrades catastrophically at traffic spikes.

---

#### Good Solution: Pre-Computed User Feed Caches

Pre-compute personalized feeds for active users in Redis (e.g., `feed:user:12345`). Background workers score and update these feeds incrementally as new articles arrive and user behavior signals are collected.

##### Challenges
- **Memory explosion:** 1,000 articles per user × 50M active users = 50 billion cache entries — 200–500× larger than category caching.
- Cache staleness when interests change rapidly.
- Personalized feeds may miss globally important breaking news.

---

#### Great Solution: Hybrid Personalization with Dynamic Feed Assembly

Store **lightweight user preference vectors** (kilobytes per user) and assemble personalized feeds on-demand by mixing pre-computed category feeds in proportions matching each user's interests.

**Example:** A user interested in technology gets a feed assembled as:
- 60% from `feed:technology:US`
- 30% from `feed:business:US`
- 10% from `feed:trending:US`

During breaking news, the system temporarily boosts trending content weights while preserving personal preferences. Machine learning continuously adjusts mixing ratios based on engagement patterns.

**Benefits:**
- Builds on existing category cache infrastructure — no new caching layer.
- ~100× lower memory requirements compared to per-user feed caches.
- Fallback strategies maintain performance during traffic spikes.

##### Challenges
- Reduced personalization depth compared to full per-user recommendation engines.
- Mixing ratios require careful tuning to balance personalization with content diversity — very narrow interests may miss important global stories.
