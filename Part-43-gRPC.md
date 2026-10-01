# Part 43: gRPC with Java/Kotlin
## ขั้นตอนที่ 2911-2980: High-Performance RPC

---

## 43.1 gRPC Overview

```
gRPC = Google Remote Procedure Call

REST vs gRPC:
  REST:   JSON over HTTP/1.1, text-based, schema optional
  gRPC:   Protocol Buffers over HTTP/2, binary, schema required

gRPC advantages:
  ✓ 7-10x faster serialization (Protobuf vs JSON)
  ✓ Streaming (unary, server, client, bidirectional)
  ✓ Strongly typed (compile-time errors, not runtime)
  ✓ Auto-generated client code
  ✓ Multiplexing over HTTP/2
  
Use cases:
  ✓ Internal microservice communication
  ✓ Streaming large datasets
  ✓ Low-latency requirements
  ✗ Browser clients (use gRPC-Web)
  ✗ Public APIs (harder to debug without tooling)
  
gRPC call types:
  Unary            = request → response (like REST)
  Server streaming = request → stream of responses
  Client streaming = stream of requests → response
  Bidirectional    = stream ↔ stream
```

---

## 43.2 Protocol Buffers Schema

```protobuf
// src/main/proto/user_service.proto

syntax = "proto3";

package com.example.grpc;

option java_package = "com.example.grpc";
option java_outer_classname = "UserServiceProto";
option java_multiple_files = true;

// ====== Messages ======
message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    UserRole role = 4;
    google.protobuf.Timestamp created_at = 5;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
    string password = 3;
    UserRole role = 4;
}

message GetUserRequest {
    int64 id = 1;
}

message GetUserByEmailRequest {
    string email = 1;
}

message UpdateUserRequest {
    int64 id = 1;
    optional string name = 2;
    optional string email = 3;
}

message DeleteUserRequest {
    int64 id = 1;
}

message DeleteUserResponse {
    bool success = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
    string search = 3;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total = 2;
    int32 page = 3;
    int32 total_pages = 4;
}

// Streaming request for batch import
message ImportUsersRequest {
    User user = 1;
}

message ImportUsersResponse {
    int32 imported = 1;
    int32 failed = 2;
    repeated string errors = 3;
}

enum UserRole {
    USER = 0;
    ADMIN = 1;
    MODERATOR = 2;
}

// ====== Service definition ======
service UserService {
    // Unary
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc GetUser(GetUserRequest) returns (User);
    rpc GetUserByEmail(GetUserByEmailRequest) returns (User);
    rpc UpdateUser(UpdateUserRequest) returns (User);
    rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    
    // Server streaming
    rpc WatchUserUpdates(GetUserRequest) returns (stream User);
    
    // Client streaming
    rpc ImportUsers(stream ImportUsersRequest) returns (ImportUsersResponse);
    
    // Bidirectional streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
    string sender = 1;
    string content = 2;
    google.protobuf.Timestamp timestamp = 3;
}

import "google/protobuf/timestamp.proto";
```

---

## 43.3 Gradle Setup

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.0.0"
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("io.grpc:grpc-netty-shaded:1.64.0")
    implementation("io.grpc:grpc-protobuf:1.64.0")
    implementation("io.grpc:grpc-stub:1.64.0")
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")
    implementation("com.google.protobuf:protobuf-kotlin:3.25.3")
    
    // For Spring Boot integration
    implementation("net.devh:grpc-server-spring-boot-starter:3.1.0.RELEASE")
    implementation("net.devh:grpc-client-spring-boot-starter:3.1.0.RELEASE")
    
    compileOnly("jakarta.annotation:jakarta.annotation-api:2.1.1")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.25.3"
    }
    plugins {
        id("grpc") {
            artifact = "io.grpc:protoc-gen-grpc-java:1.64.0"
        }
        id("grpckt") {
            artifact = "io.grpc:protoc-gen-grpc-kotlin:1.4.1:jdk8@jar"
        }
    }
    generateProtoTasks {
        all().forEach { task ->
            task.plugins {
                id("grpc")
                id("grpckt")
            }
            task.builtins {
                id("kotlin")
            }
        }
    }
}

sourceSets {
    main {
        java.srcDirs("build/generated/source/proto/main/grpc")
        java.srcDirs("build/generated/source/proto/main/java")
        kotlin.srcDirs("build/generated/source/proto/main/grpckt")
        kotlin.srcDirs("build/generated/source/proto/main/kotlin")
    }
}
```

---

## 43.4 Server Implementation

```kotlin
import com.example.grpc.*
import io.grpc.Status
import io.grpc.stub.StreamObserver
import net.devh.boot.grpc.server.service.GrpcService

