# The Art of Making Performance Problems Impossible to Ignore

Since my early days as an engineer, performance has been my passion. It signals care for the product and respect for customers' time. Trust is easily lost when timeouts and errors start popping up at scale.

Performance work is unique—it requires deep understanding of all components and how they interact. I find it fascinating, even though it's incredibly time-consuming. But here's the uncomfortable truth: when reproduction is hard or gathering information about issues is difficult, engineers pay less attention. Sometimes they ignore performance problems entirely.

I'd bet countless performance issues go unresolved simply because generating large datasets or network loads is hard. This is why attacking performance is first and foremost a **tooling problem**. We need to make it stupidly simple for developers to reproduce issues. If they can reproduce quickly, they can identify bottlenecks and iterate. Performance is a continuous battle—without the right tooling, even today's improvements will drift back tomorrow.

The goal is simple: **make invisible things visible with minimal effort**. Make it so obvious that there's no excuse to look away.

## Story 1: The Outlook Database That Couldn't Scale

In my early days at Microsoft, I worked on Business Contact Manager—an Outlook add-on that transformed it into a small business CRM. It had a shared database that multiple Outlook clients connected to. Every UI action hit this central database.

Our customers complained constantly. With just 10 employees connecting to the same database, switching search folders or adding items became painfully slow. The application was nearly unusable.

The team's approach? Automate installing Outlook on multiple desktops, connect them to the shared database, and use UI test tools to mimic folder switching so engineers could debug. Imagine coordinating 10 different PCs—the setup time alone was enormous.

I took a different approach. I stripped out the UI element from Outlook, launched just one instance, and built a load tool that spawned 10 instances of the data access layer. It mimicked activities—search folder switches, item additions—continuously. When you tried to use Outlook with this tool running, you immediately saw what customers were experiencing.

![BCM Load Tool: Before and After](markdown/blog-images/bcm-load-tool.png)

Engineers had no excuse anymore. Launch one tool, stress the database, instant reproduction. They could test fixes immediately. It became stupidly simple to reproduce, and guess what? Performance issues got resolved very quickly.

## Story 2: Why Twitter Videos Were Slow to Start

At Twitter, people complained about video start latency. The challenge was multidimensional: which CDN was used, which codec, which network provider, which mobile app version—any combination could cause delays. Customers got frustrated. Ads started late. Revenue was at risk.

The root cause was nearly impossible to find because the problem was buried in volume. We needed data from every session to investigate arbitrary issues. The data collection alone was overwhelming.

Instead of fighting the data problem head-on, we shifted focus to tooling. What if we only collected detailed telemetry for sessions with high delay or glitches, while successful sessions just delivered a summary? This tooling improvement solved the data problem and made critical information accessible to engineers immediately.

![Twitter Video Latency: Before and After Smart Sampling](markdown/blog-images/twitter-smart-sampling.png)

Suddenly, we discovered CDN routing inefficiencies. We found resolution selection bugs. We uncovered codec selection problems. Making these invisible parts visible made fixing performance issues trivial.

## Story 3: The ClickHouse Journey at Edge Delta

The same pattern emerged at Edge Delta. Customers were experiencing performance issues with large time-window queries in our log search. Queries that should return in seconds were timing out. But understanding why? Nearly impossible.

The challenge was complexity. A single page load triggered dozens of queries—search queries, facet queries, graph queries, stats queries—all hitting different tables with different routing strategies. Which query was slow? Which table was being scanned? Was the bloom filter index being used? Were we hitting the sample table or the full table? Without visibility into this, engineers were debugging blind.

This is where we applied the same philosophy: **make the invisible visible**.

We built a query profiler that captures everything about a log search session: every query that hits the database, their execution order, the exact SQL with all parameters, the EXPLAIN plan, the schema being used, I/O metrics (rows read, bytes scanned, memory usage), and timing breakdowns. Engineers could now click a "Profile" button after any search and see a complete execution timeline—which queries ran in parallel, which were sequential, and where the time actually went.

![Query Profiler: Execution Timeline and SQL Details](markdown/blog-images/query-profiler.png)

