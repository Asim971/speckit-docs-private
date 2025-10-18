# RefillService Error Reference

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Error Codes Documented**: 7 (400, 401, 404, 409, 422, 500, 503)  

---

## Error Reference Table

| HTTP Status | Error Code | Scenario | Root Cause | Solution |
|-------------|-----------|----------|-----------|----------|
| 400 | Bad Request | Invalid input data, expired prescription, no refills remaining, invalid dosage format | Request validation failed, business rule violated | Check field validation, verify prescription status |
| 401 | Unauthorized | Missing or invalid authentication token | No Bearer token or token expired/malformed | Add Bearer token to Authorization header |
| 404 | Not Found | Refill/prescription/patient doesn't exist | Resource not found in database | Verify refill ID, prescription ID, or patient ID |
| 409 | Conflict | Duplicate refill request detected | Refill already exists for same prescription | Check for existing requests before retrying |
| 422 | Unprocessable | Business logic violation, provider not authorized | Policy constraint, RBAC failure | Review policy requirements, verify provider authorization |
| 500 | Server Error | Unexpected server error | Database error, service failure | Contact support, check logs |
| 503 | Service Unavailable | Pharmacy service temporarily down | Downstream service failure | Retry after a delay |

---

## Detailed Error Scenarios

### 400 - Bad Request

**General Description**: The request contains invalid data or violates a business rule.

#### Scenario 1: Validation Error - Invalid Prescription ID

