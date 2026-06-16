# How a Cache Can Bring Down Your Database: Understanding Cache Stampede

A cache is supposed to reduce database traffic and improve performance.

But under certain conditions, a cache can actually overwhelm your database and cause outages.

This problem is known as a **Cache Stampede** (also called **Thundering Herd**).

## The Problem

Imagine your application receives thousands of requests for a popular product page. To improve performance, the product information is stored in Redis with a TTL of one hour.

![1.cache-serving-request](https://github.com/PiyushMittl/Others/blob/main/data-stampede/images/1.cache-serving-request.png)

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

![2.data-stampede](https://github.com/PiyushMittl/Others/blob/main/data-stampede/images/2.data-stampede.png)

## Solution 1: Cache Locking

One common solution is to allow only one request to rebuild the cache while others wait.

There are two main approaches:

### Approach 1: Wait and Re-check Cache (Most Common)

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

![3.1.wait-and-recheck](https://github.com/PiyushMittl/Others/blob/main/data-stampede/images/3.1.wait-and-recheck.png)

---

### Approach 2: Wait for Request 1's Result (SingleFlight Pattern)

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

![3.2.return-1st-result](https://github.com/PiyushMittl/Others/blob/main/data-stampede/images/3.2.return-1st-result.png)

### Key Difference Between Both Approaches

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

**Result: 10,000 requests = 1 database query**

Whether using Approach 1 (Wait and Re-check) or Approach 2 (SingleFlight), the objective remains the same: prevent duplicate database queries for the same cache entry.


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

![4.serve-stale-data](https://github.com/PiyushMittl/Others/blob/main/data-stampede/images/4.serve-stale-data.png)


## Solution 3: Randomized TTL (Jitter)

A common question is:

> What if thousands of cache entries expire at exactly the same time?

Consider an application that caches product information.

Suppose all cache entries are configured with a TTL of 1 hour.

```text
product:iphone    → TTL = 1 hour
product:samsung   → TTL = 1 hour
product:pixel     → TTL = 1 hour
```

At 12:00 PM, all three cache entries expire simultaneously.

```text
12:00 PM
    ↓
product:iphone expires
product:samsung expires
product:pixel expires
```

Now imagine thousands of users requesting these products.

```text
Users
   ↓
Cache Miss
   ↓
Database
```

Suddenly, the database receives a large number of requests at the same time.

Even though each cache entry is rebuilt only once, many different cache entries are being rebuilt simultaneously.

This can still create a traffic spike on the database.

### The Solution

Instead of giving every cache entry the same TTL, add a small amount of randomness.

```text
product:iphone    → TTL = 58 minutes
product:samsung   → TTL = 62 minutes
product:pixel     → TTL = 65 minutes
```

Now the cache entries expire at different times.

```text
11:58 AM → product:iphone expires
12:02 PM → product:samsung expires
12:05 PM → product:pixel expires
```

The database workload becomes spread out over time.

Instead of seeing:

```text
12:00 PM
    ↓
100 cache entries expire
    ↓
100 database queries
```

the database sees:

```text
11:58 AM → Few queries
12:02 PM → Few queries
12:05 PM → Few queries
```

The load is distributed more evenly.

![5.put-random-ttl](https://github.com/PiyushMittl/Others/blob/main/data-stampede/images/5.put-random-ttl.png)

### Key Takeaway

Randomized TTL, also known as **Jitter**, helps prevent many cache entries from expiring at the same time.

The goal is simple:

```text
Avoid:
100 cache entries expiring together
```

and instead:

```text
Spread expirations across time
```

This reduces sudden database spikes and improves system stability.

## The Best Approach

Many large-scale systems combine all three techniques:

```text
Cache Expired
	  ↓
Serve Stale Data (Fast response)
	  ↓
Acquire Lock (Prevent duplicate queries)
	  ↓
Single Database Query
	  ↓
Refresh Cache
	  ↓
Add Random Jitter to TTL (Prevent synchronized expiration)
```

This approach provides:

* Fast user experience
* Reduced database load
* Protection against traffic spikes
* Better system reliability

## Key Takeaway

The next time you design a caching strategy, remember that the biggest challenge isn't getting data into the cache.

It's preventing thousands of users from rebuilding the same cache entry at the same time — and ensuring that cache expirations don't all happen together.

A well-designed cache protects your database. A poorly designed one can take it down.
