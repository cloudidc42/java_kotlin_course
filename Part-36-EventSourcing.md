# Part 36: Event Sourcing & CQRS
## ขั้นตอนที่ 2421-2490: Advanced Architecture

---

## 36.1 Event Sourcing Concepts

```
Traditional CRUD:
  State = Current values in DB
  Update = Overwrite previous state

Event Sourcing:
  State = Result of replaying ALL events
  Events are immutable, append-only
  
  Events: [
    OrderCreated(orderId=1, userId=5, total=100)
    ItemAdded(orderId=1, productId=A, qty=2)
    ItemAdded(orderId=1, productId=B, qty=1)  
    OrderConfirmed(orderId=1)
    OrderShipped(orderId=1, tracking="ABC123")
  ]
  
  Current State = replay all events

Benefits:
  ✓ Full audit trail (every change is recorded)
  ✓ Time travel (reconstruct state at any point)
  ✓ Event-driven integration (publish events)
  ✓ Better at concurrent systems
  
CQRS = Command Query Responsibility Segregation
  Command = write (creates events)
  Query   = read (from optimized read model)
  
  Write side: events → event store
  Read side:  events → projections → read DB
```

---

## 36.2 Event and Command Definitions

```java
import java.time.*;
import java.util.*;

// ====== Base Event ======
public abstract class DomainEvent {
    protected final String eventId;
    protected final String aggregateId;
    protected final Instant occurredOn;
    protected final long version;
    
    protected DomainEvent(String aggregateId, long version) {
        this.eventId = UUID.randomUUID().toString();
        this.aggregateId = aggregateId;
        this.occurredOn = Instant.now();
        this.version = version;
    }
    
    public String getEventId() { return eventId; }
    public String getAggregateId() { return aggregateId; }
    public Instant getOccurredOn() { return occurredOn; }
    public long getVersion() { return version; }
    public abstract String getEventType();
}

// ====== Order Events ======
record OrderCreatedEvent(
    String aggregateId, long version,
    String userId, String eventId, Instant occurredOn,
    double total, List<OrderLineItem> items
) extends DomainEvent(aggregateId, version) {
    OrderCreatedEvent(String orderId, long version, String userId, 
                      double total, List<OrderLineItem> items) {
        this(orderId, version, userId, UUID.randomUUID().toString(), 
             Instant.now(), total, items);
    }
    @Override public String getEventType() { return "ORDER_CREATED"; }
}

record OrderConfirmedEvent(
    String aggregateId, long version, String eventId, Instant occurredOn
) extends DomainEvent(aggregateId, version) {
    OrderConfirmedEvent(String orderId, long version) {
        this(orderId, version, UUID.randomUUID().toString(), Instant.now());
    }
    @Override public String getEventType() { return "ORDER_CONFIRMED"; }
}

record OrderShippedEvent(
    String aggregateId, long version, String eventId, Instant occurredOn,
    String trackingNumber
) extends DomainEvent(aggregateId, version) {
    OrderShippedEvent(String orderId, long version, String trackingNumber) {
        this(orderId, version, UUID.randomUUID().toString(), Instant.now(), trackingNumber);
    }
    @Override public String getEventType() { return "ORDER_SHIPPED"; }
}

record OrderCancelledEvent(
    String aggregateId, long version, String eventId, Instant occurredOn,
    String reason
) extends DomainEvent(aggregateId, version) {
    OrderCancelledEvent(String orderId, long version, String reason) {
        this(orderId, version, UUID.randomUUID().toString(), Instant.now(), reason);
    }
    @Override public String getEventType() { return "ORDER_CANCELLED"; }
}

record OrderLineItem(String productId, String productName, int quantity, double price) {}

// ====== Commands ======
sealed interface OrderCommand {
    record CreateOrder(String userId, List<OrderLineItem> items) implements OrderCommand {}
    record ConfirmOrder(String orderId) implements OrderCommand {}
    record ShipOrder(String orderId, String trackingNumber) implements OrderCommand {}
    record CancelOrder(String orderId, String reason) implements OrderCommand {}
}
```

