# 🚀 Deployment Architecture

## 🏗️ Infrastructure Layout

### Development Environment

```
┌─────────────────────────────────────────────────────────────┐
│                    Development Environment                   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Service   │  │   Service   │  │   Service   │         │
│  │  Registry   │  │   Gateway   │  │   Keycloak  │         │
│  │  :8761      │  │   :8080     │  │   :8180     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │    User     │  │   Account   │  │ Transaction │         │
│  │   Service   │  │   Service   │  │   Service   │         │
│  │   :8081     │  │   :8082     │  │   :8083     │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐                          │
│  │Fund Transfer│  │  Sequence   │                          │
│  │   Service   │  │  Generator  │                          │
│  │   :8084     │  │   :8085     │                          │
│  └─────────────┘  └─────────────┘                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              MySQL Database Server                 │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │  users  │ │accounts │ │  txns   │ │transfers│   │   │
│  │  │   DB    │ │   DB    │ │   DB    │ │   DB    │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  │              ┌─────────┐                           │   │
│  │              │sequences│                           │   │
│  │              │   DB    │                           │   │
│  │              └─────────┘                           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Production Environment (Containerized)

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                Load Balancer                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              API Gateway Pods (3x)                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │Gateway-1│  │Gateway-2│  │Gateway-3│             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            Service Registry Pods (2x)               │   │
│  │  ┌─────────┐  ┌─────────┐                          │   │
│  │  │Registry1│  │Registry2│                          │   │
│  │  └─────────┘  └─────────┘                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Business Service Pods                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │ User-1  │  │Account-1│  │  Txn-1  │             │   │
│  │  │ User-2  │  │Account-2│  │  Txn-2  │             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  │  ┌─────────┐  ┌─────────┐                          │   │
│  │  │Transfer1│  │  Seq-1  │                          │   │
│  │  │Transfer2│  │  Seq-2  │                          │   │
│  │  └─────────┘  └─────────┘                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Database Cluster                       │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │   │
│  │  │MySQL-M  │  │MySQL-S1 │  │MySQL-S2 │             │   │
│  │  │(Master) │  │(Slave)  │  │(Slave)  │             │   │
│  │  └─────────┘  └─────────┘  └─────────┘             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 🐳 Docker Configuration

### Service Startup Order

```yaml
# docker-compose.yml startup sequence
version: '3.8'

services:
  # 1. Infrastructure Services (First)
  mysql:
    priority: 1
  
  keycloak:
    priority: 2
    depends_on: [mysql]
  
  service-registry:
    priority: 3
  
  # 2. Core Services (Second)
  sequence-generator:
    priority: 4
    depends_on: [service-registry, mysql]
  
  user-service:
    priority: 5
    depends_on: [service-registry, keycloak, mysql]
  
  # 3. Business Services (Third)
  account-service:
    priority: 6
    depends_on: [user-service, sequence-generator]
  
  transaction-service:
    priority: 7
    depends_on: [account-service, sequence-generator]
  
  fund-transfer-service:
    priority: 8
    depends_on: [account-service, transaction-service]
  
  # 4. Gateway (Last)
  api-gateway:
    priority: 9
    depends_on: [user-service, account-service, transaction-service, fund-transfer-service]
```

## 🔧 Configuration Management

### Environment-Specific Configurations

```
├── config/
│   ├── development/
│   │   ├── application-dev.yml
│   │   ├── database-dev.yml
│   │   └── security-dev.yml
│   ├── staging/
│   │   ├── application-staging.yml
│   │   ├── database-staging.yml
│   │   └── security-staging.yml
│   └── production/
│       ├── application-prod.yml
│       ├── database-prod.yml
│       └── security-prod.yml
```

### Service Configuration Pattern

```yaml
# Common configuration structure
server:
  port: ${SERVICE_PORT:8081}

spring:
  application:
    name: ${SERVICE_NAME}
  
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  
  jpa:
    hibernate:
      ddl-auto: ${DDL_AUTO:validate}
  
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}
  
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
```

## 📊 Monitoring & Observability

### Health Check Endpoints

```
Service Registry:    http://localhost:8761/actuator/health
API Gateway:         http://localhost:8080/actuator/health
User Service:        http://localhost:8081/actuator/health
Account Service:     http://localhost:8082/actuator/health
Transaction Service: http://localhost:8083/actuator/health
Fund Transfer:       http://localhost:8084/actuator/health
Sequence Generator:  http://localhost:8085/actuator/health
```

### Metrics Collection

```
Prometheus Endpoints:
- /actuator/prometheus (all services)
- /actuator/metrics (detailed metrics)

Grafana Dashboards:
- Service Health Overview
- Request Rate & Latency
- Database Connection Pools
- JVM Metrics
```

## 🔐 Security Deployment

### SSL/TLS Configuration

```
Production Security:
├── API Gateway (SSL Termination)
├── Service-to-Service (mTLS)
├── Database Connections (SSL)
└── Keycloak (HTTPS)
```

### Network Security

```
Network Segmentation:
├── Public Subnet (Load Balancer)
├── Application Subnet (Services)
├── Database Subnet (MySQL Cluster)
└── Management Subnet (Monitoring)
```

## 🚀 Scaling Strategy

### Horizontal Scaling Rules

```yaml
# Kubernetes HPA Configuration
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: banking-services-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: banking-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Database Scaling

```
Read Replicas:
- Master: Write operations
- Slave 1: Read operations (User queries)
- Slave 2: Read operations (Reporting)

Connection Pooling:
- HikariCP configuration
- Max pool size: 20 per service instance
- Connection timeout: 30 seconds
```