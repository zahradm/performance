# Message Brokers: RabbitMQ vs Kafka Deep Dive

## Part 1: Why Message Brokers?

Message brokers **decouple services** so they don't need to communicate directly.

```
WITHOUT Message Broker (Direct API Calls):
Order Service → Inventory Service (blocks waiting for response)
             → Payment Service (blocks waiting for response)
             → Notification Service (blocks waiting for response)

Problems:
- If Inventory Service is slow, Order Service is slow
- If Payment Service crashes, whole order process fails
- Order Service must know how to call 3 different services
- Hard to scale independently

WITH Message Broker:
Order Service → Message Broker ← Inventory Service
            ↓
         Kafka
            ↓
            → Payment Service
            → Notification Service

Benefits:
- Services don't need to know about each other
- If Notification Service is down, orders still process
- Can add new services without changing Order Service
- Each service processes messages at its own speed
```

## Part 2: Message Broker Guarantees

Different brokers provide different guarantees about message delivery and ordering.

### 2.1 At-Most-Once Delivery

Message is delivered **0 or 1 times**. Never duplicated.

```
Scenario:
1. Order Service sends "Order Created" message
2. Kafka delivers message to Payment Service
3. Before Payment Service acknowledges, message broker crashes
4. Payment Service never processed message
5. Message is lost

Result: Payment might never be charged, but duplicate charge is impossible
```

**Use when:** Losing a message is acceptable (analytics, non-critical notifications)

### 2.2 At-Least-Once Delivery

Message is delivered **1 or more times**. Might be duplicated.

```
Scenario:
1. Order Service sends "Order Created" message
2. Kafka delivers message to Payment Service
3. Payment Service processes and charges $100
4. Before Payment Service acknowledges, it crashes
5. Kafka resends message (thinks it failed)
6. Payment Service processes again and charges $100 again

Result: Customer charged twice, but message won't be lost
```

**Use when:** Duplicates are tolerable or can be detected and skipped

**How to handle duplicates:**

```java
// Payment Service
@KafkaListener(topics = "order-events")
public void handleOrderCreated(OrderCreatedEvent event) {
    // Check if we already processed this
    if (db.orderExists(event.orderId)) {
        logger.info("Order already processed, skipping: " + event.orderId);
        return;
    }
    
    // Process payment
    chargeCustomer(event.customerId, event.amount);
    
    // Record that we processed it
    db.recordProcessedOrder(event.orderId);
}
```

### 2.3 Exactly-Once Delivery

Message is delivered **exactly 1 time**. Hardest to implement.

```
Requirement:
1. Order Service sends "Order Created"
2. Payment Service processes exactly once
3. Notification Service processes exactly once
4. No duplicates, no lost messages
```

**How it works:**
- Message broker tracks which messages each consumer has processed
- Before giving message, checks: "Have you seen this before?"
- Consumer acknowledges **after** processing (not before)
- If consumer crashes mid-processing, message is redelivered

**Kafka exactly-once:**

```java
@KafkaListener(topics = "order-events")
@Transactional  // Exactly-once with database transaction
public void handleOrderCreated(OrderCreatedEvent event) {
    // All or nothing: either both happen, or neither
    chargeCustomer(event.customerId, event.amount);
    db.saveProcessedOrder(event.orderId);
    
    // If both succeed: commit
    // If either fails: rollback, message will be redelivered
}
```

## Part 3: RabbitMQ

**What it is:** Queue-based message broker. Producer sends message, consumer pulls from queue.

### Key Concepts

**Exchange:** Entry point for messages (like a mail sorting station)
- Direct exchange: Routes to specific queue
- Topic exchange: Routes based on pattern (like publish-subscribe)
- Fanout exchange: Routes to all queues (broadcast)

**Queue:** Storage for messages (like a mailbox)

**Binding:** Connects exchange to queue (like: "send sales events to sales-queue")

### Example: Order Processing

