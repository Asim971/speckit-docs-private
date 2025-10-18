# RefillService API Documentation

**Version**: 1.0.0  
**Status**: Production Ready  
**Last Updated**: October 16, 2025  
**Compliance**: ✅ HIPAA Business Associate Ready  

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [API Endpoints](#api-endpoints)
4. [Data Models](#data-models)
5. [Error Handling](#error-handling)
6. [HIPAA Compliance](#hipaa-compliance)

---

## Overview

The RefillService API provides a HIPAA-compliant medication refill management system handling the complete request lifecycle from patient initiation through pharmacy transmission.

### Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://api.jibonflow.com/api/v1` |
| Staging | `https://staging.jibonflow.com/api/v1` |
| Development | `http://localhost:3001/api/v1` |

### Key Features

✅ RBAC Enforcement at 6 critical checkpoints  
✅ Audit Logging with SHA-256 checksums  
✅ PII Protection (TLS 1.3, AES-256)  
✅ Real-time status tracking  
✅ Pagination support  
✅ Consistent error standards  

---

## Authentication

All API requests require JWT Bearer token authentication:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### RBAC Roles

| Role | Permissions |
|------|-------------|
| **patient** | Create own refills, view own history |
| **pharmacist** | Approve/deny refills, list all pending, transmit to pharmacy |
| **doctor** | Approve/deny refills, list all pending |
| **admin** | Full access to all operations |
| **system** | Backend automation (transmit, sync) |

---

## API Endpoints

### 1. Create Refill Request

**POST** `/refills`

**Role**: `patient` (can only create for self)

**Request**:
```json
{
  "prescriptionId": "rx-123456",
  "patientId": "patient-789",
  "quantity": 30,
  "reason": "Running low on medication"
}
```

**Response (201)**:
```json
{
  "id": "refill-abc123",
  "prescriptionId": "rx-123456",
  "patientId": "patient-789",
  "status": "pending",
  "quantityRequested": 30,
  "requestedAt": "2025-10-16T14:30:00Z"
}
```

### 2. Approve Refill

**PATCH** `/refills/{id}/approve`

**Role**: `pharmacist`, `doctor`, `admin`

**Request**:
```json
{
  "quantityApproved": 30,
  "notes": "Approved"
}
```

### 3. Deny Refill

**PATCH** `/refills/{id}/deny`

**Role**: `pharmacist`, `doctor`, `admin`

**Request**:
```json
{
  "reason": "drug_interaction",
  "notes": "Interacts with current medication"
}
```

### 4. List Refill Requests

**GET** `/refills?status=pending&limit=20&offset=0`

**Role**: All roles (RBAC-filtered)

### 5. Get Refill Status

**GET** `/refills/status/{id}`

**Role**: All roles (RBAC-filtered)

### 6. Get Refill History

**GET** `/refills/history?patientId={id}&limit=10`

**Role**: All roles (patient sees only own)

### 7. Transmit to Pharmacy

**POST** `/refills/{id}/transmit`

**Role**: `pharmacist`, `system`, `admin`

---

## Error Handling

### Error Response Format

```json
{
  "code": "ERROR_CODE",
  "message": "Human-readable message",
  "details": { "field": "value" }
}
```

### HTTP Status Codes

| Code | Error | Scenario |
|------|-------|----------|
| 400 | `INVALID_REQUEST` | Missing required fields |
| 401 | `UNAUTHORIZED` | Invalid/missing token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `DUPLICATE_REFILL` | Pending request exists |
| 422 | `MAX_REFILLS_EXCEEDED` | No refills remaining |
| 500 | `INTERNAL_ERROR` | Server error |
| 503 | `SERVICE_UNAVAILABLE` | Service down |

---

## HIPAA Compliance

### Audit Logging
All state-changing operations logged with SHA-256 checksums

### RBAC Enforcement  
6 critical checkpoints validated

### PII Protection
- TLS 1.3 in transit
- AES-256 at rest
- Redacted error messages

---

**See full documentation**: `integration-examples.md`, `error-reference.md`, `hipaa-compliance.md`, `typescript-types.md`, `refill-service.openapi.json`
