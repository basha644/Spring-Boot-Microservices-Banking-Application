# 🗄️ Database Design & Schema Architecture

## 📊 Database Architecture Overview

### Database per Service Pattern

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   User Service  │    │ Account Service │    │Transaction Svc  │
│                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │   users_db  │ │    │ │accounts_db  │ │    │ │transactions │ │
│ │             │ │    │ │             │ │    │ │    _db      │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘    └─────────────────┘

┌─────────────────┐    ┌─────────────────┐
│Fund Transfer Svc│    │Sequence Gen Svc │
│                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │ transfers   │ │    │ │ sequences   │ │
│ │    _db      │ │    │ │    _db      │ │
│ └─────────────┘ │    │ └─────────────┘ │
└─────────────────┘    └─────────────────┘
```

## 🏗️ Schema Definitions

### User Service Database Schema

```sql
-- users_db schema
CREATE DATABASE users_db;
USE users_db;

-- Users table
CREATE TABLE users (
    user_id VARCHAR(36) PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone_number VARCHAR(20),
    keycloak_user_id VARCHAR(36) UNIQUE,
    status ENUM('ACTIVE', 'INACTIVE', 'SUSPENDED') DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_username (username),
    INDEX idx_email (email),
    INDEX idx_keycloak_user_id (keycloak_user_id),
    INDEX idx_status (status)
);

