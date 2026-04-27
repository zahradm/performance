# Caching with Microservices: Patterns, Tradeoffs, and Stampedes

## Part 1: What is Caching?

Caching is simple: **store copies of data in fast places** so you don't have to fetch from slow places every time.

```
WITHOUT CACHE:
User request → Database (200ms) → Response (200ms)
At 100 requests/sec: 20 database hits/sec

WITH CACHE:
User request → Redis (1ms) → Response (1ms)
At 100 requests/sec: 100 cache hits/sec, occasional database hit

Benefit: 200x faster, database barely gets hit
```

But in microservices, caching is **much harder** because:
1. Multiple services have overlapping data (consistency nightmare)
2. Caches can become "stale" (out of date)
3. Cache stampedes can crash your system
4. Network latency adds overhead to cache operations

This guide teaches you how to cache correctly in microservices.

## Part 2: Types of Caches

### 2.1 Local Process Cache (L1 Cache)

Data stored **in your application's memory**.

**Pros:**
- Fastest (no network, just RAM access)
- Simple to implement (just a HashMap)

**Cons:**
- Not shared with other instances (if you have 10 servers, each has own copy)
- Must be small (limited RAM per process)
- Cluster invalidation is hard

**Example:**
```java
Map<String, Product> cache = new ConcurrentHashMap<>();

public Product getProduct(String id) {
    return cache.computeIfAbsent(id, productId -> {
        return db.getProduct(productId);
    });
}

// With 10 server instances:
// Product cached on Server 1
// Update product in database
// Servers 2-10 still have old version (inconsistent!)
```

**Best for:** Immutable or rarely-changing data (configurations, static content)

### 2.2 Distributed Cache (L2 Cache)

Data stored **in shared service** (Redis, Memcached) accessible from all instances.

**Pros:**
- Shared across all server instances (consistency)
- Can be larger (dedicated cache server)
- Fast network access (1-5ms)

**Cons:**
- Slower than L1 (network latency)
- Network can be unreliable
- Cache server is single point of failure

**Example:**
```java
@Autowired RedisTemplate redis;

public Product getProduct(String id) {
    // Try cache first
    Product cached = redis.opsForValue().get("product:" + id);
    if (cached != null) {
        return cached;  // Cache hit - 2ms
    }
    
    // Cache miss - go to database
    Product product = db.getProduct(id);
    redis.opsForValue().set("product:" + id, product, Duration.ofHours(1));
    return product;  // Cache miss - 200ms
}

// With 10 server instances:
// Any instance can update cache
// All instances see same data (consistent!)
```

**Best for:** Data that's shared across instances, needs consistency

### 2.3 CDN Cache (L3 Cache)

Data stored **at geographic edge locations** near users.

**Pros:**
- Very fast (cached near user, minimal latency)
- Reduces load on main servers (content served from edge)
- Distributed (survives single server failure)

**Cons:**
- Harder to invalidate (must propagate to all edges)
- Only works for static content (images, JS, CSS)
- Slight staleness is normal (eventual consistency)

**Best for:** Static assets, immutable content

## Part 3: Cache Invalidation Strategies

The hardest part of caching: **keeping cache in sync with data**.

### 3.1 Time-Based Invalidation (TTL)

Cache expires after fixed duration.

```java
redis.opsForValue().set("user:123", user, Duration.ofMinutes(5));

// After 5 minutes: key is deleted, next request hits database
```

**Pros:**
- Simple to implement
- No complex coordination needed

**Cons:**
- Stale data possible (up to 5 minutes old)
- If data changes, cache keeps old version
- If data doesn't change, cache could live longer

**Best for:** Data that changes slowly (user profile, product price)

### 3.2 Event-Based Invalidation

When data changes, immediately delete cache.

```
UPDATE users SET name = 'John' WHERE id = 123
→ Trigger: DELETE FROM cache WHERE key = 'user:123'

Next request:
→ Cache miss, fetch from database
→ Re-populate cache with fresh data
```

**Problem: Distributed Invalidation**

In microservices, your update might happen in one service, but cache is invalidated in another:

```
Service A (User Service):
- Update user in database
- Publish event: "UserUpdated:123"

Service B (Product Service):
- Has local cache with user info
- Receives UserUpdated:123 event
- Deletes cache entry
- Next request fetches fresh data
```

**Implementation:**

