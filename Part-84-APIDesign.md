# Part 84: API Design & Documentation (OpenAPI/Swagger)
## ขั้นตอนที่ 5781-5850: RESTful Best Practices, Versioning, Documentation

---

## 84.1 RESTful API Design Principles

```
REST Principles:
  1. Uniform Interface   = consistent URL patterns, HTTP verbs
  2. Stateless           = no session on server, each request self-contained
  3. Client-Server       = separated concerns
  4. Cacheable           = responses indicate cacheability
  5. Layered System      = client doesn't know about intermediate servers
  6. Code on Demand      = optional: server sends executable code

Resource Naming:
  ✓ Nouns, not verbs
  ✓ Plural for collections
  ✓ Lowercase, hyphen for words
  ✗ Actions in URL

  BAD:                    GOOD:
  /getUsers               /users
  /createUser             POST /users
  /deleteUser/1           DELETE /users/1
  /getUserOrders/1        /users/1/orders
  /process_payment        POST /orders/1/payment

HTTP Methods:
  GET     = read (safe, idempotent)
  POST    = create (not idempotent)
  PUT     = replace entire resource (idempotent)
  PATCH   = partial update (idempotent per RFC 5789)
  DELETE  = remove (idempotent)
  HEAD    = GET but only headers (checking existence)
  OPTIONS = get allowed methods (CORS preflight)

Status Codes:
  200 OK           = success with body
  201 Created      = resource created (with Location header)
  204 No Content   = success without body (DELETE, PUT)
  400 Bad Request  = invalid input
  401 Unauthorized = not authenticated
  403 Forbidden    = authenticated but no permission
  404 Not Found    = resource doesn't exist
  409 Conflict     = state conflict (duplicate, version mismatch)
  422 Unprocessable = validation failed
  429 Too Many Requests = rate limited
  500 Internal Server Error = unexpected server error
  503 Service Unavailable = temporarily unavailable
```

---

## 84.2 URL Structure & HATEOAS

```kotlin
// Resource endpoints structure
/*
Collection:
  GET    /api/v1/orders          → list orders (paged)
  POST   /api/v1/orders          → create order

Resource:
  GET    /api/v1/orders/{id}     → get single order
  PUT    /api/v1/orders/{id}     → replace order
  PATCH  /api/v1/orders/{id}     → partial update
  DELETE /api/v1/orders/{id}     → delete order

Sub-resources (relationships):
  GET    /api/v1/orders/{id}/items          → list order items
  POST   /api/v1/orders/{id}/items          → add item
  DELETE /api/v1/orders/{id}/items/{itemId} → remove item

Actions (non-CRUD operations):
  POST /api/v1/orders/{id}/submit    → submit order
  POST /api/v1/orders/{id}/cancel    → cancel order
  POST /api/v1/orders/{id}/payment   → process payment
*/

// Pagination response (standard)
data class PagedResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int,
    val hasNext: Boolean,
    val hasPrevious: Boolean,
    val links: PageLinks? = null  // HATEOAS
)

data class PageLinks(
    val self: String,
    val first: String,
    val last: String,
    val next: String?,
    val prev: String?
)

// Generic success response envelope
data class ApiResponse<T>(
    val data: T,
    val meta: Map<String, Any> = emptyMap()
)

// Error response (RFC 7807 Problem Details)
data class ProblemDetails(
    val type: String = "about:blank",   // URI identifying error type
    val title: String,                   // short description
    val status: Int,                     // HTTP status code
    val detail: String? = null,          // longer description
    val instance: String? = null,        // URI identifying this occurrence
    val errors: List<FieldError>? = null // validation errors
)

data class FieldError(
    val field: String,
    val message: String,
    val rejectedValue: Any? = null
)
```

---

## 84.3 OpenAPI 3.0 with SpringDoc

```kotlin
// build.gradle.kts
// implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")

// application.yaml
/*
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    try-it-out-enabled: true
    operations-sorter: method
  default-consumes-media-type: application/json
  default-produces-media-type: application/json
*/

// OpenAPI Configuration
@Configuration
class OpenApiConfig {
    
    @Bean
    fun openAPI(): OpenAPI = OpenAPI()
        .info(
            Info()
                .title("Shop API")
                .version("v1")
                .description("E-commerce REST API")
                .contact(Contact()
                    .name("API Team")
                    .email("api@example.com")
                )
                .license(License()
                    .name("MIT")
                    .url("https://opensource.org/licenses/MIT")
                )
        )
        .externalDocs(ExternalDocumentation()
            .description("Wiki")
            .url("https://wiki.example.com/api")
        )
        .components(Components()
            .addSecuritySchemes("bearerAuth",
                SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")
            )
        )
        .addSecurityItem(SecurityRequirement().addList("bearerAuth"))
        .servers(listOf(
            Server().url("https://api.example.com").description("Production"),
            Server().url("https://staging-api.example.com").description("Staging"),
            Server().url("http://localhost:8080").description("Local")
        ))
}
```

