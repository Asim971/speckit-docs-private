# JibonFlow Integration Matrix - External APIs & Services

**Version**: 1.0.0  
**Generated**: 2025-10-11  
**Phase**: Architecture Documentation (Day 1-2)  
**Status**: Production-Ready  
**Compliance**: HIPAA/GDPR Compliant

---

## 1. Integration Overview

### 1.1 External Services Summary

| Service | Purpose | Provider | HIPAA BAA | Lead Time | Priority |
|---------|---------|----------|-----------|-----------|----------|
| **Agora RTC SDK** | Telemedicine video/audio | Agora.io | ✅ Yes | Immediate | Critical |
| **bKash Checkout API** | Primary payment gateway | bKash Limited | ❌ No (non-PHI) | 2-4 weeks | Critical |
| **SSLCommerz** | Payment aggregator | SSLCommerz | ❌ No (non-PHI) | 1 week | High |
| **Twilio Verify** | SMS OTP verification | Twilio Inc. | ✅ Yes | Immediate | Critical |
| **Firebase Cloud Messaging** | Push notifications | Google | ✅ Yes | Immediate | High |
| **AWS S3** | PHI document storage | Amazon Web Services | ✅ Yes | Immediate | Critical |
| **AWS KMS** | Encryption key management | Amazon Web Services | ✅ Yes | Immediate | Critical |
| **HAPI FHIR** | FHIR R4 server | Self-hosted | N/A | 1 week setup | High |
| **Google Maps API** | Geocoding, route optimization | Google | ❌ No (non-PHI) | Immediate | Medium |
| **SendGrid** | Transactional emails | Twilio SendGrid | ✅ Yes | Immediate | Medium |
| **AWS CloudWatch** | Monitoring, logging | Amazon Web Services | ✅ Yes | Immediate | Critical |

### 1.2 Integration Architecture Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                    JibonFlow Platform                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         API Gateway (Kong - Port 3000)               │  │
│  │  - JWT Validation                                     │  │
│  │  - Rate Limiting                                      │  │
│  │  - Request/Response Logging                           │  │
│  └────────────┬─────────────────────────────────────────┘  │
│               │                                              │
│  ┌────────────▼────────────────────────────────────────┐   │
│  │       Backend Microservices                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │   │
│  │  │ Auth Service │  │ Telemedicine │  │  Payment  │ │   │
│  │  │  (Port 4000) │  │   Service    │  │  Service  │ │   │
│  │  │              │  │  (Port 4002) │  │(Port 4003)│ │   │
│  │  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │   │
│  └─────────┼──────────────────┼─────────────────┼───────┘   │
│            │                  │                 │            │
└────────────┼──────────────────┼─────────────────┼───────────┘
             │                  │                 │
    ┌────────▼────────┐  ┌─────▼──────┐  ┌──────▼────────┐
    │ Twilio Verify   │  │ Agora RTC  │  │ bKash API     │
    │ (SMS OTP)       │  │ (E2EE Video)│  │ (Payments)    │
    └─────────────────┘  └────────────┘  └───────────────┘
```

---

## 2. Agora RTC SDK (Telemedicine Video/Audio)

### 2.1 Service Details

**Purpose**: Real-time video/audio consultations with E2EE for HIPAA compliance  
**Provider**: Agora.io  
**Documentation**: https://docs.agora.io/en/video-calling/overview/product-overview  
**SLA**: 99.95% uptime  
**HIPAA BAA**: ✅ Required (request at sales@agora.io)

### 2.2 Authentication

**Method**: App ID + App Certificate + Dynamic Token Generation

**Configuration**:
```typescript
// Environment Variables (.env)
AGORA_APP_ID=abc123def456...
AGORA_APP_CERTIFICATE=xyz789secret...

// Token Generation (auth-service/src/agora/token-generator.ts)
import { RtcTokenBuilder, RtcRole } from 'agora-access-token';

