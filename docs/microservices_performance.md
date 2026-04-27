# Microservices Performance: Distributed Complexity

## Part 1: Why Microservices Are Hard to Optimize

Single monolithic application:
```
User Request → [Single Server] → Database → Response (100ms)
```

Microservices application:
```
User Request → [API Gateway] → [User Service] → [Order Service] → [Inventory Service]
                                     ↓              ↓                    ↓
                               [User DB]     [Order DB]           [Inventory DB]
                               
Response time = latency of ALL services COMBINED + network overhead
```

Performance challenges:
1. **Network latency adds up** - Every service call adds 10-100ms
2. **Cascading failures** - If one service is slow, whole system is slow
3. **Distributed tracing is hard** - Where is request getting slow?
4. **Thundering herd** - If service A is slow, services B, C, D all pile requests on it

## Part 2: Latency Composition

Understanding where time is spent in a request.

### Example: Simple Checkout Request

```
User clicks "Buy Now"
    ↓
API Gateway → Route to Order Service (5ms network)
    ↓
Order Service: Validate order (10ms compute)
    ↓
Call User Service (50ms network + 50ms processing)
    ↓
Call Inventory Service (50ms network + 100ms processing)
    ↓
Call Payment Service (50ms network + 200ms processing)
    ↓
Update Order DB (50ms network + 50ms database write)
    ↓
Return response (5ms)

Total: 5 + 10 + 50 + 50 + 50 + 100 + 50 + 200 + 50 + 50 + 5 = 620ms

Real perception: User waits for response, nothing cached = 620ms
```

### Latency Breakdown

```
Network: 5 + 50 + 50 + 50 + 5 = 160ms (26%)
User Service: 50ms (8%)
Inventory Service: 100ms (16%)
Payment Service: 200ms (32%)
Database: 50ms (8%)
Compute: 10ms (2%)
Other: 50ms (8%)

Bottleneck: Payment Service (32% of latency)
```

**What to optimize first?**
1. Payment Service (32%) - biggest bang for buck
2. Inventory Service (16%)
3. Network calls (26%) - parallel them if possible

## Part 3: Parallelization

If service calls are independent, run them in parallel.

### Sequential Execution (Slow)

```
Timeline:
T=0:    Call User Service
T=50:   Response from User Service
        Call Inventory Service
T=150:  Response from Inventory Service
        Call Payment Service
T=350:  Response from Payment Service
        Update DB
T=400:  Return to client

Total: 400ms
```

### Parallel Execution (Fast)

```
Timeline:
T=0:    Call User Service
        Call Inventory Service
        Call Payment Service
        All in parallel!
        
T=50ms: User Service returns
T=100ms: Inventory Service returns
T=200ms: Payment Service returns
T=250ms: Update DB (using results from all services)
T=300ms: Return to client

Total: 300ms (25% faster)
```

**Implementation:**

```java
// Sequential (blocking)
public CheckoutResponse checkout(Order order) {
    User user = userService.getUser(order.userId);  // 50ms wait
    Inventory inv = inventoryService.check(order.items);  // 100ms wait
    Payment payment = paymentService.authorize(order.amount);  // 200ms wait
    
    // Now have all info
    saveOrder(order, user, inv, payment);
    return response;
}

// Parallel (async)
public CheckoutResponse checkout(Order order) {
    CompletableFuture<User> userFuture = 
        CompletableFuture.supplyAsync(() -> userService.getUser(order.userId));
    CompletableFuture<Inventory> invFuture = 
        CompletableFuture.supplyAsync(() -> inventoryService.check(order.items));
    CompletableFuture<Payment> paymentFuture = 
        CompletableFuture.supplyAsync(() -> paymentService.authorize(order.amount));
    
    // Wait for all to complete
    User user = userFuture.join();
    Inventory inv = invFuture.join();
    Payment payment = paymentFuture.join();
    
    saveOrder(order, user, inv, payment);
    return response;
}

// Improvement: 400ms → 250ms (37% faster!)
```

## Part 4: Timeouts (Preventing Cascade Failures)

If one service hangs, it shouldn't bring down entire system.

### Without Timeout

```
Client calls Payment Service
Payment Service hangs (database is down)
Client waits forever (or for socket timeout, 30+ seconds)
Meanwhile:
  - Client thread is blocked
  - 1000 clients × 30 seconds = 30,000 thread-seconds of CPU
  - Thread pool exhausted
  - Order Service can't handle new requests
  - Everything fails

Cascade failure: 1 service down → entire system down
```

### With Timeout

```
Client calls Payment Service (with 2 second timeout)
Payment Service hangs
After 2 seconds: Timeout exception
Client returns error immediately
Thread is released to handle other requests

Result: 1 service slow → users see "Payment unavailable"
        But rest of system keeps working
```

**Implementation:**