---

## 84.4 Controller Annotations for OpenAPI

```kotlin
@RestController
@RequestMapping("/api/v1/orders")
@Tag(name = "Orders", description = "Order management endpoints")
class OrderController(private val orderService: OrderService) {
    
    @Operation(
        summary = "List orders",
        description = "Get paginated list of orders for the current user",
        responses = [
            ApiResponse(responseCode = "200", description = "Success",
                content = [Content(schema = Schema(implementation = PagedOrderResponse::class))]),
            ApiResponse(responseCode = "401", description = "Unauthorized",
                content = [Content(schema = Schema(implementation = ProblemDetails::class))])
        ]
    )
    @GetMapping
    fun listOrders(
        @Parameter(description = "Page number (0-based)") @RequestParam(defaultValue = "0") page: Int,
        @Parameter(description = "Page size") @RequestParam(defaultValue = "20") size: Int,
        @Parameter(description = "Filter by status") @RequestParam(required = false) status: OrderStatus?,
        @AuthenticationPrincipal jwt: Jwt
    ): ResponseEntity<PagedResponse<OrderDto>> {
        val result = orderService.findByUserId(jwt.subject, page, size, status)
        return ResponseEntity.ok(result.toPagedResponse())
    }
    
    @Operation(summary = "Get order by ID")
    @GetMapping("/{orderId}")
    fun getOrder(
        @Parameter(description = "Order ID", required = true, example = "ord_123abc")
        @PathVariable orderId: String,
        @AuthenticationPrincipal jwt: Jwt
    ): ResponseEntity<OrderDto> {
        val order = orderService.findById(orderId)
            ?: return ResponseEntity.notFound().build()
        
        if (order.customerId != jwt.subject) return ResponseEntity.status(403).build()
        
        return ResponseEntity.ok(order.toDto())
    }
    
    @Operation(summary = "Submit order for processing")
    @ApiResponses(value = [
        ApiResponse(responseCode = "200", description = "Order submitted"),
        ApiResponse(responseCode = "409", description = "Order cannot be submitted",
            content = [Content(schema = Schema(implementation = ProblemDetails::class))])
    ])
    @PostMapping("/{orderId}/submit")
    fun submitOrder(
        @PathVariable orderId: String
    ): ResponseEntity<OrderDto> {
        return try {
            val order = orderService.submit(orderId)
            ResponseEntity.ok(order.toDto())
        } catch (e: InvalidStateException) {
            ResponseEntity.status(409)
                .body(ProblemDetails(
                    title = "Invalid State",
                    status = 409,
                    detail = e.message
                ) as OrderDto)
        }
    }
    
    @Operation(summary = "Cancel an order")
    @DeleteMapping("/{orderId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun cancelOrder(
        @PathVariable orderId: String,
        @RequestBody @Valid request: CancelOrderRequest
    ) {
        orderService.cancel(orderId, request.reason)
    }
}

// DTO with Schema annotations
@Schema(description = "Order data transfer object")
data class OrderDto(
    @Schema(description = "Unique order ID", example = "ord_123abc")
    val id: String,
    
    @Schema(description = "Customer ID who placed the order")
    val customerId: String,
    
    @Schema(description = "Current order status", allowableValues = ["DRAFT", "SUBMITTED", "CONFIRMED", "CANCELLED"])
    val status: String,
    
    @Schema(description = "Order total in THB cents", example = "19999")
    val totalCents: Long,
    
    @Schema(description = "Order line items")
    val items: List<OrderItemDto>,
    
    @Schema(description = "Order creation timestamp", example = "2024-01-15T10:30:00Z")
    val createdAt: java.time.Instant
)
```

---

## 84.5 API Versioning Strategies

```kotlin
// Strategy 1: URL Path Versioning (most common)
// GET /api/v1/orders
// GET /api/v2/orders
@RequestMapping("/api/v1/orders")  // explicit version in URL
class OrderControllerV1

@RequestMapping("/api/v2/orders")  // new version with different response
class OrderControllerV2

// Strategy 2: Header Versioning
// GET /api/orders  + header: API-Version: 2
@GetMapping("/api/orders")
fun getOrders(@RequestHeader("API-Version", defaultValue = "1") version: String) {
    return when (version) {
        "2" -> orderService.findAllV2()
        else -> orderService.findAll()
    }
}

// Strategy 3: Accept Header (Content Negotiation)
// GET /api/orders  + Accept: application/vnd.example.v2+json
@GetMapping("/api/orders", 
    produces = ["application/vnd.example.v1+json"])
fun getOrdersV1(): List<OrderDtoV1> = ...

@GetMapping("/api/orders",
    produces = ["application/vnd.example.v2+json"])
fun getOrdersV2(): List<OrderDtoV2> = ...

// API Version via class config (recommended for Spring)
@Configuration
class ApiVersionConfig {
    @Bean
    fun apiVersionRequestMappingHandlerMapping(): RequestMappingHandlerMapping {
        return object : RequestMappingHandlerMapping() {
            override fun getMappingForMethod(method: Method, handlerType: Class<*>): RequestMappingInfo? {
                val mappingInfo = super.getMappingForMethod(method, handlerType) ?: return null
                val version = handlerType.getAnnotation(ApiVersion::class.java) 
                    ?: return mappingInfo
                return RequestMappingInfo.paths("/api/v${version.value}")
                    .combine(mappingInfo)
            }
        }
    }
}

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class ApiVersion(val value: Int)

@RestController
@ApiVersion(2)
@RequestMapping("/orders")  // combined with prefix = /api/v2/orders
class OrderControllerV2
```

