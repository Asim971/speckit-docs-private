## 3. Backend Microservices Architecture

### 3.1 Service Overview

| Service | Port | Technology | Database | External Dependencies | Status |
|---------|------|------------|----------|----------------------|---------|
| **auth-service** | 4000 | Node.js 18 + Express | PostgreSQL + Redis | Twilio (OTP), AWS KMS | ✅ Exists |
| **patient-management** | 4001 | Node.js 18 + Express | PostgreSQL | FHIR Server, S3 | ✅ Exists |
| **telemedicine-service** | 4002 | Node.js 18 + Express | PostgreSQL | Agora RTC, S3 | ✅ Exists |
| **payment-service** | 4003 | Node.js 18 + Express | PostgreSQL | bKash, SSLCommerz | ❌ Create |
| **medicine-verification** | 4004 | Node.js 18 + Express | PostgreSQL | - | ❌ Create |
| **logistics-tracking** | 4005 | Node.js 18 + Express | PostgreSQL + Redis | Google Maps, Twilio | ❌ Create |
| **loyalty-rewards** | 4006 | Node.js 18 + Express | PostgreSQL + Redis | - | ❌ Create |
| **notification-service** | 4007 | Node.js 18 + Express | PostgreSQL + Redis | Twilio, Firebase, SendGrid | ❌ Create |
| **audit-logging** | 4008 | Node.js 18 + Express | PostgreSQL | S3, CloudWatch | ❌ Create |
| **api-gateway** | 3000 | Kong / AWS ALB | - | All services | ❌ Create |

---

### 3.2 Auth Service (Port 4000)

**Responsibility**: User authentication, authorization, session management

**API Endpoints**:
```typescript
POST   /api/auth/register                    // Create account (phone + OTP)
POST   /api/auth/login                       // Login (phone + password)
POST   /api/auth/otp/send                    // Send OTP via SMS
POST   /api/auth/otp/verify                  // Verify OTP code
POST   /api/auth/refresh                     // Refresh JWT token
POST   /api/auth/logout                      // Invalidate session
GET    /api/auth/profile                     // Get current user profile
PUT    /api/auth/profile                     // Update user profile
POST   /api/auth/password/reset              // Request password reset
PUT    /api/auth/password/reset/:token       // Complete password reset
GET    /api/auth/sessions                    // List active sessions
DELETE /api/auth/sessions/:id                // Revoke session
```

**Database Tables**:
- `users` (id, phone, email, password_hash, role, mfa_enabled, created_at)
- `sessions` (id, user_id, token_hash, expires_at, device_info)
- `otp_codes` (id, phone, code_hash, expires_at, verified_at)
- `password_resets` (id, user_id, token_hash, expires_at, used_at)

**Redis Cache**:
- `session:{token}` → User session data (TTL: 8 hours)
- `otp:{phone}` → OTP code hash (TTL: 5 minutes)
- `rate_limit:{ip}:{endpoint}` → API rate limiting

**Security Features**:
- ✅ bcrypt password hashing (cost factor: 12)
- ✅ JWT tokens with RS256 signing
- ✅ Session timeout: 15min idle / 8hr absolute
- ✅ Rate limiting: 5 login attempts per 15min per IP
- ✅ MFA support (TOTP via Twilio Authy)
- ✅ Device fingerprinting
- ✅ HIPAA audit logging for all auth events

**Integration Points**:
- **Twilio Verify API**: OTP delivery via SMS
- **AWS KMS**: JWT signing key storage
- **Redis**: Session storage + rate limiting
- **audit-logging service**: Auth event logging

---

### 3.3 Patient Management Service (Port 4001)

**Responsibility**: Patient data CRUD, FHIR R4 resource management, consent tracking