---

## 36.3 Aggregate (Write Model)

```java
import java.util.*;

public class OrderAggregate {
    
    public enum Status { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
    
    private String id;
    private String userId;
    private Status status;
    private List<OrderLineItem> items = new ArrayList<>();
    private double total;
    private String trackingNumber;
    private long version = 0;
    
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();
    
    // ====== Constructor from scratch (new aggregate) ======
    private OrderAggregate() {}
    
    // ====== Reconstitute from event history ======
    public static OrderAggregate reconstitute(String id, List<DomainEvent> events) {
        OrderAggregate order = new OrderAggregate();
        order.id = id;
        events.forEach(order::apply);
        return order;
    }
    
    // ====== Command handlers ======
    public static OrderAggregate handle(OrderCommand.CreateOrder cmd) {
        OrderAggregate order = new OrderAggregate();
        order.id = UUID.randomUUID().toString();
        
        double total = cmd.items().stream()
            .mapToDouble(item -> item.price() * item.quantity())
            .sum();
        
        order.raise(new OrderCreatedEvent(order.id, 1, cmd.userId(), total, cmd.items()));
        return order;
    }
    
    public void handle(OrderCommand.ConfirmOrder cmd) {
        if (status != Status.PENDING) {
            throw new IllegalStateException("Can only confirm PENDING orders");
        }
        raise(new OrderConfirmedEvent(id, version + 1));
    }
    
    public void handle(OrderCommand.ShipOrder cmd) {
        if (status != Status.CONFIRMED) {
            throw new IllegalStateException("Can only ship CONFIRMED orders");
        }
        raise(new OrderShippedEvent(id, version + 1, cmd.trackingNumber()));
    }
    
    public void handle(OrderCommand.CancelOrder cmd) {
        if (status == Status.SHIPPED || status == Status.DELIVERED) {
            throw new IllegalStateException("Cannot cancel " + status + " orders");
        }
        raise(new OrderCancelledEvent(id, version + 1, cmd.reason()));
    }
    
    // ====== Event application (mutate state) ======
    private void apply(DomainEvent event) {
        version = event.getVersion();
        switch (event) {
            case OrderCreatedEvent e -> {
                this.userId = e.userId();
                this.items = new ArrayList<>(e.items());
                this.total = e.total();
                this.status = Status.PENDING;
            }
            case OrderConfirmedEvent e -> this.status = Status.CONFIRMED;
            case OrderShippedEvent e -> {
                this.status = Status.SHIPPED;
                this.trackingNumber = e.trackingNumber();
            }
            case OrderCancelledEvent e -> this.status = Status.CANCELLED;
            default -> throw new UnsupportedOperationException("Unknown event: " + event.getEventType());
        }
    }
    
    private void raise(DomainEvent event) {
        apply(event);
        uncommittedEvents.add(event);
    }
    
    public List<DomainEvent> getUncommittedEvents() { return List.copyOf(uncommittedEvents); }
    public void markEventsAsCommitted() { uncommittedEvents.clear(); }
    
    public String getId() { return id; }
    public Status getStatus() { return status; }
    public double getTotal() { return total; }
    public String getUserId() { return userId; }
    public long getVersion() { return version; }
}
```

---

## 36.4 Event Store