---

## 84.6 Request Validation

```kotlin
// Jakarta Bean Validation with custom validators

data class CreateOrderRequest(
    @NotBlank @Size(max = 50)
    val customerId: String,
    
    @NotEmpty
    val items: List<@Valid OrderItemRequest>,
    
    @ValidDeliveryDate
    val deliveryDate: java.time.LocalDate?
)

data class OrderItemRequest(
    @NotBlank
    val productId: String,
    
    @Positive @Max(100)
    val quantity: Int,
    
    @NotNull
    val unitPriceCents: Long?
)

// Custom validator
@Constraint(validatedBy = [DeliveryDateValidator::class])
@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class ValidDeliveryDate(
    val message: String = "Delivery date must be in the future and not a holiday",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

class DeliveryDateValidator : ConstraintValidator<ValidDeliveryDate, java.time.LocalDate?> {
    override fun isValid(value: java.time.LocalDate?, context: ConstraintValidatorContext): Boolean {
        if (value == null) return true  // optional field
        
        val today = java.time.LocalDate.now()
        if (value.isBefore(today.plusDays(1))) {
            context.disableDefaultConstraintViolation()
            context.buildConstraintViolationWithTemplate("Must be at least 1 day in the future")
                .addConstraintViolation()
            return false
        }
        
        if (value.dayOfWeek in listOf(java.time.DayOfWeek.SATURDAY, java.time.DayOfWeek.SUNDAY)) {
            context.disableDefaultConstraintViolation()
            context.buildConstraintViolationWithTemplate("Weekend delivery not available")
                .addConstraintViolation()
            return false
        }
        
        return true
    }
}

// Global exception handler for validation errors
@RestControllerAdvice
class ValidationExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationErrors(ex: MethodArgumentNotValidException): ResponseEntity<ProblemDetails> {
        val errors = ex.bindingResult.fieldErrors.map { error ->
            FieldError(
                field = error.field,
                message = error.defaultMessage ?: "Invalid value",
                rejectedValue = error.rejectedValue
            )
        }
        
        return ResponseEntity.status(422).body(
            ProblemDetails(
                title = "Validation Failed",
                status = 422,
                detail = "Request body validation failed",
                errors = errors
            )
        )
    }
    
    @ExceptionHandler(ConstraintViolationException::class)
    fun handleConstraintViolation(ex: ConstraintViolationException): ResponseEntity<ProblemDetails> {
        val errors = ex.constraintViolations.map { violation ->
            FieldError(
                field = violation.propertyPath.toString(),
                message = violation.message
            )
        }
        
        return ResponseEntity.status(422).body(
            ProblemDetails(
                title = "Constraint Violation",
                status = 422,
                errors = errors
            )
        )
    }
}
```

---

## สรุป Part 84

```
API Design Best Practices:

URL Design:
  /resources (plural noun)
  /resources/{id}
  /resources/{id}/sub-resources
  /resources/{id}/actions (POST for state changes)

HTTP Semantics:
  GET    = safe (no side effects), cacheable
  POST   = create, non-idempotent
  PUT    = full replace, idempotent
  PATCH  = partial update, idempotent
  DELETE = remove, idempotent

Response Codes:
  200 = success with body
  201 = created (POST success)
  204 = no content (DELETE, PUT success)
  400 = bad syntax
  401 = not authenticated (send 401 Unauthorized)
  403 = no permission (send 403 Forbidden)
  404 = not found
  422 = validation failed
  409 = conflict (duplicate, wrong state)
  429 = rate limited
  5xx = server errors

Documentation:
  OpenAPI 3.0 = standard spec
  SpringDoc = auto-generate from code
  @Operation, @Schema, @Parameter = annotate
  Try-it-out = test from Swagger UI

Versioning:
  URL path /v1/, /v2/ = simplest, most visible
  Header = cleaner URLs but harder to test
  Deprecation: X-Deprecated header + sunset date

Always:
  Paginate list endpoints
  Validate input (Bean Validation)
  Return RFC 7807 Problem Details for errors
  Include traceId in error responses
```

➡️ [Part 85: Advanced Database Patterns](./Part-85-DatabasePatterns.md)