**API Endpoints**:
```typescript
// Patient Resources (FHIR R4 Compliant)
POST   /api/patients                         // Create patient (FHIR Patient)
GET    /api/patients/:id                     // Get patient by ID
PUT    /api/patients/:id                     // Update patient
GET    /api/patients/search                  // Search patients (phone, name, MRN)
POST   /api/patients/:id/consents            // Record consent
GET    /api/patients/:id/consents            // List consents
PUT    /api/patients/:id/consents/:consentId // Revoke consent

// Medical Records (FHIR Observation)
POST   /api/patients/:id/observations        // Add vital signs, lab results
GET    /api/patients/:id/observations        // Get medical history

// Prescriptions (FHIR MedicationRequest)
POST   /api/patients/:id/prescriptions       // Create prescription
GET    /api/patients/:id/prescriptions       // List prescriptions
PUT    /api/patients/:id/prescriptions/:rxId // Update prescription status

// GDPR Data Subject Rights
GET    /api/patients/:id/export              // Export all patient data (FHIR Bundle)
DELETE /api/patients/:id                     // Soft delete + anonymize PHI
GET    /api/patients/:id/access-log          // View access history
```

**Database Tables**:
- `patients` (id, mrn, fhir_patient_json, phone, email, dob, gender, address, insurance_id, created_at)
- `consents` (id, patient_id, type, granted_at, revoked_at, scope)
- `medical_observations` (id, patient_id, fhir_observation_json, type, value, unit, recorded_at)
- `prescriptions` (id, patient_id, provider_id, fhir_medication_request_json, status, issued_at)
- `patient_access_log` (id, patient_id, accessed_by_user_id, accessed_at, purpose)

**FHIR R4 Integration**:
- Store full FHIR resources as JSON (PostgreSQL JSONB column)
- Validate against FHIR R4 schemas using `@types/fhir`
- Support FHIR search parameters (name, phone, birthdate)
- Export patient data as FHIR Bundle for interoperability

**HIPAA/GDPR Features**:
- ✅ AES-256 encryption for PHI fields (using AWS KMS)
- ✅ Purpose-of-use tracking for every data access
- ✅ Consent management with granular scopes
- ✅ Right to access: FHIR Bundle export
- ✅ Right to erasure: Soft delete + PHI anonymization
- ✅ Right to portability: Standard FHIR format
- ✅ Audit logging: All PHI operations logged

**Integration Points**:
- **FHIR Server** (HAPI FHIR or AWS HealthLake): Store/retrieve FHIR resources
- **AWS S3**: Store patient documents (insurance cards, medical images)
- **AWS KMS**: Encrypt/decrypt PHI fields
- **audit-logging service**: Log all PHI access

---

### 3.4 Telemedicine Service (Port 4002)

**Responsibility**: Video consultation scheduling, Agora RTC token generation, session management

**API Endpoints**:
```typescript
// Consultation Management
POST   /api/consultations                    // Schedule consultation
GET    /api/consultations/:id                // Get consultation details
PUT    /api/consultations/:id                // Update (reschedule, cancel)
POST   /api/consultations/:id/start          // Start video session (generate Agora token)
POST   /api/consultations/:id/end            // End session + save notes
GET    /api/consultations/:id/recording      // Get session recording URL

// Agora RTC Integration
POST   /api/agora/token                      // Generate Agora RTC token
POST   /api/agora/recording/start            // Start cloud recording
POST   /api/agora/recording/stop             // Stop recording
GET    /api/agora/recording/:sid             // Get recording status

// Provider Availability
GET    /api/providers/:id/availability       // Get available time slots
POST   /api/providers/:id/availability       // Set availability
```

**Database Tables**:
- `consultations` (id, patient_id, provider_id, scheduled_at, duration_min, status, agora_channel_name, recording_url)
- `consultation_notes` (id, consultation_id, provider_id, notes_encrypted, diagnosis, prescription_ids)
- `provider_availability` (id, provider_id, day_of_week, start_time, end_time, timezone)
- `agora_sessions` (id, consultation_id, channel_name, uid, token, started_at, ended_at)

**Agora RTC Integration**:
```typescript
// Token generation (E2EE AES-128-GCM)
const AgoraToken = require('agora-access-token');

function generateRTCToken(channelName: string, uid: number, role: 'publisher' | 'subscriber') {
  const appId = process.env.AGORA_APP_ID;
  const appCertificate = process.env.AGORA_APP_CERTIFICATE;
  const expirationTime = 3600; // 1 hour
  
  const token = AgoraToken.RtcTokenBuilder.buildTokenWithUid(
    appId,
    appCertificate,
    channelName,
    uid,
    role === 'publisher' ? AgoraToken.RtcRole.PUBLISHER : AgoraToken.RtcRole.SUBSCRIBER,
    Math.floor(Date.now() / 1000) + expirationTime
  );
  
  return token;
}
```

