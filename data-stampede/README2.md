## A Common Question: What Happens to the Waiting Requests?

When discussing cache locking, a common question is:

> If Request 2, Request 3, and thousands of other requests are waiting while Request 1 fetches data from the database, won't they eventually hit the database as well?

In a well-designed system, the answer is **no**.

The goal of cache locking is to ensure that only one request rebuilds the cache while the others reuse the result.

There are two common approaches.

### Option 1: Wait and Re-check Cache (Most Common)

The first request acquires the lock and refreshes the cache.

```text
Request 1
   ↓
Acquire Lock
   ↓
Query Database
   ↓
Update Cache
```

Meanwhile, other requests detect that a refresh is already in progress.

```text
Request 2
   ↓
Cache Miss
   ↓
Lock Exists
   ↓
Wait
```

```text
Request 3
   ↓
Cache Miss
   ↓
Lock Exists
   ↓
Wait
```

After Request 1 updates the cache, the waiting requests check the cache again.

```text
Request 2
   ↓
Check Cache Again
   ↓
Cache Hit
```

```text
Request 3
   ↓
Check Cache Again
   ↓
Cache Hit
```

As a result, only one database query is executed.

```text
10,000 Requests
       ↓
1 Database Query
       ↓
10,000 Responses
```

---

### Option 2: Wait for Request 1's Result (SingleFlight Pattern)

Another approach is called **SingleFlight**.

Instead of making waiting requests repeatedly check the cache, the system allows them to share the result of the first request.

Suppose multiple requests arrive for the same product:

```text
Request 1 -> product:iphone
Request 2 -> product:iphone
Request 3 -> product:iphone
```

Request 1 encounters a cache miss and starts a database query.

```text
Request 1
   ↓
Query Database
```

The system keeps track of in-flight requests using the cache key.

```text
{
   "product:iphone" -> Database Query In Progress
}
```

When Request 2 arrives, it notices that another request is already fetching data for the same cache key.

```text
Request 2
   ↓
Join Existing Request
   ↓
Wait
```

Request 3 behaves the same way.

```text
Request 3
   ↓
Join Existing Request
   ↓
Wait
```

Once the database query completes:

```text
Product = iPhone
```

the result is shared with all waiting requests.

```text
Request 1 ← iPhone
Request 2 ← iPhone
Request 3 ← iPhone
```

The cache is then updated for future requests.

### How Does the System Know Which Requests Can Share Results?

The grouping is typically done using the cache key.

For example:

```text
product:iphone
product:samsung
product:pixel
```

Requests for the same key share a single database query.

```text
Request 1 -> product:iphone
Request 2 -> product:iphone
Request 3 -> product:iphone
```

However, requests for different keys execute independently.

```text
Request 4 -> product:samsung
Request 5 -> product:pixel
```

This means the system may execute:

```text
1 Query for iPhone
1 Query for Samsung
1 Query for Pixel
```

instead of thousands of duplicate queries for the same resource.

### Key Takeaway

Whether the system uses:

* Wait and Re-check Cache
* SingleFlight Result Sharing

the objective remains the same:

```text
10,000 Requests
       ↓
1 Database Query
```

instead of:

```text
10,000 Requests
       ↓
10,000 Database Queries
```

A cache should reduce database traffic. The purpose of cache locking and SingleFlight is to ensure that an expired cache entry does not accidentally create a database overload.