Suddenly the bottlenecks were obvious. We could see queries scanning 5TB of data when they should have been using skip indexes. We could see facet queries taking longer than the main search. We could see routing decisions sending queries to the wrong table. With this visibility, my colleagues could attack the actual problems. On the backend, we dove deep into ClickHouse internals, optimizing query execution paths, improving partition pruning, and tuning how we structure data for time-range queries. On the frontend, we ensured the UI remained responsive during long-running queries, implementing progressive loading and fixing rendering bottlenecks that made slow queries feel even slower.

The results spoke for themselves. Queries that previously timed out now complete in seconds. What changed wasn't just the code—it was our ability to see, measure, and iterate on performance continuously.

## The Pattern

Three companies, three different problems, one consistent lesson: **performance work is tooling work**.

When you make reproduction trivial, engineers can't ignore problems. When you make measurements automatic, performance becomes part of the development cycle, not an afterthought. When you make the invisible visible, fixes become obvious.

If you're struggling with performance issues in your product, don't start by optimizing code. Start by asking: "How easy is it for any engineer on my team to reproduce this problem right now?" If the answer is anything other than "trivially easy," that's your first problem to solve.

Make it stupidly simple. Leave no excuses.

---

## Appendix: Technical Details of ClickHouse/ClickHouse Optimizations

Below is a comprehensive compilation of the performance improvements made to Log Search, ordered by impact. The work spans backend query optimization, frontend rendering, and profiling/tooling.

---

### 1. Smart Query Routing with hasToken Optimization — **HIGH IMPACT**

**Problem:** Full-text searches were scanning the entire ordered table even when token-based filtering could dramatically reduce the search space.

**Solution:** Implemented intelligent query routing that analyzes search tokens and routes queries to the optimal table (ordered vs. sample) based on token rarity. Added `hasToken()` optimization that leverages ClickHouse's inverted indexes for initial filtering before full-text search.

**Key Changes:**
- Built a routing planner that evaluates token rarity and query complexity
- Added `hasToken()` calls in PREWHERE clauses to leverage skip indexes
- Routes rare-token queries to ordered table, common-token queries to sample table
- Token rarity dependent routing biased toward ordered table for better index utilization

#### How It Works: The Two-Phase Filter Strategy

When a user searches for `"connection refused error"`, we now split the query into two phases:

**Phase 1: PREWHERE with hasToken (Bloom Filter)**
```sql
SELECT * FROM logs
PREWHERE hasToken(ibody, 'connection')
    AND hasToken(ibody, 'refused')
    AND hasToken(ibody, 'error')
WHERE position(body, 'connection refused error') > 0
```

The `hasToken()` function uses ClickHouse's bloom filter indexes—these are probabilistic data structures that can quickly tell us "this block definitely doesn't contain this token" without reading the actual data. This eliminates 90%+ of data blocks before we even start scanning.

**Phase 2: WHERE with position() (Exact Match)**

The `position()` check in WHERE verifies the exact phrase match. This is necessary because:
- `hasToken('error')` will match both "error" and "errors"
- We need to verify the tokens appear in the correct order/phrase

#### The Tokenizer: Matching ClickHouse's Behavior

A critical detail was ensuring our tokenizer produces the same tokens as ClickHouse's internal tokenizer:

```go
func TokenizeSearchTerm(term string) []string {
    seen := make(map[string]bool)
    tokens := []string{}
    var current strings.Builder

    for _, r := range strings.ToLower(term) {
        if (r >= 'a' && r <= 'z') || (r >= '0' && r <= '9') {
            current.WriteRune(r)
        } else {
            // Non-alphanumeric char, flush current token
            if current.Len() > 0 {
                t := current.String()
                if !seen[t] {
                    tokens = append(tokens, t)
                    seen[t] = true
                }
            }
            current.Reset()
        }
    }
    // ... flush final token
    return tokens
}
```

For example, `"user@example.com"` tokenizes to `["user", "example", "com"]`.

#### The Routing Decision: Which Table to Query?

Not all queries benefit equally from the ordered table. We built a routing planner that considers:

```go
// If token is so rare we can't satisfy LIMIT, use resource-ordered table
if tokenCount < uint64(p.limit) {
    decision.TableName = fmt.Sprintf("ed_log_%s%s", p.orgID, resTableSuffix)
    decision.RoutingReason = "rare_token_insufficient_results"
    return decision
}

// Calculate estimated rows to scan
tokenDensity := float64(tokenCount) / float64(rowsScope)
approxRowsToScan := float64(p.limit) / tokenDensity

// If estimated scan > 10x total size, token is too rare for T_ts
if estimatedScan > uint64(float64(rowsTotal) * 10.0) {
    // Rare token: use resource-ordered table for locality
    decision.TableName = fmt.Sprintf("ed_log_%s%s", p.orgID, resTableSuffix)
} else {
    // Common token: timestamp-ordered table can short-circuit efficiently
    decision.TableName = fmt.Sprintf("ed_log_%s", p.orgID)
}
```

