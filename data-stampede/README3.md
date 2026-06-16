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