**HIPAA Compliance**:
- ✅ End-to-end encryption (Agora E2EE)
- ✅ Cloud recording encrypted at rest (S3 SSE-KMS)
- ✅ Access control: Only patient + provider can join channel
- ✅ Audit logging: All session starts/ends logged
- ✅ Business Associate Agreement (BAA) signed with Agora

**Performance Optimization (3G Networks)**:
- ✅ Adaptive bitrate (384kbps → 128kbps auto-adjust)
- ✅ Audio-only fallback if video quality <240p
- ✅ <200ms latency target on 3G
- ✅ Connection quality monitoring with UI feedback

**Integration Points**:
- **Agora RTC SDK**: Video/audio streaming infrastructure
- **AWS S3**: Store encrypted session recordings
- **notification-service**: Send consultation reminders
- **patient-management**: Link prescriptions to consultations

---

### 3.5 Payment Service (Port 4003) ❌ TO CREATE

**Responsibility**: Payment gateway integration (bKash, SSLCommerz, COD), transaction management

**API Endpoints**:
```typescript
// Payment Initiation
POST   /api/payments/bkash/initiate          // Start bKash payment
POST   /api/payments/sslcommerz/initiate     // Start SSLCommerz payment
POST   /api/payments/cod/create              // Create COD order

// Payment Callbacks (Webhooks)
POST   /api/payments/bkash/callback          // bKash IPN
POST   /api/payments/sslcommerz/callback     // SSLCommerz IPN
POST   /api/payments/bkash/webhook           // bKash webhook

// Payment Status
GET    /api/payments/:txnId                  // Get payment status
POST   /api/payments/:txnId/refund           // Initiate refund
GET    /api/payments/user/:userId            // Get user payment history

// Admin
GET    /api/payments/reconcile               // Daily reconciliation report
```

**Database Tables**:
- `payments` (id, user_id, order_id, amount, currency, gateway, status, txn_id, initiated_at, completed_at)
- `refunds` (id, payment_id, amount, reason, status, initiated_at, completed_at)
- `payment_methods` (id, user_id, gateway, token, card_last4, expiry, is_default)

**bKash Integration** (Primary - 65% Bangladesh market share):
```typescript
// bKash Checkout API Flow
const axios = require('axios');

async function initiateBKashPayment(amount: number, invoiceNumber: string) {
  // Step 1: Grant token
  const tokenResponse = await axios.post('https://checkout.pay.bka.sh/v1.2.0-beta/checkout/token/grant', {
    app_key: process.env.BKASH_APP_KEY,
    app_secret: process.env.BKASH_APP_SECRET,
  });
  
  const { id_token } = tokenResponse.data;
  
  // Step 2: Create payment
  const paymentResponse = await axios.post('https://checkout.pay.bka.sh/v1.2.0-beta/checkout/payment/create', {
    amount: amount.toFixed(2),
    currency: 'BDT',
    intent: 'sale',
    merchantInvoiceNumber: invoiceNumber,
  }, {
    headers: {
      'Authorization': `Bearer ${id_token}`,
      'X-APP-Key': process.env.BKASH_APP_KEY,
    }
  });
  
  return {
    paymentID: paymentResponse.data.paymentID,
    bkashURL: paymentResponse.data.bkashURL, // Redirect user here
  };
}
```

**SSLCommerz Integration** (Secondary - Nagad, Rocket, cards):
```typescript
async function initiateSSLCommerzPayment(amount: number, orderId: string, customerInfo: any) {
  const sslcz = new SSLCommerzPayment(
    process.env.SSLCOMMERZ_STORE_ID,
    process.env.SSLCOMMERZ_STORE_PASSWORD,
    false // false = sandbox, true = live
  );
  
  const data = {
    total_amount: amount,
    currency: 'BDT',
    tran_id: orderId,
    success_url: `${process.env.API_BASE_URL}/api/payments/sslcommerz/callback`,
    fail_url: `${process.env.API_BASE_URL}/api/payments/sslcommerz/callback`,
    cancel_url: `${process.env.API_BASE_URL}/api/payments/sslcommerz/callback`,
    ipn_url: `${process.env.API_BASE_URL}/api/payments/sslcommerz/webhook`,
    ...customerInfo,
  };
  
  const apiResponse = await sslcz.init(data);
  return apiResponse.GatewayPageURL; // Redirect user here
}
```

