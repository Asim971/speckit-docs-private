# RefillService API - Quick Start Guide

**Latest Version**: 1.0.0  
**Status**: Production Ready (Phase 5C Complete)  
**Compliance**: ✅ HIPAA Business Associate Ready  

---

## 📚 Table of Contents

- [Installation](#installation)
- [Authentication](#authentication)
- [First API Call](#first-api-call)
- [Common Tasks](#common-tasks)
- [Error Handling](#error-handling)
- [Troubleshooting](#troubleshooting)
- [Related Documentation](#related-documentation)

---

## Installation

### Node.js / TypeScript

Install the RefillService package from npm:

```bash
npm install @jibonflow/refill-service
```

Import into your project:

```typescript
import { RefillService } from '@jibonflow/refill-service';

const refillService = new RefillService({
  apiUrl: 'https://api.jibonflow.com/api/v1',
  apiKey: process.env.JIBONFLOW_API_KEY
});
```

### Python

Install via pip:

```bash
pip install jibonflow-refill-service
```

Import into your project:

```python
from jibonflow.refill_service import RefillService

refill_service = RefillService(
    api_url='https://api.jibonflow.com/api/v1',
    api_key=os.getenv('JIBONFLOW_API_KEY')
)
```

### JavaScript (Browser)

Include via CDN:

```html
<script src="https://cdn.jibonflow.com/refill-service@1.0.0/dist/index.js"></script>

<script>
  const refillService = new JibonFlow.RefillService({
    apiUrl: 'https://api.jibonflow.com/api/v1',
    apiKey: process.env.JIBONFLOW_API_KEY
  });
</script>
```

---

## Authentication

### Setting Up Bearer Token

**Step 1: Obtain Authentication Credentials**

Contact your JibonFlow account manager to get:
- `CLIENT_ID`
- `CLIENT_SECRET`
- Initial access token

**Step 2: Configure Your Environment**

Create `.env` file in your project:

```bash
JIBONFLOW_API_URL=https://api.jibonflow.com/api/v1
JIBONFLOW_API_KEY=sk_live_abcdef1234567890xyz
JIBONFLOW_CLIENT_ID=client_id_here
JIBONFLOW_CLIENT_SECRET=client_secret_here
```

**Step 3: Use Bearer Token in Requests**

All API requests require the `Authorization` header:

```bash
curl -X GET https://api.jibonflow.com/api/v1/refills/status/refill-123 \
  -H "Authorization: Bearer sk_live_abcdef1234567890xyz" \
  -H "Content-Type: application/json"
```

### Token Refresh (If Needed)

```typescript
// Automatic token refresh in SDK
const response = await refillService.getRefillStatus('refill-123');
// SDK automatically refreshes token if expired

// Manual refresh
const newToken = await refillService.refreshToken();
process.env.JIBONFLOW_API_KEY = newToken;
```

---

## First API Call

### Simple Example: Get Refill Status

**TypeScript/JavaScript**:

```typescript
import { RefillService } from '@jibonflow/refill-service';

const refillService = new RefillService({
  apiUrl: 'https://api.jibonflow.com/api/v1',
  apiKey: process.env.JIBONFLOW_API_KEY
});

// Get status of a specific refill
const status = await refillService.getRefillStatus('refill-1635534600-abc123');

console.log('Refill Status:', status.status);
console.log('Current Quantity:', status.quantityRequested);
console.log('Last Updated:', status.updatedAt);
```

**Python**:

```python
from jibonflow.refill_service import RefillService
import os

refill_service = RefillService(
    api_url='https://api.jibonflow.com/api/v1',
    api_key=os.getenv('JIBONFLOW_API_KEY')
)

# Get status of a specific refill
status = refill_service.get_refill_status('refill-1635534600-abc123')

print(f"Refill Status: {status['status']}")
print(f"Current Quantity: {status['quantityRequested']}")
print(f"Last Updated: {status['updatedAt']}")
```

**cURL**:

```bash
curl -X GET "https://api.jibonflow.com/api/v1/refills/status/refill-1635534600-abc123" \
  -H "Authorization: Bearer sk_live_abcdef1234567890xyz" \
  -H "Content-Type: application/json"
```

**Response**:

```json
{
  "status": "approved_pending_transmission",
  "quantityRequested": 30,
  "quantityRemaining": 5,
  "lastRefillDate": "2025-09-16T10:00:00Z",
  "nextRefillDate": "2025-10-21T10:00:00Z",
  "createdAt": "2025-10-16T14:30:00Z",
  "updatedAt": "2025-10-16T15:45:00Z"
}
```

---

## Common Tasks

### Task 1: Create a Refill Request

Patient requests a prescription refill:

```typescript
const refillRequest = await refillService.createRefillRequest({
  prescriptionId: 'rx-123456',
  quantityRequested: 30,
  reason: 'Running low on medication'
}, 'patient-789');

console.log('Refill Request Created:', refillRequest.id);
console.log('Status:', refillRequest.status);  // "pending"
```

**API Details**:
- **Endpoint**: `POST /api/v1/refills`
- **Role Required**: Patient (own prescription only)
- **Response**: RefillRequest object with `id` and `status: "pending"`
- **Error Codes**: 400, 403, 404, 422

---

### Task 2: Provider Approves Refill

Provider reviews and approves a pending refill:

```typescript
const approval = await refillService.approveRefill(
  'refill-1635534600-abc123',
  {
    dosageModification: true,
    previousDose: '20mg',
    newDose: '10mg',
    reason: 'Recent lab results indicate sensitivity'
  },
  'provider-456'
);

console.log('Refill Approved:', approval.id);
console.log('New Status:', approval.status);  // "approved_pending_transmission"
console.log('Dosage Changed:', approval.dosageModified);
```

**API Details**:
- **Endpoint**: `PATCH /api/v1/refills/{id}/approve`
- **Role Required**: Provider (authorized for patient)
- **Response**: RefillRequest object with `status: "approved_pending_transmission"`
- **Error Codes**: 403, 404, 409, 422

---

### Task 3: Provider Denies Refill

Provider denies a refill request with structured reason:

```typescript
const denial = await refillService.denyRefill(
  'refill-1635534600-abc123',
  {
    reason: 'drug_interaction',
    notes: 'Potential interaction with patient\'s recent blood pressure medication'
  },
  'provider-456'
);

console.log('Refill Denied:', denial.id);
console.log('Denial Reason:', denial.denialReason);
console.log('New Status:', denial.status);  // "denied"
```

**Denial Reasons**:
- `contraindication` - Medication inappropriate for patient
- `drug_interaction` - Conflicts with current medications
- `dosage_concern` - Dosage requires provider review
- `patient_non_compliant` - Patient not following treatment plan
- `provider_request` - Provider re-evaluation needed
- `other` - Other reason (see notes)

**API Details**:
- **Endpoint**: `PATCH /api/v1/refills/{id}/deny`
- **Role Required**: Provider (authorized for patient)
- **Response**: RefillRequest object with `status: "denied"`
- **Error Codes**: 403, 404, 409

---

### Task 4: List Pending Refills (Provider)

Provider retrieves all pending refills for authorized patients:

```typescript
const pendingRefills = await refillService.getRefillRequests(
  {
    status: 'pending',
    limit: 10,
    offset: 0
  },
  'provider-456'
);

console.log('Pending Refills:', pendingRefills.data.length);
pendingRefills.data.forEach(refill => {
  console.log(`- ${refill.id}: ${refill.status} (${refill.quantityRequested} units)`);
});
```

**Filtering Options**:
- `status` - Filter by status (pending, approved, denied, etc.)
- `patientId` - Filter by patient (optional, defaults to all authorized)
- `createdAfter` - Filter by creation date (ISO 8601)
- `limit` - Records per page (default: 20, max: 100)
- `offset` - Pagination offset (default: 0)

**API Details**:
- **Endpoint**: `GET /api/v1/refills?status=pending&limit=10`
- **Role Required**: Provider (limited to authorized patients)
- **Response**: Paginated RefillRequest array
- **Error Codes**: 400, 401

---

### Task 5: Check Refill History (Patient)

Patient views their complete refill history:

```typescript
const history = await refillService.getRefillHistory(
  'patient-789',
  {
    daysBack: 365,  // Last 365 days (default)
    limit: 50,
    offset: 0
  }
);

console.log('Total Refills:', history.total);
history.data.forEach(refill => {
  console.log(`${refill.prescriptionId}: ${refill.status} on ${refill.createdAt}`);
});
```

**Filtering Options**:
- `daysBack` - Look back N days (default: 365)
- `status` - Filter by status
- `prescriptionId` - Filter by prescription
- `limit` - Records per page (default: 20)
- `offset` - Pagination offset

**API Details**:
- **Endpoint**: `GET /api/v1/refills/history?daysBack=365`
- **Role Required**: Patient (own history only)
- **Response**: Paginated RefillRequest array
- **Error Codes**: 400, 401, 403

---

## Error Handling

### Catching & Handling Errors

**TypeScript/JavaScript**:

```typescript
try {
  const refill = await refillService.approveRefill(
    'refill-123',
    { dosageModification: false },
    'provider-456'
  );
} catch (error) {
  if (error.code === 403) {
    console.error('Not authorized to approve this refill');
  } else if (error.code === 404) {
    console.error('Refill not found');
  } else if (error.code === 409) {
    console.error('Refill already processed');
  } else {
    console.error('Unexpected error:', error.message);
  }
}
```

**Python**:

```python
try:
    refill = refill_service.approve_refill(
        'refill-123',
        {'dosageModification': False},
        'provider-456'
    )
except jibonflow.Unauthorized as e:
    print(f"Not authorized: {e.message}")
except jibonflow.NotFound as e:
    print(f"Refill not found: {e.message}")
except jibonflow.Conflict as e:
    print(f"Refill already processed: {e.message}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

### Common Error Codes

| Code | Meaning | Solution |
|------|---------|----------|
| 400 | Bad Request | Check request format and required fields |
| 401 | Unauthorized | Verify API key and Bearer token are valid |
| 403 | Forbidden | User role doesn't have permission for this action |
| 404 | Not Found | Refill/prescription/patient doesn't exist |
| 409 | Conflict | Refill already processed or in conflicting state |
| 422 | Unprocessable | Request data failed validation |
| 500 | Server Error | Internal error - contact support if persists |
| 503 | Unavailable | Service temporarily down - retry in 30 seconds |

For detailed error scenarios, see `error-reference.md`.

---

## Troubleshooting

### Issue: "401 Unauthorized"

**Cause**: Invalid or expired API key

**Solution**:
1. Verify `JIBONFLOW_API_KEY` is set correctly in `.env`
2. Check token hasn't expired (tokens expire after 24 hours)
3. Refresh token: `refillService.refreshToken()`
4. Contact support if issue persists

### Issue: "403 Forbidden"

**Cause**: User role doesn't have permission

**Solution**:
1. Verify user role is appropriate (Provider can't create refills for others)
2. Check if provider is authorized for the patient
3. Ensure patient isn't trying to access another patient's data
4. See `hipaa-compliance.md` for RBAC matrix

### Issue: "404 Not Found"

**Cause**: Resource doesn't exist

**Solution**:
1. Verify IDs are spelled correctly (case-sensitive)
2. Check prescription status (may be inactive/expired)
3. Confirm refill hasn't been deleted
4. Try getting resource list to verify it exists

### Issue: "422 Unprocessable Entity"

**Cause**: Request validation failed

**Solution**:
1. Check all required fields are provided
2. Verify field types match schema (e.g., quantity is integer)
3. Validate dates are ISO 8601 format
4. See `typescript-types.md` for complete schema

### Issue: High Latency / Timeouts

**Cause**: Network or service issues

**Solution**:
1. Implement retry logic with exponential backoff
2. Check service status at `https://status.jibonflow.com`
3. Verify network connectivity to API endpoint
4. Increase timeout to 30 seconds for large requests

**Retry Example**:
```typescript
async function retryRequest<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  backoffMs = 1000
): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, backoffMs * Math.pow(2, i)));
    }
  }
  throw new Error('Max retries exceeded');
}

// Usage
const refill = await retryRequest(() => 
  refillService.getRefillStatus('refill-123')
);
```

---

## Integration Examples

### Frontend Integration (React)

```typescript
import { useState, useEffect } from 'react';
import { RefillService } from '@jibonflow/refill-service';

export function RefillStatus({ refillId }) {
  const [status, setStatus] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const refillService = new RefillService({
      apiUrl: process.env.REACT_APP_API_URL,
      apiKey: process.env.REACT_APP_API_KEY
    });

    refillService
      .getRefillStatus(refillId)
      .then(setStatus)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [refillId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h3>Refill Status: {status.status}</h3>
      <p>Quantity: {status.quantityRequested}</p>
      <p>Last Updated: {new Date(status.updatedAt).toLocaleString()}</p>
    </div>
  );
}
```

### Backend Integration (Express)

```typescript
import express from 'express';
import { RefillService } from '@jibonflow/refill-service';

const app = express();
const refillService = new RefillService({
  apiUrl: process.env.JIBONFLOW_API_URL,
  apiKey: process.env.JIBONFLOW_API_KEY
});

// List pending refills for authenticated provider
app.get('/provider/pending-refills', async (req, res) => {
  try {
    const refills = await refillService.getRefillRequests(
      { status: 'pending', limit: 20 },
      req.user.providerId
    );
    res.json(refills);
  } catch (error) {
    res.status(error.code || 500).json({ error: error.message });
  }
});

// Approve refill
app.post('/provider/approve/:refillId', async (req, res) => {
  try {
    const refill = await refillService.approveRefill(
      req.params.refillId,
      req.body.approval,
      req.user.providerId
    );
    res.json(refill);
  } catch (error) {
    res.status(error.code || 500).json({ error: error.message });
  }
});

app.listen(3000);
```

---

## Webhooks (Advanced)

Subscribe to refill status changes:

```typescript
// Register webhook
await refillService.registerWebhook({
  url: 'https://yourapp.com/webhooks/refill-status',
  events: ['refill.approved', 'refill.denied', 'refill.transmitted']
});

// Your webhook handler
app.post('/webhooks/refill-status', (req, res) => {
  const { event, refill } = req.body;
  
  switch (event) {
    case 'refill.approved':
      console.log(`Refill ${refill.id} approved`);
      // Update UI, send notification, etc.
      break;
    case 'refill.denied':
      console.log(`Refill ${refill.id} denied: ${refill.denialReason}`);
      // Notify patient, suggest alternatives
      break;
    case 'refill.transmitted':
      console.log(`Refill ${refill.id} transmitted to pharmacy`);
      // Update prescription status
      break;
  }
  
  res.json({ success: true });
});
```

---

## Related Documentation

### Complete API Reference
👉 See `api-documentation.md` for detailed endpoint documentation, all parameters, and complete examples

### OpenAPI Specification
👉 See `refill-service.openapi.json` for machine-readable OpenAPI 3.0 spec (import into Postman, Swagger UI, etc.)

### TypeScript Types
👉 See `typescript-types.md` for all interface definitions, DTOs, and type-safe development

### Error Reference
👉 See `error-reference.md` for all error codes, scenarios, and solutions

### HIPAA Compliance
👉 See `hipaa-compliance.md` for audit logging, RBAC, and PII protection details

### Integration Examples
👉 See `integration-examples.md` for 20+ code samples across curl, Python, and JavaScript

---

## Support & Contact

- **Documentation**: https://docs.jibonflow.com
- **API Status**: https://status.jibonflow.com
- **Support Email**: support@jibonflow.com
- **GitHub Issues**: https://github.com/jibonflow/refill-service/issues

---

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Status**: ✅ Production Ready (Phase 5C Complete)  
**Next**: Frontend integration for Phase 5C implementation