**Commits:**
- `a8779b1d` - Add routing logic + hasToken optimization (+1,815 lines)
- `9bfe0a8a` - Change token rarity dependent routing logic
- `01f72ddf` - Utilize hasToken and prewhere optimization (+591 lines)
- `6621e9b1` - Change tokenizer code to mimic ClickHouse exactly

**Impact:** 5-10x improvement on searches with selective tokens; eliminates full table scans for most queries.

---

### 2. Aggregation and Sample Table Architecture — **HIGH IMPACT**

**Problem:** Large time-window queries were scanning massive amounts of data even for simple aggregations and facet counting.

**Solution:** Introduced a tiered table architecture with pre-aggregated (agg) tables and sampled tables. Queries are routed to the appropriate tier based on time range and query type.

**Key Changes:**
- Support for hitting agg and sample tables for logs
- New endpoint for grouping facets with optimized data access
- Added sampling factor multiplication to compensate for sampled data
- Ability to fetch facet keys from cache + agg table
- Return 404 when lookback dates are not within sample table data range

#### The Three-Tier Data Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Query Router                                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│   Raw Table   │ │ Sample Table  │ │   Agg Table   │
│  (Full Data)  │ │  (1% sample)  │ │ (Pre-computed)│
├───────────────┤ ├───────────────┤ ├───────────────┤
│ Use for:      │ │ Use for:      │ │ Use for:      │
│ - Last 1 hour │ │ - 1h to 24h   │ │ - Facet counts│
│ - Exact match │ │ - Trends      │ │ - Histograms  │
│ - Full results│ │ - Estimates   │ │ - Time series │
└───────────────┘ └───────────────┘ └───────────────┘
```

#### Sample Table with Compensation

When querying the sample table, we multiply results by the sampling factor:

```go
// Add sampling factor multiplication to compensate for sampled data
if useSampleTable && samplingFactor > 1 {
    query = fmt.Sprintf(`
        SELECT
            timestamp,
            count * %d as count,  -- Compensate for 1%% sampling
            facet_key,
            facet_value
        FROM sample_logs
        WHERE ...
    `, samplingFactor)
}
```

A 1% sample table is 100x smaller—queries that would scan 100GB now scan 1GB, with statistically accurate results for aggregations.

#### Smart Routing Based on Query Type

| Query Type | Time Range | Table Used | Why |
|------------|------------|------------|-----|
| Search "error" | Last 15m | Raw | Need exact matches |
| Search "error" | Last 24h | Sample → Raw | Quick scan, then drill down |
| Facet counts | Any | Agg | Pre-computed, instant |
| Time series graph | Last 7d | Agg | Pre-aggregated rollups |

**Commits:**
- `daa93e36` - Support hitting agg and sample tables for logs (+1,607 lines)
- `6fd3cfbb` - Add sample table based routing
- `39175acd` - Add ability to fetch facet keys from cache + agg table
- `b18c324e` - Add sampling factor multiplication to db query
- `688190904` - Add routing to facet_graph endpoint

**Impact:** Orders of magnitude improvement for wide time-range queries (hours to days); reduced memory pressure on ClickHouse.

---

### 3. Server-Sent Events (SSE) Streaming for Log Search — **HIGH IMPACT**

**Problem:** Large result sets caused timeouts and poor UX—users waited for entire query completion before seeing any results.

**Solution:** Implemented row-by-row SSE streaming that sends results as they're fetched from ClickHouse. Users see first results in milliseconds, with continuous streaming until completion.

**Key Changes:**
- Introduced SSE endpoint for log search API
- Row-by-row streaming with heartbeats
- Added compression on log search streaming
- Frontend streaming service with proper cancellation handling
- Feature flag for gradual rollout

#### How It Works: Progressive Result Delivery

Instead of the traditional request-response pattern:

```
Client: GET /logs/search?query=error
Server: [waits 10s for all results]
Server: {"items": [10000 log entries]}
```

We now stream results as windows complete:

```
Client: GET /logs/search/stream?query=error
Server: event: window
        data: {"window_id":"w1","items":[...100 logs...],"window_index":0}

