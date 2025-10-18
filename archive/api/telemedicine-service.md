# Telemedicine Service API Documentation

The Telemedicine Service provides secure, HIPAA-compliant video consultation capabilities using Agora RTC with End-to-End Encryption (E2EE). Optimized for low-bandwidth networks common in Bangladesh.

## Base URL
```
Production: https://api.jibonflow.com/telemedicine
Development: http://localhost:4002/telemedicine
```

## Features
- 🔒 **E2EE Video Calls** - AES-256 encryption for all communications
- 📱 **Low-Bandwidth Optimized** - Works on 3G networks with <200ms latency
- 🏥 **HIPAA Compliant** - Full audit logging and data protection
- 👥 **Multi-Party Sessions** - Support for patient, provider, observers
- 📋 **FHIR Integration** - Automatic medical record updates
- 🎥 **Screen Sharing** - Share medical images and documents
- 📊 **Quality Monitoring** - Real-time connection quality assessment

## Authentication
All endpoints require JWT authentication:
```
Authorization: Bearer <access_token>
```

---

## 🎥 Consultation Management

### Create Consultation Session
Creates a new secure video consultation session.

**POST** `/api/telemedicine/consultations`
**Auth Required:** Yes (Provider or Admin only)

**Request Body:**
```json
{
  "patientId": "patient-uuid-v4",
  "providerId": "provider-uuid-v4", 
  "duration": 3600,
  "appointmentId": "appointment-uuid-v4",
  "consultationType": "initial",
  "specialization": "general-practice"
}
```

**Parameters:**
- `patientId` (required): UUID of the patient
- `providerId` (required): UUID of the healthcare provider
- `duration` (optional): Session duration in seconds (300-7200, default: 3600)
- `appointmentId` (optional): Associated appointment ID
- `consultationType` (optional): Type of consultation (`initial`, `follow-up`, `emergency`)
- `specialization` (optional): Medical specialization

**Success Response (201):**
```json
{
  "success": true,
  "session": {
    "sessionId": "session-uuid-v4",
    "channelName": "encrypted-channel-name",
    "expiresAt": "2024-01-01T01:00:00.000Z",
    "status": "created",
    "participants": {
      "patient": {
        "id": "patient-uuid-v4",
        "name": "John Doe",
        "status": "pending"
      },
      "provider": {
        "id": "provider-uuid-v4", 
        "name": "Dr. Smith",
        "status": "pending"
      }
    },
    "encryptionEnabled": true,
    "qualityProfile": "low-bandwidth"
  },
  "message": "Consultation session created successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

### Join Consultation Session
Joins an existing consultation session and receives connection credentials.

**POST** `/api/telemedicine/consultations/:sessionId/join`
**Auth Required:** Yes

**Path Parameters:**
- `sessionId`: Session UUID to join

**Request Body:**
```json
{
  "role": "patient",
  "deviceInfo": {
    "type": "mobile",
    "os": "android",
    "version": "12",
    "capabilities": ["video", "audio", "screen-share"]
  }
}
```

**Parameters:**
- `role` (required): Participant role (`patient`, `provider`, `observer`)
- `deviceInfo` (optional): Device capabilities for optimization

**Success Response (200):**
```json
{
  "success": true,
  "connection": {
    "token": "agora-rtc-token",
    "channelName": "encrypted-channel-name",
    "appId": "agora-app-id",
    "uid": 12345,
    "encryptionKey": "aes-256-encryption-key",
    "hasEncryptionKey": true,
    "serverConfig": {
      "iceServers": [
        {
          "urls": "stun:stun.jibonflow.com",
          "username": "user",
          "credential": "pass"
        }
      ]
    }
  },
  "sessionInfo": {
    "duration": 3600,
    "remainingTime": 3580,
    "participants": [
      {
        "uid": 12345,
        "role": "patient",
        "name": "John Doe",
        "status": "connected"
      },
      {
        "uid": 67890,
        "role": "provider", 
        "name": "Dr. Smith",
        "status": "connected"
      }
    ]
  },
  "qualitySettings": {
    "videoProfile": "240p_1",
    "audioProfile": "music_standard",
    "adaptiveBitrate": true
  },
  "message": "Joined session successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

### Leave Consultation Session
Leaves the current consultation session.

**POST** `/api/telemedicine/consultations/:sessionId/leave`
**Auth Required:** Yes

**Path Parameters:**
- `sessionId`: Session UUID to leave

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "sessionDuration": 1850,
    "disconnectedAt": "2024-01-01T00:30:50.000Z"
  },
  "message": "Left session successfully",
  "timestamp": "2024-01-01T00:30:50.000Z",
  "requestId": "uuid-v4"
}
```

---

### End Consultation Session
Ends the consultation session for all participants (Provider only).

**POST** `/api/telemedicine/consultations/:sessionId/end`
**Auth Required:** Yes (Provider or Admin only)

**Path Parameters:**
- `sessionId`: Session UUID to end

**Request Body:**
```json
{
  "reason": "consultation_completed",
  "notes": "Follow-up required in 2 weeks",
  "duration": 1800
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "sessionId": "session-uuid-v4",
    "endedAt": "2024-01-01T00:30:00.000Z",
    "totalDuration": 1800,
    "participants": [
      {
        "uid": 12345,
        "role": "patient",
        "joinedAt": "2024-01-01T00:00:00.000Z",
        "leftAt": "2024-01-01T00:30:00.000Z",
        "duration": 1800
      }
    ],
    "qualityMetrics": {
      "averageLatency": 180,
      "packetsLost": 0.1,
      "averageBitrate": 256,
      "connectionQuality": "excellent"
    },
    "recordingAvailable": false
  },
  "message": "Session ended successfully",
  "timestamp": "2024-01-01T00:30:00.000Z",
  "requestId": "uuid-v4"
}
```

---

## 📋 Session Management

### Get Session Details
Retrieves detailed information about a consultation session.

**GET** `/api/telemedicine/consultations/:sessionId`
**Auth Required:** Yes

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "session": {
      "sessionId": "session-uuid-v4",
      "channelName": "encrypted-channel-name",
      "status": "active",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "expiresAt": "2024-01-01T01:00:00.000Z",
      "duration": 3600,
      "remainingTime": 2400,
      "participants": [
        {
          "uid": 12345,
          "userId": "patient-uuid-v4",
          "role": "patient",
          "name": "John Doe",
          "status": "connected",
          "joinedAt": "2024-01-01T00:10:00.000Z",
          "connectionQuality": "good"
        }
      ],
      "encryptionEnabled": true,
      "recordingEnabled": false,
      "qualityProfile": "low-bandwidth"
    }
  },
  "message": "Session details retrieved successfully",
  "timestamp": "2024-01-01T00:40:00.000Z",
  "requestId": "uuid-v4"
}
```