```java
// Service A: User Service
@PostMapping("/users/{id}")
public User updateUser(@PathVariable String id, @RequestBody User user) {
    db.update(user);
    
    // Publish event for other services
    kafkaTemplate.send("user-events", new UserUpdatedEvent(id, user));
    
    return user;
}

// Service B: Any service caching user data
@KafkaListener(topics = "user-events")
public void handleUserUpdated(UserUpdatedEvent event) {
    // Invalidate cache when user is updated
    cache.delete("user:" + event.userId);
}
```

**Pros:**
- Data is never stale
- Consistent across all services

**Cons:**
- Requires event infrastructure (Kafka, RabbitMQ)
- Coordinating events across services is complex
- If event is lost, cache becomes stale

**Best for:** Critical data that must be current (user permissions, inventory)

### 3.3 Lazy Invalidation

Cache serves stale data but is aware it might be stale. Validation happens when accessed.

```java
class CachedUser {
    User data;
    Instant lastUpdated;
    
    public boolean isStale() {
        return Duration.between(lastUpdated, now()).seconds() > 300;
    }
}

public User getUser(String id) {
    CachedUser cached = cache.get("user:" + id);
    
    if (cached == null || cached.isStale()) {
        // Cache miss or stale - fetch from database
        // But include version in database: getIfChanged(id, cached.version)
        User fresh = db.getUser(id);
        cache.set("user:" + id, fresh);
        return fresh;
    }
    
    return cached.data;
}
```

**Pros:**
- Balances freshness and performance
- Gracefully handles eventual consistency

**Cons:**
- Some requests see slightly stale data
- Still requires database hits

**Best for:** Data where slight staleness is acceptable (analytics, recommendations)

## Part 4: Cache Stampede Problem

### The Problem

```
Scenario: Popular product (gets 1000 requests/sec)
- Cache expires at time T
- 1000 requests arrive at time T+1ms
- Cache miss for all 1000
- All 1000 ask database for product
- Database gets hammered with 1000 identical queries

Database load:
- Normal: 2 queries/sec (cache hit rate 99.8%)
- Cache expires: 1000 queries/sec
- Database melts (not designed for 500x traffic spike)
```

### Why It Happens

```
Time 0:  Product cached (TTL = 60 seconds)
Time 30: 1000 users hit cache (1ms response)
Time 60: Cache expires
Time 60.001: 100 requests arrive simultaneously
             All miss cache
             All query database
             Database gets hammered
```

### Solution 1: Probabilistic Expiration

Instead of all items expiring at once, spread expiration.

```java
// Don't use fixed TTL
redis.opsForValue().set("product:123", product, Duration.ofSeconds(60));

// Use variable TTL
int baseSeconds = 60;
int jitter = random(0, 20);  // 0-20 seconds variation
redis.opsForValue().set("product:123", product, Duration.ofSeconds(baseSeconds + jitter));

// Result:
// Product A expires at 60s
// Product B expires at 72s
// Product C expires at 65s
// Expiration is spread out, no stampede
```

### Solution 2: Lazy Expiration (Refresh Before Expire)

Start refreshing cache **before** it expires.

```java
class CacheEntry<T> {
    T data;
    Instant createdAt;
    
    public boolean shouldRefresh() {
        // If older than 80% of TTL, start refreshing
        return Duration.between(createdAt, now()).seconds() > (60 * 0.8);
    }
    
    public boolean isExpired() {
        return Duration.between(createdAt, now()).seconds() > 60;
    }
}

public Product getProduct(String id) {
    CacheEntry cached = cache.get("product:" + id);
    
    if (cached == null) {
        // First request - load from DB
        return loadAndCache(id);
    }
    
    if (cached.shouldRefresh()) {
        // Proactively refresh in background
        executor.execute(() -> {
            Product fresh = db.getProduct(id);
            cache.set("product:" + id, fresh);
        });
        
        // Return stale data immediately
        return cached.data;
    }
    
    return cached.data;
}

// Result:
// At T=48s: Cache is refreshed in background
// At T=60s: Fresh data is in cache
// At T=60.001: New request gets fresh data
// No stampede!
```

### Solution 3: Lock + Reload

Only one request fetches from database, others wait.

```java
ConcurrentHashMap<String, ReentrantLock> locks = new ConcurrentHashMap<>();

public Product getProduct(String id) {
    Product cached = cache.get("product:" + id);
    
    if (cached == null) {
        // Cache miss - get lock for this product
        ReentrantLock lock = locks.computeIfAbsent(id, k -> new ReentrantLock());
        
        lock.lock();
        try {
            // Check again (another thread might have loaded)
            cached = cache.get("product:" + id);
            if (cached == null) {
                // Load from database once
                Product product = db.getProduct(id);
                cache.set("product:" + id, product);
                cached = product;
            }
        } finally {
            lock.unlock();
        }
    }
    
    return cached;
}

// Result:
// 1000 requests arrive simultaneously
// First request: Gets lock, loads from DB, puts in cache
// Requests 2-1000: Wait for lock, then return cached data
// Database gets 1 query instead of 1000
```

