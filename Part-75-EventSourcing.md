# Part 75: Event Sourcing & CQRS
## ขั้นตอนที่ 5151-5220: Event Store, Projections, Eventual Consistency

---

## 75.1 Event Sourcing คืออะไร

```
Traditional (State-based):
  orders table: id=1, status=DELIVERED, total=500
  → เก็บแค่ state ปัจจุบัน
  → ประวัติหายหมด

Event Sourcing:
  events table:
    id=1, OrderCreated  {orderId=1, userId=U1, items=[...]}
    id=2, ItemAdded     {orderId=1, product=P1, qty=2}
    id=3, OrderSubmitted {orderId=1}
    id=4, PaymentCharged {orderId=1, amount=500}
    id=5, OrderDelivered {orderId=1}
  
  → State ได้จากการ replay events
  → ไม่มีข้อมูลหาย

Benefits:
  ✓ Complete audit trail
  ✓ Temporal queries ("state ของ order เมื่อ 3 วันก่อน")
  ✓ Replay events to rebuild any projection
  ✓ Event-driven natural fit
  ✓ Debugging: replay events to reproduce bug

Drawbacks:
  ✗ Complexity สูงกว่า CRUD
  ✗ Query ยากขึ้น (ต้องใช้ CQRS)
  ✗ Schema evolution (old events + new fields)

CQRS (Command Query Responsibility Segregation):
  Commands → Write Model (Event Store)
  Queries  → Read Model (Projection/Materialized View)
```

---

## 75.2 Event Store Implementation

```java
import com.fasterxml.jackson.databind.*;
import jakarta.persistence.*;
import org.springframework.stereotype.*;

// Generic Event Store
@Entity
@Table(name = "domain_events",
    indexes = {
        @Index(name = "idx_events_aggregate", columnList = "aggregate_type, aggregate_id, version"),
        @Index(name = "idx_events_created_at", columnList = "created_at")
    }
)
class DomainEventRecord {
    
    @Id
    @GeneratedValue
    private java.util.UUID id;
    
    @Column(name = "aggregate_type", nullable = false)
    private String aggregateType;
    
    @Column(name = "aggregate_id", nullable = false)
    private String aggregateId;
    
    @Column(nullable = false)
    private long version;
    
    @Column(name = "event_type", nullable = false)
    private String eventType;
    
    @Column(columnDefinition = "jsonb", nullable = false)
    private String payload;
    
    @Column(name = "occurred_at", nullable = false)
    private java.time.Instant occurredAt;
    
    @Column(name = "created_at", nullable = false)
    private java.time.Instant createdAt;
    
    // getters/constructors
}

@Repository
interface DomainEventJpaRepository extends JpaRepository<DomainEventRecord, java.util.UUID> {
    
    List<DomainEventRecord> findByAggregateTypeAndAggregateIdOrderByVersionAsc(
        String aggregateType, String aggregateId);
    
    List<DomainEventRecord> findByAggregateTypeAndAggregateIdAndVersionGreaterThanOrderByVersionAsc(
        String aggregateType, String aggregateId, long fromVersion);
    
    @Query("SELECT MAX(e.version) FROM DomainEventRecord e WHERE e.aggregateType = :type AND e.aggregateId = :id")
    Long findMaxVersion(@Param("type") String type, @Param("id") String id);
}

@Service
class EventStore {
    
    private final DomainEventJpaRepository repository;
    private final ObjectMapper objectMapper;
    
    EventStore(DomainEventJpaRepository repository, ObjectMapper objectMapper) {
        this.repository = repository;
        this.objectMapper = objectMapper;
    }
    
    @org.springframework.transaction.annotation.Transactional
    public void append(String aggregateType, String aggregateId, 
                       List<DomainEvent> events, long expectedVersion) {
        
        Long currentVersion = repository.findMaxVersion(aggregateType, aggregateId);
        long actualVersion = currentVersion != null ? currentVersion : -1;
        
        // Optimistic concurrency check
        if (actualVersion != expectedVersion) {
            throw new ConcurrencyException(
                "Expected version " + expectedVersion + " but found " + actualVersion);
        }
        
        long nextVersion = actualVersion + 1;
        for (var event : events) {
            var record = new DomainEventRecord(
                aggregateType,
                aggregateId,
                nextVersion++,
                event.getClass().getSimpleName(),
                serialize(event),
                event.occurredAt(),
                java.time.Instant.now()
            );
            repository.save(record);
        }
    }
    
    public List<DomainEvent> load(String aggregateType, String aggregateId) {
        return repository.findByAggregateTypeAndAggregateIdOrderByVersionAsc(
                aggregateType, aggregateId)
            .stream()
            .map(this::deserialize)
            .toList();
    }
    
    private String serialize(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize event", e);
        }
    }
    
    private DomainEvent deserialize(DomainEventRecord record) {
        try {
            Class<?> eventClass = Class.forName("com.example.events." + record.getEventType());
            return (DomainEvent) objectMapper.readValue(record.getPayload(), eventClass);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize event", e);
        }
    }
}
```

