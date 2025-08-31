# 🔄 Service Flow Documentation

## 📋 Business Process Flows

### 1. User Registration Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant U as User Service
    participant K as Keycloak
    participant DB as MySQL

    C->>G: POST /users/register
    G->>G: Validate JWT
    G->>U: Forward request
    U->>K: Create user in Keycloak
    K-->>U: User created
    U->>DB: Save user details
    DB-->>U: User saved
    U-->>G: Registration response
    G-->>C: Success response
```

### 2. Account Creation Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant A as Account Service
    participant U as User Service
    participant S as Sequence Generator
    participant DB as MySQL

    C->>G: POST /accounts
    G->>A: Create account request
    A->>U: Validate user exists
    U-->>A: User validation response
    A->>S: Generate account number
    S-->>A: Account number
    A->>DB: Save account
    DB-->>A: Account saved
    A-->>G: Account created
    G-->>C: Success response
```

### 3. Fund Transfer Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant F as Fund Transfer Service
    participant A as Account Service
    participant T as Transaction Service
    participant S as Sequence Generator
    participant DB as MySQL

    C->>G: POST /transfers
    G->>F: Transfer request
    F->>A: Validate source account
    A-->>F: Account valid
    F->>A: Validate target account
    A-->>F: Account valid
    F->>A: Check balance
    A-->>F: Balance sufficient
    F->>S: Generate transfer ID
    S-->>F: Transfer ID
    F->>T: Create debit transaction
    T-->>F: Debit recorded
    F->>T: Create credit transaction
    T-->>F: Credit recorded
    F->>A: Update source balance
    F->>A: Update target balance
    F->>DB: Save transfer record
    DB-->>F: Transfer saved
    F-->>G: Transfer completed
    G-->>C: Success response
```

### 4. Transaction History Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant T as Transaction Service
    participant A as Account Service
    participant DB as MySQL

    C->>G: GET /transactions/{accountId}
    G->>T: Get transactions request
    T->>A: Validate account access
    A-->>T: Access granted
    T->>DB: Query transactions
    DB-->>T: Transaction list
    T-->>G: Transaction response
    G-->>C: Transaction history
```

## 🔐 Security Flow

### Authentication & Authorization Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant K as Keycloak
    participant S as Service

    C->>K: Login request
    K-->>C: JWT Token
    C->>G: API request + JWT
    G->>K: Validate token
    K-->>G: Token valid
    G->>S: Forward request
    S-->>G: Service response
    G-->>C: API response
```

## 📊 Data Flow Patterns

### Service Dependencies

```
User Service (Independent)
    ↓
Account Service → User Service, Sequence Generator
    ↓
Transaction Service → Account Service, Sequence Generator
    ↓
Fund Transfer Service → Account Service, Transaction Service, Sequence Generator
```

### Database Interaction Pattern

```
Service Layer
    ↓
Repository Layer (Spring Data JPA)
    ↓
MySQL Database (Service-specific schema)
```

## 🚨 Error Handling Flow

### Service Error Propagation

```mermaid
graph LR
    A[Service Error] --> B[Exception Handler]
    B --> C[Error Response DTO]
    C --> D[HTTP Status Code]
    D --> E[Client Response]
```

### Common Error Scenarios

1. **User Not Found**: 404 - User does not exist
2. **Insufficient Balance**: 400 - Cannot process transaction
3. **Account Inactive**: 403 - Account is closed/suspended
4. **Service Unavailable**: 503 - Downstream service failure
5. **Validation Error**: 400 - Invalid request data

## 🔄 Retry & Circuit Breaker Patterns

### Service Resilience

```mermaid
graph TB
    A[Service Call] --> B{Service Available?}
    B -->|Yes| C[Execute Request]
    B -->|No| D[Circuit Breaker Open]
    D --> E[Fallback Response]
    C --> F{Success?}
    F -->|Yes| G[Return Response]
    F -->|No| H[Retry Logic]
    H --> I{Max Retries?}
    I -->|No| A
    I -->|Yes| E
```

## 📈 Performance Optimization Flows

### Caching Strategy

```
Request → Cache Check → Cache Hit? → Return Cached Data
                    ↓
                Cache Miss → Database Query → Cache Update → Return Data
```

### Connection Pooling

```
Service Request → Connection Pool → Available Connection? → Execute Query
                                ↓
                            Wait/Create New → Execute Query
```