@GrpcService
class UserGrpcService(
    private val userRepository: UserRepository
) : UserServiceGrpc.UserServiceImplBase() {
    
    // ====== Unary ======
    override fun createUser(
        request: CreateUserRequest,
        responseObserver: StreamObserver<User>
    ) {
        try {
            val saved = userRepository.save(
                UserEntity(
                    name = request.name,
                    email = request.email,
                    role = request.role.name
                )
            )
            
            responseObserver.onNext(saved.toProto())
            responseObserver.onCompleted()
            
        } catch (e: Exception) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.message).asRuntimeException()
            )
        }
    }
    
    override fun getUser(
        request: GetUserRequest,
        responseObserver: StreamObserver<User>
    ) {
        val user = userRepository.findById(request.id)
        
        if (user == null) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("User not found: ${request.id}")
                    .asRuntimeException()
            )
            return
        }
        
        responseObserver.onNext(user.toProto())
        responseObserver.onCompleted()
    }
    
    override fun listUsers(
        request: ListUsersRequest,
        responseObserver: StreamObserver<ListUsersResponse>
    ) {
        val users = userRepository.findAll(request.page, request.pageSize, request.search)
        val total = userRepository.count(request.search)
        
        val response = ListUsersResponse.newBuilder()
            .addAllUsers(users.map { it.toProto() })
            .setTotal(total.toInt())
            .setPage(request.page)
            .setTotalPages(((total + request.pageSize - 1) / request.pageSize).toInt())
            .build()
        
        responseObserver.onNext(response)
        responseObserver.onCompleted()
    }
    
    // ====== Server streaming ======
    override fun watchUserUpdates(
        request: GetUserRequest,
        responseObserver: StreamObserver<User>
    ) {
        // Simulate live updates (in practice, subscribe to event bus)
        for (i in 1..5) {
            val user = userRepository.findById(request.id) ?: break
            responseObserver.onNext(user.toProto())
            Thread.sleep(1000)
        }
        responseObserver.onCompleted()
    }
    
    // ====== Client streaming ======
    override fun importUsers(
        responseObserver: StreamObserver<ImportUsersResponse>
    ): StreamObserver<ImportUsersRequest> {
        
        val importedUsers = mutableListOf<UserEntity>()
        val errors = mutableListOf<String>()
        
        return object : StreamObserver<ImportUsersRequest> {
            override fun onNext(request: ImportUsersRequest) {
                try {
                    val entity = UserEntity(
                        name = request.user.name,
                        email = request.user.email
                    )
                    importedUsers.add(entity)
                } catch (e: Exception) {
                    errors.add("Failed to import ${request.user.email}: ${e.message}")
                }
            }
            
            override fun onError(t: Throwable) {
                // Client disconnected
                responseObserver.onError(t)
            }
            
            override fun onCompleted() {
                // Batch save all at once
                val saved = userRepository.saveAll(importedUsers)
                
                responseObserver.onNext(
                    ImportUsersResponse.newBuilder()
                        .setImported(saved.size)
                        .setFailed(errors.size)
                        .addAllErrors(errors)
                        .build()
                )
                responseObserver.onCompleted()
            }
        }
    }
}

// Extension function: Entity → Proto
fun UserEntity.toProto(): User = User.newBuilder()
    .setId(id)
    .setName(name)
    .setEmail(email)
    .setRole(UserRole.valueOf(role))
    .build()
```

---

## 43.5 Client

```kotlin
import com.example.grpc.*
import io.grpc.ManagedChannelBuilder
import net.devh.boot.grpc.client.inject.GrpcClient
import org.springframework.stereotype.Service

// Spring Boot auto-configured client
@Service
class UserGrpcClient {
    
    @GrpcClient("user-service")  // name from application.yml
    private lateinit var userServiceStub: UserServiceGrpc.UserServiceBlockingStub
    
    @GrpcClient("user-service")
    private lateinit var userServiceAsync: UserServiceGrpc.UserServiceStub
    
    fun getUser(id: Long): User {
        return userServiceStub
            .withDeadlineAfter(5, java.util.concurrent.TimeUnit.SECONDS)
            .getUser(GetUserRequest.newBuilder().setId(id).build())
    }
    
    fun listUsers(page: Int, size: Int): ListUsersResponse {
        return userServiceStub.listUsers(
            ListUsersRequest.newBuilder()
                .setPage(page)
                .setPageSize(size)
                .build()
        )
    }
    
    // Server streaming
    fun watchUser(userId: Long, observer: io.grpc.stub.StreamObserver<User>) {
        userServiceAsync.watchUserUpdates(
            GetUserRequest.newBuilder().setId(userId).build(),
            observer
        )
    }
    
