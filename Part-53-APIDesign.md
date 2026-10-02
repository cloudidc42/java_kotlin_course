# Part 53: API Design & REST Best Practices
## ขั้นตอนที่ 3611-3680: REST Principles, Versioning, OpenAPI

---

## 53.1 REST Design Principles

```
REST Resource Naming:
  Good:
    GET    /api/v1/orders              # list
    POST   /api/v1/orders              # create
    GET    /api/v1/orders/{id}         # get one
    PUT    /api/v1/orders/{id}         # replace
    PATCH  /api/v1/orders/{id}         # partial update
    DELETE /api/v1/orders/{id}         # delete
    
  Sub-resources:
    GET    /api/v1/orders/{id}/items   # list items of order
    POST   /api/v1/orders/{id}/items   # add item to order
    
  Actions (when needed):
    POST   /api/v1/orders/{id}/confirm     # state transition
    POST   /api/v1/orders/{id}/cancel      # state transition
    POST   /api/v1/orders/bulk-import      # batch operation
    
  Bad (verb in URL):
    GET    /api/v1/getOrder/{id}        # No: already GET
    POST   /api/v1/createOrder         # No: already POST
    POST   /api/v1/deleteOrder/{id}    # No: use DELETE method

HTTP Status Codes:
  2xx Success:
    200 OK             = success (GET, PUT, PATCH, POST returning body)
    201 Created        = new resource created (POST), with Location header
    202 Accepted       = async processing started
    204 No Content     = success, no body (DELETE, PUT returning nothing)
    
  4xx Client Error:
    400 Bad Request    = invalid input (validation failed)
    401 Unauthorized   = not authenticated
    403 Forbidden      = authenticated but not authorized
    404 Not Found      = resource doesn't exist
    409 Conflict       = duplicate, version conflict
    422 Unprocessable  = business rule violation
    429 Too Many Req   = rate limited
    
  5xx Server Error:
    500 Internal Error = unexpected error
    502 Bad Gateway    = upstream service error
    503 Unavailable    = overloaded, maintenance
    504 Gateway Timeout = upstream timeout
```

---

## 53.2 Spring REST Controller

```kotlin
import org.springframework.http.*
import org.springframework.web.bind.annotation.*
import jakarta.validation.Valid

@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrder: CreateOrderUseCase,
    private val getOrder: GetOrderUseCase,
    private val cancelOrder: CancelOrderUseCase,
    private val listOrders: ListOrdersUseCase
) {
    
    @GetMapping
    fun listOrders(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(required = false) status: String?,
        @RequestParam(required = false) cursor: String?,
        @org.springframework.security.core.annotation.AuthenticationPrincipal
        principal: org.springframework.security.core.userdetails.UserDetails
    ): ResponseEntity<PageResponse<OrderSummaryDTO>> {
        val result = listOrders.execute(ListOrdersUseCase.Query(
            userId = principal.username,
            status = status,
            cursor = cursor,
            size = minOf(size, 100)  // cap at 100
        ))
        return ResponseEntity.ok(result)
    }
    
    @PostMapping
    fun createOrder(
        @Valid @RequestBody request: CreateOrderRequest,
        @org.springframework.security.core.annotation.AuthenticationPrincipal
        principal: org.springframework.security.core.userdetails.UserDetails
    ): ResponseEntity<OrderDTO> {
        val order = createOrder.execute(
            CreateOrderUseCase.Command(
                userId = principal.username,
                items = request.items.map {
                    CreateOrderUseCase.OrderItemCommand(it.productId, it.quantity)
                }
            )
        )
        return ResponseEntity
            .created(java.net.URI("/api/v1/orders/${order.id}"))
            .body(order)
    }
    
    @GetMapping("/{id}")
    fun getOrder(
        @PathVariable id: String,
        @org.springframework.security.core.annotation.AuthenticationPrincipal
        principal: org.springframework.security.core.userdetails.UserDetails
    ): ResponseEntity<OrderDetailDTO> {
        val order = getOrder.execute(id, principal.username)
        return ResponseEntity.ok(order)
    }
    
    @PostMapping("/{id}/cancel")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun cancelOrder(
        @PathVariable id: String,
        @Valid @RequestBody request: CancelOrderRequest,
        @org.springframework.security.core.annotation.AuthenticationPrincipal
        principal: org.springframework.security.core.userdetails.UserDetails
    ) {
        cancelOrder.execute(
            CancelOrderUseCase.Command(
                orderId = id,
                userId = principal.username,
                reason = request.reason
            )
        )
    }
}

// Request/Response DTOs
data class CreateOrderRequest(
    @field:jakarta.validation.Valid
    @field:jakarta.validation.constraints.NotEmpty(message = "Items cannot be empty")
    val items: List<OrderItemRequest>
)

data class OrderItemRequest(
    @field:jakarta.validation.constraints.NotBlank val productId: String,
    @field:jakarta.validation.constraints.Min(1) val quantity: Int
)

data class CancelOrderRequest(
    @field:jakarta.validation.constraints.NotBlank
    @field:jakarta.validation.constraints.Size(min = 3, max = 500)
    val reason: String
)
```