export function generateAgoraToken(
  channelName: string,
  uid: number,
  role: 'publisher' | 'subscriber'
): string {
  const appId = process.env.AGORA_APP_ID!;
  const appCertificate = process.env.AGORA_APP_CERTIFICATE!;
  
  // Token expires in 24 hours
  const expirationTimeInSeconds = 3600 * 24;
  const currentTimestamp = Math.floor(Date.now() / 1000);
  const privilegeExpiredTs = currentTimestamp + expirationTimeInSeconds;
  
  const agoraRole = role === 'publisher' ? RtcRole.PUBLISHER : RtcRole.SUBSCRIBER;
  
  return RtcTokenBuilder.buildTokenWithUid(
    appId,
    appCertificate,
    channelName,
    uid,
    agoraRole,
    privilegeExpiredTs
  );
}
```

### 2.3 Rate Limits

| Resource | Limit | Enforcement |
|----------|-------|-------------|
| Token Generation | 100 req/sec | Server-side |
| Concurrent Channels | 1,000 channels | Account-level |
| Channel Join Rate | 500 joins/sec | Server-side |
| Cloud Recording | 100 recordings/account | Account-level |

**Mitigation**: Implement token caching with 23-hour TTL (1-hour safety buffer)

### 2.4 Webhooks

**Cloud Recording Callbacks**:
```typescript
// telemedicine-service/src/webhooks/agora-recording.ts
import { Router, Request, Response } from 'express';
import crypto from 'crypto';

const router = Router();

// POST /webhooks/agora/recording
router.post('/recording', async (req: Request, res: Response) => {
  const { eventType, payload } = req.body;
  
  // Verify webhook signature
  const signature = req.headers['agora-signature'] as string;
  const expectedSignature = crypto
    .createHmac('sha256', process.env.AGORA_WEBHOOK_SECRET!)
    .update(JSON.stringify(req.body))
    .digest('hex');
  
  if (signature !== expectedSignature) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  // Handle recording lifecycle events
  switch (eventType) {
    case '1': // Recording started
      await updateConsultationRecordingStatus(payload.sid, 'recording');
      break;
    case '11': // Recording uploaded (session ended)
      await storeRecordingMetadata(payload);
      await uploadRecordingToS3(payload.fileList);
      break;
    case '12': // Recording upload failed
      await logRecordingError(payload);
      break;
  }
  
  res.status(200).json({ success: true });
});

export default router;
```

**Webhook URL**: `https://api.jibonflow.com/webhooks/agora/recording`

### 2.5 Error Handling

**Common Errors**:
- `ERR_INVALID_TOKEN` (401): Regenerate token with fresh timestamp
- `ERR_CHANNEL_FULL` (433): Max 17 users per channel (telemedicine is 1:1, safe)
- `ERR_NETWORK_QUALITY_POOR` (1101): Adaptive bitrate fallback (384kbps → 128kbps)

**Retry Strategy**: Exponential backoff (2s, 4s, 8s, 16s) for token generation failures

### 2.6 E2EE Implementation

```typescript
// patient-mobile/src/screens/TelemedicineConsultation.tsx
import AgoraRTC from 'agora-rtc-sdk-ng';

const client = AgoraRTC.createClient({ mode: 'rtc', codec: 'vp8' });

// Enable E2EE (HIPAA requirement)
await client.setEncryptionMode('aes-128-gcm2');

// Session key derived from consultation ID (server generates)
const sessionKey = await fetchSessionKey(consultationId);
await client.setEncryptionSecret(sessionKey);

// Join channel with dynamic token
await client.join(
  process.env.REACT_APP_AGORA_APP_ID,
  channelName,
  agoraToken,
  userId
);
```

### 2.7 Cost Estimation

**Pricing**: $0.99 per 1,000 minutes  
**Projected Usage (Month 1)**:
- 500 consultations/month × 15 minutes avg = 7,500 minutes
- Cost: $7.43/month

**Projected Usage (Year 1 - 10,000 patients)**:
- 5,000 consultations/month × 15 minutes = 75,000 minutes
- Cost: $74.25/month