---

### List User Sessions
Retrieves consultation sessions for the authenticated user.

**GET** `/api/telemedicine/consultations`
**Auth Required:** Yes

**Query Parameters:**
- `status`: Filter by status (`scheduled`, `active`, `completed`, `cancelled`)
- `limit`: Number of results (default: 20, max: 100)
- `offset`: Pagination offset (default: 0)
- `startDate`: Filter from date (ISO 8601)
- `endDate`: Filter to date (ISO 8601)

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "sessions": [
      {
        "sessionId": "session-uuid-v4",
        "status": "completed",
        "createdAt": "2024-01-01T00:00:00.000Z",
        "endedAt": "2024-01-01T00:30:00.000Z",
        "duration": 1800,
        "participants": {
          "patient": {
            "id": "patient-uuid-v4",
            "name": "John Doe"
          },
          "provider": {
            "id": "provider-uuid-v4",
            "name": "Dr. Smith"
          }
        },
        "consultationType": "follow-up",
        "specialization": "cardiology"
      }
    ],
    "pagination": {
      "total": 1,
      "limit": 20,
      "offset": 0,
      "hasMore": false
    }
  },
  "message": "Sessions retrieved successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

## 🔊 Real-time Communication

### Update Connection Quality
Reports connection quality metrics during an active session.

**POST** `/api/telemedicine/consultations/:sessionId/quality`
**Auth Required:** Yes