---

## 53.3 Error Handling

```kotlin
import org.springframework.web.bind.annotation.*
import org.springframework.http.*
import org.springframework.web.bind.*

// Centralized error handling
@RestControllerAdvice
class GlobalExceptionHandler {
    
    // Validation errors
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val fieldErrors = ex.bindingResult.fieldErrors
            .associate { it.field to (it.defaultMessage ?: "Invalid") }
        
        return ResponseEntity
            .badRequest()
            .body(ErrorResponse(
                code = "VALIDATION_FAILED",
                message = "Request validation failed",
                details = fieldErrors
            ))
    }
    
    // Business rule violations
    @ExceptionHandler(BusinessException::class)
    fun handleBusiness(ex: BusinessException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(ErrorResponse(code = ex.code, message = ex.message ?: "Business error"))
    }
    
    // Not found
    @ExceptionHandler(ResourceNotFoundException::class)
    fun handleNotFound(ex: ResourceNotFoundException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(code = "NOT_FOUND", message = ex.message ?: "Resource not found"))
    }
    
    // Unauthorized access
    @ExceptionHandler(AccessDeniedException::class)
    fun handleAccessDenied(ex: AccessDeniedException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.FORBIDDEN)
            .body(ErrorResponse(code = "FORBIDDEN", message = "Access denied"))
    }
    
    // Generic catch-all
    @ExceptionHandler(Exception::class)
    fun handleGeneric(ex: Exception): ResponseEntity<ErrorResponse> {
        // Log full exception but hide internals from client
        org.slf4j.LoggerFactory.getLogger(javaClass).error("Unhandled error", ex)
        
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse(code = "INTERNAL_ERROR", message = "An unexpected error occurred"))
    }
}

data class ErrorResponse(
    val code: String,
    val message: String,
    val details: Map<String, String> = emptyMap(),
    val timestamp: java.time.Instant = java.time.Instant.now()
)

class BusinessException(val code: String, message: String) : RuntimeException(message)
class ResourceNotFoundException(resource: String, id: Any) :
    RuntimeException("$resource not found: $id")
```

---

## 53.4 API Versioning

```kotlin
// Strategy 1: URL versioning (most common, visible)
@RestController
@RequestMapping("/api/v1/orders")
class OrderControllerV1 { /* ... */ }

@RestController
@RequestMapping("/api/v2/orders")
class OrderControllerV2 {
    // Breaking changes in v2:
    // - Renamed field: 'total' → 'totalAmount'
    // - Added required field
    // - Changed response structure
}

// Strategy 2: Header versioning (clean URLs)
@RestController
@RequestMapping("/api/orders")
class OrderController {
    
    @GetMapping(headers = "API-Version=1")
    fun getOrderV1(@PathVariable id: String): OrderDTOv1 { TODO() }
    
    @GetMapping(headers = "API-Version=2")
    fun getOrderV2(@PathVariable id: String): OrderDTOv2 { TODO() }
}

// Strategy 3: Content negotiation
@RestController
@RequestMapping("/api/orders")
class ContentNegotiationController {
    
    @GetMapping(produces = ["application/vnd.company.v1+json"])
    fun getV1(): OrderDTOv1 { TODO() }
    
    @GetMapping(produces = ["application/vnd.company.v2+json"])
    fun getV2(): OrderDTOv2 { TODO() }
}

// Version negotiation in filter
@Component
class ApiVersionFilter : jakarta.servlet.Filter {
    
    override fun doFilter(
        request: jakarta.servlet.ServletRequest,
        response: jakarta.servlet.ServletResponse,
        chain: jakarta.servlet.FilterChain
    ) {
        val httpRequest = request as jakarta.servlet.http.HttpServletRequest
        val version = httpRequest.getHeader("API-Version") ?: "2"
        
        // Add version to request attributes for controllers to read
        httpRequest.setAttribute("apiVersion", version)
        chain.doFilter(request, response)
    }
}
```

---

## 53.5 OpenAPI Documentation

