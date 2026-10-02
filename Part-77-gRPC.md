# Part 77: gRPC & Protocol Buffers
## ขั้นตอนที่ 5291-5360: High-Performance RPC, Streaming, Service Contracts

---

## 77.1 gRPC Overview

```
gRPC = Google Remote Procedure Call

ทำไมใช้ gRPC แทน REST?
  REST:      Text/JSON, HTTP/1.1, request-response
  gRPC:      Binary/Protobuf, HTTP/2, streaming support

Comparison:
  Feature         REST JSON    gRPC Protobuf
  Payload size    100%         ~20% (5x smaller)
  Speed           baseline     ~5-10x faster parsing
  Schema          OpenAPI      .proto file
  Streaming       SSE/WS       native (4 types)
  Type safety     weak         strong
  Browser support excellent    limited (grpc-web)

gRPC Streaming Types:
  1. Unary            = 1 request → 1 response (like REST)
  2. Server streaming = 1 request → many responses
  3. Client streaming = many requests → 1 response
  4. Bidirectional   = many requests ↔ many responses

Best For:
  ✓ Internal microservice communication (low latency)
  ✓ Large data transfer (binary compression)
  ✓ Streaming (real-time data feeds)
  ✓ Polyglot environments (proto = language-agnostic)
```

---

## 77.2 Protocol Buffers (Proto3)

```protobuf
// src/main/proto/order_service.proto

syntax = "proto3";

package com.example.order;

option java_package = "com.example.order.grpc";
option java_multiple_files = true;

// Message definitions
message CreateOrderRequest {
    string customer_id = 1;
    repeated OrderItemRequest items = 2;
}

message OrderItemRequest {
    string product_id = 1;
    int32 quantity = 2;
}

message Order {
    string id = 1;
    string customer_id = 2;
    OrderStatus status = 3;
    repeated OrderItem items = 4;
    int64 total_cents = 5;
    string created_at = 6;  // ISO-8601
}

message OrderItem {
    string product_id = 1;
    string product_name = 2;
    int64 unit_price_cents = 3;
    int32 quantity = 4;
}

message GetOrderRequest {
    string order_id = 1;
}

message GetOrdersRequest {
    string customer_id = 1;
    int32 page = 2;
    int32 page_size = 3;
    repeated OrderStatus status_filter = 4;
}

message GetOrdersResponse {
    repeated Order orders = 1;
    int32 total_count = 2;
    bool has_next_page = 3;
}

message OrderStatusUpdate {
    string order_id = 1;
    OrderStatus old_status = 2;
    OrderStatus new_status = 3;
    int64 timestamp = 4;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_DRAFT = 1;
    ORDER_STATUS_SUBMITTED = 2;
    ORDER_STATUS_CONFIRMED = 3;
    ORDER_STATUS_CANCELLED = 4;
    ORDER_STATUS_COMPLETED = 5;
}

// Service definition
service OrderService {
    // Unary RPC
    rpc CreateOrder (CreateOrderRequest) returns (Order);
    rpc GetOrder (GetOrderRequest) returns (Order);
    rpc GetOrders (GetOrdersRequest) returns (GetOrdersResponse);
    
    // Server streaming: stream order updates
    rpc WatchOrderStatus (GetOrderRequest) returns (stream OrderStatusUpdate);
    
    // Client streaming: bulk create orders
    rpc BulkCreateOrders (stream CreateOrderRequest) returns (GetOrdersResponse);
    
    // Bidirectional streaming: live order feed
    rpc OrderFeed (stream GetOrdersRequest) returns (stream Order);
}
```

---

## 77.3 Spring Boot gRPC Server

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

```yaml
# application.yaml
grpc:
  server:
    port: 9090
    max-inbound-message-size: 10MB
```