Server: event: window
        data: {"window_id":"w2","items":[...100 logs...],"window_index":1}

... continues as data is fetched ...

Server: event: complete
        data: {"total_items":10000,"hit_windows":100}
```

#### The Backend SSE Writer

```go
type sseWriter struct {
    w       http.ResponseWriter
    flusher http.Flusher
}

func setupSSEHeaders(w http.ResponseWriter) {
    w.Header().Set("Content-Type", "text/event-stream; charset=utf-8")
    w.Header().Set("Cache-Control", "no-cache, no-store, must-revalidate")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no")  // Disable nginx buffering
}

func (s *sseWriter) sendEvent(event string, data any) error {
    if err := sse.Encode(s.w, sse.Event{Event: event, Data: data}); err != nil {
        return err
    }
    s.flusher.Flush()  // Immediately push to client
    return nil
}
```

The key insight: `X-Accel-Buffering: no` is crucial—without it, nginx will buffer the entire response, defeating the purpose of streaming.

#### Why This Matters for UX

| Scenario | Before (Batch) | After (Streaming) |
|----------|----------------|-------------------|
| 100 results | 2s wait, then all | 200ms first results |
| 10,000 results | 15s wait (often timeout) | 200ms first, continuous |
| Query with no results | 10s wait, then empty | 500ms "no results" |

Users can start analyzing results immediately. If the first results aren't what they want, they can refine the query without waiting for completion.

**Commits:**
- `b8f5ea5e` - Introduce SSE to log search API (+647 lines)
- `87ddac75` - Implement log search row-by-row streaming
- `733c20da` - Use compression on log search streaming
- `f226ad5d` - Add streaming log search service (frontend)
- `f3b14d76` - UX fixes for log search streaming

**Impact:** Time-to-first-result improved from 5-30s to <500ms; eliminated timeout errors for large result sets.

---

### 4. VList Virtual Rendering Engine — **HIGH IMPACT**

**Problem:** Rendering large log result sets (10K+ rows) caused browser jank, memory bloat, and UI freezes.

**Solution:** Built a custom virtual list component that renders only visible items, with smart prerendering, anchoring strategies, and Fenwick tree-based offset calculations.

**Key Changes:**
- Lazy rendering that only renders required React elements
- Fenwick tree for O(log n) offset calculations
- Smart prerendering during browser idle time
- Anchoring strategies for smooth scrolling in both directions
- Support for external scroll elements (e.g., window virtualization)
- Can now render up to 300K items smoothly

#### The Problem with Naive Virtual Lists

Most virtual list implementations use a simple approach:
```javascript
// Naive: O(n) to find item at scroll position
function getItemAtOffset(offset) {
    let cumulative = 0;
    for (let i = 0; i < items.length; i++) {
        cumulative += items[i].height;
        if (cumulative > offset) return i;
    }
}
```

With 300K items of variable height, this becomes a performance nightmare—every scroll event triggers O(n) calculations.

#### The Fenwick Tree Solution

We implemented a [Fenwick Tree](https://en.wikipedia.org/wiki/Fenwick_tree) (Binary Indexed Tree) for O(log n) prefix-sum queries:

```typescript
export class FenwickTree {
  private tree: number[];
  private vals: number[];
  private _total: number = 0;

  /** Sum of values[0..index) — O(log n) */
  prefixSum(index: number): number {
    let sum = 0;
    for (let i = index; i > 0; i -= i & -i) {
      sum += this.tree[i];
    }
    return sum;
  }

  /**
   * Find the largest index where prefixSum(index) <= target — O(log n)
   * This is the key operation for "which item is at scroll position Y?"
   */
  lowerBound(target: number, stride: number = 0): number {
    let pos = 0;
    let sum = 0;
    for (let bit = this._highBit; bit > 0; bit >>= 1) {
      const next = pos + bit;
      if (next <= this.vals.length &&
          sum + this.tree[next] + next * stride <= target) {
        pos = next;
        sum += this.tree[next];
      }
    }
    return pos;
  }