**Request Body:**
```json
{
  "latency": 180,
  "packetsLost": 0.5,
  "bitrate": 256,
  "resolution": "240p",
  "frameRate": 15,
  "audioQuality": "good",
  "networkType": "wifi"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "qualityScore": 85,
    "recommendations": [
      "Consider reducing video quality for better performance"
    ]
  },
  "message": "Quality metrics updated",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

### Send Session Message
Sends a secure chat message during consultation.

**POST** `/api/telemedicine/consultations/:sessionId/messages`
**Auth Required:** Yes

**Request Body:**
```json
{
  "message": "Please describe your symptoms in detail",
  "messageType": "text",
  "isPrivate": false
}
```

**Success Response (201):**
```json
{
  "success": true,
  "data": {
    "messageId": "message-uuid-v4",
    "message": "Please describe your symptoms in detail",
    "senderId": "provider-uuid-v4",
    "senderName": "Dr. Smith",
    "timestamp": "2024-01-01T00:15:00.000Z",
    "encrypted": true
  },
  "message": "Message sent successfully",
  "timestamp": "2024-01-01T00:15:00.000Z",
  "requestId": "uuid-v4"
}
```

---

## 📱 Mobile & Low-Bandwidth Optimization

### Get Network Optimization
Retrieves optimized settings based on network conditions.

**GET** `/api/telemedicine/optimize`
**Auth Required:** Yes

**Query Parameters:**
- `networkType`: Network type (`2g`, `3g`, `4g`, `wifi`)
- `deviceType`: Device type (`mobile`, `tablet`, `desktop`)
- `batteryLevel`: Battery percentage (0-100)

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "videoProfile": "240p_1",
    "audioProfile": "speech_standard",
    "maxBitrate": 200,
    "adaptiveBitrate": true,
    "enableEcho": false,
    "enableNoise": true,
    "batteryOptimization": true,
    "recommendedDuration": 1800
  },
  "message": "Optimization settings retrieved",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

## 🏥 FHIR Integration

### Get Consultation FHIR Data
Retrieves FHIR-compliant consultation data.

**GET** `/api/telemedicine/consultations/:sessionId/fhir`
**Auth Required:** Yes (Provider only)

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "resourceType": "Encounter",
    "id": "session-uuid-v4",
    "status": "finished",
    "class": {
      "system": "http://terminology.hl7.org/CodeSystem/v3-ActCode",
      "code": "VR",
      "display": "virtual"
    },
    "type": [
      {
        "coding": [
          {
            "system": "http://snomed.info/sct",
            "code": "185317003",
            "display": "Telemedicine consultation"
          }
        ]
      }
    ],
    "subject": {
      "reference": "Patient/patient-uuid-v4",
      "display": "John Doe"
    },
    "participant": [
      {
        "type": [
          {
            "coding": [
              {
                "system": "http://terminology.hl7.org/CodeSystem/v3-ParticipationType",
                "code": "PPRF",
                "display": "primary performer"
              }
            ]
          }
        ],
        "individual": {
          "reference": "Practitioner/provider-uuid-v4",
          "display": "Dr. Smith"
        }
      }
    ],
    "period": {
      "start": "2024-01-01T00:00:00.000Z",
      "end": "2024-01-01T00:30:00.000Z"
    }
  },
  "message": "FHIR data retrieved successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

## 🔒 Security & Compliance

### Key Features
- **E2EE Encryption**: AES-256 encryption for all video/audio streams
- **HIPAA Compliance**: Complete audit trails and data protection
- **Access Control**: Role-based permissions (patient/provider/observer)
- **Session Security**: Encrypted channel names and secure token generation
- **Network Security**: STUN/TURN servers with authentication
- **Data Protection**: Automatic session cleanup and secure key rotation

### Audit Events
All telemedicine activities are logged:
- Session creation, join, leave, end
- Quality degradation events
- Security violations
- Encryption key rotations
- Participant authentication events

---

## 📊 Monitoring & Analytics

### Quality Metrics
The service tracks:
- Connection latency and packet loss
- Video/audio quality scores
- Session duration and completion rates
- Network performance by region/provider
- Device compatibility statistics

### Health Endpoints

**GET** `/health` - Service health check
**GET** `/health/detailed` - Detailed health with dependencies

---

## Error Codes

| Code | Description |
|------|-------------|
| `SESSION_NOT_FOUND` | Consultation session not found |
| `SESSION_EXPIRED` | Session has expired |
| `SESSION_FULL` | Maximum participants reached |
| `INVALID_ROLE` | Invalid participant role |
| `UNAUTHORIZED_ACCESS` | User not authorized for session |
| `ENCRYPTION_FAILED` | E2EE encryption setup failed |
| `NETWORK_ERROR` | Network connectivity issues |
| `QUALITY_DEGRADED` | Connection quality below threshold |
| `DEVICE_INCOMPATIBLE` | Device not supported |
| `BANDWIDTH_INSUFFICIENT` | Network bandwidth too low |

---

## Development & Testing

### Test Environment
```bash
# Start telemedicine service
npm run dev

# Run with mock Agora (no real video)
MOCK_AGORA=true npm run dev

# Enable debug logging
DEBUG=telemedicine:* npm run dev
```

### Test Accounts
```json
{
  "patient": {
    "id": "patient-test-uuid",
    "token": "patient-jwt-token"
  },
  "provider": {
    "id": "provider-test-uuid", 
    "token": "provider-jwt-token"
  }
}
```

### cURL Examples

**Create Session:**
```bash
curl -X POST http://localhost:4002/api/telemedicine/consultations \
  -H "Authorization: Bearer <provider-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "patientId": "patient-uuid",
    "providerId": "provider-uuid",
    "duration": 1800
  }'
```

**Join Session:**
```bash
curl -X POST http://localhost:4002/api/telemedicine/consultations/{sessionId}/join \
  -H "Authorization: Bearer <user-token>" \
  -H "Content-Type: application/json" \
  -d '{"role": "patient"}'
```