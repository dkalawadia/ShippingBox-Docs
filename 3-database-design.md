# Database Design Specification
## ShippingBox.com
### Version 1.0
### Date: January 19, 2025

## 1. Database Architecture

### 1.1 Overview
```plaintext
┌─────────────────┐     ┌──────────────┐     ┌───────────────┐
│ Primary Database│────▶│   Replicas   │────▶│  Read Replicas│
└─────────────────┘     └──────────────┘     └───────────────┘
        │                                            │
        │                                            │
        ▼                                            ▼
┌─────────────────┐                         ┌───────────────┐
│  Backup Store   │                         │  Cache Layer  │
└─────────────────┘                         └───────────────┘
```

### 1.2 Database Systems
- Primary: PostgreSQL 14
- Cache: Redis 6
- Search: Elasticsearch 8
- Time Series: TimescaleDB (PostgreSQL extension)

## 2. Schema Design

### 2.1 Core Tables

#### 2.1.1 Users and Authentication
```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company_name VARCHAR(200),
    role VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    email_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP WITH TIME ZONE,
    preferences JSONB DEFAULT '{}',
    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT valid_status CHECK (status IN ('active', 'inactive', 'suspended'))
);

-- Authentication tokens
CREATE TABLE auth_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    token_type VARCHAR(50) NOT NULL,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP WITH TIME ZONE,
    revoked_at TIMESTAMP WITH TIME ZONE,
    CONSTRAINT valid_token_type CHECK (token_type IN ('access', 'refresh', 'reset'))
);
```

#### 2.1.2 Shipping and Calculations
```sql
-- Box templates
CREATE TABLE box_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    length DECIMAL(10,2) NOT NULL,
    width DECIMAL(10,2) NOT NULL,
    height DECIMAL(10,2) NOT NULL,
    weight_limit DECIMAL(10,2) NOT NULL,
    internal_length DECIMAL(10,2) NOT NULL,
    internal_width DECIMAL(10,2) NOT NULL,
    internal_height DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2) NOT NULL,
    material VARCHAR(50) NOT NULL,
    strength_rating VARCHAR(50) NOT NULL,
    supplier_id UUID REFERENCES suppliers(id),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_dimensions CHECK (
        length > 0 AND width > 0 AND height > 0 AND
        internal_length > 0 AND internal_width > 0 AND internal_height > 0 AND
        internal_length < length AND internal_width < width AND internal_height < height
    )
);

-- Calculations
CREATE TABLE calculations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    input_data JSONB NOT NULL,
    result JSONB NOT NULL,
    selected_box_id UUID REFERENCES box_templates(id),
    status VARCHAR(50) NOT NULL DEFAULT 'completed',
    error_message TEXT,
    calculation_time INTERVAL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_status CHECK (status IN ('pending', 'completed', 'failed'))
);

-- Shipping rates
CREATE TABLE shipping_rates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calculation_id UUID NOT NULL REFERENCES calculations(id) ON DELETE CASCADE,
    carrier_code VARCHAR(50) NOT NULL,
    service_code VARCHAR(50) NOT NULL,
    service_name VARCHAR(200) NOT NULL,
    base_rate DECIMAL(10,2) NOT NULL,
    total_rate DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    transit_days INTEGER,
    guaranteed_delivery BOOLEAN DEFAULT false,
    rate_details JSONB NOT NULL,
    valid_until TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

#### 2.1.3 Address and Location
```sql
-- Addresses
CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    address_type VARCHAR(50) NOT NULL,
    company_name VARCHAR(200),
    attention VARCHAR(100),
    street1 VARCHAR(200) NOT NULL,
    street2 VARCHAR(200),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    country VARCHAR(2) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(255),
    is_residential BOOLEAN DEFAULT true,
    is_default BOOLEAN DEFAULT false,
    validated_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT valid_address_type CHECK (address_type IN ('shipping', 'billing', 'both'))
);

-- Postal codes
CREATE TABLE postal_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country VARCHAR(2) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6),
    timezone VARCHAR(50),
    active BOOLEAN DEFAULT true,
    UNIQUE(country, postal_code)
);
```

## 3. Indexing Strategy

### 3.1 Primary Indexes
```sql
-- Users indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status) WHERE status != 'inactive';
CREATE INDEX idx_users_company ON users(company_name) WHERE company_name IS NOT NULL;

-- Calculations indexes
CREATE INDEX idx_calculations_user_date ON calculations(user_id, created_at DESC);
CREATE INDEX idx_calculations_status ON calculations(status, created_at DESC);
CREATE INDEX idx_calculations_box ON calculations(selected_box_id) WHERE selected_box_id IS NOT NULL;

-- Shipping rates indexes
CREATE INDEX idx_shipping_rates_calc ON shipping_rates(calculation_id);
CREATE INDEX idx_shipping_rates_carrier ON shipping_rates(carrier_code, service_code);
CREATE INDEX idx_shipping_rates_validity ON shipping_rates(valid_until) 
    WHERE valid_until > CURRENT_TIMESTAMP;