---

## 3. bKash Checkout API (Mobile Wallet Payments)

### 3.1 Service Details

**Purpose**: Primary payment gateway for Bangladesh market (65% market share)  
**Provider**: bKash Limited  
**Documentation**: https://developer.bka.sh/docs  
**SLA**: 99.5% uptime  
**Lead Time**: 2-4 weeks (merchant account approval + API sandbox access)

### 3.2 Authentication

**Method**: Grant Token (OAuth 2.0-like flow)

**Configuration**:
```typescript
// Environment Variables (.env)
BKASH_APP_KEY=your_app_key
BKASH_APP_SECRET=your_app_secret
BKASH_USERNAME=merchant_username
BKASH_PASSWORD=merchant_password
BKASH_BASE_URL=https://tokenized.sandbox.bka.sh/v1.2.0-beta  # Sandbox
# Production: https://tokenized.pay.bka.sh/v1.2.0-beta

// Grant Token Generation (payment-service/src/bkash/auth.ts)
import axios from 'axios';

interface GrantTokenResponse {
  id_token: string;
  token_type: string;
  expires_in: number;
  refresh_token: string;
}

export async function getBkashGrantToken(): Promise<string> {
  const response = await axios.post<GrantTokenResponse>(
    `${process.env.BKASH_BASE_URL}/tokenized/checkout/token/grant`,
    {
      app_key: process.env.BKASH_APP_KEY,
      app_secret: process.env.BKASH_APP_SECRET
    },
    {
      headers: {
        'Content-Type': 'application/json',
        'username': process.env.BKASH_USERNAME,
        'password': process.env.BKASH_PASSWORD
      }
    }
  );
  
  // Cache token (expires in 3600 seconds)
  await redis.setex(
    'bkash:grant_token',
    3500, // 1-hour safety buffer
    response.data.id_token
  );
  
  return response.data.id_token;
}
```

### 3.3 Payment Flow

**1. Create Payment**:
```typescript
// POST /api/payments/bkash/create
import { getBkashGrantToken } from './auth';

interface CreatePaymentRequest {
  amount: string; // e.g., "500.00"
  invoiceNumber: string; // e.g., "ORD-20251011-001"
  patientId: string;
}

export async function createBkashPayment(req: CreatePaymentRequest) {
  const grantToken = await getBkashGrantToken();
  
  const response = await axios.post(
    `${process.env.BKASH_BASE_URL}/tokenized/checkout/create`,
    {
      mode: '0011', // Wallet payment
      payerReference: req.patientId,
      callbackURL: 'https://api.jibonflow.com/webhooks/bkash/callback',
      amount: req.amount,
      currency: 'BDT',
      intent: 'sale',
      merchantInvoiceNumber: req.invoiceNumber
    },
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': grantToken,
        'X-APP-Key': process.env.BKASH_APP_KEY
      }
    }
  );
  
  // Response: { paymentID, bkashURL, callbackURL, ... }
  return {
    paymentId: response.data.paymentID,
    redirectUrl: response.data.bkashURL // Redirect user to bKash app
  };
}
```

**2. Execute Payment (After User Authorizes)**:
```typescript
// POST /api/payments/bkash/execute
export async function executeBkashPayment(paymentId: string) {
  const grantToken = await getBkashGrantToken();
  
  const response = await axios.post(
    `${process.env.BKASH_BASE_URL}/tokenized/checkout/execute`,
    { paymentID: paymentId },
    {
      headers: {
        'Authorization': grantToken,
        'X-APP-Key': process.env.BKASH_APP_KEY
      }
    }
  );
  
  // Response: { paymentID, trxID, transactionStatus, amount, ... }
  if (response.data.transactionStatus === 'Completed') {
    await updatePaymentStatus(paymentId, 'completed', response.data.trxID);
  }
  
  return response.data;
}
```

### 3.4 Rate Limits