**PCI-DSS Compliance**:
- ✅ No card data stored (tokenization via SSLCommerz)
- ✅ TLS 1.3 for all payment API calls
- ✅ Payment gateway credentials encrypted with AWS KMS
- ✅ Webhook signature verification
- ✅ Idempotency keys to prevent duplicate charges

**Integration Points**:
- **bKash API**: Mobile wallet payments (merchant account required - 2-4 week approval)
- **SSLCommerz API**: Multi-gateway aggregator
- **notification-service**: Payment confirmation emails/SMS
- **audit-logging**: Log all payment transactions

---

### 3.6 Medicine Verification Service (Port 4004) ❌ TO CREATE

**Responsibility**: QR/barcode scanning for medicine authenticity, counterfeit reporting

**API Endpoints**:
```typescript
POST   /api/medicines/verify                 // Verify medicine by QR/barcode
GET    /api/medicines/:id                    // Get medicine details
POST   /api/medicines/report-counterfeit     // Report fake medicine
GET    /api/medicines/verified               // List verified medicines
```

**Database Tables**:
- `medicines` (id, name, manufacturer, batch_number, qr_code_hash, barcode, verified_at)
- `verification_logs` (id, medicine_id, user_id, scanned_at, location, result)
- `counterfeit_reports` (id, medicine_id, user_id, photo_url, location, reported_at)

**QR Code Format**:
```
{
  "medicine_id": "MED-12345",
  "batch_number": "BT-2025-001",
  "manufacturer": "Square Pharmaceuticals",
  "expiry_date": "2026-12-31",
  "signature": "SHA256_HMAC_SIGNATURE"
}
```

**Integration Points**:
- Pharmacy databases for authenticity verification
- AWS S3 for counterfeit medicine photos

---

### 3.7 Logistics Tracking Service (Port 4005) ❌ TO CREATE

**Responsibility**: Real-time GPS tracking for medicine deliveries, delivery person assignment

**API Endpoints**:
```typescript
POST   /api/deliveries                       // Create delivery order
GET    /api/deliveries/:id                   // Get delivery status
PUT    /api/deliveries/:id/location          // Update delivery person location (GPS)
POST   /api/deliveries/:id/complete          // Mark delivery complete
GET    /api/deliveries/active                // Get active deliveries (for delivery person)
```

**Database Tables**:
- `deliveries` (id, order_id, pharmacy_id, patient_id, delivery_person_id, status, created_at, delivered_at)
- `delivery_locations` (id, delivery_id, lat, lng, recorded_at)
- `delivery_persons` (id, name, phone, vehicle_type, is_available)

**Real-Time Tracking**:
- WebSocket connection for live location updates
- Redis Pub/Sub for location broadcasts
- Google Maps API for route optimization

**Integration Points**:
- **Google Maps API**: Geocoding, route optimization
- **Twilio SMS**: Delivery notifications
- **Redis Pub/Sub**: Real-time location broadcasting

---

### 3.8 Loyalty & Rewards Service (Port 4006) ❌ TO CREATE

**Responsibility**: Gamification, referral tracking, discount management

**API Endpoints**:
```typescript
GET    /api/loyalty/points/:userId            // Get user points balance
POST   /api/loyalty/points/earn               // Award points (order completion, referral)
POST   /api/loyalty/points/redeem             // Redeem points for discount
GET    /api/loyalty/referrals/:userId         // Get referral status
POST   /api/loyalty/referrals/generate        // Generate referral code
```

**Database Tables**:
- `loyalty_accounts` (id, user_id, points_balance, tier)
- `loyalty_transactions` (id, account_id, points, type, description, created_at)
- `referrals` (id, referrer_id, referred_id, code, status, reward_points, created_at)