## Part 5: Cache Coherence in Microservices

When one service updates data, other services' caches must know about it.

### Scenario: Order Service Updates Inventory

```
Order Service: User buys item
  → Update inventory count in Inventory Service
  → Inventory Service updates database
  → But Product Service has cached inventory count

Problem: Product Service returns stale inventory
         Customer sees "In Stock" but item is actually sold out
```

### Solution: Event-Driven Invalidation

```
1. Order Service: receives purchase
   - Calls Inventory Service API

2. Inventory Service: updates inventory
   - Publishes "InventoryChanged" event to Kafka

3. Product Service: listening to events
   - Receives InventoryChanged event
   - Deletes product cache entry
   - Next request fetches fresh inventory from Inventory Service

Code:

// Inventory Service
@PostMapping("/inventory/{productId}/purchase")
public void purchase(@PathVariable String productId, int quantity) {
    db.updateInventory(productId, -quantity);
    kafkaTemplate.send("inventory-events", 
        new InventoryChangedEvent(productId));
}

// Product Service
@KafkaListener(topics = "inventory-events")
public void handleInventoryChanged(InventoryChangedEvent event) {
    cache.delete("product:" + event.productId);
}
```

## Part 6: Cache Strategies for Different Data Types

### Immutable Data (Product Catalog)

```java
// Cache forever (or very long TTL)
redis.opsForValue().set("product:123", product, Duration.ofDays(30));

// Safe because data never changes
// If it does change, manually invalidate:
// cache.delete("product:123");
```

### Frequently Changing Data (User Profile)

```java
// Short TTL + time-based invalidation
redis.opsForValue().set("user:456", user, Duration.ofMinutes(5));

// Or event-based invalidation
// When user updates profile, delete cache

// Or combination:
redis.opsForValue().set("user:456", user, Duration.ofMinutes(5));
kafkaTemplate.send("user-events", new UserUpdatedEvent(userId));
// Invalidate immediately on update, or wait for TTL
```

### Real-Time Data (Live Inventory)

```java
// Don't cache, or very short TTL
redis.opsForValue().set("inventory:789", count, Duration.ofSeconds(10));

// Or calculate on demand (no cache)
// High database load but data is always fresh

// Best: Use database row-level locking
// Customer A: SELECT count FROM inventory WHERE id=789 FOR UPDATE
// Ensures atomic read-modify-write, no stale data
```

## Part 7: Cache Monitoring

Know when cache is working:

```
Metrics to track:
- Hit rate: % of requests served from cache (target: >95%)
- Miss rate: % of requests needing database (target: <5%)
- Eviction rate: How often entries are removed (target: low)
- Memory usage: % of cache capacity (target: <80%)
- Stale data incidents: Times when stale data caused problems (target: 0)

Example report:
Hit rate: 98.2% (good, cache working)
Miss rate: 1.8% (normal, new products)
Database load: 20 queries/sec (low, cache is protecting)
Cache size: 2.3 GB of 4 GB (healthy)
Stale data incidents: 0 (good)
```

## Part 8: Interview Q&A

**Q: When should you NOT use caching?**

A: Don't cache:
1. Data that's accessed once then never again (waste of cache)
2. Data that must always be current (financial transactions, inventory)
3. Data too large to cache efficiently
4. When network latency to cache is high (local process cache is better)

**Q: How do you prevent cache stampede?**

A: Use one or more of:
1. Probabilistic expiration (spread TTL across time)
2. Lazy expiration (refresh before expiring)
3. Locks (only one thread loads on cache miss)
4. Circuit breaker (if database is down, serve stale data)

**Q: What's the difference between cache invalidation and eviction?**

A: 
- Invalidation: Manually delete cache when data changes
- Eviction: Automatically remove old entries when cache is full

Invalidation is intentional; eviction is automatic when space is needed.

**Q: How do you handle cache in a distributed system?**

A: Use event-driven invalidation:
1. Service A updates data, publishes event
2. All other services receive event, delete their cache
3. Next request gets fresh data
4. Ensures consistency across services

**Q: Is it better to cache at the client, server, or CDN?**

A: Depends:
- Client cache: Fastest for that user, but not shared
- Server cache: Shared, faster than database
- CDN cache: Shared geographically, but for static content only
- Usually use all three: client → server → database