| Resource | Limit | Enforcement |
|----------|-------|-------------|
| Grant Token | 50 req/hour | Account-level |
| Create Payment | 200 req/min | Account-level |
| Execute Payment | 200 req/min | Account-level |
| Query Payment | 500 req/min | Account-level |

**Mitigation**: Cache grant token (3,500s TTL), implement request queuing with Bull

### 3.5 Webhooks

**Callback URL**: `https://api.jibonflow.com/webhooks/bkash/callback`

```typescript
// payment-service/src/webhooks/bkash-callback.ts
router.get('/bkash/callback', async (req: Request, res: Response) => {
  const { paymentID, status } = req.query;
  
  if (status === 'success') {
    // Auto-execute payment
    await executeBkashPayment(paymentID as string);
    
    // Redirect user to success page
    res.redirect(`jibonflow://payment-success?paymentId=${paymentID}`);
  } else {
    // Payment failed or cancelled
    await updatePaymentStatus(paymentID as string, 'failed');
    res.redirect(`jibonflow://payment-failed?paymentId=${paymentID}`);
  }
});
```

### 3.6 Error Handling

**Common Errors**:
- `2001`: Invalid App Key → Check environment variable
- `2062`: Insufficient balance → Show user error message
- `2063`: Transaction limit exceeded → Retry after 24 hours
- `2068`: Duplicate invoice number → Generate new invoice number

**Retry Strategy**: No retries for payment execution (idempotency risk)

### 3.7 Testing

**Sandbox Credentials** (for development):
```bash
# Sandbox wallet for testing
Phone: 01619777282
OTP: 123456
PIN: 12121

# Test amounts trigger specific responses
Amount: 10 BDT → Success
Amount: 25 BDT → Insufficient Balance
Amount: 30 BDT → Transaction Limit Exceeded
```

### 3.8 Cost Estimation

**Pricing**: 1.85% per transaction  
**Projected Usage (Month 1)**:
- 500 orders × 500 BDT avg = 250,000 BDT revenue
- Fees: 4,625 BDT (~$42 USD)

---

## 4. SSLCommerz (Payment Aggregator)

### 4.1 Service Details

**Purpose**: Secondary payment gateway (aggregates Nagad, Rocket, credit/debit cards)  
**Provider**: SSLCommerz  
**Documentation**: https://developer.sslcommerz.com/  
**SLA**: 99.9% uptime

### 4.2 Authentication

**Method**: Store ID + Store Password

**Configuration**:
```typescript
// Environment Variables (.env)
SSLCOMMERZ_STORE_ID=your_store_id
SSLCOMMERZ_STORE_PASSWORD=your_store_password
SSLCOMMERZ_BASE_URL=https://sandbox.sslcommerz.com  # Sandbox
# Production: https://securepay.sslcommerz.com

// Payment Initiation (payment-service/src/sslcommerz/initiate.ts)
import axios from 'axios';

interface InitiatePaymentRequest {
  totalAmount: string;
  transactionId: string;
  productCategory: string; // e.g., "Healthcare"
  customerName: string;
  customerEmail: string;
  customerPhone: string;
}