```java
// With timeout
@HystrixCommand(commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "2000")
})
public Payment authorize(Order order) {
    return paymentService.authorize(order);
}

// If payment service doesn't respond in 2 seconds, throw exception
// This timeout is caught, fallback is used, request completes
```

## Part 5: Circuit Breaker (Preventing Overwhelming Failing Service)

If a service is failing, don't keep sending it requests.

### Without Circuit Breaker

```
Payment Service starts failing (10% error rate)
Client keeps retrying
After 1000 requests:
  - 100 fail (wasted)
  - Payment Service is even more overloaded
  - Error rate goes to 30%
  - More clients retry
  - Error rate goes to 50%
  
Cascade: Service gets slower, more retries, more slowness
```

### With Circuit Breaker

```
Payment Service fails (10% error rate)
After 5 failures:
  - Circuit breaker opens
  - Subsequent requests fail fast (no network call)
  - Error returned immediately
  
Result:
  - Payment Service gets respite (no new requests)
  - Clients fail fast (2ms error) instead of 2 sec timeout
  - Both recover faster
  
Circuit breaker states:
- CLOSED: Normal (requests go through)
- OPEN: Failing (requests fail immediately)
- HALF-OPEN: Recovering (allow test requests through)
```

**Implementation:**

```java
@HystrixCommand(
    fallbackMethod = "paymentAuthorizeFallback",
    commandProperties = {
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "5")
    }
)
public Payment authorize(Order order) {
    return paymentService.authorize(order);
}

public Payment paymentAuthorizeFallback(Order order) {
    // Return cached payment or placeholder
    return new Payment(order.id, Payment.PENDING);
}
```

## Part 6: Caching in Microservices

Every redundant network call is latency.

### Example: User Information

```
Current flow:
Order Service → User Service → User Database
                               (response: 50ms)
                               
With 1000 concurrent orders:
1000 requests to User Service per second
Database: 50ms response time
Queue forms, cascade failure

Solution: Cache user data in Order Service
Order Service → (Check local cache)
             → If miss, call User Service (50ms)
             → Cache for 5 minutes
             → Next 100 requests: 0.5ms (cache hit)
             
Throughput improvement:
- Without cache: 50 requests/sec
- With cache: 1000+ requests/sec (50x!)
```

But be careful: **Cache invalidation is hard in microservices**

```
User updates name in User Service
Order Service still has old cached name
Order emails customer with wrong name

Solution: Event-based invalidation
User Service publishes "UserUpdated" event
Order Service listens, clears cache
Next request gets fresh data
```

## Part 7: Database Performance in Microservices

Each service has its own database (no shared database).

```
Problem: Service A and B both need user data

Option 1: Shared database
  - Easy to implement
  - But tight coupling (Service A affects Service B's performance)
  - Bad for scale (one database bottleneck)

Option 2: Each service owns data
  - Service A owns Users table, exposes API
  - Service B calls Service A API for users
  - Or: Service B caches user data locally

Better option: Service A exposes Users API
  - Service B calls: GET /users/123 (cached locally)
  - If user updates, Service A publishes event
  - Service B invalidates cache
  - No shared database, services are independent
```

## Part 8: Monitoring Microservices Performance

Track latency at each hop:

```
Distributed tracing (using Jaeger, Zipkin):

User Request → API Gateway [5ms]
            → Order Service [100ms]
              → User Service [50ms]
                → User DB [20ms]
              → Inventory Service [80ms]
                → Inventory DB [30ms]
              → Payment Service [200ms]
                → Payment API [200ms]
              → Order DB [10ms]
            → Return response [5ms]

Total: 500ms

Can see:
- Payment Service is bottleneck (200ms)
- User DB is fast (20ms)
- Where to optimize next
```

## Part 9: Interview Q&A

**Q: How do you optimize a microservices call chain?**

A: Follow this process:
1. Measure latency of each service (distributed tracing)
2. Identify bottleneck (biggest latency)
3. Parallelize independent calls (not sequential)
4. Add caching (if appropriate)
5. Add timeout + circuit breaker (prevent failures)
6. Measure again (confirm improvement)

**Q: Why is network latency different from database latency?**

A:
- Network latency: Distance + routing + switch hops (50-100ms)
- Database latency: Query optimization + disk I/O (5-50ms)
- Network is typically slower, but less controllable

**Q: When should you use caching in microservices?**

A: Use caching for:
- Data that's read frequently, changes rarely (user profile)
- Data that's accessed by multiple services (product catalog)
- Don't cache for:
- Real-time data (live inventory)
- Personal/sensitive data (passwords)

**Q: How do you prevent cascade failures?**

A:
1. Timeout: Don't wait forever for slow service
2. Circuit breaker: Don't keep hammering failing service
3. Fallback: Have plan B when service is down
4. Bulkheads: Limit resources per service (thread pools)

All work together to isolate failures.