-- User profiles table (extended information)
CREATE TABLE user_profiles (
    profile_id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    date_of_birth DATE,
    address_line1 VARCHAR(100),
    address_line2 VARCHAR(100),
    city VARCHAR(50),
    state VARCHAR(50),
    postal_code VARCHAR(20),
    country VARCHAR(50),
    occupation VARCHAR(100),
    annual_income DECIMAL(15,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
);
```

### Account Service Database Schema

```sql
-- accounts_db schema
CREATE DATABASE accounts_db;
USE accounts_db;

-- Accounts table
CREATE TABLE accounts (
    account_id VARCHAR(36) PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    account_type ENUM('SAVINGS', 'CHECKING', 'BUSINESS', 'INVESTMENT') NOT NULL,
    balance DECIMAL(15,2) DEFAULT 0.00,
    currency_code VARCHAR(3) DEFAULT 'USD',
    status ENUM('ACTIVE', 'INACTIVE', 'CLOSED', 'SUSPENDED') DEFAULT 'ACTIVE',
    overdraft_limit DECIMAL(15,2) DEFAULT 0.00,
    interest_rate DECIMAL(5,4) DEFAULT 0.0000,
    minimum_balance DECIMAL(15,2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    closed_at TIMESTAMP NULL,
    
    INDEX idx_account_number (account_number),
    INDEX idx_user_id (user_id),
    INDEX idx_account_type (account_type),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- Account balance history (for audit trail)
CREATE TABLE account_balance_history (
    history_id VARCHAR(36) PRIMARY KEY,
    account_id VARCHAR(36) NOT NULL,
    previous_balance DECIMAL(15,2) NOT NULL,
    new_balance DECIMAL(15,2) NOT NULL,
    change_amount DECIMAL(15,2) NOT NULL,
    change_type ENUM('CREDIT', 'DEBIT') NOT NULL,
    reference_id VARCHAR(36),
    reference_type ENUM('TRANSACTION', 'TRANSFER', 'ADJUSTMENT') NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (account_id) REFERENCES accounts(account_id) ON DELETE CASCADE,
    INDEX idx_account_id (account_id),
    INDEX idx_reference_id (reference_id),
    INDEX idx_created_at (created_at)
);
```

### Transaction Service Database Schema

```sql
-- transactions_db schema
CREATE DATABASE transactions_db;
USE transactions_db;

-- Transactions table
CREATE TABLE transactions (
    transaction_id VARCHAR(36) PRIMARY KEY,
    account_id VARCHAR(36) NOT NULL,
    transaction_type ENUM('DEPOSIT', 'WITHDRAWAL', 'TRANSFER_IN', 'TRANSFER_OUT', 'FEE', 'INTEREST') NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency_code VARCHAR(3) DEFAULT 'USD',
    balance_before DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    description TEXT,
    reference_number VARCHAR(50),
    external_reference_id VARCHAR(36),
    status ENUM('PENDING', 'COMPLETED', 'FAILED', 'CANCELLED') DEFAULT 'PENDING',
    transaction_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_account_id (account_id),
    INDEX idx_transaction_type (transaction_type),
    INDEX idx_reference_number (reference_number),
    INDEX idx_external_reference_id (external_reference_id),
    INDEX idx_status (status),
    INDEX idx_transaction_date (transaction_date),
    INDEX idx_created_at (created_at)
);

-- Transaction fees table
CREATE TABLE transaction_fees (
    fee_id VARCHAR(36) PRIMARY KEY,
    transaction_id VARCHAR(36) NOT NULL,
    fee_type ENUM('TRANSFER_FEE', 'OVERDRAFT_FEE', 'MAINTENANCE_FEE', 'ATM_FEE') NOT NULL,
    fee_amount DECIMAL(15,2) NOT NULL,
    currency_code VARCHAR(3) DEFAULT 'USD',
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id) ON DELETE CASCADE,
    INDEX idx_transaction_id (transaction_id),
    INDEX idx_fee_type (fee_type)
);
```

### Fund Transfer Service Database Schema

```sql
-- transfers_db schema
CREATE DATABASE transfers_db;
USE transfers_db;

-- Fund transfers table
CREATE TABLE fund_transfers (
    transfer_id VARCHAR(36) PRIMARY KEY,
    transfer_reference VARCHAR(50) UNIQUE NOT NULL,
    from_account_id VARCHAR(36) NOT NULL,
    to_account_id VARCHAR(36) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency_code VARCHAR(3) DEFAULT 'USD',
    transfer_type ENUM('INTERNAL', 'EXTERNAL', 'WIRE', 'ACH') DEFAULT 'INTERNAL',
    status ENUM('INITIATED', 'PROCESSING', 'COMPLETED', 'FAILED', 'CANCELLED') DEFAULT 'INITIATED',
    description TEXT,
    fee_amount DECIMAL(15,2) DEFAULT 0.00,
    exchange_rate DECIMAL(10,6) DEFAULT 1.000000,
    initiated_by VARCHAR(36) NOT NULL,
    approved_by VARCHAR(36),
    scheduled_date TIMESTAMP NULL,
    processed_date TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_transfer_reference (transfer_reference),
    INDEX idx_from_account_id (from_account_id),
    INDEX idx_to_account_id (to_account_id),
    INDEX idx_transfer_type (transfer_type),
    INDEX idx_status (status),
    INDEX idx_initiated_by (initiated_by),
    INDEX idx_scheduled_date (scheduled_date),
    INDEX idx_created_at (created_at)
);

-- Transfer status history
CREATE TABLE transfer_status_history (
    history_id VARCHAR(36) PRIMARY KEY,
    transfer_id VARCHAR(36) NOT NULL,
    previous_status ENUM('INITIATED', 'PROCESSING', 'COMPLETED', 'FAILED', 'CANCELLED'),
    new_status ENUM('INITIATED', 'PROCESSING', 'COMPLETED', 'FAILED', 'CANCELLED') NOT NULL,
    status_reason TEXT,
    changed_by VARCHAR(36),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (transfer_id) REFERENCES fund_transfers(transfer_id) ON DELETE CASCADE,
    INDEX idx_transfer_id (transfer_id),
    INDEX idx_new_status (new_status),
    INDEX idx_changed_at (changed_at)
);
```

### Sequence Generator Service Database Schema

```sql
-- sequences_db schema
CREATE DATABASE sequences_db;
USE sequences_db;

-- Sequence generators table
CREATE TABLE sequence_generators (
    sequence_id VARCHAR(36) PRIMARY KEY,
    entity_name VARCHAR(50) UNIQUE NOT NULL,
    current_value BIGINT NOT NULL DEFAULT 0,
    increment_by INT NOT NULL DEFAULT 1,
    prefix VARCHAR(10),
    suffix VARCHAR(10),
    min_value BIGINT DEFAULT 1,
    max_value BIGINT DEFAULT 999999999999,
    cycle_flag BOOLEAN DEFAULT FALSE,
    format_pattern VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_entity_name (entity_name)
);

-- Sequence allocation history
CREATE TABLE sequence_allocations (
    allocation_id VARCHAR(36) PRIMARY KEY,
    sequence_id VARCHAR(36) NOT NULL,
    allocated_value BIGINT NOT NULL,
    formatted_value VARCHAR(50),
    requested_by_service VARCHAR(50) NOT NULL,
    allocated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (sequence_id) REFERENCES sequence_generators(sequence_id) ON DELETE CASCADE,
    INDEX idx_sequence_id (sequence_id),
    INDEX idx_requested_by_service (requested_by_service),
    INDEX idx_allocated_at (allocated_at)
);
```

## 🔗 Cross-Service Data Relationships

### Logical Relationships (No Foreign Keys Across Services)

```
User Service (user_id) ←→ Account Service (user_id)
Account Service (account_id) ←→ Transaction Service (account_id)
Account Service (account_id) ←→ Fund Transfer Service (from_account_id, to_account_id)
All Services ←→ Sequence Generator Service (entity references)
```

## 📊 Data Consistency Patterns

### Eventual Consistency Strategy

```sql
-- Event sourcing table (optional for audit)
CREATE TABLE domain_events (
    event_id VARCHAR(36) PRIMARY KEY,
    aggregate_id VARCHAR(36) NOT NULL,
    aggregate_type VARCHAR(50) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSON NOT NULL,
    event_version INT NOT NULL,
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP NULL,
    
    INDEX idx_aggregate_id (aggregate_id),
    INDEX idx_aggregate_type (aggregate_type),
    INDEX idx_event_type (event_type),
    INDEX idx_occurred_at (occurred_at)
);
```

## 🔧 Database Configuration

### Connection Pool Settings

```yaml
# Application configuration for each service
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 1200000
      connection-timeout: 20000
      leak-detection-threshold: 60000
      
  jpa:
    hibernate:
      ddl-auto: validate
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true
        use_sql_comments: true
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
```

### Indexing Strategy

```sql
-- Performance optimization indexes
-- User Service
CREATE INDEX idx_users_composite ON users(status, created_at);
CREATE INDEX idx_user_profiles_composite ON user_profiles(user_id, country);

-- Account Service
CREATE INDEX idx_accounts_composite ON accounts(user_id, status, account_type);
CREATE INDEX idx_balance_history_composite ON account_balance_history(account_id, created_at);

-- Transaction Service
CREATE INDEX idx_transactions_composite ON transactions(account_id, transaction_date, status);
CREATE INDEX idx_transactions_amount ON transactions(amount, transaction_type);

-- Fund Transfer Service
CREATE INDEX idx_transfers_composite ON fund_transfers(from_account_id, status, created_at);
CREATE INDEX idx_transfers_date_range ON fund_transfers(created_at, processed_date);

-- Sequence Generator Service
CREATE INDEX idx_sequences_composite ON sequence_generators(entity_name, current_value);
```

## 🚨 Data Backup & Recovery

### Backup Strategy

```bash
# Daily backup script for each database
#!/bin/bash
DATABASES=("users_db" "accounts_db" "transactions_db" "transfers_db" "sequences_db")
BACKUP_DIR="/backup/mysql/$(date +%Y%m%d)"

for db in "${DATABASES[@]}"; do
    mysqldump --single-transaction --routines --triggers \
              --user=$DB_USER --password=$DB_PASS \
              $db > $BACKUP_DIR/${db}_$(date +%Y%m%d_%H%M%S).sql
done
```

### Point-in-Time Recovery

```sql
-- Enable binary logging for point-in-time recovery
SET GLOBAL log_bin = ON;
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL sync_binlog = 1;
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```