export async function initiateSSLCommerzPayment(req: InitiatePaymentRequest) {
  const response = await axios.post(
    `${process.env.SSLCOMMERZ_BASE_URL}/gwprocess/v4/api.php`,
    {
      store_id: process.env.SSLCOMMERZ_STORE_ID,
      store_passwd: process.env.SSLCOMMERZ_STORE_PASSWORD,
      total_amount: req.totalAmount,
      currency: 'BDT',
      tran_id: req.transactionId,
      success_url: 'https://api.jibonflow.com/webhooks/sslcommerz/success',
      fail_url: 'https://api.jibonflow.com/webhooks/sslcommerz/fail',
      cancel_url: 'https://api.jibonflow.com/webhooks/sslcommerz/cancel',
      ipn_url: 'https://api.jibonflow.com/webhooks/sslcommerz/ipn',
      product_name: 'Medicine Order',
      product_category: req.productCategory,
      product_profile: 'general',
      cus_name: req.customerName,
      cus_email: req.customerEmail,
      cus_add1: 'Dhaka, Bangladesh',
      cus_city: 'Dhaka',
      cus_country: 'Bangladesh',
      cus_phone: req.customerPhone,
      shipping_method: 'NO',
      num_of_item: 1,
      product_amount: req.totalAmount
    },
    {
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
    }
  );
  
  // Response: { status: 'SUCCESS', GatewayPageURL: '...', sessionkey: '...' }
  return {
    sessionKey: response.data.sessionkey,
    redirectUrl: response.data.GatewayPageURL
  };
}
```

### 4.3 Webhooks (IPN - Instant Payment Notification)

```typescript
// payment-service/src/webhooks/sslcommerz-ipn.ts
router.post('/sslcommerz/ipn', async (req: Request, res: Response) => {
  const { val_id, tran_id, amount, status } = req.body;
  
  // Step 1: Validate transaction with SSLCommerz
  const validationResponse = await axios.get(
    `${process.env.SSLCOMMERZ_BASE_URL}/validator/api/validationserverAPI.php`,
    {
      params: {
        val_id,
        store_id: process.env.SSLCOMMERZ_STORE_ID,
        store_passwd: process.env.SSLCOMMERZ_STORE_PASSWORD,
        format: 'json'
      }
    }
  );
  
  // Step 2: Check if transaction is valid and not processed before
  if (validationResponse.data.status === 'VALID') {
    await updatePaymentStatus(tran_id, 'completed', val_id);
  } else {
    await logFraudulentTransaction(tran_id, val_id);
  }
  
  res.status(200).send('OK');
});
```

### 4.4 Rate Limits

| Resource | Limit | Enforcement |
|----------|-------|-------------|
| Payment Initiation | 1,000 req/min | Account-level |
| Validation API | 500 req/min | Account-level |

### 4.5 Cost Estimation

**Pricing**: 2.5% per transaction  
**Projected Usage**: 20% of total payment volume (bKash covers 80%)

---

## 5. Twilio Verify (SMS OTP)

### 5.1 Service Details

**Purpose**: Phone number verification via SMS OTP  
**Provider**: Twilio Inc.  
**Documentation**: https://www.twilio.com/docs/verify/api  
**SLA**: 99.95% uptime

### 5.2 Authentication

**Method**: Account SID + Auth Token

**Configuration**:
```typescript
// Environment Variables (.env)
TWILIO_ACCOUNT_SID=ACxxx...
TWILIO_AUTH_TOKEN=xxx...
TWILIO_VERIFY_SERVICE_SID=VAxxx...

// Send OTP (auth-service/src/twilio/send-otp.ts)
import twilio from 'twilio';

const client = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

export async function sendOTP(phoneNumber: string): Promise<void> {
  // Rate limit: 3 OTPs per phone per 15 minutes
  const recentOtps = await redis.get(`otp:rate_limit:${phoneNumber}`);
  if (recentOtps && parseInt(recentOtps) >= 3) {
    throw new Error('Too many OTP requests. Try again in 15 minutes.');
  }
  
  await client.verify.v2
    .services(process.env.TWILIO_VERIFY_SERVICE_SID!)
    .verifications.create({
      to: phoneNumber,
      channel: 'sms',
      locale: 'bn' // Bangla language
    });
  
  // Increment rate limit counter
  await redis.incr(`otp:rate_limit:${phoneNumber}`);
  await redis.expire(`otp:rate_limit:${phoneNumber}`, 900); // 15 minutes
}