```java
import net.devh.boot.grpc.server.service.*;
import io.grpc.stub.*;

@GrpcService
class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {
    
    private final OrderService orderService;
    private final StreamObserver<OrderStatusUpdate> statusBroadcaster;
    
    OrderGrpcService(OrderService orderService) {
        this.orderService = orderService;
    }
    
    // ====== Unary RPC ======
    @Override
    public void createOrder(CreateOrderRequest request,
                             StreamObserver<Order> responseObserver) {
        try {
            var order = orderService.createOrder(request.getCustomerId());
            responseObserver.onNext(toProto(order));
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(io.grpc.Status.INTERNAL
                .withDescription(e.getMessage())
                .asRuntimeException());
        }
    }
    
    @Override
    public void getOrder(GetOrderRequest request,
                          StreamObserver<Order> responseObserver) {
        try {
            var order = orderService.findById(request.getOrderId());
            if (order == null) {
                responseObserver.onError(io.grpc.Status.NOT_FOUND
                    .withDescription("Order not found: " + request.getOrderId())
                    .asRuntimeException());
                return;
            }
            responseObserver.onNext(toProto(order));
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(io.grpc.Status.INTERNAL
                .withDescription(e.getMessage())
                .asRuntimeException());
        }
    }
    
    // ====== Server streaming: send multiple responses ======
    @Override
    public void watchOrderStatus(GetOrderRequest request,
                                  StreamObserver<OrderStatusUpdate> responseObserver) {
        
        String orderId = request.getOrderId();
        
        // Register observer, remove on client disconnect
        var subscription = orderStatusBus.subscribe(orderId, update -> {
            try {
                responseObserver.onNext(update);
            } catch (Exception e) {
                // Client disconnected
                orderStatusBus.unsubscribe(orderId);
            }
        });
        
        // Send current status immediately
        var currentOrder = orderService.findById(orderId);
        if (currentOrder != null) {
            responseObserver.onNext(OrderStatusUpdate.newBuilder()
                .setOrderId(orderId)
                .setNewStatus(toProtoStatus(currentOrder.getStatus()))
                .setTimestamp(java.time.Instant.now().toEpochMilli())
                .build());
        }
    }
    
    // ====== Client streaming: receive multiple requests ======
    @Override
    public StreamObserver<CreateOrderRequest> bulkCreateOrders(
            StreamObserver<GetOrdersResponse> responseObserver) {
        
        var createdOrders = new java.util.ArrayList<Order>();
        
        return new StreamObserver<CreateOrderRequest>() {
            @Override
            public void onNext(CreateOrderRequest request) {
                // Process each incoming request
                var order = orderService.createOrder(request.getCustomerId());
                createdOrders.add(toProto(order));
            }
            
            @Override
            public void onError(Throwable t) {
                responseObserver.onError(t);
            }
            
            @Override
            public void onCompleted() {
                // All requests received, send summary
                responseObserver.onNext(GetOrdersResponse.newBuilder()
                    .addAllOrders(createdOrders)
                    .setTotalCount(createdOrders.size())
                    .setHasNextPage(false)
                    .build());
                responseObserver.onCompleted();
            }
        };
    }
    
    // Convert domain object to proto
    private Order toProto(com.example.domain.Order order) {
        return Order.newBuilder()
            .setId(order.getId())
            .setCustomerId(order.getCustomerId())
            .setStatus(toProtoStatus(order.getStatus()))
            .setTotalCents(order.getTotal().cents())
            .setCreatedAt(order.getCreatedAt().toString())
            .build();
    }
    
    private OrderStatus toProtoStatus(com.example.domain.OrderStatus status) {
        return switch (status) {
            case DRAFT     -> OrderStatus.ORDER_STATUS_DRAFT;
            case SUBMITTED -> OrderStatus.ORDER_STATUS_SUBMITTED;
            case CONFIRMED -> OrderStatus.ORDER_STATUS_CONFIRMED;
            case CANCELLED -> OrderStatus.ORDER_STATUS_CANCELLED;
            case COMPLETED -> OrderStatus.ORDER_STATUS_COMPLETED;
        };
    }
}
```

---

## 77.4 gRPC Client

```java
import net.devh.boot.grpc.client.inject.*;

@Service
class OrderGrpcClient {
    
    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceBlockingStub orderStub;
    
    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceStub asyncOrderStub;
    
    // Unary call
    public Order createOrder(String customerId) {
        try {
            return orderStub.withDeadlineAfter(5, java.util.concurrent.TimeUnit.SECONDS)
                .createOrder(CreateOrderRequest.newBuilder()
                    .setCustomerId(customerId)
                    .build());
        } catch (io.grpc.StatusRuntimeException e) {
            if (e.getStatus().getCode() == io.grpc.Status.Code.NOT_FOUND) {
                throw new NotFoundException("Order not found");
            }
            throw new RuntimeException("gRPC call failed: " + e.getStatus());
        }
    }
    
    // Server streaming: receive updates async
    public void watchOrder(String orderId,
                           java.util.function.Consumer<OrderStatusUpdate> onUpdate) {
        asyncOrderStub.watchOrderStatus(
            GetOrderRequest.newBuilder().setOrderId(orderId).build(),
            new StreamObserver<OrderStatusUpdate>() {
                @Override public void onNext(OrderStatusUpdate update) { onUpdate.accept(update); }
                @Override public void onError(Throwable t) { log.error("Watch error", t); }
                @Override public void onCompleted() { log.info("Watch completed"); }
            }
        );
    }
}

// application.yaml for client
/*
grpc:
  client:
    order-service:
      address: static://order-service:9090
      negotiation-type: plaintext    # or TLS
      keep-alive-time: 30s
      keep-alive-timeout: 5s
*/
```

---

## สรุป Part 77

```
gRPC Summary:

Proto Definition:
  .proto file = service contract (like OpenAPI but stronger)
  message = data structure (like class)
  service = RPC methods
  
  Field numbers (1, 2, 3...) = binary encoding key
  Never reuse field numbers!

Streaming Types:
  rpc Unary(Req) returns (Res)
  rpc ServerStream(Req) returns (stream Res)
  rpc ClientStream(stream Req) returns (Res)
  rpc BidiStream(stream Req) returns (stream Res)

Spring Boot Setup:
  grpc-server-spring-boot-starter (server)
  grpc-client-spring-boot-starter (client)
  @GrpcService on server implementation
  @GrpcClient on client injection

Error Handling:
  io.grpc.Status.NOT_FOUND → 404
  io.grpc.Status.INVALID_ARGUMENT → 400
  io.grpc.Status.INTERNAL → 500
  Always check StatusRuntimeException.getStatus().getCode()

When gRPC vs REST:
  gRPC:
    ✓ Microservice-to-microservice (internal)
    ✓ Performance-critical paths
    ✓ Streaming data (real-time)
  REST:
    ✓ Public APIs (browsers support)
    ✓ Simple CRUD
    ✓ Partner integrations (easier tooling)
```

➡️ [Part 78: Testing Strategies (Comprehensive)](./Part-78-TestingStrategies.md)