    // Client streaming: import batch
    fun importUsers(users: List<User>): ImportUsersResponse {
        val resultHolder = java.util.concurrent.CompletableFuture<ImportUsersResponse>()
        
        val requestObserver = userServiceAsync.importUsers(
            object : io.grpc.stub.StreamObserver<ImportUsersResponse> {
                override fun onNext(value: ImportUsersResponse) {
                    resultHolder.complete(value)
                }
                override fun onError(t: Throwable) {
                    resultHolder.completeExceptionally(t)
                }
                override fun onCompleted() {}
            }
        )
        
        users.forEach { user ->
            requestObserver.onNext(
                ImportUsersRequest.newBuilder().setUser(user).build()
            )
        }
        requestObserver.onCompleted()
        
        return resultHolder.get(30, java.util.concurrent.TimeUnit.SECONDS)
    }
}

// Manual channel creation (without Spring)
fun createGrpcClient(): UserServiceGrpc.UserServiceBlockingStub {
    val channel = ManagedChannelBuilder.forAddress("localhost", 9090)
        .usePlaintext()  // no TLS (development only)
        .keepAliveTime(30, java.util.concurrent.TimeUnit.SECONDS)
        .keepAliveTimeout(5, java.util.concurrent.TimeUnit.SECONDS)
        .build()
    
    return UserServiceGrpc.newBlockingStub(channel)
}
```

---

## 43.6 gRPC Configuration

```yaml
# application.yml (server)
grpc:
  server:
    port: 9090
    enable-reflection: true    # for grpcurl debugging

# application.yml (client)
grpc:
  client:
    user-service:
      address: static://user-service:9090
      negotiation-type: plaintext    # or TLS
      keep-alive-time: 30s
      keep-alive-timeout: 5s
    
    # With service discovery (Eureka)
    order-service:
      address: discovery:///order-service
      negotiation-type: plaintext
```

```bash
# Debug with grpcurl CLI:
# List services
grpcurl -plaintext localhost:9090 list

# Describe service
grpcurl -plaintext localhost:9090 describe com.example.grpc.UserService

# Call method
grpcurl -plaintext -d '{"id": 1}' localhost:9090 com.example.grpc.UserService/GetUser

# Call with auth header
grpcurl -plaintext -H 'Authorization: Bearer token123' \
  -d '{"page": 0, "page_size": 10}' \
  localhost:9090 com.example.grpc.UserService/ListUsers
```

---

## 43.7 gRPC Interceptors

```kotlin
import io.grpc.*
import net.devh.boot.grpc.server.interceptor.GrpcGlobalServerInterceptor

@GrpcGlobalServerInterceptor
class AuthInterceptor(private val jwtService: JwtService) : ServerInterceptor {
    
    companion object {
        val USER_ID_KEY: Context.Key<Long> = Context.key("userId")
    }
    
    override fun <ReqT, RespT> interceptCall(
        call: ServerCall<ReqT, RespT>,
        headers: Metadata,
        next: ServerCallHandler<ReqT, RespT>
    ): ServerCall.Listener<ReqT> {
        
        val token = headers.get(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER))
        
        if (token == null || !token.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED.withDescription("No auth token"), Metadata())
            return object : ServerCall.Listener<ReqT>() {}
        }
        
        return try {
            val userId = jwtService.extractUserId(token.removePrefix("Bearer "))
            val ctx = Context.current().withValue(USER_ID_KEY, userId)
            Contexts.interceptCall(ctx, call, headers, next)
        } catch (e: Exception) {
            call.close(Status.UNAUTHENTICATED.withDescription("Invalid token"), Metadata())
            object : ServerCall.Listener<ReqT>() {}
        }
    }
}

// Logging interceptor
@GrpcGlobalServerInterceptor
class LoggingInterceptor : ServerInterceptor {
    
    override fun <ReqT, RespT> interceptCall(
        call: ServerCall<ReqT, RespT>,
        headers: Metadata,
        next: ServerCallHandler<ReqT, RespT>
    ): ServerCall.Listener<ReqT> {
        val methodName = call.methodDescriptor.fullMethodName
        val start = System.currentTimeMillis()
        
        println("gRPC call: $methodName")
        
        return object : ForwardingServerCallListener.SimpleForwardingServerCallListener<ReqT>(
            next.startCall(object : ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
                override fun close(status: Status, trailers: Metadata) {
                    println("gRPC $methodName completed: ${status.code} (${System.currentTimeMillis() - start}ms)")
                    super.close(status, trailers)
                }
            }, headers)
        ) {}
    }
}
```

---

## สรุป Part 43

| Concept | คำอธิบาย |
|---------|---------|
| `.proto` file | Schema definition (IDL) |
| `protoc` | Compiler generates Java/Kotlin code |
| `StreamObserver` | Callback for streaming |
| `@GrpcService` | Server implementation |
| `@GrpcClient` | Inject auto-configured client |
| `ServerInterceptor` | Middleware (auth, logging) |

**gRPC Call Types:**
```
Unary:              client.getUser(req)
Server streaming:   client.watchUser(req) → stream<User>
Client streaming:   stream<req> → client.importUsers()
Bidirectional:      stream<Message> ↔ stream<Message>
```

➡️ [Part 44: Security Best Practices](./Part-44-Security.md)
