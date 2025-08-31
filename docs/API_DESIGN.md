# 🔌 API Design & Integration Patterns

## 🌐 API Gateway Routing Configuration

### Route Definitions

```yaml
# API Gateway Routes
spring:
  cloud:
    gateway:
      routes:
        # User Service Routes
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

        # Account Service Routes
        - id: account-service
          uri: lb://ACCOUNT-SERVICE
          predicates:
            - Path=/api/accounts/**
          filters:
            - StripPrefix=2
            - name: CircuitBreaker
              args:
                name: account-service-cb
                fallbackUri: forward:/fallback/accounts

        # Transaction Service Routes
        - id: transaction-service
          uri: lb://TRANSACTION-SERVICE
          predicates:
            - Path=/api/transactions/**
          filters:
            - StripPrefix=2

        # Fund Transfer Service Routes
        - id: fund-transfer-service
          uri: lb://FUND-TRANSFER-SERVICE
          predicates:
            - Path=/api/transfers/**
          filters:
            - StripPrefix=2

        # Sequence Generator Routes
        - id: sequence-generator-service
          uri: lb://SEQUENCE-GENERATOR-SERVICE
          predicates:
            - Path=/api/sequences/**
          filters:
            - StripPrefix=2
```

## 📋 RESTful API Standards

### HTTP Methods & Status Codes

```
GET    /api/users           → 200 OK (List users)
GET    /api/users/{id}      → 200 OK | 404 Not Found
POST   /api/users           → 201 Created | 400 Bad Request
PUT    /api/users/{id}      → 200 OK | 404 Not Found
DELETE /api/users/{id}      → 204 No Content | 404 Not Found

GET    /api/accounts        → 200 OK
GET    /api/accounts/{id}   → 200 OK | 404 Not Found
POST   /api/accounts        → 201 Created | 400 Bad Request
PUT    /api/accounts/{id}   → 200 OK | 404 Not Found
DELETE /api/accounts/{id}   → 204 No Content | 404 Not Found

GET    /api/transactions    → 200 OK
GET    /api/transactions/{id} → 200 OK | 404 Not Found
POST   /api/transactions    → 201 Created | 400 Bad Request

GET    /api/transfers       → 200 OK
GET    /api/transfers/{id}  → 200 OK | 404 Not Found
POST   /api/transfers       → 201 Created | 400 Bad Request

POST   /api/sequences       → 201 Created
GET    /api/sequences/{entity} → 200 OK
```

### Request/Response Patterns

#### Standard Response Format

```json
{
  "status": "success|error",
  "message": "Operation completed successfully",
  "data": {
    // Response payload
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/api/users/123"
}
```

#### Error Response Format

```json
{
  "status": "error",
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email format"
    }
  ],
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/api/users"
}
```

## 🔄 Inter-Service Communication

### Feign Client Configuration

```java
// Account Service calling User Service
@FeignClient(name = "USER-SERVICE", path = "/users")
public interface UserServiceClient {
    
    @GetMapping("/{userId}")
    ResponseEntity<UserDto> getUserById(@PathVariable String userId);
    
    @GetMapping("/{userId}/exists")
    ResponseEntity<Boolean> userExists(@PathVariable String userId);
}

// Fund Transfer Service calling Account Service
@FeignClient(name = "ACCOUNT-SERVICE", path = "/accounts")
public interface AccountServiceClient {
    
    @GetMapping("/{accountId}")
    ResponseEntity<AccountDto> getAccountById(@PathVariable String accountId);
    
    @PostMapping("/{accountId}/balance/debit")
    ResponseEntity<Void> debitAccount(@PathVariable String accountId, 
                                    @RequestBody AmountDto amount);
    
    @PostMapping("/{accountId}/balance/credit")
    ResponseEntity<Void> creditAccount(@PathVariable String accountId, 
                                     @RequestBody AmountDto amount);
}
```

### Circuit Breaker Pattern

```java
@Component
public class AccountServiceClientFallback implements AccountServiceClient {
    
    @Override
    public ResponseEntity<AccountDto> getAccountById(String accountId) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(AccountDto.builder()
                        .accountId(accountId)
                        .status("UNAVAILABLE")
                        .build());
    }
    
    @Override
    public ResponseEntity<Void> debitAccount(String accountId, AmountDto amount) {
        throw new ServiceUnavailableException("Account service is currently unavailable");
    }
}
```

## 🔐 Security Integration

### JWT Token Validation