```java
import java.util.*;
import java.util.concurrent.*;

// ====== Event Store interface ======
interface EventStore {
    void append(String aggregateId, List<DomainEvent> events, long expectedVersion);
    List<DomainEvent> loadEvents(String aggregateId);
    List<DomainEvent> loadEvents(String aggregateId, long fromVersion, long toVersion);
}

// ====== In-Memory Event Store ======
class InMemoryEventStore implements EventStore {
    
    private final Map<String, List<DomainEvent>> store = new ConcurrentHashMap<>();
    private final List<DomainEvent> globalLog = new java.util.concurrent.CopyOnWriteArrayList<>();
    
    @Override
    public synchronized void append(String aggregateId, List<DomainEvent> events, 
                                    long expectedVersion) {
        List<DomainEvent> existing = store.getOrDefault(aggregateId, new ArrayList<>());
        
        // Optimistic concurrency check
        long currentVersion = existing.isEmpty() ? 0 : 
            existing.get(existing.size() - 1).getVersion();
        
        if (currentVersion != expectedVersion) {
            throw new ConcurrencyException(
                "Concurrency conflict for " + aggregateId + 
                ": expected " + expectedVersion + " but was " + currentVersion
            );
        }
        
        List<DomainEvent> updated = new ArrayList<>(existing);
        updated.addAll(events);
        store.put(aggregateId, updated);
        globalLog.addAll(events);
        
        System.out.println("Stored " + events.size() + " events for " + aggregateId);
    }
    
    @Override
    public List<DomainEvent> loadEvents(String aggregateId) {
        return List.copyOf(store.getOrDefault(aggregateId, List.of()));
    }
    
    @Override
    public List<DomainEvent> loadEvents(String aggregateId, long from, long to) {
        return store.getOrDefault(aggregateId, List.of()).stream()
            .filter(e -> e.getVersion() >= from && e.getVersion() <= to)
            .toList();
    }
    
    public List<DomainEvent> getGlobalLog() { return List.copyOf(globalLog); }
}

class ConcurrencyException extends RuntimeException {
    ConcurrencyException(String msg) { super(msg); }
}
```

---

## 36.5 Projections (Read Model)

```java
import java.util.*;
import java.util.concurrent.*;

// Read model for queries
record OrderReadModel(
    String id, String userId, String status,
    double total, String trackingNumber, java.time.Instant createdAt,
    List<OrderLineItem> items, int itemCount
) {}

// Projection: listens to events and builds read model
class OrderProjection {
    
    private final Map<String, OrderReadModel> readModels = new ConcurrentHashMap<>();
    
    // Called when events are persisted
    public void on(DomainEvent event) {
        switch (event) {
            case OrderCreatedEvent e -> {
                var model = new OrderReadModel(
                    e.getAggregateId(), e.userId(), "PENDING",
                    e.total(), null, e.occurredOn(), e.items(), e.items().size()
                );
                readModels.put(e.getAggregateId(), model);
            }
            case OrderConfirmedEvent e -> {
                var existing = readModels.get(e.getAggregateId());
                if (existing != null) {
                    readModels.put(e.getAggregateId(), new OrderReadModel(
                        existing.id(), existing.userId(), "CONFIRMED",
                        existing.total(), null, existing.createdAt(), 
                        existing.items(), existing.itemCount()
                    ));
                }
            }
            case OrderShippedEvent e -> {
                var existing = readModels.get(e.getAggregateId());
                if (existing != null) {
                    readModels.put(e.getAggregateId(), new OrderReadModel(
                        existing.id(), existing.userId(), "SHIPPED",
                        existing.total(), e.trackingNumber(), existing.createdAt(), 
                        existing.items(), existing.itemCount()
                    ));
                }
            }
            case OrderCancelledEvent e -> {
                var existing = readModels.get(e.getAggregateId());
                if (existing != null) {
                    readModels.put(e.getAggregateId(), new OrderReadModel(
                        existing.id(), existing.userId(), "CANCELLED",
                        existing.total(), null, existing.createdAt(), 
                        existing.items(), existing.itemCount()
                    ));
                }
            }
            default -> {}
        }
    }
    
    // Query methods
    public Optional<OrderReadModel> findById(String orderId) {
        return Optional.ofNullable(readModels.get(orderId));
    }
    
    public List<OrderReadModel> findByUserId(String userId) {
        return readModels.values().stream()
            .filter(m -> m.userId().equals(userId))
            .toList();
    }
    
    public List<OrderReadModel> findByStatus(String status) {
        return readModels.values().stream()
            .filter(m -> m.status().equals(status))
            .toList();
    }
}
```

---

## 36.6 Full Demo

