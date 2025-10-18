# JibonFlow Database Schema - PostgreSQL 14 + Redis 7

**Generated**: 2025-10-11  
**Database Version**: PostgreSQL 14.x  
**Cache Layer**: Redis 7.x  
**Compliance**: HIPAA-compliant schema design with encrypted PHI fields

---

## Table of Contents

1. [Schema Overview](#1-schema-overview)
2. [Core Tables](#2-core-tables)
3. [Healthcare Tables](#3-healthcare-tables)
4. [Payment Tables](#4-payment-tables)
5. [B2B Tables](#5-b2b-tables)
6. [System Tables](#6-system-tables)
7. [Indexes & Performance](#7-indexes--performance)
8. [Redis Cache Strategy](#8-redis-cache-strategy)
9. [Backup & Recovery](#9-backup--recovery)

---

## 1. Schema Overview

### Database Structure

```
jibonflow_production (PostgreSQL 14)
├── public (default schema)
│   ├── users                    # All user accounts
│   ├── sessions                 # Active sessions
│   ├── patients                 # Patient profiles
│   ├── providers                # Healthcare providers
│   ├── chws                     # Community Health Workers
│   ├── pharmacies               # Pharmacy partners
│   ├── pharma_companies         # Pharmaceutical companies
│   ├── consultations            # Telemedicine sessions
│   ├── prescriptions            # Medicine prescriptions
│   ├── payments                 # Payment transactions
│   ├── deliveries               # Medicine deliveries
│   ├── medicines                # Medicine catalog
│   ├── audit_logs               # HIPAA audit trails
│   └── notifications            # Multi-channel notifications
│
└── Redis (in-memory cache)
    ├── sessions:*               # User sessions
    ├── api_cache:*              # API response cache
    ├── rate_limit:*             # Rate limiting counters
    └── realtime:*               # WebSocket data
```

### Encryption Strategy

**PHI Fields** (HIPAA-protected):
- Encrypted at rest using PostgreSQL `pgcrypto` extension + AWS KMS
- Columns: `name`, `phone`, `email`, `address`, `medical_notes`, `prescriptions`
- Encryption: AES-256-GCM
- Key management: AWS KMS (automatic rotation every 90 days)

**Example Encrypted Column**:
```sql
-- Encrypted column type
CREATE EXTENSION IF NOT EXISTS pgcrypto;

ALTER TABLE patients 
  ADD COLUMN phone_encrypted BYTEA,
  ADD COLUMN email_encrypted BYTEA,
  ADD COLUMN address_encrypted BYTEA;

-- Encryption function (uses KMS-managed key)
CREATE FUNCTION encrypt_phi(plaintext TEXT, key_id TEXT) 
RETURNS BYTEA AS $$
  -- Call AWS KMS API via pgx_aws extension
  -- Returns AES-256-GCM encrypted ciphertext
$$ LANGUAGE plpgsql;
```

---

## 2. Core Tables

### 2.1 `users` Table

**Purpose**: Central user authentication and authorization

```sql
CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone             VARCHAR(20) UNIQUE NOT NULL,
  phone_encrypted   BYTEA,                    -- HIPAA encrypted
  email             VARCHAR(255) UNIQUE,
  email_encrypted   BYTEA,                    -- HIPAA encrypted
  password_hash     VARCHAR(255) NOT NULL,     -- bcrypt hash
  role              VARCHAR(50) NOT NULL,      -- patient, provider, chw, pharmacy_admin, pharma_admin, system_admin
  mfa_enabled       BOOLEAN DEFAULT false,
  mfa_secret        VARCHAR(255),              -- TOTP secret (encrypted)
  verified_at       TIMESTAMP,
  last_login_at     TIMESTAMP,
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW(),
  deleted_at        TIMESTAMP,                 -- Soft delete
  
  CONSTRAINT valid_role CHECK (role IN ('patient', 'provider', 'chw', 'pharmacy_admin', 'pharma_admin', 'system_admin'))
);

CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

**Audit Logging**: All `INSERT`, `UPDATE`, `DELETE` operations logged to `audit_logs` table

---

### 2.2 `sessions` Table

**Purpose**: JWT session management with device tracking

```sql
CREATE TABLE sessions (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash        VARCHAR(255) NOT NULL UNIQUE,  -- SHA-256 hash of JWT
  device_info       JSONB,                         -- {device_type, os, browser, ip}
  ip_address        INET NOT NULL,
  expires_at        TIMESTAMP NOT NULL,
  last_activity_at  TIMESTAMP DEFAULT NOW(),
  created_at        TIMESTAMP DEFAULT NOW(),
  revoked_at        TIMESTAMP,
  
  CONSTRAINT session_timeout CHECK (expires_at > created_at)
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token_hash ON sessions(token_hash);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);

-- Auto-delete expired sessions (daily cron job)
CREATE OR REPLACE FUNCTION cleanup_expired_sessions() 
RETURNS void AS $$
  DELETE FROM sessions WHERE expires_at < NOW() - INTERVAL '7 days';
$$ LANGUAGE sql;
```

---

### 2.3 `otp_codes` Table

**Purpose**: OTP verification for phone-based authentication

```sql
CREATE TABLE otp_codes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone             VARCHAR(20) NOT NULL,
  code_hash         VARCHAR(255) NOT NULL,     -- bcrypt hash of OTP
  purpose           VARCHAR(50) NOT NULL,      -- 'registration', 'login', 'password_reset'
  attempts          INT DEFAULT 0,
  max_attempts      INT DEFAULT 3,
  expires_at        TIMESTAMP NOT NULL,        -- 5 minutes from creation
  verified_at       TIMESTAMP,
  created_at        TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT valid_purpose CHECK (purpose IN ('registration', 'login', 'password_reset'))
);

CREATE INDEX idx_otp_phone ON otp_codes(phone);
CREATE INDEX idx_otp_expires_at ON otp_codes(expires_at);

-- Rate limiting: Max 3 OTP requests per phone per 15 minutes
CREATE OR REPLACE FUNCTION check_otp_rate_limit(phone_number VARCHAR) 
RETURNS BOOLEAN AS $$
DECLARE
  request_count INT;
BEGIN
  SELECT COUNT(*) INTO request_count 
  FROM otp_codes 
  WHERE phone = phone_number 
    AND created_at > NOW() - INTERVAL '15 minutes';
  
  RETURN request_count < 3;
END;
$$ LANGUAGE plpgsql;
```

---

## 3. Healthcare Tables

### 3.1 `patients` Table

**Purpose**: Patient demographic and medical information (FHIR R4 compliant)

```sql
CREATE TABLE patients (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  mrn                 VARCHAR(50) UNIQUE NOT NULL,  -- Medical Record Number
  fhir_patient_json   JSONB NOT NULL,               -- Full FHIR R4 Patient resource
  
  -- Encrypted PHI fields
  full_name_encrypted BYTEA,
  phone_encrypted     BYTEA,
  email_encrypted     BYTEA,
  dob_encrypted       BYTEA,
  gender              VARCHAR(20),                  -- 'male', 'female', 'other', 'prefer_not_to_say'
  address_encrypted   BYTEA,
  
  -- Medical info
  blood_group         VARCHAR(10),                  -- A+, B-, O+, etc.
  insurance_id        UUID REFERENCES insurance_policies(id),
  primary_language    VARCHAR(10) DEFAULT 'bn',     -- 'en', 'bn' (Bangla)
  
  -- Consent tracking
  consented_at        TIMESTAMP,
  consent_version     VARCHAR(20),
  
  created_at          TIMESTAMP DEFAULT NOW(),
  updated_at          TIMESTAMP DEFAULT NOW(),
  deleted_at          TIMESTAMP,                    -- Soft delete (GDPR)
  
  CONSTRAINT valid_gender CHECK (gender IN ('male', 'female', 'other', 'prefer_not_to_say'))
);

CREATE INDEX idx_patients_user_id ON patients(user_id);
CREATE INDEX idx_patients_mrn ON patients(mrn);
CREATE INDEX idx_patients_created_at ON patients(created_at DESC);

-- FHIR search support (GIN index for JSONB)
CREATE INDEX idx_patients_fhir_gin ON patients USING GIN (fhir_patient_json);
```

---

### 3.2 `providers` Table

**Purpose**: Healthcare providers (doctors, nurses, specialists)

```sql
CREATE TABLE providers (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  license_number      VARCHAR(100) UNIQUE NOT NULL,
  specialization      VARCHAR(100) NOT NULL,      -- 'general_practitioner', 'cardiologist', etc.
  years_experience    INT,
  consultation_fee    DECIMAL(10,2) NOT NULL,     -- in BDT
  is_verified         BOOLEAN DEFAULT false,
  verified_at         TIMESTAMP,
  verified_by         UUID REFERENCES users(id),
  rating              DECIMAL(3,2) DEFAULT 0.00,  -- Average rating (0-5)
  total_consultations INT DEFAULT 0,
  created_at          TIMESTAMP DEFAULT NOW(),
  updated_at          TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_providers_user_id ON providers(user_id);
CREATE INDEX idx_providers_specialization ON providers(specialization);
CREATE INDEX idx_providers_rating ON providers(rating DESC);
```

---

### 3.3 `consultations` Table

**Purpose**: Telemedicine consultation sessions

```sql
CREATE TABLE consultations (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id          UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
  provider_id         UUID NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  scheduled_at        TIMESTAMP NOT NULL,
  duration_minutes    INT DEFAULT 30,
  status              VARCHAR(50) NOT NULL DEFAULT 'scheduled',
  
  -- Agora RTC session info
  agora_channel_name  VARCHAR(255) UNIQUE,
  agora_recording_sid VARCHAR(255),
  recording_url       VARCHAR(500),              -- S3 URL (encrypted)
  
  -- Session metadata
  started_at          TIMESTAMP,
  ended_at            TIMESTAMP,
  patient_joined_at   TIMESTAMP,
  provider_joined_at  TIMESTAMP,
  
  -- Consultation outcome
  diagnosis           TEXT,                      -- Encrypted
  prescription_id     UUID REFERENCES prescriptions(id),
  follow_up_required  BOOLEAN DEFAULT false,
  follow_up_date      DATE,
  
  -- Payment
  payment_id          UUID REFERENCES payments(id),
  amount              DECIMAL(10,2) NOT NULL,
  
  created_at          TIMESTAMP DEFAULT NOW(),
  updated_at          TIMESTAMP DEFAULT NOW(),
  cancelled_at        TIMESTAMP,
  cancellation_reason TEXT,
  
  CONSTRAINT valid_status CHECK (status IN ('scheduled', 'in_progress', 'completed', 'cancelled', 'no_show'))
);

CREATE INDEX idx_consultations_patient_id ON consultations(patient_id);
CREATE INDEX idx_consultations_provider_id ON consultations(provider_id);
CREATE INDEX idx_consultations_scheduled_at ON consultations(scheduled_at);
CREATE INDEX idx_consultations_status ON consultations(status);
```

---

### 3.4 `prescriptions` Table

**Purpose**: Medicine prescriptions (FHIR MedicationRequest)

```sql
CREATE TABLE prescriptions (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  consultation_id         UUID REFERENCES consultations(id),
  patient_id              UUID NOT NULL REFERENCES patients(id),
  provider_id             UUID NOT NULL REFERENCES providers(id),
  fhir_medication_request JSONB NOT NULL,           -- FHIR R4 MedicationRequest
  
  -- Prescription details
  medicine_id             UUID REFERENCES medicines(id),
  dosage                  VARCHAR(255),             -- "500mg twice daily"
  quantity                INT,
  duration_days           INT,
  refills_allowed         INT DEFAULT 0,
  refills_remaining       INT DEFAULT 0,
  
  -- Status
  status                  VARCHAR(50) DEFAULT 'active',
  issued_at               TIMESTAMP DEFAULT NOW(),
  expires_at              TIMESTAMP,
  dispensed_at            TIMESTAMP,
  dispensed_by_pharmacy   UUID REFERENCES pharmacies(id),
  
  created_at              TIMESTAMP DEFAULT NOW(),
  updated_at              TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT valid_status CHECK (status IN ('active', 'completed', 'cancelled', 'expired'))
);

CREATE INDEX idx_prescriptions_patient_id ON prescriptions(patient_id);
CREATE INDEX idx_prescriptions_provider_id ON prescriptions(provider_id);
CREATE INDEX idx_prescriptions_status ON prescriptions(status);
CREATE INDEX idx_prescriptions_issued_at ON prescriptions(issued_at DESC);
```

---

## 4. Payment Tables

### 4.1 `payments` Table

**Purpose**: Payment transactions (bKash, SSLCommerz, COD)

```sql
CREATE TABLE payments (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id           UUID NOT NULL REFERENCES users(id),
  order_id          UUID,                        -- References orders table (if exists)
  consultation_id   UUID REFERENCES consultations(id),
  
  -- Payment details
  amount            DECIMAL(10,2) NOT NULL,
  currency          VARCHAR(3) DEFAULT 'BDT',
  gateway           VARCHAR(50) NOT NULL,        -- 'bkash', 'sslcommerz', 'cod'
  status            VARCHAR(50) NOT NULL DEFAULT 'pending',
  
  -- Gateway transaction IDs
  txn_id            VARCHAR(255) UNIQUE,         -- Gateway transaction ID
  invoice_number    VARCHAR(100) UNIQUE,
  
  -- Timestamps
  initiated_at      TIMESTAMP DEFAULT NOW(),
  completed_at      TIMESTAMP,
  failed_at         TIMESTAMP,
  refunded_at       TIMESTAMP,
  
  -- Metadata
  gateway_response  JSONB,                       -- Raw gateway response
  failure_reason    TEXT,
  
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT valid_gateway CHECK (gateway IN ('bkash', 'sslcommerz', 'nagad', 'rocket', 'cod')),
  CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded'))
);

CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_payments_txn_id ON payments(txn_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_initiated_at ON payments(initiated_at DESC);
```

---

### 4.2 `refunds` Table

**Purpose**: Payment refund tracking

```sql
CREATE TABLE refunds (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id        UUID NOT NULL REFERENCES payments(id) ON DELETE CASCADE,
  amount            DECIMAL(10,2) NOT NULL,
  reason            TEXT NOT NULL,
  status            VARCHAR(50) DEFAULT 'pending',
  gateway_refund_id VARCHAR(255),
  initiated_at      TIMESTAMP DEFAULT NOW(),
  completed_at      TIMESTAMP,
  created_at        TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'completed', 'failed'))
);

CREATE INDEX idx_refunds_payment_id ON refunds(payment_id);
CREATE INDEX idx_refunds_status ON refunds(status);
```

---

## 5. B2B Tables

### 5.1 `pharmacies` Table

**Purpose**: Pharmacy partner information

```sql
CREATE TABLE pharmacies (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  admin_user_id     UUID NOT NULL REFERENCES users(id),
  name              VARCHAR(255) NOT NULL,
  license_number    VARCHAR(100) UNIQUE NOT NULL,
  phone             VARCHAR(20) NOT NULL,
  email             VARCHAR(255),
  address           TEXT NOT NULL,
  location          GEOGRAPHY(POINT),          -- PostGIS for lat/lng
  is_verified       BOOLEAN DEFAULT false,
  verified_at       TIMESTAMP,
  rating            DECIMAL(3,2) DEFAULT 0.00,
  total_orders      INT DEFAULT 0,
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_pharmacies_location ON pharmacies USING GIST(location);
CREATE INDEX idx_pharmacies_verified ON pharmacies(is_verified);
```

---

### 5.2 `medicines` Table

**Purpose**: Medicine catalog with QR codes

```sql
CREATE TABLE medicines (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name              VARCHAR(255) NOT NULL,
  generic_name      VARCHAR(255),
  manufacturer_id   UUID REFERENCES pharma_companies(id),
  category          VARCHAR(100),              -- 'antibiotic', 'pain_reliever', etc.
  dosage_form       VARCHAR(50),               -- 'tablet', 'syrup', 'injection'
  strength          VARCHAR(50),               -- '500mg', '10ml'
  price             DECIMAL(10,2) NOT NULL,
  qr_code_hash      VARCHAR(255) UNIQUE,       -- SHA-256 of QR code data
  barcode           VARCHAR(100),
  is_prescription_required BOOLEAN DEFAULT true,
  is_verified       BOOLEAN DEFAULT false,
  created_at        TIMESTAMP DEFAULT NOW(),
  updated_at        TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_medicines_name ON medicines USING GIN(to_tsvector('english', name));
CREATE INDEX idx_medicines_manufacturer_id ON medicines(manufacturer_id);
CREATE INDEX idx_medicines_qr_code_hash ON medicines(qr_code_hash);
```

---

## 6. System Tables

### 6.1 `audit_logs` Table

**Purpose**: HIPAA-compliant audit trail (immutable, 6-year retention)

```sql
CREATE TABLE audit_logs (
  id                BIGSERIAL PRIMARY KEY,       -- Use BIGSERIAL for high volume
  user_id           UUID REFERENCES users(id),
  resource_type     VARCHAR(100) NOT NULL,       -- 'patient', 'prescription', 'consultation'
  resource_id       UUID NOT NULL,
  action            VARCHAR(50) NOT NULL,        -- 'view', 'create', 'update', 'delete', 'export'
  metadata_encrypted BYTEA,                      -- Encrypted JSON metadata
  ip_address        INET NOT NULL,
  user_agent        TEXT,
  purpose_of_use    VARCHAR(255),                -- HIPAA requirement
  timestamp         TIMESTAMP DEFAULT NOW() NOT NULL
);

-- Partitioning by month for performance (6-year retention = 72 partitions)
CREATE TABLE audit_logs_2025_10 PARTITION OF audit_logs
  FOR VALUES FROM ('2025-10-01') TO ('2025-11-01');

-- Auto-create partitions (monthly cron job)
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp DESC);

-- Immutable table (prevent DELETE/UPDATE)
CREATE OR REPLACE RULE audit_logs_no_delete AS
  ON DELETE TO audit_logs DO INSTEAD NOTHING;

CREATE OR REPLACE RULE audit_logs_no_update AS
  ON UPDATE TO audit_logs DO INSTEAD NOTHING;
```

---

## 7. Indexes & Performance

### 7.1 Full-Text Search Indexes

**Medicine Search**:
```sql
CREATE INDEX idx_medicines_fulltext ON medicines
  USING GIN(to_tsvector('english', name || ' ' || COALESCE(generic_name, '')));

-- Usage:
SELECT * FROM medicines
WHERE to_tsvector('english', name || ' ' || COALESCE(generic_name, '')) 
      @@ to_tsquery('english', 'paracetamol | acetaminophen');
```

**Patient Search** (encrypted fields require decryption first - performance hit):
```sql
-- Create materialized view for search (refreshed hourly)
CREATE MATERIALIZED VIEW patient_search_mv AS
SELECT 
  id,
  mrn,
  pgp_sym_decrypt(phone_encrypted, current_setting('app.encryption_key')) as phone,
  pgp_sym_decrypt(full_name_encrypted, current_setting('app.encryption_key')) as full_name
FROM patients
WHERE deleted_at IS NULL;

CREATE INDEX idx_patient_search_phone ON patient_search_mv(phone);
CREATE INDEX idx_patient_search_name ON patient_search_mv USING GIN(to_tsvector('english', full_name));

-- Refresh every hour
REFRESH MATERIALIZED VIEW CONCURRENTLY patient_search_mv;
```

---

### 7.2 Query Performance Optimization

**Partitioning Strategy**:
- `audit_logs`: Partition by month (range partitioning)
- `consultations`: Partition by quarter (range partitioning on `scheduled_at`)
- `payments`: Partition by year (range partitioning on `initiated_at`)

**Example Partitioning**:
```sql
-- Create parent table
CREATE TABLE consultations_partitioned (
  LIKE consultations INCLUDING ALL
) PARTITION BY RANGE (scheduled_at);

-- Create partitions (Q1 2025)
CREATE TABLE consultations_2025_q1 PARTITION OF consultations_partitioned
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Create partitions (Q2 2025)
CREATE TABLE consultations_2025_q2 PARTITION OF consultations_partitioned
  FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');
```

---

## 8. Redis Cache Strategy

### 8.1 Cache Keys Structure

**Session Cache** (TTL: 8 hours):
```redis
SET session:{token_hash} "{user_id, role, permissions}" EX 28800
```

**API Response Cache** (TTL: 5 minutes):
```redis
SET api_cache:{endpoint}:{params_hash} "{response_json}" EX 300
```

**Rate Limiting** (TTL: 15 minutes):
```redis
INCR rate_limit:{ip}:{endpoint}
EXPIRE rate_limit:{ip}:{endpoint} 900
```

**Real-Time Data** (Pub/Sub):
```redis
PUBLISH realtime:delivery:{delivery_id} "{lat, lng, eta}"
```

---

### 8.2 Cache Invalidation

**Write-Through Pattern**:
```typescript
async function updatePatient(patientId: string, data: any) {
  // 1. Update database
  await db.query('UPDATE patients SET ... WHERE id = $1', [patientId, ...]);
  
  // 2. Invalidate cache
  await redis.del(`patient:${patientId}`);
  
  // 3. Publish cache invalidation event
  await redis.publish('cache:invalidate', JSON.stringify({ type: 'patient', id: patientId }));
}
```

---

## 9. Backup & Recovery

### 9.1 Backup Strategy

**Daily Automated Backups** (AWS RDS):
- Full snapshot: Every day at 2 AM UTC (30-day retention)
- Transaction logs: Continuous archival to S3 (Point-in-time recovery)
- Encryption: AES-256 at rest (AWS KMS)

**Manual Backups**:
```bash
# Export full database
pg_dump -h jibonflow-prod.rds.amazonaws.com -U postgres -Fc jibonflow_production > backup_$(date +%Y%m%d).dump

# Restore from backup
pg_restore -h localhost -U postgres -d jibonflow_production backup_20251011.dump
```

---

### 9.2 Disaster Recovery

**RPO (Recovery Point Objective)**: 1 hour (transaction logs archived every 5 minutes)  
**RTO (Recovery Time Objective)**: 4 hours (includes restore + validation)

**Multi-Region Replication** (Future):
- Primary: ap-southeast-1 (Singapore)
- DR: ap-south-1 (Mumbai)
- Read replicas in both regions

---

**Schema Version**: 1.0.0  
**Last Updated**: 2025-10-11  
**Maintained by**: SpecKit Architecture Agent