**Gamification Features**:
- Points for: First order (+100), Referral (+50), Reviews (+10)
- Tiers: Bronze (<500 pts), Silver (500-1000), Gold (>1000)
- Rewards: 10% discount (500 pts), Free delivery (200 pts)

---

### 3.9 Notification Service (Port 4007) ❌ TO CREATE

**Responsibility**: Multi-channel notifications (Email, SMS, Push, In-App)

**API Endpoints**:
```typescript
POST   /api/notifications/send                // Send notification (any channel)
POST   /api/notifications/email               // Send email
POST   /api/notifications/sms                 // Send SMS
POST   /api/notifications/push                // Send push notification
GET    /api/notifications/:userId             // Get user notifications
PUT    /api/notifications/:id/read            // Mark as read
```

**Database Tables**:
- `notifications` (id, user_id, type, channel, subject, body, sent_at, read_at)
- `notification_preferences` (id, user_id, email_enabled, sms_enabled, push_enabled)

**Integration Points**:
- **Twilio**: SMS notifications
- **SendGrid**: Email notifications
- **Firebase Cloud Messaging**: Push notifications

---

### 3.10 Audit Logging Service (Port 4008) ❌ TO CREATE

**Responsibility**: HIPAA-compliant audit trails, tamper-proof logging

**API Endpoints**:
```typescript
POST   /api/audit/log                         // Create audit log entry
GET    /api/audit/logs                        // Search audit logs (admin only)
GET    /api/audit/logs/user/:userId           // Get user's access history
GET    /api/audit/logs/resource/:resourceId   // Get resource access history
```

**Database Tables**:
- `audit_logs` (id, user_id, resource_type, resource_id, action, metadata_encrypted, ip_address, timestamp)

**HIPAA Requirements**:
- ✅ 6-year retention minimum
- ✅ Immutable logs (append-only table)
- ✅ Encrypted log metadata (AWS KMS)
- ✅ Monthly exports to S3 (compliance archive)

**Integration Points**:
- **AWS S3**: Long-term audit log archival
- **CloudWatch**: Real-time log analysis
- All services must call audit-logging for PHI operations

---

### 3.11 Service Communication Patterns

**Synchronous (REST)**:
- Frontend apps → API Gateway → Backend services
- Service-to-service: Direct HTTP calls with circuit breaker (use `axios` + `opossum`)

**Asynchronous (Message Queue)**:
- Bull Queue (Redis-backed) for background jobs:
  - Email sending
  - SMS notifications
  - Prescription refill reminders
  - Daily reconciliation reports

**Event-Driven (Pub/Sub)**:
- Redis Pub/Sub for real-time features:
  - Delivery location updates
  - Consultation status changes
  - In-app notifications

**Service Mesh** (Optional - Future):
- Istio or Linkerd for:
  - Automatic mTLS between services
  - Distributed tracing
  - Traffic splitting (A/B testing)

---

### 3.12 API Gateway Configuration

**Technology**: Kong Gateway or AWS Application Load Balancer

**Responsibilities**:
- Request routing to backend services
- Rate limiting (100 req/min per user)
- JWT validation
- Request/response logging
- CORS configuration

**Kong Plugins**:
```yaml
plugins:
  - name: jwt
    config:
      secret_is_base64: false
      key_claim_name: sub
  
  - name: rate-limiting
    config:
      minute: 100
      policy: local
  
  - name: cors
    config:
      origins: ["https://jibonflow.com", "jibonflow://app"]
      credentials: true
  
  - name: request-transformer
    config:
      add:
        headers: ["X-Request-ID:$(uuid)"]
```

**Routing Rules**:
```
/api/auth/*          → auth-service:4000
/api/patients/*      → patient-management:4001
/api/consultations/* → telemedicine-service:4002
/api/payments/*      → payment-service:4003
/api/medicines/*     → medicine-verification:4004
/api/deliveries/*    → logistics-tracking:4005
/api/loyalty/*       → loyalty-rewards:4006
/api/notifications/* → notification-service:4007
/api/audit/*         → audit-logging:4008
```

---
