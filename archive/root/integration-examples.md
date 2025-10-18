# RefillService Integration Examples

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Example Count**: 21+ (7 curl + 7 Python + 7 JavaScript + 3 workflows)  

---

## Table of Contents

1. [Curl Examples](#curl-examples)
2. [Python Examples](#python-examples)
3. [JavaScript/TypeScript Examples](#javascripttypescript-examples)
4. [Workflow Integration](#workflow-integration)
5. [Error Handling](#error-handling)
6. [Performance Considerations](#performance-considerations)

---

## Curl Examples

### Example 1: Create Refill Request

```bash
curl -X POST http://localhost:3001/api/v1/refills \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwYXRpZW50LTc4OSIsInJvbGUiOiJwYXRpZW50IiwicGF0aWVudElkIjoicGF0aWVudC03ODkiLCJpYXQiOjE2MzU1MzQ2MDAsImV4cCI6MTYzNTUzODIwMH0.signature" \
  -d '{
    "prescriptionId": "rx-123456",
    "patientId": "patient-789",
    "quantity": 30,
    "reason": "Running low on medication"
  }'

# Expected Response (201 Created):
# {
#   "id": "refill-1635534600-abc123",
#   "prescriptionId": "rx-123456",
#   "patientId": "patient-789",
#   "status": "pending",
#   "quantityRequested": 30,
#   "maxRefillsRemaining": 5,
#   "reasonForRequest": "Running low on medication",
#   "requestedAt": "2025-10-16T14:30:00Z",
#   "createdAt": "2025-10-16T14:30:00Z"
# }
```

### Example 2: List Pending Refills (Provider)

```bash
curl -X GET "http://localhost:3001/api/v1/refills?status=pending&limit=20&offset=0" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwcm92aWRlci00NTYiLCJyb2xlIjoicHJvdmlkZXIiLCJwcm92aWRlcklkIjoicHJvdmlkZXItNDU2IiwiaWF0IjoxNjM1NTM0NjAwLCJleHAiOjE2MzU1MzgyMDB9.signature"

# Expected Response (200 OK):
# {
#   "data": [
#     {
#       "id": "refill-1635534600-abc123",
#       "prescriptionId": "rx-123456",
#       "patientId": "patient-789",
#       "status": "pending",
#       "quantityRequested": 30,
#       "reasonForRequest": "Running low on medication",
#       "requestedAt": "2025-10-16T14:30:00Z"
#     }
#   ],
#   "pagination": {
#     "count": 1,
#     "hasMore": false,
#     "limit": 20,
#     "offset": 0
#   }
# }
```

### Example 3: Get Refill Status

```bash
curl -X GET http://localhost:3001/api/v1/refills/refill-1635534600-abc123 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwYXRpZW50LTc4OSIsInJvbGUiOiJwYXRpZW50IiwicGF0aWVudElkIjoicGF0aWVudC03ODkiLCJpYXQiOjE2MzU1MzQ2MDAsImV4cCI6MTYzNTUzODIwMH0.signature"

# Expected Response (200 OK):
# {
#   "id": "refill-1635534600-abc123",
#   "prescriptionId": "rx-123456",
#   "patientId": "patient-789",
#   "status": "approved_transmission_sent",
#   "statusDetail": "Sent to pharmacy (pending acknowledgment)",
#   "transmissionStatus": "sent",
#   "lastUpdated": "2025-10-16T15:45:00Z"
# }
```

### Example 4: Approve Refill Request

```bash
curl -X PATCH http://localhost:3001/api/v1/refills/refill-1635534600-abc123/approve \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwcm92aWRlci00NTYiLCJyb2xlIjoicHJvdmlkZXIiLCJwcm92aWRlcklkIjoicHJvdmlkZXItNDU2IiwiaWF0IjoxNjM1NTM0NjAwLCJleHAiOjE2MzU1MzgyMDB9.signature" \
  -d '{
    "status": "approved",
    "dosageChange": {
      "doseQuantity": 10,
      "doseUnit": "mg"
    },
    "notes": "Approved. Reduced dose due to recent lab results."
  }'

# Expected Response (200 OK):
# {
#   "id": "refill-1635534600-abc123",
#   "prescriptionId": "rx-123456",
#   "patientId": "patient-789",
#   "status": "approved_pending_transmission",
#   "quantityApproved": 30,
#   "dosageChange": {
#     "doseQuantity": 10,
#     "doseUnit": "mg"
#   },
#   "approvedById": "provider-456",
#   "approvedAt": "2025-10-16T15:45:00Z"
# }
```

### Example 5: Deny Refill Request

```bash
curl -X PATCH http://localhost:3001/api/v1/refills/refill-1635534600-abc123/deny \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwcm92aWRlci00NTYiLCJyb2xlIjoicHJvdmlkZXIiLCJwcm9=""IZciOiJwcm92aWRlci00NTYiLCJpYXQiOjE2MzU1MzQ2MDAsImV4cCI6MTYzNTUzODIwMH0.signature" \
  -d '{
    "status": "denied",
    "reason": "drug_interaction",
    "notes": "Interacts with current blood pressure medication (lisinopril)",
    "recommendation": "Switch to alternative antihypertensive. Consult cardiologist."
  }'

# Expected Response (200 OK):
# {
#   "id": "refill-1635534600-abc123",
#   "prescriptionId": "rx-123456",
#   "patientId": "patient-789",
#   "status": "denied",
#   "denialReason": "drug_interaction",
#   "denialNotes": "Interacts with current blood pressure medication (lisinopril)",
#   "recommendation": "Switch to alternative antihypertensive. Consult cardiologist.",
#   "deniedById": "provider-456",
#   "deniedAt": "2025-10-16T16:00:00Z"
# }
```

### Example 6: Get Patient History

```bash
curl -X GET "http://localhost:3001/api/v1/refills/patient/patient-789/history?limit=50&offset=0" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJwYXRpZW50LTc4OSIsInJvbGUiOiJwYXRpZW50IiwicGF0aWVudElkIjoicGF0aWVudC03ODkiLCJpYXQiOjE2MzU1MzQ2MDAsImV4cCI6MTYzNTUzODIwMH0.signature"

# Expected Response (200 OK):
# {
#   "data": [
#     {
#       "id": "refill-1635534600-abc123",
#       "prescriptionId": "rx-123456",
#       "status": "fulfilled",
#       "requestedAt": "2025-10-16T14:30:00Z",
#       "approvedAt": "2025-10-16T15:45:00Z",
#       "filledAt": "2025-10-17T09:30:00Z"
#     }
#   ],
#   "pagination": {
#     "count": 1,
#     "hasMore": false,
#     "limit": 50,
#     "offset": 0
#   }
# }
```

### Example 7: Transmit Refill to Pharmacy

```bash
curl -X POST http://localhost:3001/api/v1/refills/refill-1635534600-abc123/transmit \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzeXN0ZW0iLCJyb2xlIjoic3lzdGVtIiwiaWF0IjoxNjM1NTM0NjAwLCJleHAiOjE2MzU1MzgyMDB9.signature"

# Expected Response (200 OK):
# {
#   "id": "refill-1635534600-abc123",
#   "status": "approved_transmission_sent",
#   "transmissionStatus": "sent",
#   "transmissionSentAt": "2025-10-16T16:00:00Z",
#   "message": "Refill successfully transmitted to pharmacy"
# }
```

---

## Python Examples

### Example 1: Create Refill Request

```python
import aiohttp
import json
from datetime import datetime

async def create_refill_request():
    """
    Patient-initiated refill request creation.
    Returns newly created refill with 'pending' status.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    payload = {
        "prescriptionId": "rx-123456",
        "patientId": "patient-789",
        "quantity": 30,
        "reason": "Running low on medication"
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.post(
            "http://localhost:3001/api/v1/refills",
            headers=headers,
            json=payload
        ) as response:
            if response.status == 201:
                result = await response.json()
                print(f"✅ Refill created: {result['id']}")
                print(f"   Status: {result['status']}")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
                raise Exception(f"Failed to create refill: {error}")
```

### Example 2: List Pending Refills

```python
import aiohttp

async def list_pending_refills(provider_id: str, limit: int = 20):
    """
    Provider lists all pending refill requests.
    Results filtered to authorized patients only.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Authorization": f"Bearer {token}"
    }
    
    params = {
        "status": "pending",
        "limit": limit,
        "offset": 0
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.get(
            "http://localhost:3001/api/v1/refills",
            headers=headers,
            params=params
        ) as response:
            if response.status == 200:
                result = await response.json()
                print(f"✅ Found {len(result['data'])} pending refills")
                for refill in result['data']:
                    print(f"   - {refill['id']}: {refill['reasonForRequest']}")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
```

### Example 3: Get Refill Status

```python
async def get_refill_status(refill_id: str):
    """
    Check current status of a refill request.
    Returns status with human-readable detail.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Authorization": f"Bearer {token}"
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"http://localhost:3001/api/v1/refills/{refill_id}",
            headers=headers
        ) as response:
            if response.status == 200:
                result = await response.json()
                print(f"✅ Status: {result['statusDetail']}")
                print(f"   Status: {result['status']}")
                if result.get('transmissionStatus'):
                    print(f"   Transmission: {result['transmissionStatus']}")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
```

### Example 4: Approve Refill with Dosage Change

```python
async def approve_refill(refill_id: str, new_dose: float, dose_unit: str):
    """
    Provider approves refill with optional dosage modification.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    payload = {
        "status": "approved",
        "dosageChange": {
            "doseQuantity": new_dose,
            "doseUnit": dose_unit
        },
        "notes": f"Approved with dosage adjustment to {new_dose}{dose_unit}"
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.patch(
            f"http://localhost:3001/api/v1/refills/{refill_id}/approve",
            headers=headers,
            json=payload
        ) as response:
            if response.status == 200:
                result = await response.json()
                print(f"✅ Refill approved: {result['id']}")
                if result.get('dosageChange'):
                    print(f"   Dosage: {result['dosageChange']['doseQuantity']}{result['dosageChange']['doseUnit']}")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
```

### Example 5: Deny Refill with Reason

```python
async def deny_refill(refill_id: str, reason: str, recommendation: str):
    """
    Provider denies refill with structured reason and recommendation.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}"
    }
    
    payload = {
        "status": "denied",
        "reason": reason,  # contraindication, drug_interaction, dosage_concern, etc.
        "notes": f"Denied due to {reason}",
        "recommendation": recommendation
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.patch(
            f"http://localhost:3001/api/v1/refills/{refill_id}/deny",
            headers=headers,
            json=payload
        ) as response:
            if response.status == 200:
                result = await response.json()
                print(f"✅ Refill denied: {result['id']}")
                print(f"   Reason: {result['denialReason']}")
                print(f"   Recommendation: {result.get('recommendation')}")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
```

### Example 6: Get Patient History

```python
async def get_patient_history(patient_id: str, days_back: int = 365):
    """
    Retrieve patient's refill history.
    Patients can only access their own history.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Authorization": f"Bearer {token}"
    }
    
    params = {
        "limit": 50,
        "offset": 0
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.get(
            f"http://localhost:3001/api/v1/refills/patient/{patient_id}/history",
            headers=headers,
            params=params
        ) as response:
            if response.status == 200:
                result = await response.json()
                print(f"✅ History: {len(result['data'])} refills")
                for entry in result['data']:
                    print(f"   - {entry['id']}: {entry['status']} ({entry['requestedAt']})")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
```

### Example 7: Transmit to Pharmacy

```python
async def transmit_refill(refill_id: str):
    """
    System endpoint to transmit approved refill to pharmacy.
    """
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    headers = {
        "Authorization": f"Bearer {token}"
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.post(
            f"http://localhost:3001/api/v1/refills/{refill_id}/transmit",
            headers=headers
        ) as response:
            if response.status == 200:
                result = await response.json()
                print(f"✅ Transmitted: {result['transmissionStatus']}")
                print(f"   Sent at: {result['transmissionSentAt']}")
                return result
            else:
                error = await response.text()
                print(f"❌ Error: {response.status} - {error}")
```

---

## JavaScript/TypeScript Examples

### Example 1: Create Refill Request

```javascript
async function createRefillRequest() {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const response = await fetch("http://localhost:3001/api/v1/refills", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${token}`
    },
    body: JSON.stringify({
      prescriptionId: "rx-123456",
      patientId: "patient-789",
      quantity: 30,
      reason: "Running low on medication"
    })
  });

  if (response.status === 201) {
    const refill = await response.json();
    console.log(`✅ Refill created: ${refill.id}`);
    console.log(`   Status: ${refill.status}`);
    return refill;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
    throw new Error(`Failed to create refill: ${error}`);
  }
}
```

### Example 2: List Pending Refills

```javascript
async function listPendingRefills(limit = 20) {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const params = new URLSearchParams({
    status: "pending",
    limit: limit
  });
  
  const response = await fetch(
    `http://localhost:3001/api/v1/refills?${params}`,
    {
      headers: {
        "Authorization": `Bearer ${token}`
      }
    }
  );

  if (response.status === 200) {
    const data = await response.json();
    console.log(`✅ Found ${data.data.length} pending refills`);
    data.data.forEach(refill => {
      console.log(`   - ${refill.id}: ${refill.reasonForRequest}`);
    });
    return data;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
  }
}
```

### Example 3: Get Refill Status

```javascript
async function getRefillStatus(refillId) {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const response = await fetch(
    `http://localhost:3001/api/v1/refills/${refillId}`,
    {
      headers: {
        "Authorization": `Bearer ${token}`
      }
    }
  );

  if (response.status === 200) {
    const refill = await response.json();
    console.log(`✅ Status: ${refill.statusDetail}`);
    console.log(`   Status: ${refill.status}`);
    if (refill.transmissionStatus) {
      console.log(`   Transmission: ${refill.transmissionStatus}`);
    }
    return refill;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
  }
}
```

### Example 4: Approve Refill with TypeScript

```typescript
import type { ApprovalDTO, RefillRequest } from '@jibonflow/refill-service';

async function approveRefill(
  refillId: string,
  newDose: number,
  doseUnit: string
): Promise<RefillRequest> {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const approval: ApprovalDTO = {
    status: 'approved',
    dosageChange: {
      doseQuantity: newDose,
      doseUnit: doseUnit
    },
    notes: `Approved with dosage adjustment to ${newDose}${doseUnit}`
  };
  
  const response = await fetch(
    `http://localhost:3001/api/v1/refills/${refillId}/approve`,
    {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}`
      },
      body: JSON.stringify(approval)
    }
  );

  if (response.status === 200) {
    const refill = await response.json() as RefillRequest;
    console.log(`✅ Refill approved: ${refill.id}`);
    if (refill.dosageChange) {
      console.log(`   Dosage: ${refill.dosageChange.doseQuantity}${refill.dosageChange.doseUnit}`);
    }
    return refill;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
    throw new Error(`Failed to approve refill: ${error}`);
  }
}
```

### Example 5: Deny Refill

```javascript
async function denyRefill(refillId, reason, recommendation) {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const response = await fetch(
    `http://localhost:3001/api/v1/refills/${refillId}/deny`,
    {
      method: "PATCH",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}`
      },
      body: JSON.stringify({
        status: "denied",
        reason: reason,
        notes: `Denied due to ${reason}`,
        recommendation: recommendation
      })
    }
  );

  if (response.status === 200) {
    const refill = await response.json();
    console.log(`✅ Refill denied: ${refill.id}`);
    console.log(`   Reason: ${refill.denialReason}`);
    console.log(`   Recommendation: ${refill.recommendation}`);
    return refill;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
  }
}
```

### Example 6: Get Patient History with Pagination

```javascript
async function getPatientHistory(patientId, limit = 50, offset = 0) {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const params = new URLSearchParams({ limit, offset });
  
  const response = await fetch(
    `http://localhost:3001/api/v1/refills/patient/${patientId}/history?${params}`,
    {
      headers: {
        "Authorization": `Bearer ${token}`
      }
    }
  );

  if (response.status === 200) {
    const data = await response.json();
    console.log(`✅ History: ${data.data.length} refills (${data.pagination.count} total)`);
    data.data.forEach(entry => {
      console.log(`   - ${entry.id}: ${entry.status} (${new Date(entry.requestedAt).toLocaleDateString()})`);
    });
    return data;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
  }
}
```

### Example 7: Transmit Refill to Pharmacy

```javascript
async function transmitRefill(refillId) {
  const token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";
  
  const response = await fetch(
    `http://localhost:3001/api/v1/refills/${refillId}/transmit`,
    {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${token}`
      }
    }
  );

  if (response.status === 200) {
    const result = await response.json();
    console.log(`✅ Transmitted: ${result.transmissionStatus}`);
    console.log(`   Sent at: ${result.transmissionSentAt}`);
    return result;
  } else {
    const error = await response.text();
    console.error(`❌ Error: ${response.status} - ${error}`);
  }
}
```

---

## Workflow Integration

### Workflow 1: Complete Happy Path (Create → Approve → Transmit)

**Scenario**: Patient requests refill, provider approves, system transmits to pharmacy

```javascript
/**
 * Complete refill workflow demonstration
 */
async function completeRefillWorkflow() {
  try {
    // Step 1: Patient creates refill request
    console.log("\n📝 Step 1: Creating refill request...");
    const refill = await createRefillRequest();
    const refillId = refill.id;
    console.log(`✅ Created: ${refillId} (Status: ${refill.status})`);

    // Step 2: Provider reviews pending refills
    console.log("\n🔍 Step 2: Provider reviewing pending refills...");
    const pending = await listPendingRefills();
    console.log(`✅ Found: ${pending.data.length} pending refills`);

    // Step 3: Provider approves refill
    console.log("\n✔️ Step 3: Provider approving refill...");
    const approved = await approveRefill(refillId, 10, "mg");
    console.log(`✅ Approved: ${approved.id} (Status: ${approved.status})`);

    // Step 4: System transmits to pharmacy
    console.log("\n📤 Step 4: System transmitting to pharmacy...");
    const transmitted = await transmitRefill(refillId);
    console.log(`✅ Transmitted: ${refillId}`);

    // Step 5: Patient checks final status
    console.log("\n📊 Step 5: Patient checking final status...");
    const finalStatus = await getRefillStatus(refillId);
    console.log(`✅ Final Status: ${finalStatus.statusDetail}`);

    console.log("\n✅ Workflow completed successfully!");
    return { refill, approved, transmitted, finalStatus };
  } catch (error) {
    console.error("❌ Workflow failed:", error.message);
    throw error;
  }
}
```

### Workflow 2: Denied Refill Workflow

**Scenario**: Provider reviews and denies refill with recommendation

```javascript
async function deniedRefillWorkflow() {
  try {
    console.log("\n📝 Step 1: Patient creates refill request...");
    const refill = await createRefillRequest();
    const refillId = refill.id;

    console.log("\n🔍 Step 2: Provider reviewing refill...");
    const review = await getRefillStatus(refillId);
    console.log(`Review: ${review.statusDetail}`);

    console.log("\n❌ Step 3: Provider denying refill...");
    const denied = await denyRefill(
      refillId,
      "drug_interaction",
      "Switch to alternative antihypertensive. Consult cardiologist."
    );
    console.log(`✅ Denied with reason: ${denied.denialReason}`);

    console.log("\n📧 Step 4: Patient notified of denial...");
    console.log(`   Recommendation: ${denied.recommendation}`);

    console.log("\n✅ Denial workflow completed!");
    return { refill, denied };
  } catch (error) {
    console.error("❌ Workflow failed:", error.message);
    throw error;
  }
}
```

### Workflow 3: Patient History Pagination

**Scenario**: Patient views refill history with pagination

```javascript
async function patientHistoryWorkflow() {
  try {
    const patientId = "patient-789";
    let offset = 0;
    const pageSize = 10;
    let allResults = [];

    console.log("\n📜 Retrieving complete refill history...\n");

    while (true) {
      console.log(`Fetching page at offset ${offset}...`);
      const result = await getPatientHistory(patientId, pageSize, offset);

      allResults = [...allResults, ...result.data];

      if (!result.pagination.hasMore) {
        break;
      }

      offset += pageSize;
    }

    console.log(`\n✅ Retrieved ${allResults.length} total refills`);
    console.log("Status breakdown:");
    const statusCounts = {};
    allResults.forEach(r => {
      statusCounts[r.status] = (statusCounts[r.status] || 0) + 1;
    });
    Object.entries(statusCounts).forEach(([status, count]) => {
      console.log(`   ${status}: ${count}`);
    });

    return allResults;
  } catch (error) {
    console.error("❌ History retrieval failed:", error.message);
    throw error;
  }
}
```

---

## Error Handling

### Example: Comprehensive Error Handling

```javascript
async function robustRefillOperation(refillId) {
  try {
    // Attempt to approve refill
    const approval = {
      status: 'approved',
      dosageChange: {
        doseQuantity: 10,
        doseUnit: 'mg'
      }
    };

    const response = await fetch(
      `http://localhost:3001/api/v1/refills/${refillId}/approve`,
      {
        method: "PATCH",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${token}`
        },
        body: JSON.stringify(approval)
      }
    );

    // Handle specific error codes
    switch (response.status) {
      case 200:
        console.log("✅ Success");
        return await response.json();

      case 400:
        const badRequest = await response.json();
        console.error(`❌ Bad Request: ${badRequest.error.message}`);
        if (badRequest.error.details?.field === 'dosageChange') {
          console.log("   → Invalid dosage format. Check dose quantity and unit.");
        }
        throw new Error(`Validation error: ${badRequest.error.message}`);

      case 401:
        console.error("❌ Unauthorized - Token expired or invalid");
        // Trigger token refresh or re-authentication
        throw new Error("Authentication failed. Please log in again.");

      case 403:
        console.error("❌ Forbidden - Insufficient permissions");
        throw new Error("You don't have permission to approve this refill.");

      case 404:
        console.error("❌ Not Found - Refill doesn't exist");
        throw new Error(`Refill ${refillId} not found.`);

      case 422:
        const unprocessable = await response.json();
        console.error(`❌ Business Logic Violation: ${unprocessable.error.message}`);
        if (unprocessable.error.message.includes("not authorized")) {
          console.log("   → Provider not authorized for this patient.");
        }
        throw new Error(`Policy violation: ${unprocessable.error.message}`);

      case 503:
        console.error("❌ Service Unavailable - Try again later");
        throw new Error("Pharmacy service temporarily unavailable. Retry in a few moments.");

      default:
        const error = await response.text();
        console.error(`❌ Unexpected error: ${response.status}`);
        throw new Error(`Server error: ${error}`);
    }
  } catch (error) {
    console.error("Operation failed:", error.message);
    // Log for monitoring
    recordError({
      operation: "approveRefill",
      refillId,
      error: error.message,
      timestamp: new Date()
    });
    throw error;
  }
}
```

---

## Performance Considerations

### Example: Batch Pagination with Efficient Loading

```javascript
/**
 * Efficiently retrieve large result sets with minimal bandwidth
 */
async function efficientBatchRetrieval() {
  const pageSize = 100; // Max allowed
  let offset = 0;
  let allRefills = [];
  let hasMore = true;

  console.log("🚀 Starting efficient batch retrieval...\n");

  while (hasMore) {
    try {
      console.log(`Fetching batch at offset ${offset}...`);
      
      const startTime = Date.now();
      
      const response = await fetch(
        `http://localhost:3001/api/v1/refills?status=pending&limit=${pageSize}&offset=${offset}`,
        { headers: { "Authorization": `Bearer ${token}` } }
      );

      const elapsed = Date.now() - startTime;
      
      if (response.status === 200) {
        const data = await response.json();
        allRefills = [...allRefills, ...data.data];
        
        console.log(`   ✅ Retrieved ${data.data.length} items in ${elapsed}ms`);
        
        hasMore = data.pagination.hasMore;
        offset += pageSize;
      } else {
        console.error(`Error: ${response.status}`);
        hasMore = false;
      }
    } catch (error) {
      console.error(`Batch retrieval failed: ${error.message}`);
      break;
    }
  }

  console.log(`\n✅ Complete: ${allRefills.length} total refills retrieved`);
  return allRefills;
}
```

### Example: Caching Strategy

```javascript
/**
 * Cache refill status to reduce API calls
 */
class RefillStatusCache {
  constructor(ttlSeconds = 60) {
    this.cache = new Map();
    this.ttl = ttlSeconds * 1000;
  }

  async getRefillStatus(refillId) {
    // Check cache first
    if (this.cache.has(refillId)) {
      const cached = this.cache.get(refillId);
      if (Date.now() - cached.timestamp < this.ttl) {
        console.log(`📦 Cache hit: ${refillId}`);
        return cached.data;
      }
      this.cache.delete(refillId);
    }

    // Fetch fresh data
    console.log(`🔄 Fetching: ${refillId}`);
    const data = await getRefillStatus(refillId);

    // Store in cache
    this.cache.set(refillId, {
      data,
      timestamp: Date.now()
    });

    return data;
  }

  clear() {
    this.cache.clear();
    console.log("Cache cleared");
  }
}

// Usage
const statusCache = new RefillStatusCache(60); // 60-second TTL
const status1 = await statusCache.getRefillStatus("refill-123"); // Fetch
const status2 = await statusCache.getRefillStatus("refill-123"); // Cache hit
```

---

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Total Examples**: 21+ (7 curl + 7 Python + 7 JavaScript + 3 workflows + 2 error handling)  
**Status**: ✅ Production Ready
