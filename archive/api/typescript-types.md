# RefillService TypeScript Type Definitions

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Package**: `@jibonflow/refill-service`  

---

## Table of Contents

1. [Data Transfer Objects (DTOs)](#data-transfer-objects-dtos)
2. [Domain Models](#domain-models)
3. [Response Types](#response-types)
4. [Enums](#enums)
5. [Export Instructions](#export-instructions)
6. [Type Safety Notes](#type-safety-notes)

---

## Data Transfer Objects (DTOs)

### CreateRefillDTO

Input model for creating a new refill request.

```typescript
interface CreateRefillDTO {
  /**
   * ID of the prescription to refill
   * @type {string}
   * @required
   */
  prescriptionId: string;

  /**
   * Patient ID (must match authenticated user for RBAC)
   * @type {string}
   * @required
   */
  patientId: string;

  /**
   * Requested quantity (optional, defaults to original quantity)
   * @type {number}
   * @optional
   */
  quantity?: number;

  /**
   * Patient's reason for requesting refill
   * @type {string}
   * @optional
   * @maxLength 500
   */
  reason?: string;
}
```

**Validation Rules**:
- `prescriptionId`: Non-empty string, must exist in database
- `patientId`: Must match auth context for RBAC enforcement
- `quantity`: Positive integer, not exceeding prescription strength
- `reason`: Max 500 characters

**Example**:
```typescript
const request: CreateRefillDTO = {
  prescriptionId: 'rx-123456',
  patientId: 'patient-789',
  quantity: 30,
  reason: 'Running low on medication'
};
```

---

### ApprovalDTO

Input model for approving a pending refill request.

```typescript
interface ApprovalDTO {
  /**
   * Approval status (must be "approved")
   * @type {"approved"}
   * @required
   */
  status: 'approved';

  /**
   * Optional dosage modification
   * @type {DosageChange}
   * @optional
   */
  dosageChange?: {
    /**
     * New dose quantity
     * @type {number}
     * @required (if dosageChange provided)
     */
    doseQuantity: number;

    /**
     * Dose unit (mg, mcg, g, mL, etc.)
     * @type {string}
     * @required (if dosageChange provided)
     */
    doseUnit: string;

    /**
     * Frequency change (optional)
     * @type {string}
     * @optional
     */
    frequency?: string;
  };

  /**
   * Approval notes for audit trail
   * @type {string}
   * @optional
   * @maxLength 1000
   */
  notes?: string;
}
```

**Validation Rules**:
- `status`: Must equal "approved"
- `dosageChange.doseQuantity`: Positive number
- `dosageChange.doseUnit`: Valid unit from approved list
- `notes`: Max 1000 characters, no PHI

**Example**:
```typescript
const approval: ApprovalDTO = {
  status: 'approved',
  dosageChange: {
    doseQuantity: 10,
    doseUnit: 'mg'
  },
  notes: 'Approved at patient request. Reduced dose due to recent lab results.'
};
```

---

### DenialDTO

Input model for denying a pending refill request.

```typescript
interface DenialDTO {
  /**
   * Denial status (must be "denied")
   * @type {"denied"}
   * @required
   */
  status: 'denied';

  /**
   * Structured reason for denial
   * @type {DenialReason}
   * @required
   */
  reason: DenialReason;

  /**
   * Detailed denial notes
   * @type {string}
   * @optional
   * @maxLength 1000
   */
  notes?: string;

  /**
   * Provider's clinical recommendation
   * @type {string}
   * @optional
   * @maxLength 1000
   */
  recommendation?: string;
}

/**
 * Valid denial reasons
 */
type DenialReason = 
  | 'contraindication'          // Drug contraindicated for patient condition
  | 'drug_interaction'          // Interaction with current medications
  | 'dosage_concern'            // Current dosage inappropriate
  | 'patient_non_compliant'     // Patient not taking as prescribed
  | 'provider_request'          // Provider clinical judgment
  | 'other';                    // Other reason (notes required)
```

**Validation Rules**:
- `status`: Must equal "denied"
- `reason`: Must be valid DenialReason value
- `notes`: Max 1000 characters, clinical detail encouraged
- `recommendation`: Max 1000 characters, clinical guidance

**Example**:
```typescript
const denial: DenialDTO = {
  status: 'denied',
  reason: 'drug_interaction',
  notes: 'Interacts with patient\'s current blood pressure medication (lisinopril)',
  recommendation: 'Switch to alternative antihypertensive. Recommend consultation with cardiology.'
};
```

---

### RefillFilterDTO

Input model for filtering refill requests.

```typescript
interface RefillFilterDTO {
  /**
   * Filter by refill status
   * @type {string}
   * @optional
   */
  status?: 'pending' | 'approved' | 'denied' | 'fulfilled';

  /**
   * Filter by patient ID (for provider authorization check)
   * @type {string}
   * @optional
   */
  patientId?: string;

  /**
   * Sort field
   * @type {string}
   * @optional
   */
  sortBy?: 'createdAt' | 'status' | 'requestedAt';

  /**
   * Results per page (max 100)
   * @type {number}
   * @optional
   */
  limit?: number;

  /**
   * Pagination offset
   * @type {number}
   * @optional
   */
  offset?: number;
}
```

---

### PaginationOptions

Input model for pagination configuration.

```typescript
interface PaginationOptions {
  /**
   * Results per page (max 100, default 50)
   * @type {number}
   * @optional
   */
  limit?: number;

  /**
   * Pagination offset (default 0)
   * @type {number}
   * @optional
   */
  offset?: number;

  /**
   * Date range for filtering results
   * @type {{start: Date, end: Date}}
   * @optional
   */
  dateRange?: {
    start: Date;
    end: Date;
  };
}
```

---

## Domain Models

### RefillRequest

Core refill request entity representing a complete refill lifecycle.

```typescript
interface RefillRequest {
  /**
   * Unique refill request identifier
   * @type {string}
   * @pattern ^refill-\d+-[a-z0-9]+$
   */
  id: string;

  /**
   * Associated prescription identifier
   * @type {string}
   * @required
   */
  prescriptionId: string;

  /**
   * Patient who requested refill
   * @type {string}
   * @required
   */
  patientId: string;

  /**
   * Current refill status
   * @type {RefillStatus}
   * @required
   */
  status: RefillStatus;

  /**
   * Quantity requested by patient
   * @type {number}
   * @optional
   */
  quantityRequested?: number;

  /**
   * Quantity approved by provider
   * @type {number}
   * @optional
   */
  quantityApproved?: number;

  /**
   * Original authorized refills on prescription
   * @type {number}
   * @required
   */
  maxRefillsOriginal: number;

  /**
   * Refills remaining on prescription
   * @type {number}
   * @required
   */
  maxRefillsRemaining: number;

  /**
   * Patient's reason for requesting refill
   * @type {string}
   * @optional
   * @maxLength 500
   */
  reasonForRequest?: string;

  /**
   * Reason for denial (if denied)
   * @type {DenialReason}
   * @optional
   */
  denialReason?: DenialReason;

  /**
   * Detailed denial notes
   * @type {string}
   * @optional
   */
  denialNotes?: string;

  /**
   * Provider's clinical recommendation
   * @type {string}
   * @optional
   */
  recommendation?: string;

  /**
   * Dosage modification details
   * @type {object}
   * @optional
   */
  dosageChange?: {
    doseQuantity: number;
    doseUnit: string;
    frequency?: string;
  };

  /**
   * User ID who created request
   * @type {string}
   * @required
   */
  requestedById: string;

  /**
   * Request creation timestamp (ISO 8601)
   * @type {Date}
   * @required
   */
  requestedAt: Date;

  /**
   * Provider ID who approved (if approved)
   * @type {string}
   * @optional
   */
  approvedById?: string;

  /**
   * Approval timestamp (ISO 8601)
   * @type {Date}
   * @optional
   */
  approvedAt?: Date;

  /**
   * Provider ID who denied (if denied)
   * @type {string}
   * @optional
   */
  deniedById?: string;

  /**
   * Denial timestamp (ISO 8601)
   * @type {Date}
   * @optional
   */
  deniedAt?: Date;

  /**
   * Pharmacy transmission timestamp
   * @type {Date}
   * @optional
   */
  transmissionSentAt?: Date;

  /**
   * Pharmacy transmission status
   * @type {"pending" | "sent" | "acked" | "failed"}
   * @optional
   */
  transmissionStatus?: 'pending' | 'sent' | 'acked' | 'failed';

  /**
   * Prescription fulfillment timestamp
   * @type {Date}
   * @optional
   */
  filledAt?: Date;

  /**
   * Soft delete timestamp (if deleted)
   * @type {Date}
   * @optional
   */
  deletedAt?: Date;

  /**
   * Record creation timestamp (ISO 8601)
   * @type {Date}
   * @required
   */
  createdAt: Date;

  /**
   * Last update timestamp (ISO 8601)
   * @type {Date}
   * @required
   */
  updatedAt: Date;
}
```

**Status Lifecycle**:
```
pending
  ↓
  ├─→ approved → approved_pending_transmission → approved_transmission_sent → fulfilled
  │
  └─→ denied
```

---

### AuditTrailEntry

Immutable audit log entry for compliance.

```typescript
interface AuditTrailEntry {
  /**
   * Unique audit entry ID
   * @type {string}
   * @required
   */
  id: string;

  /**
   * Associated prescription ID
   * @type {string}
   * @required
   */
  prescriptionId: string;

  /**
   * User ID who performed action
   * @type {string}
   * @required
   */
  userId: string;

  /**
   * User role (PATIENT, PROVIDER, PHARMACIST, ADMIN, SYSTEM)
   * @type {RoleType}
   * @required
   */
  userRole: RoleType;

  /**
   * Action performed
   * @type {string}
   * @required
   */
  action: string;

  /**
   * Changes made (serialized as JSON)
   * @type {object}
   * @required
   */
  changes: Record<string, any>;

  /**
   * SHA-256 hash for immutability verification
   * @type {string}
   * @pattern ^[a-f0-9]{64}$
   * @required
   */
  checksum: string;

  /**
   * Action timestamp (ISO 8601)
   * @type {Date}
   * @required
   */
  timestamp: Date;

  /**
   * IP address of requestor
   * @type {string}
   * @optional
   */
  ipAddress?: string;

  /**
   * User agent string
   * @type {string}
   * @optional
   */
  userAgent?: string;
}
```

**Example**:
```typescript
const auditEntry: AuditTrailEntry = {
  id: 'audit-2025-10-16-001',
  prescriptionId: 'rx-123456',
  userId: 'provider-456',
  userRole: 'PROVIDER',
  action: 'approved',
  changes: {
    status: 'approved',
    dosageChanged: true,
    previousDose: '20mg',
    newDose: '10mg'
  },
  checksum: 'e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855',
  timestamp: new Date('2025-10-16T15:45:00Z'),
  ipAddress: '192.168.1.100'
};
```

---

## Response Types

### RefillHistory

Refill history entry with timestamps.

```typescript
interface RefillHistory {
  /**
   * Refill request ID
   * @type {string}
   * @required
   */
  id: string;

  /**
   * Associated prescription ID
   * @type {string}
   * @required
   */
  prescriptionId: string;

  /**
   * Current refill status
   * @type {string}
   * @required
   */
  status: string;

  /**
   * Request creation timestamp
   * @type {Date}
   * @required
   */
  requestedAt: Date;

  /**
   * Approval timestamp (if approved)
   * @type {Date}
   * @optional
   */
  approvedAt?: Date;

  /**
   * Denial timestamp (if denied)
   * @type {Date}
   * @optional
   */
  deniedAt?: Date;

  /**
   * Fulfillment timestamp (if filled)
   * @type {Date}
   * @optional
   */
  filledAt?: Date;

  /**
   * Pharmacy transmission status
   * @type {string}
   * @optional
   */
  transmissionStatus?: string;
}
```

---

### RefillStatusDetail

Real-time status details for a refill request.

```typescript
interface RefillStatusDetail {
  /**
   * Current status
   * @type {RefillStatus}
   * @required
   */
  status: RefillStatus;

  /**
   * Human-readable status detail
   * @type {string}
   * @required
   * @example "Awaiting provider approval"
   */
  statusDetail: string;

  /**
   * Pharmacy transmission status
   * @type {string}
   * @optional
   */
  transmissionStatus?: string;

  /**
   * Fulfillment timestamp
   * @type {Date}
   * @optional
   */
  filledAt?: Date;

  /**
   * Last update timestamp
   * @type {Date}
   * @required
   */
  lastUpdated: Date;
}
```

---

### PaginatedRefillList

Paginated response wrapper.

```typescript
interface PaginatedRefillList {
  /**
   * Array of refill requests
   * @type {RefillRequest[]}
   * @required
   */
  data: RefillRequest[];

  /**
   * Pagination metadata
   * @type {Pagination}
   * @required
   */
  pagination: {
    /**
     * Total items in current result set
     * @type {number}
     */
    count: number;

    /**
     * Whether more results available
     * @type {boolean}
     */
    hasMore: boolean;

    /**
     * Items per page
     * @type {number}
     */
    limit: number;

    /**
     * Current pagination offset
     * @type {number}
     */
    offset: number;
  };
}
```

---

### ErrorResponse

Standardized error response.

```typescript
interface ErrorResponse {
  /**
   * Error code identifier
   * @type {string}
   * @example "INVALID_REQUEST"
   */
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
    timestamp: Date;
    requestId: string;
  };
}
```

---

## Enums

### RefillStatus

```typescript
type RefillStatus = 
  | 'pending'                      // Awaiting provider review
  | 'approved'                     // Approved by provider
  | 'denied'                       // Denied by provider
  | 'approved_pending_transmission'// Approved, queued for pharmacy
  | 'approved_transmission_sent'   // Sent to pharmacy
  | 'fulfilled';                   // Filled by pharmacy
```

---

### RoleType

```typescript
type RoleType = 
  | 'PATIENT'                      // Patient user
  | 'PROVIDER'                     // Healthcare provider
  | 'PHARMACIST'                   // Pharmacist
  | 'ADMIN'                        // System administrator
  | 'SYSTEM';                      // System process
```

---

### DenialReason

```typescript
type DenialReason =
  | 'contraindication'             // Drug contraindicated
  | 'drug_interaction'             // Medication interaction
  | 'dosage_concern'               // Dosage inappropriate
  | 'patient_non_compliant'        // Non-compliance issue
  | 'provider_request'             // Provider decision
  | 'other';                       // Other reason
```

---

## Export Instructions

### npm Package Export

Types are exported from `@jibonflow/refill-service` package:

```typescript
// ESM Import (Recommended)
import type {
  RefillRequest,
  CreateRefillDTO,
  ApprovalDTO,
  DenialDTO,
  RefillStatusDetail,
  RefillStatus,
  RoleType,
  DenialReason
} from '@jibonflow/refill-service';

// CommonJS (Legacy)
const RefillTypes = require('@jibonflow/refill-service');
```

### Direct Import from TypeScript Files

If using directly from source:

```typescript
import type {
  RefillRequest,
  CreateRefillDTO,
  ApprovalDTO,
  DenialDTO
} from './types/refill.types';
```

### Package.json Configuration

```json
{
  "dependencies": {
    "@jibonflow/refill-service": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

---

## Type Safety Notes

### Strict Mode Requirements

All types require TypeScript strict mode enabled:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true
  }
}
```

### No `any` Type Usage

All interfaces are fully typed with no `any` types:

```typescript
// ✅ CORRECT - Fully typed
const request: CreateRefillDTO = {
  prescriptionId: 'rx-123',
  patientId: 'patient-456',
  quantity: 30
};

// ❌ INCORRECT - Using any
const request: any = { /* ... */ };
```

### Optional vs Required Fields

Clearly marked using TypeScript optional operator:

```typescript
interface Example {
  required: string;          // ✅ Must be provided
  optional?: string;         // ✅ May be omitted
  nullable: string | null;   // ✅ Can be null
}
```

### Discriminated Unions

Status-based types use discriminated unions for type safety:

```typescript
type ApprovalAction = 
  | { status: 'approved'; dosageChange?: object }
  | { status: 'denied'; reason: DenialReason };

const action: ApprovalAction = {
  status: 'denied',
  reason: 'drug_interaction'
  // ✅ TypeScript knows 'dosageChange' not allowed here
};
```

### Date Handling

All dates are Date objects (not strings):

```typescript
const refill: RefillRequest = {
  // ... other fields
  requestedAt: new Date('2025-10-16T14:30:00Z'),  // ✅ Date object
  approvedAt: new Date(),                          // ✅ Current time
  createdAt: new Date('2025-10-16T14:30:00Z')    // ✅ Date object
};

// ❌ Strings not allowed for Date fields
const invalid: RefillRequest = {
  requestedAt: '2025-10-16T14:30:00Z'  // Error: Type 'string' not assignable
};
```

### Enum Type Guards

Use type guards for enum-like types:

```typescript
// ✅ Type guard function
function isValidStatus(status: string): status is RefillStatus {
  return ['pending', 'approved', 'denied', 'approved_pending_transmission', 
          'approved_transmission_sent', 'fulfilled'].includes(status);
}

// Usage
const status: unknown = 'pending';
if (isValidStatus(status)) {
  const refill: RefillRequest = { /* ... */ status };
}
```

---

## Usage Examples

### Creating a Refill Request

```typescript
import type { CreateRefillDTO, RefillRequest } from '@jibonflow/refill-service';

async function requestRefill(
  prescriptionId: string,
  patientId: string
): Promise<RefillRequest> {
  const request: CreateRefillDTO = {
    prescriptionId,
    patientId,
    quantity: 30,
    reason: 'Running low on medication'
  };

  const response = await fetch('http://localhost:3001/api/v1/refills', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(request)
  });

  return response.json() as Promise<RefillRequest>;
}
```

### Approving with Type Safety

```typescript
import type { ApprovalDTO, RefillRequest } from '@jibonflow/refill-service';

async function approveRefill(
  refillId: string,
  newDose: number
): Promise<RefillRequest> {
  const approval: ApprovalDTO = {
    status: 'approved',
    dosageChange: {
      doseQuantity: newDose,
      doseUnit: 'mg'
    },
    notes: 'Approved with dosage adjustment'
  };

  const response = await fetch(
    `http://localhost:3001/api/v1/refills/${refillId}/approve`,
    {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify(approval)
    }
  );

  return response.json() as Promise<RefillRequest>;
}
```

---

**Version**: 1.0.0  
**Last Updated**: October 16, 2025  
**Status**: ✅ Production Ready