```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        if (isSecuredPath(request.getPath().toString())) {
            String token = extractToken(request);
            
            if (token == null || !validateToken(token)) {
                return handleUnauthorized(exchange);
            }
            
            // Add user context to request headers
            ServerHttpRequest modifiedRequest = request.mutate()
                    .header("X-User-Id", extractUserId(token))
                    .header("X-User-Roles", extractRoles(token))
                    .build();
            
            exchange = exchange.mutate().request(modifiedRequest).build();
        }
        
        return chain.filter(exchange);
    }
}
```

### OAuth2 Resource Server Configuration

```java
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) {
        return http
                .authorizeExchange(exchanges -> exchanges
                        .pathMatchers("/actuator/**").permitAll()
                        .pathMatchers("/api/users/register").permitAll()
                        .pathMatchers(HttpMethod.GET, "/api/users/**").hasRole("USER")
                        .pathMatchers(HttpMethod.POST, "/api/accounts/**").hasRole("USER")
                        .pathMatchers("/api/transfers/**").hasRole("USER")
                        .anyExchange().authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(jwt -> jwt.jwtDecoder(jwtDecoder()))
                )
                .csrf().disable()
                .build();
    }
}
```

## 📊 Data Transfer Objects (DTOs)

### User Service DTOs

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserRegistrationDto {
    @NotBlank(message = "Username is required")
    private String username;
    
    @Email(message = "Invalid email format")
    private String email;
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
    
    @NotBlank(message = "First name is required")
    private String firstName;
    
    @NotBlank(message = "Last name is required")
    private String lastName;
    
    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", message = "Invalid phone number")
    private String phoneNumber;
}

@Data
@Builder
public class UserResponseDto {
    private String userId;
    private String username;
    private String email;
    private String firstName;
    private String lastName;
    private String phoneNumber;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### Account Service DTOs

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AccountCreationDto {
    @NotBlank(message = "User ID is required")
    private String userId;
    
    @NotNull(message = "Account type is required")
    private AccountType accountType;
    
    @DecimalMin(value = "0.0", inclusive = false, message = "Initial deposit must be positive")
    private BigDecimal initialDeposit;
}

@Data
@Builder
public class AccountResponseDto {
    private String accountId;
    private String accountNumber;
    private String userId;
    private AccountType accountType;
    private BigDecimal balance;
    private AccountStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### Transaction Service DTOs

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class TransactionRequestDto {
    @NotBlank(message = "Account ID is required")
    private String accountId;
    
    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.0", inclusive = false, message = "Amount must be positive")
    private BigDecimal amount;
    
    @NotNull(message = "Transaction type is required")
    private TransactionType transactionType;
    
    private String description;
    private String referenceNumber;
}

@Data
@Builder
public class TransactionResponseDto {
    private String transactionId;
    private String accountId;
    private BigDecimal amount;
    private TransactionType transactionType;
    private TransactionStatus status;
    private String description;
    private String referenceNumber;
    private LocalDateTime transactionDate;
    private BigDecimal balanceAfter;
}
```

## 🔄 Pagination & Filtering

### Pagination Standards

```java
@GetMapping("/api/transactions")
public ResponseEntity<PagedResponse<TransactionResponseDto>> getTransactions(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "transactionDate") String sortBy,
        @RequestParam(defaultValue = "desc") String sortDir,
        @RequestParam(required = false) String accountId,
        @RequestParam(required = false) TransactionType type,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate fromDate,
        @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate toDate) {
    
    Pageable pageable = PageRequest.of(page, size, 
            Sort.by(Sort.Direction.fromString(sortDir), sortBy));
    
    Page<TransactionResponseDto> transactions = transactionService
            .getTransactions(accountId, type, fromDate, toDate, pageable);
    
    return ResponseEntity.ok(PagedResponse.<TransactionResponseDto>builder()
            .content(transactions.getContent())
            .page(transactions.getNumber())
            .size(transactions.getSize())
            .totalElements(transactions.getTotalElements())
            .totalPages(transactions.getTotalPages())
            .first(transactions.isFirst())
            .last(transactions.isLast())
            .build());
}
```

## 🚨 Error Handling Strategy

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        return ResponseEntity.badRequest()
                .body(ErrorResponse.builder()
                        .status("error")
                        .message("Validation failed")
                        .errors(ex.getErrors())
                        .timestamp(LocalDateTime.now())
                        .build());
    }
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.notFound()
                .build();
    }
    
    @ExceptionHandler(InsufficientBalanceException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientBalance(InsufficientBalanceException ex) {
        return ResponseEntity.badRequest()
                .body(ErrorResponse.builder()
                        .status("error")
                        .message("Insufficient balance")
                        .timestamp(LocalDateTime.now())
                        .build());
    }
}
```