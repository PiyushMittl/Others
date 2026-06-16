# How a Cache Can Bring Down Your Database: Understanding Cache Stampede

A cache is supposed to reduce database traffic and improve performance.

But under certain conditions, a cache can actually overwhelm your database and cause outages.

This problem is known as a **Cache Stampede**.

Imagine your application receives thousands of requests for a popular product page. To improve performance, the product information is stored in Redis with a TTL of one hour.

Everything works perfectly until the cache entry expires.

At that exact moment, thousands of users request the same product.

Since the cache is empty, every request misses the cache and attempts to fetch data from the database.

```text
Request 1 → Database
Request 2 → Database
Request 3 → Database
...
Request 10000 → Database
```

Instead of reducing database load, the cache expiration creates a sudden traffic spike.

This surge can increase latency, overload the database, and in extreme cases cause service outages.

## Solution 1: Cache Locking

One common solution is to allow only one request to rebuild the cache.

When the first request encounters a cache miss, it acquires a lock and fetches data from the database.

```text
Request 1
   ↓
Acquire Lock
   ↓
Query Database
   ↓
Update Cache
```

Meanwhile, other requests detect that a cache refresh is already in progress.

```text
Request 2
   ↓
Lock Exists
   ↓
Wait
```

```text
Request 3
   ↓
Lock Exists
   ↓
Wait
```

Once the cache is rebuilt, the waiting requests check the cache again and retrieve the data from there.

```text
Request 2
   ↓
Cache Hit
```

```text
Request 3
   ↓
Cache Hit
```

As a result, only one database query is executed instead of thousands.

## Solution 2: Serve Stale Data

Another popular approach is to serve slightly outdated data while refreshing the cache in the background.

Instead of immediately removing expired data, the system temporarily continues serving it.

```text
User Request
    ↓
Serve Stale Data
    ↓
Trigger Background Refresh
```

A background process fetches fresh data from the database and updates the cache.

Users continue receiving responses without waiting, and the database is protected from sudden traffic spikes.

This technique works particularly well for:

* Product catalogs
* Trending content
* News feeds
* Leaderboards

where a few minutes of stale data is usually acceptable.

## The Best Approach

Many large-scale systems combine both techniques.

```text
Cache Expired
      ↓
Serve Stale Data
      ↓
Acquire Lock
      ↓
Single Database Query
      ↓
Refresh Cache
```

This approach provides:

* Fast user experience
* Reduced database load
* Protection against traffic spikes
* Better system reliability

The next time you design a caching strategy, remember that the biggest challenge isn't getting data into the cache.

It's preventing thousands of users from rebuilding the same cache entry at the same time.