```java
public class EventSourcingDemo {
    
    public static void main(String[] args) throws Exception {
        
        InMemoryEventStore eventStore = new InMemoryEventStore();
        OrderProjection projection = new OrderProjection();
        
        System.out.println("=== Event Sourcing Demo ===\n");
        
        // ====== Create order ======
        System.out.println("-- Creating order --");
        OrderAggregate order = OrderAggregate.handle(new OrderCommand.CreateOrder(
            "user-001",
            List.of(
                new OrderLineItem("PROD-A", "Laptop", 1, 75000),
                new OrderLineItem("PROD-B", "Mouse", 2, 1500)
            )
        ));
        
        // Save to event store
        eventStore.append(order.getId(), order.getUncommittedEvents(), 0);
        order.getUncommittedEvents().forEach(projection::on);
        order.markEventsAsCommitted();
        
        System.out.println("Order created: " + order.getId());
        System.out.println("Status: " + order.getStatus());
        System.out.println("Total: " + order.getTotal());
        
        // ====== Confirm order ======
        System.out.println("\n-- Confirming order --");
        order.handle(new OrderCommand.ConfirmOrder(order.getId()));
        eventStore.append(order.getId(), order.getUncommittedEvents(), 1);
        order.getUncommittedEvents().forEach(projection::on);
        order.markEventsAsCommitted();
        System.out.println("Status: " + order.getStatus());
        
        // ====== Ship order ======
        System.out.println("\n-- Shipping order --");
        order.handle(new OrderCommand.ShipOrder(order.getId(), "DHL-12345-TH"));
        eventStore.append(order.getId(), order.getUncommittedEvents(), 2);
        order.getUncommittedEvents().forEach(projection::on);
        order.markEventsAsCommitted();
        System.out.println("Status: " + order.getStatus());
        
        // ====== Reconstitute from events ======
        System.out.println("\n-- Reconstituting from events --");
        List<DomainEvent> events = eventStore.loadEvents(order.getId());
        OrderAggregate restored = OrderAggregate.reconstitute(order.getId(), events);
        System.out.println("Restored status: " + restored.getStatus());
        System.out.println("Events replayed: " + events.size());
        
        // ====== Time travel: what was state after event 1? ======
        System.out.println("\n-- Time Travel: state after event 1 --");
        List<DomainEvent> firstEvent = eventStore.loadEvents(order.getId(), 1, 1);
        OrderAggregate timeTravel = OrderAggregate.reconstitute(order.getId(), firstEvent);
        System.out.println("State at v1: " + timeTravel.getStatus());
        
        // ====== Query read model ======
        System.out.println("\n-- Query Read Model --");
        projection.findById(order.getId()).ifPresent(m -> {
            System.out.println("Order: " + m.id());
            System.out.println("Status: " + m.status());
            System.out.println("Tracking: " + m.trackingNumber());
        });
        
        // ====== Event log ======
        System.out.println("\n-- Event Log --");
        eventStore.getGlobalLog().forEach(e -> 
            System.out.printf("[%s] %s @ %s%n",
                e.getVersion(), e.getEventType(), e.getOccurredOn())
        );
    }
}
```

---

## สรุป Part 36

```
Event Sourcing Key Components:

1. DomainEvent     = immutable record of what happened
2. Command         = intent to change state
3. Aggregate       = apply events, enforce business rules
4. EventStore      = append-only log of events
5. Projection      = build read model from events

CQRS Flow:
  UI → Command → CommandHandler → Aggregate
                                       ↓
                                 EventStore.append()
                                       ↓
                              EventBus.publish()
                                       ↓
                              Projection.on(event)
                                       ↓
  UI ← Query ← QueryHandler ← Read Model (DB)

When to use:
  ✓ Audit requirements
  ✓ Complex domain logic
  ✓ Multi-team with bounded contexts
  
When NOT to use:
  ✗ Simple CRUD apps
  ✗ Small teams
  ✗ When simplicity matters more
```

➡️ [Part 37: Apache Kafka](./Part-37-Kafka.md)