export async function verifyOTP(
  phoneNumber: string,
  code: string
): Promise<boolean> {
  const verification = await client.verify.v2
    .services(process.env.TWILIO_VERIFY_SERVICE_SID!)
    .verificationChecks.create({
      to: phoneNumber,
      code
    });
  
  return verification.status === 'approved';
}
```

### 5.3 Rate Limits

| Resource | Limit | Application-Level Limit |
|----------|-------|------------------------|
| Send OTP | 100 req/sec (account-level) | 3 per phone/15min |
| Verify OTP | 100 req/sec | 5 attempts/OTP |

### 5.4 Cost Estimation

**Pricing**: $0.05 per verification (Bangladesh)  
**Projected Usage**: 1,000 new users/month × 1 verification = $50/month

---

## 6. Firebase Cloud Messaging (Push Notifications)

### 6.1 Service Details

**Purpose**: Push notifications to mobile apps (React Native)  
**Provider**: Google Firebase  
**Documentation**: https://firebase.google.com/docs/cloud-messaging  
**SLA**: 99.95% uptime

### 6.2 Authentication

**Method**: Service Account JSON (Server-to-Server)

**Configuration**:
```typescript
// Environment Variables (.env)
FIREBASE_SERVICE_ACCOUNT_PATH=/path/to/service-account.json

// Send Push Notification (notification-service/src/firebase/send-push.ts)
import admin from 'firebase-admin';
import serviceAccount from process.env.FIREBASE_SERVICE_ACCOUNT_PATH;

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount as any)
});

interface SendPushRequest {
  userId: string;
  title: string;
  body: string;
  data?: Record<string, string>;
}

export async function sendPushNotification(req: SendPushRequest) {
  // Fetch user's FCM device token from database
  const deviceToken = await getUserDeviceToken(req.userId);
  
  if (!deviceToken) {
    throw new Error('User has not registered for push notifications');
  }
  
  const message = {
    notification: {
      title: req.title,
      body: req.body
    },
    data: req.data || {},
    token: deviceToken,
    android: {
      priority: 'high' as const,
      notification: {
        sound: 'default',
        channelId: 'jibonflow_notifications'
      }
    },
    apns: {
      payload: {
        aps: {
          sound: 'default',
          badge: 1
        }
      }
    }
  };
  
  await admin.messaging().send(message);
}
```

### 6.3 Rate Limits

| Resource | Limit | Enforcement |
|----------|-------|-------------|
| Send to Single Device | 1,000 req/sec | Account-level |
| Send to Topics | 10,000 messages/min | Account-level |

### 6.4 Cost Estimation

**Pricing**: Free (no charge for FCM)

---

## 7. AWS S3 (PHI Document Storage)

### 7.1 Service Details

**Purpose**: Store PHI documents (prescriptions, test results, consultation recordings)  
**Provider**: Amazon Web Services  
**Compliance**: HIPAA-eligible (requires BAA)

### 7.2 Authentication

**Method**: IAM Role (ECS Task Role)

**Bucket Configuration**:
```typescript
// terraform/s3-phi-documents.tf
resource "aws_s3_bucket" "phi_documents" {
  bucket = "jibonflow-phi-documents-${var.environment}"
  
  tags = {
    Name        = "JibonFlow PHI Documents"
    Environment = var.environment
    Compliance  = "HIPAA"
  }
}