**When**: POST `/api/v1/refills`  
**Root Cause**: `prescriptionId` field is empty or invalid  
**Response**:
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Prescription ID is required",
    "details": {
      "field": "prescriptionId",
      "reason": "Field is required"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify prescription ID is not empty
2. Ensure prescription ID follows correct format
3. Check that prescription ID exists in system

**Example Fix**:
```javascript
// ❌ WRONG
const request = {
  prescriptionId: "",  // Empty string
  patientId: "patient-789"
};

// ✅ CORRECT
const request = {
  prescriptionId: "rx-123456",
  patientId: "patient-789"
};
```

---

#### Scenario 2: Business Rule - Prescription Expired

**When**: POST `/api/v1/refills`  
**Root Cause**: Prescription has passed expiration date  
**Response**:
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Prescription has expired",
    "details": {
      "field": "prescriptionId",
      "reason": "Expiration date: 2025-09-15",
      "expirationDate": "2025-09-15T23:59:59Z"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify prescription expiration date
2. Contact provider for prescription renewal
3. Request new prescription before attempting refill

**Example Check**:
```javascript
async function createRefillSafely(prescriptionId) {
  // First, check prescription details
  const rx = await getPrescriptionDetails(prescriptionId);
  
  if (new Date(rx.expirationDate) < new Date()) {
    console.error("Prescription expired. Contact provider for renewal.");
    return null;
  }
  
  // Proceed with refill
  return await createRefillRequest({ prescriptionId });
}
```

---

#### Scenario 3: Business Rule - No Refills Remaining

**When**: POST `/api/v1/refills`  
**Root Cause**: Prescription has no refills remaining  
**Response**:
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "No refills remaining for this prescription",
    "details": {
      "field": "prescriptionId",
      "maxRefillsOriginal": 3,
      "maxRefillsRemaining": 0
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Contact healthcare provider for new prescription or refill authorization
2. Wait for provider to authorize additional refills
3. Request prescription modification

---

#### Scenario 4: Validation Error - Invalid Dosage Format

**When**: PATCH `/api/v1/refills/{id}/approve`  
**Root Cause**: `dosageChange` object missing required fields  
**Response**:
```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Invalid dosage change format",
    "details": {
      "field": "dosageChange",
      "reason": "Missing required fields: doseQuantity and doseUnit"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Ensure `doseQuantity` is provided as a number
2. Ensure `doseUnit` is provided (e.g., "mg", "mcg")
3. Verify format matches expected schema

**Example Fix**:
```javascript
// ❌ WRONG - Missing doseUnit
const approval = {
  status: 'approved',
  dosageChange: {
    doseQuantity: 10
    // doseUnit missing!
  }
};

// ✅ CORRECT
const approval = {
  status: 'approved',
  dosageChange: {
    doseQuantity: 10,
    doseUnit: 'mg'
  }
};
```

---

### 401 - Unauthorized

**General Description**: Authentication failed. Token is missing, invalid, or expired.

#### Scenario 1: Missing Authorization Header

**When**: Any API call  
**Root Cause**: Authorization header not provided  
**Response**:
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Missing authorization header",
    "details": {
      "reason": "Authorization header not found in request"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Add `Authorization` header with Bearer token
2. Ensure header format is exactly: `Authorization: Bearer {token}`
3. Verify token is present and not empty

**Example Fix**:
```javascript
// ❌ WRONG - No Authorization header
const response = await fetch("http://localhost:3001/api/v1/refills", {
  method: "GET"
});

// ✅ CORRECT
const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
const response = await fetch("http://localhost:3001/api/v1/refills", {
  method: "GET",
  headers: {
    "Authorization": `Bearer ${token}`
  }
});
```

---

#### Scenario 2: Invalid Token Format

**When**: Any API call  
**Root Cause**: Token doesn't match Bearer format or is malformed  
**Response**:
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid token format",
    "details": {
      "reason": "Token must be a valid JWT"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify Bearer token is valid JWT
2. Check token hasn't been corrupted in transmission
3. Re-authenticate to get fresh token

---

#### Scenario 3: Token Expired

**When**: Any API call  
**Root Cause**: JWT token has exceeded expiration time  
**Response**:
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Token has expired",
    "details": {
      "reason": "Token expired at 2025-10-16T17:00:00Z",
      "expiredAt": "2025-10-16T17:00:00Z"
    },
    "timestamp": "2025-10-16T17:15:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Implement token refresh mechanism
2. Re-authenticate user
3. Store tokens with expiration tracking

**Example Refresh Logic**:
```javascript
async function makeAuthenticatedRequest(endpoint) {
  let token = getStoredToken();
  
  // Check if token expired
  if (isTokenExpired(token)) {
    console.log("Token expired, refreshing...");
    token = await refreshToken();
    storeToken(token);
  }
  
  const response = await fetch(endpoint, {
    headers: {
      "Authorization": `Bearer ${token}`
    }
  });
  
  return response;
}
```

---

### 404 - Not Found

**General Description**: Requested resource doesn't exist in the system.

#### Scenario 1: Refill Request Not Found

**When**: GET/PATCH `/api/v1/refills/{id}`  
**Root Cause**: Refill ID doesn't exist  
**Response**:
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Refill request not found",
    "details": {
      "resourceType": "Refill",
      "resourceId": "refill-invalid-123"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify refill ID is correct and complete
2. Confirm refill exists in system
3. Check for typos in ID

**Example Verification**:
```javascript
// First verify refill exists
async function safeGetRefillStatus(refillId) {
  try {
    const response = await fetch(`http://localhost:3001/api/v1/refills/${refillId}`, {
      headers: { "Authorization": `Bearer ${token}` }
    });
    
    if (response.status === 404) {
      console.error(`Refill ${refillId} not found`);
      // Maybe list available refills to help user
      await listPendingRefills();
      return null;
    }
    
    return await response.json();
  } catch (error) {
    console.error("Error retrieving refill:", error);
  }
}
```

---

#### Scenario 2: Prescription Not Found

**When**: POST `/api/v1/refills`  
**Root Cause**: Prescription ID references non-existent prescription  
**Response**:
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Prescription not found",
    "details": {
      "resourceType": "Prescription",
      "resourceId": "rx-invalid-456"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify prescription ID is correct
2. Confirm prescription has been created in system
3. Check patient's active prescriptions

---

#### Scenario 3: Patient Not Found

**When**: GET `/api/v1/refills/patient/{id}/history`  
**Root Cause**: Patient ID doesn't exist  
**Response**:
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Patient not found",
    "details": {
      "resourceType": "Patient",
      "resourceId": "patient-invalid"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify patient ID matches authentication context
2. Confirm patient exists in system

---

### 409 - Conflict

**General Description**: Resource conflict - typically duplicate request.

#### Scenario: Duplicate Refill Request

**When**: POST `/api/v1/refills`  
**Root Cause**: Patient already has pending refill for same prescription  
**Response**:
```json
{
  "error": {
    "code": "CONFLICT",
    "message": "Duplicate refill request detected",
    "details": {
      "reason": "Pending refill already exists for this prescription",
      "existingRefillId": "refill-1635534600-abc123",
      "existingRefillStatus": "pending"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Check for existing pending refills before creating new one
2. Use existing refill ID instead
3. Wait for provider decision on pending refill
4. Contact support if duplicate occurred in error

**Example Prevention**:
```javascript
async function createRefillWithDuplicateCheck(prescriptionId) {
  // List existing refills for prescription
  const existing = await listRefillsByPrescription(prescriptionId);
  
  const pending = existing.find(r => r.status === 'pending');
  
  if (pending) {
    console.warn(`Pending refill already exists: ${pending.id}`);
    console.log(`Check status at: /api/v1/refills/${pending.id}`);
    return pending;
  }
  
  // No pending refill, safe to create new one
  return await createRefillRequest({ prescriptionId });
}
```

---

### 422 - Unprocessable Entity

**General Description**: Business logic violation or policy constraint failed.

#### Scenario 1: Provider Not Authorized for Patient

**When**: PATCH `/api/v1/refills/{id}/approve` or `/deny`  
**Root Cause**: Provider has no authorized relationship with patient  
**Response**:
```json
{
  "error": {
    "code": "UNPROCESSABLE",
    "message": "Provider not authorized for this patient",
    "details": {
      "providerId": "provider-456",
      "patientId": "patient-789",
      "reason": "No provider-patient relationship found"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Verify provider is listed as patient's healthcare provider
2. Add provider-patient relationship through patient's EHR
3. Patient needs to add provider to their care team
4. Use different provider with authorization

**Example Check**:
```javascript
async function checkProviderAuthorization(providerId, patientId) {
  // Check provider has access to patient
  const auth = await getProviderPatientRelationship(providerId, patientId);
  
  if (!auth) {
    console.error(
      `Provider ${providerId} is not authorized for patient ${patientId}. ` +
      `Patient must add provider to their care team.`
    );
    return false;
  }
  
  return true;
}
```

---

#### Scenario 2: Refill Already Processed

**When**: PATCH `/api/v1/refills/{id}/approve` or `/deny`  
**Root Cause**: Refill no longer in 'pending' status  
**Response**:
```json
{
  "error": {
    "code": "UNPROCESSABLE",
    "message": "Refill request is already approved",
    "details": {
      "refillId": "refill-123",
      "currentStatus": "approved",
      "reason": "Cannot process refill that's already been processed"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Check refill status before attempting action
2. Don't attempt to approve/deny already-processed refill
3. If reversal needed, contact support

**Example Check**:
```javascript
async function safeApproveRefill(refillId) {
  const refill = await getRefillStatus(refillId);
  
  if (refill.status !== 'pending') {
    console.error(`Cannot approve refill in ${refill.status} status`);
    console.log(`Current status: ${refill.statusDetail}`);
    return null;
  }
  
  return await approveRefill(refillId);
}
```

---

#### Scenario 3: Policy Violation

**When**: PATCH `/api/v1/refills/{id}/approve`  
**Root Cause**: Dosage modification violates clinical policy  
**Response**:
```json
{
  "error": {
    "code": "UNPROCESSABLE",
    "message": "Dosage change violates clinical policy",
    "details": {
      "field": "dosageChange.doseQuantity",
      "proposedValue": 500,
      "maximumAllowed": 100,
      "reason": "Dosage exceeds maximum safe dose"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Review clinical guidelines for medication
2. Use dosage within approved range
3. Contact clinical team if different dosage medically necessary
4. Document clinical justification

---

### 500 - Server Error

**General Description**: Unexpected server-side error. Typically not caused by client request.

#### Response Example:
```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "An unexpected error occurred",
    "details": {
      "reason": "Database connection failed"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Wait a moment and retry request
2. Check status page for service incidents
3. Contact support if error persists
4. Include requestId in support ticket

---

### 503 - Service Unavailable

**General Description**: Pharmacy service or critical dependency temporarily unavailable.

#### Response Example:
```json
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Pharmacy service temporarily unavailable",
    "details": {
      "service": "pharmacy-integration",
      "retryAfter": 30
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Solutions**:
1. Retry after indicated delay (30 seconds in example)
2. Check Pharmacy Partner status page
3. Contact support if service unavailable for extended period
4. Implement exponential backoff retry strategy

**Example Retry Logic**:
```javascript
async function transmitWithRetry(refillId, maxRetries = 3) {
  let attempt = 0;
  
  while (attempt < maxRetries) {
    try {
      const response = await fetch(
        `http://localhost:3001/api/v1/refills/${refillId}/transmit`,
        {
          method: 'POST',
          headers: { 'Authorization': `Bearer ${token}` }
        }
      );
      
      if (response.status === 503) {
        attempt++;
        const retryAfter = 30 * Math.pow(2, attempt); // Exponential backoff
        console.log(`Service unavailable. Retrying in ${retryAfter}s...`);
        await sleep(retryAfter * 1000);
        continue;
      }
      
      if (response.status === 200) {
        return await response.json();
      }
      
      throw new Error(`Unexpected status: ${response.status}`);
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;
      attempt++;
    }
  }
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

## Error Response Format

All errors follow this standardized format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "User-friendly error message",
    "details": {
      "field": "field_name",
      "reason": "Specific reason or context"
    },
    "timestamp": "2025-10-16T16:00:00Z",
    "requestId": "req-12345-abcde"
  }
}
```

**Fields**:
- `code`: Machine-readable error identifier
- `message`: Human-readable description
- `details`: Additional context (optional)
- `timestamp`: ISO 8601 timestamp of error
- `requestId`: Unique request identifier for tracking

---

## HTTP Status Code Summary

```
2xx Success
  200 OK - Request succeeded
  201 Created - Resource created successfully

4xx Client Error
  400 Bad Request - Invalid input or business rule violated
  401 Unauthorized - Authentication failed
  403 Forbidden - Permission denied (RBAC)
  404 Not Found - Resource doesn't exist
  409 Conflict - Resource conflict
  422 Unprocessable Entity - Business logic violation

5xx Server Error
  500 Internal Server Error - Unexpected server error
  503 Service Unavailable - Downstream service failure
```

---

## Debugging Checklist

When troubleshooting errors:

- [ ] Check HTTP status code to identify error category
- [ ] Read error message and details
- [ ] Verify request format and required fields
- [ ] Confirm authentication token is valid and not expired
- [ ] Verify user has required permissions (role, RBAC)
- [ ] Check resource IDs (refill, prescription, patient)
- [ ] Review request body for typos or invalid data
- [ ] Check API documentation for endpoint specifics
- [ ] Save requestId for support inquiry
- [ ] Check service status page for incidents

---

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Error Codes Documented**: 7 (all HTTP 4xx/5xx responses)  
**Status**: ✅ Production Ready