  /** Update single item height — O(log n) */
  set(index: number, value: number): void {
    const delta = value - this.vals[index];
    if (delta === 0) return;
    this.vals[index] = value;
    this._total += delta;
    for (let i = index + 1; i <= this.vals.length; i += i & -i) {
      this.tree[i] += delta;
    }
  }
}
```

The magic of `i & -i`: This bit trick isolates the lowest set bit, which determines the tree structure. It's what makes updates and queries O(log n).

#### Smart Prerendering During Idle Time

We don't just render visible items—we prerender nearby items during browser idle time:

```typescript
// Request idle callback for prerendering
requestIdleCallback(() => {
  const prerenderRange = calculatePrerenderRange(visibleRange);
  prerenderItems(prerenderRange);
}, { timeout: 100 });
```

This means when users scroll, the items are often already rendered, making scrolling buttery smooth.

#### Memory Comparison

| Items | Naive DOM Rendering | VList |
|-------|---------------------|-------|
| 1,000 | 50MB | 5MB |
| 10,000 | 500MB | 8MB |
| 100,000 | Browser crash | 15MB |
| 300,000 | N/A | 25MB |

**Commits:**
- `7b91b8bf` - Log table and VList performance improvements (+2,889 lines)
- `cf97d41c` - Implement custom fast table component for log search (+943 lines)
- `3513dba7` - Add VList inspector page and improve performance
- `1b6861f0` - VList performance improvements and component tests
- `05f7ef6d` - Implement lazy rendering for VList

**Impact:** UI now handles 100K+ log lines without jank; memory usage reduced by 80% for large result sets.

---

### 5. Query Merging for Metric Graphs — **MEDIUM-HIGH IMPACT**

**Problem:** Dashboard pages with multiple graphs sent individual queries for each metric, causing N round trips and redundant data scanning.

**Solution:** Merge compatible metric queries into single ClickHouse requests, reducing round trips and leveraging shared partition scans.

**Key Changes:**
- Query parameter to control merging behavior
- Server timing headers to expose per-query performance
- Metric evaluator that batches compatible queries

#### The Problem: Death by a Thousand Queries

A typical dashboard might show:
- CPU usage by host (5 hosts)
- Memory usage by host (5 hosts)
- Request latency p50, p95, p99 (3 metrics)
- Error rate by service (10 services)

That's 23 separate queries! Each one:
1. Makes a network round trip
2. Scans overlapping time partitions
3. Competes for ClickHouse resources

#### The Solution: Query Batching by Merge Key

We identify queries that differ only in the metric name but share everything else:

```go
// metricMergeKey computes a key that identifies queries that can be merged
func metricMergeKey(metricResult *extractor.MetricResult, qp FormulaParameters) string {
    groupBys := make([]string, len(metricResult.GroupBy()))
    copy(groupBys, metricResult.GroupBy())
    sort.Strings(groupBys)

    var filters []string
    for _, f := range metricResult.Filters() {
        filters = append(filters, fmt.Sprintf("%s=%s", f.Field, f.Value))
    }
    sort.Strings(filters)

    return fmt.Sprintf("%s|%s|%s|%s|%v|%v|%v|%s|%d",
        metricResult.Aggregation,    // e.g., "avg"
        strings.Join(filters, ","),  // e.g., "env=prod"
        strings.Join(groupBys, ","), // e.g., "host"
        metricResult.Rollup,         // e.g., "1m"
        qp.From.UnixNano(),
        qp.To.UnixNano(),
        qp.Rollup,
        qp.Order,
        qp.Limit,
    )
}
```

Queries with the same merge key get batched into a single DB call:

```sql
-- Before: 5 separate queries
SELECT avg(cpu_usage) FROM metrics WHERE host='h1' GROUP BY time
SELECT avg(cpu_usage) FROM metrics WHERE host='h2' GROUP BY time
-- ... 3 more