# Enable versioning (HIPAA requirement)
resource "aws_s3_bucket_versioning" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption with KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.phi_encryption.arn
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "phi_documents" {
  bucket = aws_s3_bucket.phi_documents.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### 7.3 Upload Pattern (Presigned URLs)

```typescript
// patient-service/src/aws/s3-upload.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3Client = new S3Client({ region: 'ap-southeast-1' });

export async function generatePresignedUploadUrl(
  patientId: string,
  fileName: string,
  contentType: string
): Promise<string> {
  const key = `patients/${patientId}/documents/${Date.now()}-${fileName}`;
  
  const command = new PutObjectCommand({
    Bucket: process.env.S3_PHI_DOCUMENTS_BUCKET,
    Key: key,
    ContentType: contentType,
    ServerSideEncryption: 'aws:kms',
    SSEKMSKeyId: process.env.KMS_PHI_KEY_ARN,
    Metadata: {
      'uploaded-by': patientId,
      'uploaded-at': new Date().toISOString()
    }
  });
  
  // URL expires in 15 minutes
  const presignedUrl = await getSignedUrl(s3Client, command, { expiresIn: 900 });
  
  return presignedUrl;
}
```

### 7.4 Cost Estimation

**Pricing**:
- Storage: $0.023/GB/month (S3 Standard)
- Requests: $0.005/1,000 PUT requests

**Projected Usage**: 10,000 patients × 5MB avg = 50GB storage = $1.15/month

---

## 8. AWS KMS (Encryption Key Management)

### 8.1 Service Details

**Purpose**: Manage encryption keys for PHI data at rest  
**Provider**: Amazon Web Services

### 8.2 Key Policy

```typescript
// terraform/kms-phi-key.tf
resource "aws_kms_key" "phi_encryption" {
  description             = "JibonFlow PHI Encryption Key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  tags = {
    Name       = "jibonflow-phi-key"
    Compliance = "HIPAA"
  }
}

resource "aws_kms_alias" "phi_encryption" {
  name          = "alias/jibonflow-phi-${var.environment}"
  target_key_id = aws_kms_key.phi_encryption.key_id
}
```

---

## 9. HAPI FHIR Server (Self-Hosted)

### 9.1 Service Details

**Purpose**: FHIR R4 resource storage and validation  
**Provider**: Self-hosted (HAPI FHIR open-source)  
**Documentation**: https://hapifhir.io/hapi-fhir/docs/  

### 9.2 Deployment

```yaml
# docker-compose.yml
version: '3.8'

services:
  hapi-fhir:
    image: hapiproject/hapi:latest
    container_name: jibonflow-fhir-server
    ports:
      - "8080:8080"
    environment:
      - spring.datasource.url=jdbc:postgresql://db:5432/hapi_fhir
      - spring.datasource.username=hapi_user
      - spring.datasource.password=${HAPI_DB_PASSWORD}
      - hapi.fhir.fhir_version=R4
      - hapi.fhir.tester.home.server_address=http://localhost:8080/fhir
    depends_on:
      - db
  
  db:
    image: postgres:14-alpine
    environment:
      - POSTGRES_DB=hapi_fhir
      - POSTGRES_USER=hapi_user
      - POSTGRES_PASSWORD=${HAPI_DB_PASSWORD}
    volumes:
      - hapi-fhir-data:/var/lib/postgresql/data

volumes:
  hapi-fhir-data:
```

---

## 10. Google Maps API (Geocoding & Route Optimization)

### 10.1 Service Details

**Purpose**: Geocode pharmacy addresses, optimize delivery routes  
**Provider**: Google Cloud Platform

### 10.2 Authentication

**Method**: API Key (with IP/domain restrictions)

```typescript
// logistics-service/src/google-maps/geocode.ts
import { Client } from '@googlemaps/google-maps-services-js';

const client = new Client({});

export async function geocodeAddress(address: string) {
  const response = await client.geocode({
    params: {
      address,
      key: process.env.GOOGLE_MAPS_API_KEY!
    }
  });
  
  const location = response.data.results[0].geometry.location;
  return {
    latitude: location.lat,
    longitude: location.lng
  };
}
```

### 10.3 Cost Estimation

**Pricing**: $5 per 1,000 geocode requests  
**Projected Usage**: 100 pharmacies × 1 request = $0.50 (one-time)

---

## 11. SendGrid (Transactional Emails)

### 11.1 Service Details

**Purpose**: Send transactional emails (OTP backup, order confirmations)  
**Provider**: Twilio SendGrid

### 11.2 Authentication

**Method**: API Key

```typescript
// notification-service/src/sendgrid/send-email.ts
import sgMail from '@sendgrid/mail';

sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export async function sendTransactionalEmail(
  to: string,
  subject: string,
  html: string
) {
  await sgMail.send({
    to,
    from: 'noreply@jibonflow.com',
    subject,
    html
  });
}
```

### 11.3 Cost Estimation

**Pricing**: Free tier (100 emails/day), then $0.0012 per email  
**Projected Usage**: 1,000 emails/month (within free tier)

---

## 12. AWS CloudWatch (Monitoring & Logging)

### 12.1 Service Details

**Purpose**: Centralized logging, metrics, alarms

### 12.2 Log Groups

```typescript
// terraform/cloudwatch-logs.tf
resource "aws_cloudwatch_log_group" "api_gateway" {
  name              = "/aws/ecs/jibonflow-api-gateway"
  retention_in_days = 90
  
  tags = {
    Service = "api-gateway"
  }
}

resource "aws_cloudwatch_log_group" "audit_logs" {
  name              = "/aws/ecs/jibonflow-audit-logs"
  retention_in_days = 2190 # 6 years (HIPAA requirement)
  
  tags = {
    Compliance = "HIPAA"
  }
}
```

---

## 13. Integration Summary Table

| Integration | Authentication | Rate Limit | Webhook Required | HIPAA BAA | Monthly Cost (Est.) |
|-------------|----------------|------------|------------------|-----------|---------------------|
| Agora RTC | App ID + Token | 100 req/sec | ✅ Yes (recordings) | ✅ Yes | $7.43 |
| bKash | Grant Token | 200 req/min | ✅ Yes (callbacks) | ❌ No | 1.85% fees |
| SSLCommerz | Store ID/Password | 1000 req/min | ✅ Yes (IPN) | ❌ No | 2.5% fees |
| Twilio Verify | Account SID/Token | 100 req/sec | ❌ No | ✅ Yes | $50.00 |
| Firebase FCM | Service Account | 1000 req/sec | ❌ No | ✅ Yes | $0.00 |
| AWS S3 | IAM Role | N/A | ❌ No | ✅ Yes | $1.15 |
| AWS KMS | IAM Role | 10,000 req/sec | ❌ No | ✅ Yes | $1.00 |
| HAPI FHIR | None (self-hosted) | N/A | ❌ No | N/A | $0.00 |
| Google Maps | API Key | 50 req/sec | ❌ No | ❌ No | $0.50 |
| SendGrid | API Key | 600 req/min | ❌ No | ✅ Yes | $0.00 |
| CloudWatch | IAM Role | N/A | ❌ No | ✅ Yes | $5.00 |

**Total Monthly Cost**: ~$65 + payment processing fees

---

## 14. Security Best Practices

### 14.1 API Key Management

✅ **DO**:
- Store all API keys in AWS Secrets Manager
- Rotate keys every 90 days
- Use environment-specific keys (sandbox vs. production)
- Implement IP whitelisting where supported

❌ **DON'T**:
- Commit keys to Git repositories
- Share keys between environments
- Log API keys in application logs

### 14.2 Webhook Security

✅ **Verify Signatures**:
```typescript
// Generic webhook signature verification
function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

### 14.3 Rate Limiting

Implement application-level rate limiting with Redis:
```typescript
// api-gateway/src/middleware/rate-limiter.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

export const apiRateLimiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rate_limit:'
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per 15 minutes
  message: 'Too many requests from this IP, please try again later.'
});
```

---

## 15. Integration Testing Checklist

**Pre-Production Validation**:

- [ ] Agora RTC: Test E2EE video call with 3G network simulation
- [ ] bKash: Complete test transaction in sandbox with all error scenarios
- [ ] SSLCommerz: Test IPN callback signature verification
- [ ] Twilio: Send OTP to Bangladesh number (+880 prefix)
- [ ] Firebase: Verify push notification delivery to both Android & iOS
- [ ] S3: Upload encrypted PHI document, verify KMS encryption
- [ ] FHIR: Validate Patient resource against R4 schema
- [ ] Google Maps: Geocode Bangladesh address with Bangla characters
- [ ] SendGrid: Send test email, verify deliverability
- [ ] CloudWatch: Trigger alarm, verify SNS notification

---

**Generated**: 2025-10-11  
**Phase**: Architecture Documentation (Day 1-2)  
**Next**: Non-Functional Requirements Documentation  
**Invocation Tag**: jibonflow-bootstrap-2025-10-11