```kotlin
import io.swagger.v3.oas.annotations.*
import io.swagger.v3.oas.annotations.tags.*
import io.swagger.v3.oas.annotations.responses.*
import io.swagger.v3.oas.annotations.media.*
import io.swagger.v3.oas.annotations.parameters.*

@RestController
@RequestMapping("/api/v1/orders")
@Tag(name = "Orders", description = "Order management APIs")
class DocumentedOrderController(
    private val createOrder: CreateOrderUseCase,
    private val getOrder: GetOrderUseCase
) {
    
    @Operation(
        summary = "Create a new order",
        description = "Creates a new order for the authenticated user. Automatically processes payment.",
        security = [SecurityRequirement(name = "bearerAuth")]
    )
    @ApiResponses(
        ApiResponse(
            responseCode = "201",
            description = "Order created successfully",
            headers = [Header(name = "Location", description = "URL of created order")],
            content = [Content(schema = Schema(implementation = OrderDTO::class))]
        ),
        ApiResponse(
            responseCode = "400",
            description = "Validation failed",
            content = [Content(schema = Schema(implementation = ErrorResponse::class))]
        ),
        ApiResponse(responseCode = "401", description = "Unauthorized"),
        ApiResponse(
            responseCode = "422",
            description = "Business rule violation (e.g. insufficient stock)",
            content = [Content(schema = Schema(implementation = ErrorResponse::class))]
        )
    )
    @PostMapping
    fun createOrder(
        @io.swagger.v3.oas.annotations.parameters.RequestBody(
            description = "Order creation request",
            required = true
        )
        @Valid @RequestBody request: CreateOrderRequest
    ): ResponseEntity<OrderDTO> {
        TODO()
    }
    
    @Operation(summary = "Get order by ID")
    @Parameter(name = "id", description = "Order ID", required = true, example = "ord_123abc")
    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: String): OrderDetailDTO {
        TODO()
    }
}

// OpenAPI configuration
@Configuration
class OpenApiConfig {
    
    @Bean
    fun openApiDefinition(): io.swagger.v3.oas.models.OpenAPI {
        return io.swagger.v3.oas.models.OpenAPI()
            .info(
                io.swagger.v3.oas.models.info.Info()
                    .title("Order Service API")
                    .version("2.0")
                    .description("RESTful API for order management")
                    .contact(io.swagger.v3.oas.models.info.Contact()
                        .name("API Team")
                        .email("api@example.com")
                    )
            )
            .components(
                io.swagger.v3.oas.models.Components()
                    .addSecuritySchemes("bearerAuth",
                        io.swagger.v3.oas.models.security.SecurityScheme()
                            .type(io.swagger.v3.oas.models.security.SecurityScheme.Type.HTTP)
                            .scheme("bearer")
                            .bearerFormat("JWT")
                    )
            )
            .addSecurityItem(
                io.swagger.v3.oas.models.security.SecurityRequirement()
                    .addList("bearerAuth")
            )
    }
}
```

---

## 53.6 Rate Limiting

```java
import org.springframework.web.servlet.HandlerInterceptor;

// Token bucket rate limiter per API key
@Component
class RateLimitInterceptor implements HandlerInterceptor {
    
    private final io.github.resilience4j.ratelimiter.RateLimiterRegistry rateLimiterRegistry;
    
    RateLimitInterceptor(io.github.resilience4j.ratelimiter.RateLimiterRegistry registry) {
        this.rateLimiterRegistry = registry;
    }
    
    @Override
    public boolean preHandle(jakarta.servlet.http.HttpServletRequest request,
                             jakarta.servlet.http.HttpServletResponse response,
                             Object handler) throws Exception {
        
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey == null) {
            response.setStatus(401);
            return false;
        }
        
        var rateLimiter = rateLimiterRegistry.rateLimiter(apiKey,
            io.github.resilience4j.ratelimiter.RateLimiterConfig.custom()
                .limitForPeriod(100)
                .limitRefreshPeriod(java.time.Duration.ofMinutes(1))
                .timeoutDuration(java.time.Duration.ZERO)  // fail immediately
                .build()
        );
        
        if (!rateLimiter.acquirePermission()) {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.setHeader("X-RateLimit-Reset",
                String.valueOf(System.currentTimeMillis() / 1000 + 60));
            return false;
        }
        
        return true;
    }
}
```

---

## สรุป Part 53

```
REST API Design Checklist:

URLs:
  ✓ Plural nouns for resources (/orders, /users)
  ✓ Sub-resources for nested data (/orders/{id}/items)
  ✓ Actions for state transitions (/orders/{id}/cancel)
  ✗ No verbs in URL (/getOrder, /createUser)

HTTP Methods + Status Codes:
  POST   → 201 Created + Location header
  GET    → 200 OK
  DELETE → 204 No Content
  400    = validation failed (return field-level details)
  401    = not authenticated
  403    = authenticated, not authorized
  422    = business rule violation

Error Response:
  Always return { code, message, details, timestamp }
  Never expose stack traces to client

Versioning:
  URL versioning (/api/v1, /api/v2) = most common
  Header versioning = cleaner URLs but less visible

OpenAPI:
  @Operation @ApiResponse @Schema annotations
  Swagger UI at /swagger-ui.html
  OpenAPI JSON at /v3/api-docs

Rate Limiting:
  Token bucket per API key
  429 response with Retry-After header
```

➡️ [Part 54: Microservices Communication Patterns](./Part-54-Microservices.md)