---

## 75.3 Aggregate with Event Sourcing

```java
// Domain Events
interface DomainEvent {
    String aggregateId();
    java.time.Instant occurredAt();
}

record OrderCreated(String orderId, String customerId, java.time.Instant occurredAt) 
    implements DomainEvent {
    @Override public String aggregateId() { return orderId; }
}

record ItemAdded(String orderId, String productId, String productName,
                 long unitPriceCents, int quantity, java.time.Instant occurredAt)
    implements DomainEvent {
    @Override public String aggregateId() { return orderId; }
}

record OrderSubmitted(String orderId, java.time.Instant occurredAt) 
    implements DomainEvent {
    @Override public String aggregateId() { return orderId; }
}

record PaymentProcessed(String orderId, String paymentId, long amountCents, 
                        java.time.Instant occurredAt)
    implements DomainEvent {
    @Override public String aggregateId() { return orderId; }
}

// Aggregate that tracks uncommitted events
class Order {
    
    private String id;
    private String customerId;
    private List<OrderItem> items = new java.util.ArrayList<>();
    private OrderStatus status;
    private long version = -1;
    
    private final List<DomainEvent> uncommittedEvents = new java.util.ArrayList<>();
    
    // ====== Static factory: create new aggregate ======
    public static Order create(String customerId) {
        var order = new Order();
        order.applyNewEvent(new OrderCreated(
            java.util.UUID.randomUUID().toString(),
            customerId,
            java.time.Instant.now()
        ));
        return order;
    }
    
    // ====== Reconstitute from events ======
    public static Order reconstitute(List<DomainEvent> events) {
        var order = new Order();
        events.forEach(order::apply);
        return order;
    }
    
    // ====== Commands ======
    public void addItem(String productId, String productName, long unitPriceCents, int quantity) {
        if (status != OrderStatus.DRAFT)
            throw new IllegalStateException("Can only add items to DRAFT orders");
        
        applyNewEvent(new ItemAdded(id, productId, productName, unitPriceCents, quantity,
            java.time.Instant.now()));
    }
    
    public void submit() {
        if (status != OrderStatus.DRAFT)
            throw new IllegalStateException("Only DRAFT orders can be submitted");
        if (items.isEmpty())
            throw new IllegalStateException("Cannot submit empty order");
        
        applyNewEvent(new OrderSubmitted(id, java.time.Instant.now()));
    }
    
    // ====== Event application (mutate state) ======
    private void applyNewEvent(DomainEvent event) {
        apply(event);
        uncommittedEvents.add(event);
    }
    
    private void apply(DomainEvent event) {
        version++;
        switch (event) {
            case OrderCreated e -> {
                this.id = e.orderId();
                this.customerId = e.customerId();
                this.status = OrderStatus.DRAFT;
            }
            case ItemAdded e -> {
                items.add(new OrderItem(e.productId(), e.productName(),
                    e.unitPriceCents(), e.quantity()));
            }
            case OrderSubmitted e -> this.status = OrderStatus.SUBMITTED;
            case PaymentProcessed e -> this.status = OrderStatus.CONFIRMED;
            default -> throw new IllegalArgumentException("Unknown event: " + event.getClass());
        }
    }
    
    public List<DomainEvent> getUncommittedEvents() { return List.copyOf(uncommittedEvents); }
    public void clearUncommittedEvents() { uncommittedEvents.clear(); }
    public String getId() { return id; }
    public long getVersion() { return version; }
}

// Repository that uses EventStore
@Service
class EventSourcedOrderRepository {
    
    private final EventStore eventStore;
    
    EventSourcedOrderRepository(EventStore eventStore) {
        this.eventStore = eventStore;
    }
    
    public Order load(String orderId) {
        var events = eventStore.load("Order", orderId);
        if (events.isEmpty()) throw new RuntimeException("Order not found: " + orderId);
        return Order.reconstitute(events);
    }
    
    @org.springframework.transaction.annotation.Transactional
    public void save(Order order) {
        var uncommitted = order.getUncommittedEvents();
        if (uncommitted.isEmpty()) return;
        
        eventStore.append("Order", order.getId(), uncommitted, 
            order.getVersion() - uncommitted.size());
        
        order.clearUncommittedEvents();
    }
}
```