```

### 3.2 Full Text Search Indexes
```sql
-- Address search
CREATE INDEX idx_addresses_fts ON addresses USING gin(
    to_tsvector('english',
        COALESCE(company_name, '') || ' ' ||
        COALESCE(street1, '') || ' ' ||
        COALESCE(street2, '') || ' ' ||
        COALESCE(city, '') || ' ' ||
        COALESCE(state, '') || ' ' ||
        COALESCE(postal_code, '')
    )
);

-- Box template search
CREATE INDEX idx_box_templates_fts ON box_templates USING gin(
    to_tsvector('english',
        name || ' ' ||
        material || ' ' ||
        strength_rating
    )
);
```

### 3.3 JSONB Indexes
```sql
-- Calculation input/result indexes
CREATE INDEX idx_calculations_input ON calculations USING gin(input_data);
CREATE INDEX idx_calculations_result ON calculations USING gin(result);

-- User preferences index
CREATE INDEX idx_users_preferences ON users USING gin(preferences);

-- Shipping rate details index
CREATE INDEX idx_shipping_rates_details ON shipping_rates USING gin(rate_details);
```

## 4. Partitioning Strategy

### 4.1 Time-Based Partitioning
```sql
-- Calculations partitioning
CREATE TABLE calculations_partition (
    LIKE calculations INCLUDING DEFAULTS INCLUDING CONSTRAINTS
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE calculations_y2025m01 PARTITION OF calculations_partition
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE calculations_y2025m02 PARTITION OF calculations_partition
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Shipping rates partitioning
CREATE TABLE shipping_rates_partition (
    LIKE shipping_rates INCLUDING DEFAULTS INCLUDING CONSTRAINTS
) PARTITION BY RANGE (created_at);
```

### 4.2 Partition Maintenance
```sql
-- Partition maintenance function
CREATE OR REPLACE FUNCTION maintain_partitions()
RETURNS void AS $$
DECLARE
    future_date DATE;
    partition_name TEXT;
    partition_start DATE;
    partition_end DATE;
BEGIN
    -- Create partitions 3 months ahead
    future_date := DATE_TRUNC('month', NOW() + INTERVAL '3 months');
    
    WHILE future_date >= DATE_TRUNC('month', NOW()) LOOP
        partition_name := 'calculations_y' || 
            TO_CHAR(future_date, 'YYYY') ||
            'm' || TO_CHAR(future_date, 'MM');
            
        partition_start := DATE_TRUNC('month', future_date);
        partition_end := partition_start + INTERVAL '1 month';
        
        -- Create partition if it doesn't exist
        IF NOT EXISTS (
            SELECT 1
            FROM pg_class c
            JOIN pg_namespace n ON n.oid = c.relnamespace
            WHERE c.relname = partition_name
        ) THEN
            EXECUTE format(
                'CREATE TABLE %I PARTITION OF calculations_partition
                FOR VALUES FROM (%L) TO (%L)',
                partition_name, partition_start, partition_end
            );
        END IF;
        
        future_date := future_date - INTERVAL '1 month';
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

## 5. Optimization Strategies

### 5.1 Materialized Views
```sql
-- Shipping rate analytics
CREATE MATERIALIZED VIEW mv_shipping_rate_analytics AS
SELECT 
    carrier_code,
    service_code,
    date_trunc('day', created_at) as rate_date,
    COUNT(*) as quote_count,
    AVG(base_rate) as avg_base_rate,
    AVG(total_rate) as avg_total_rate,
    MIN(total_rate) as min_rate,
    MAX(total_rate) as max_rate
FROM shipping_rates
GROUP BY carrier_code, service_code, date_trunc('day', created_at);

CREATE UNIQUE INDEX idx_mv_shipping_rate_analytics 
ON mv_shipping_rate_analytics(carrier_code, service_code, rate_date);

-- Box usage statistics
CREATE MATERIALIZED VIEW mv_box_usage_stats AS
SELECT
    bt.id as box_template_id,
    bt.name as box_name,
    COUNT(c.id) as usage_count,
    AVG(EXTRACT(EPOCH FROM c.calculation_time)) as avg_calc_time,
    COUNT(DISTINCT c.user_id) as unique_users
FROM box_templates bt
LEFT JOIN calculations c ON c.selected_box_id = bt.id
WHERE c.created_at >= NOW() - INTERVAL '30 days'
GROUP BY bt.id, bt.name;
```

### 5.2 Query Optimization
```sql
-- Analyze frequently used tables
ANALYZE users;
ANALYZE calculations;
ANALYZE shipping_rates;
ANALYZE box_templates;

-- Update statistics
ALTER TABLE calculations ALTER COLUMN status SET STATISTICS 1000;
ALTER TABLE shipping_rates ALTER COLUMN carrier_code SET STATISTICS 1000;
```