```
Order Service (Producer):
    message: {"orderId": 123, "amount": 500}
                 ↓
            RabbitMQ
                 ↓
    Exchange: "order-events" (topic)
                 ↓
    [Binding: route-key = "order.created" → queue "payment-queue"]
    [Binding: route-key = "order.created" → queue "notification-queue"]
                 ↓
    Payment Service Consumer (pulls from payment-queue)
    Notification Service Consumer (pulls from notification-queue)
```

### Configuration

```java
// Order Service (Producer)
@Configuration
public class RabbitConfig {
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-events");
    }
}

@Service
public class OrderService {
    @Autowired RabbitTemplate rabbitTemplate;
    
    public void createOrder(Order order) {
        // Save to database
        db.save(order);
        
        // Send message
        rabbitTemplate.convertAndSend(
            "order-events",
            "order.created",
            new OrderCreatedEvent(order)
        );
    }
}

// Payment Service (Consumer)
@Service
public class PaymentService {
    @RabbitListener(queues = "payment-queue")
    public void handleOrderCreated(OrderCreatedEvent event) {
        chargeCustomer(event.customerId, event.amount);
    }
}
```

### Pros
- Simple for request-response patterns
- Good for task queues (background jobs)
- Reliable delivery with acknowledgments

### Cons
- Queues don't scale well beyond one consumer (need multiple queues)
- Not designed for large-scale streaming
- Each message is stored, then removed (no history)

**Use RabbitMQ for:** Background jobs, task distribution, one-to-one or one-to-few messaging

## Part 4: Kafka

**What it is:** Event streaming platform. Producer publishes events to topics, multiple consumers read from topic.

### Key Concepts

**Topic:** A stream of events (like a TV channel)
- "order-events" topic receives all order events

**Partition:** Topic split into partitions for parallelism
- Partition 0: orders from users A-L
- Partition 1: orders from users M-Z
- Enables parallel processing

**Consumer Group:** Group of consumers reading same topic
- All consumers in group read all partitions
- Each partition read by exactly one consumer in the group

**Offset:** Position in the stream
- Kafka remembers: "Consumer read up to message 1000"
- Consumer can resume from message 1001 if it crashes

### Example: Event Streaming

```
Order Service produces:
    Event: {"orderId": 123, "customerId": 456, "amount": 500}
                 ↓
            Kafka Topic: "order-events"
           [Partition 0] [Partition 1] [Partition 2]
                 ↓
    Consumer Group A:
        - Consumer A1 reads Partition 0
        - Consumer A2 reads Partition 1
        - Consumer A3 reads Partition 2
    
    Consumer Group B:
        - Consumer B1 reads all partitions (different group)

Result:
- Multiple services can consume same events independently
- Each consumer tracks its own progress
- Events are never deleted from topic (history available)
```

### Configuration

```java
// Order Service (Producer)
@Service
public class OrderService {
    @Autowired KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;
    
    public void createOrder(Order order) {
        db.save(order);
        kafkaTemplate.send("order-events", order.getId(), 
            new OrderCreatedEvent(order));
    }
}

// Payment Service (Consumer)
@Service
public class PaymentService {
    @KafkaListener(
        topics = "order-events",
        groupId = "payment-service",
        concurrency = "3"  // 3 consumers in parallel
    )
    public void handleOrderCreated(OrderCreatedEvent event) {
        chargeCustomer(event.customerId, event.amount);
    }
}

// Notification Service (Another consumer, same events)
@Service
public class NotificationService {
    @KafkaListener(
        topics = "order-events",
        groupId = "notification-service"
    )
    public void handleOrderCreated(OrderCreatedEvent event) {
        sendEmail(event.customerId, "Order confirmed");
    }
}
```

### Pros
- Designed for streaming at scale (millions of events/sec)
- Events are stored (can replay history)
- Multiple independent consumers (different groups)
- Partitions enable parallel processing
- Excellent for event sourcing

### Cons
- More complex to understand and operate
- Higher latency than RabbitMQ (100ms+ typical)
- Requires more infrastructure
- Rebalancing when consumers join/leave (brief pause)

**Use Kafka for:** Event streaming, audit logs, data pipeline, real-time analytics

## Part 5: Comparison Table