---

## 75.4 CQRS Read Model (Projection)

```java
// Read Model: optimized for queries
@Entity
@Table(name = "order_view")
class OrderView {
    
    @Id
    private String orderId;
    private String customerId;
    private String status;
    private long totalCents;
    private int itemCount;
    private java.time.Instant createdAt;
    private java.time.Instant updatedAt;
    
    // getters/setters
}

// Projection: builds read model from events
@Service
@org.springframework.transaction.event.TransactionalEventListener
class OrderProjection {
    
    private final OrderViewRepository viewRepository;
    
    OrderProjection(OrderViewRepository viewRepository) {
        this.viewRepository = viewRepository;
    }
    
    @org.springframework.context.event.EventListener
    void on(OrderCreated event) {
        var view = new OrderView();
        view.setOrderId(event.orderId());
        view.setCustomerId(event.customerId());
        view.setStatus("DRAFT");
        view.setTotalCents(0);
        view.setItemCount(0);
        view.setCreatedAt(event.occurredAt());
        view.setUpdatedAt(event.occurredAt());
        viewRepository.save(view);
    }
    
    @org.springframework.context.event.EventListener
    void on(ItemAdded event) {
        viewRepository.findById(event.orderId()).ifPresent(view -> {
            view.setTotalCents(view.getTotalCents() + event.unitPriceCents() * event.quantity());
            view.setItemCount(view.getItemCount() + 1);
            view.setUpdatedAt(event.occurredAt());
            viewRepository.save(view);
        });
    }
    
    @org.springframework.context.event.EventListener
    void on(OrderSubmitted event) {
        viewRepository.findById(event.orderId()).ifPresent(view -> {
            view.setStatus("SUBMITTED");
            view.setUpdatedAt(event.occurredAt());
            viewRepository.save(view);
        });
    }
}
```

---

## สรุป Part 75

```
Event Sourcing + CQRS:

Event Sourcing:
  Store EVENTS, not state
  State = replay of all events
  Commands → produce events → apply to aggregate
  
  Aggregate:
    applyNewEvent() = apply + add to uncommitted
    reconstitute()  = replay events from store
    getUncommittedEvents() = events to save

CQRS:
  Write side: EventStore (append-only)
  Read side:  Projection (materialized view, optimized for query)
  
  Commands → Write Model → Events → Projections → Read Model

When to use Event Sourcing:
  ✓ Audit requirements (banking, medical, legal)
  ✓ Temporal queries ("what was the state at time T?")
  ✓ Complex business domains with event-driven design
  ✗ Simple CRUD (over-engineering)
  ✗ High write throughput with simple queries

Snapshot Pattern (for performance):
  Don't replay 10,000 events every time
  → Snapshot state every N events
  → Load latest snapshot + events after it

  snapshots table:
    aggregate_id, version, state_json
  
  load = find latest snapshot + events after snapshot.version
```

➡️ [Part 76: GraphQL API](./Part-76-GraphQL.md)