-- After: 1 merged query
SELECT host, avg(cpu_usage) FROM metrics
WHERE host IN ('h1','h2','h3','h4','h5')
GROUP BY host, time
```

#### Real-World Impact

| Dashboard Type | Queries Before | Queries After | Reduction |
|----------------|---------------|---------------|-----------|
| Host Overview (10 hosts) | 30 | 6 | 80% |
| Service Dashboard | 45 | 12 | 73% |
| Multi-metric comparison | 20 | 4 | 80% |

**Commits:**
- `6811eb9e` - Speed-up metric graph requests by merging multiple queries (+4,705 lines)
- `fb7fab68` - Merge multiple graph queries together

**Impact:** Dashboard load times reduced by 40-60% for pages with multiple graphs.

---

### 6. Query Profiler and Visibility Tooling — **MEDIUM IMPACT (ENABLER)**

**Problem:** Engineers couldn't see query execution details, making it impossible to identify bottlenecks or validate optimizations.

**Solution:** Built comprehensive query profiling UI that shows query plans, execution timing, table access patterns, and routing decisions.

**Key Changes:**
- Log search profile button with full query breakdown
- EXPLAIN button for query plans
- Schema fetching for query profiler
- TTFI (Time-To-First-Item) metric in waterfall view
- AI analysis of query log profiles
- Downloadable HTML reports for sharing

#### What the Profiler Shows

Every log search now has a "Profile" button that reveals:

```
┌─────────────────────────────────────────────────────────────────┐
│ Query Profile: "ed.tag:api-gateway error"                       │
├─────────────────────────────────────────────────────────────────┤
│ Routing Decision: ordered_table (rare_token_large_scan)         │
│ Table: ed_log_org123_tag_ordered                                │
│                                                                 │
│ Tokens Extracted: ["api", "gateway", "error"]                   │
│ Rarest Token: "gateway" (12,847 occurrences in scope)           │
│                                                                 │
│ ──────────────── Timing Waterfall ────────────────              │
│ Parse CQL:        ████ 12ms                                     │
│ Route Decision:   ██ 5ms                                        │
│ Token Stats:      ████████ 45ms                                 │
│ Execute Query:    ████████████████████████████████ 234ms        │
│ Serialize:        ██ 8ms                                        │
│ ────────────────────────────────────────────────                │
│ TTFI:             89ms (time to first item)                     │
│ Total:            304ms                                         │
│                                                                 │
│ [View EXPLAIN Plan] [Download HTML] [AI Analysis]               │
└─────────────────────────────────────────────────────────────────┘
```

#### The EXPLAIN Plan View

Clicking "View EXPLAIN Plan" shows the actual ClickHouse execution plan:

```sql
EXPLAIN PIPELINE
SELECT * FROM ed_log_org123_tag_ordered
PREWHERE hasToken(ibody, 'api') AND hasToken(ibody, 'gateway')
WHERE position(body, 'error') > 0
LIMIT 1000

-- Output:
(Expression)
  (SettingQuotaAndLimits)
    (ReadFromMergeTree)
      Indexes:
        MinMax: 847 parts → 234 parts (72% filtered)
        Partition: 234 parts → 89 parts (62% filtered)
        PrimaryKey: 89 parts → 23 parts (74% filtered)
        Skip (bloom_filter): 23 parts → 4 parts (83% filtered)
```

This makes it immediately obvious:
- Is the bloom filter being used? (Skip index filtering)
- How much data is being scanned?
- Which indexes are helping?

#### Query-Session Correlation

Every query now includes a comment that links it to the user session:

```sql
/* session_id=abc123 user_id=user@example.com query_purpose=log_search */
SELECT ...
```

This allows us to find any query in ClickHouse's `system.query_log` and correlate it back to user actions.

**Commits:**
- `c256cecf` - Introduce log search profile button (+1,786 lines)
- `7f828bc0` - Add explain button for query plans
- `7edf665e` - Introduce schema fetching for query profiler
- `cbf89eff` - Add TTFI metric for query profiler waterfall
- `d3c86795` - Add AI analysis query log profiles
- `735b0187` - Add log comment section to correlate queries to session

**Impact:** Enabled all other optimizations by making query performance visible; reduced debugging time from hours to minutes.

---

## Summary

These six optimizations transformed our log search from timing out on large queries to delivering sub-second results. The key insight wasn't any single technique—it was the combination of making performance visible through tooling, then systematically attacking bottlenecks at every layer: smarter query routing with bloom filters, tiered data architecture for different access patterns, streaming to eliminate perceived latency, virtual rendering to handle massive result sets, query batching to reduce round trips, and profiling to keep us honest. Time-to-first-result improved from 5-30 seconds to under 500ms. Large query timeouts dropped by 95%. The UI now handles 300K log lines without breaking a sweat. Most importantly, we built the infrastructure to keep iterating—because performance is never "done," it's a continuous battle that requires constant visibility.