| Feature | RabbitMQ | Kafka |
|---------|----------|-------|
| **Type** | Task queue | Event stream |
| **Message retention** | Deleted after consume | Stored (replay possible) |
| **Throughput** | 50K-100K msg/sec | 1M+ msg/sec |
| **Latency** | 10-100ms | 100ms-1s |
| **Consumer scalability** | Limited (task distribution) | Excellent (independent groups) |
| **Replay capability** | No | Yes |
| **Complexity** | Simple | Complex |
| **Best for** | Background jobs | Event streaming |
| **Failure handling** | Queue persistence | Partition replication |

## Part 6: Ordering Guarantees

### RabbitMQ Ordering

```
Single queue guarantees order:
    Message 1 → 2 → 3 → 4 → Consumer

Multiple consumers on same queue:
    Consumer A gets: 1, 3, 5 (out of order possible)
    Consumer B gets: 2, 4, 6
```

### Kafka Ordering

```
Single partition:
    Message 1 → 2 → 3 → 4 → Consumer A

Multiple partitions (multiple consumers):
    Partition 0: 1 → 5 → 9 → Consumer A
    Partition 1: 2 → 6 → 10 → Consumer B
    Partition 2: 3 → 7 → 11 → Consumer C
    
    Global order: 1 2 3 4 5 6 7 8 9...
    But order within partition is guaranteed
```

**If you need global ordering with Kafka:**
- Use single partition (lose parallelism)
- Or use key-based partitioning: all messages with same key go to same partition

```java
kafkaTemplate.send(
    "order-events",
    order.getId(),  // ← Key: ensures order for same customer
    new OrderCreatedEvent(order)
);

// Orders from customer 123 always go to same partition
// Consumer processes in order for that customer
```

## Part 7: Failure Scenarios

### Message Loss in RabbitMQ

```
Producer → RabbitMQ → Consumer
              |
           (Crash!)
              
If RabbitMQ crashes before consumer acknowledges:
- Message is lost (not persisted to disk)

Solution: Enable queue persistence
rabbitTemplate.convertAndSend(message, msg -> {
    msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
    return msg;
});
```

### Message Loss in Kafka

```
Producer → Broker 1 (Leader)
                ↓
            [Write to log]
                ↓
            Broker 2, Broker 3 (Replicas)
```

Kafka replicates to multiple brokers:
- Message is written to leader
- Leader replicates to follower brokers
- Only acknowledged when replicated to N brokers
- If leader crashes, follower becomes new leader
- Message is safe (stored on multiple brokers)

## Part 8: Interview Q&A

**Q: When would you use RabbitMQ instead of Kafka?**

A: Use RabbitMQ when:
- You need simple point-to-point messaging
- Throughput is modest (< 100K msg/sec)
- You don't need message history
- You have background jobs with dependencies
- Operational complexity should be minimal

Use Kafka when:
- You need event streaming at scale
- Multiple independent consumers need same events
- You need to replay events (audit trail)
- You need millions of events per second

**Q: How do you handle duplicate messages from a broker?**

A: Three strategies:
1. Idempotent consumer: Duplicate processing gives same result
2. Deduplication table: Track message IDs, skip duplicates
3. Exactly-once semantics: Use broker's exactly-once guarantee (complex)

**Q: What happens when a consumer crashes while processing a message?**

A:
- RabbitMQ: Message returns to queue, redelivered to another consumer
- Kafka: Consumer lag increases (other consumers in group read ahead), crashed consumer rebalanced

**Q: Why does Kafka have higher latency than RabbitMQ?**

A: Because:
- Kafka writes to disk (durable)
- RabbitMQ can keep in memory
- Kafka replicates to other brokers (network I/O)
- RabbitMQ is optimized for low latency
- Trade-off: RabbitMQ is faster, Kafka is more reliable

**Q: How do you ensure ordering in Kafka?**

A: Use key-based partitioning:
```java
kafkaTemplate.send("orders", 
    customer.getId(),  // ← Key ensures same customer's messages go to same partition
    orderEvent
);
```

Messages with same key always go to same partition, which guarantees order.
