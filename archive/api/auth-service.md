# Auth Service API Documentation

The Auth Service handles all authentication and authorization functionality for the JibonFlow platform, including user registration, login, JWT token management, and Multi-Factor Authentication (MFA).

## Base URL
```
Production: https://api.jibonflow.com/auth
Development: http://localhost:4000/auth
```

## Authentication
Protected endpoints require a valid JWT access token in the Authorization header:
```
Authorization: Bearer <access_token>
```

## Rate Limiting
- General auth endpoints: 10 requests per minute
- Login attempts: 5 attempts per 15 minutes (per IP)

---

## Endpoints

### 🔐 Core Authentication

#### Register User
Creates a new user account with email verification.

**POST** `/auth/register`

**Request Body:**
```json
{
  "email": "patient@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "+8801712345678",
  "userType": "patient"
}
```

**Validation Rules:**
- `email`: Valid email format, 3-254 characters
- `password`: 12-128 characters, must contain uppercase, lowercase, number, and special character
- `firstName`: 1-100 characters, required
- `lastName`: 1-100 characters, required  
- `phone`: Optional, international format (+8801234567890)
- `userType`: One of: `patient`, `provider`, `pharmacy`, `pharma`, `chw`, `admin`

**Success Response (201):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid-v4",
      "email": "patient@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "userType": "patient",
      "isEmailVerified": false,
      "createdAt": "2024-01-01T00:00:00.000Z"
    },
    "tokens": {
      "accessToken": "jwt-access-token",
      "refreshToken": "jwt-refresh-token",
      "expiresIn": 3600
    }
  },
  "message": "Registration successful. Please verify your email.",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

#### Login User
Authenticates user credentials and returns JWT tokens.

**POST** `/auth/login`

**Request Body:**
```json
{
  "email": "patient@example.com",
  "password": "SecurePass123!",
  "mfaToken": "123456"
}
```

**Parameters:**
- `email`: User's email address (required)
- `password`: User's password (required) 
- `mfaToken`: 6-digit MFA code (required if MFA enabled)

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid-v4",
      "email": "patient@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "userType": "patient",
      "isEmailVerified": true,
      "mfaEnabled": false,
      "lastLoginAt": "2024-01-01T00:00:00.000Z"
    },
    "tokens": {
      "accessToken": "jwt-access-token",
      "refreshToken": "jwt-refresh-token", 
      "expiresIn": 3600
    },
    "session": {
      "id": "session-uuid",
      "deviceInfo": "Mozilla/5.0...",
      "ipAddress": "192.168.1.100",
      "createdAt": "2024-01-01T00:00:00.000Z"
    }
  },
  "message": "Login successful",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

**Error Response (401):**
```json
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid email or password",
    "details": []
  },
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

#### Refresh Token
Generates a new access token using a valid refresh token.

**POST** `/auth/refresh`

**Request Body:**
```json
{
  "refreshToken": "jwt-refresh-token"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "tokens": {
      "accessToken": "new-jwt-access-token",
      "refreshToken": "new-jwt-refresh-token",
      "expiresIn": 3600
    }
  },
  "message": "Token refreshed successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

#### Logout User
Invalidates the current session and tokens.

**POST** `/auth/logout`
**Auth Required:** Yes

**Success Response (200):**
```json
{
  "success": true,
  "data": null,
  "message": "Logout successful",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

### 👤 User Profile

#### Get User Profile
Retrieves the current authenticated user's profile information.

**GET** `/auth/profile`
**Auth Required:** Yes

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid-v4",
      "email": "patient@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "phone": "+8801712345678",
      "userType": "patient",
      "isEmailVerified": true,
      "mfaEnabled": false,
      "createdAt": "2024-01-01T00:00:00.000Z",
      "updatedAt": "2024-01-01T00:00:00.000Z",
      "lastLoginAt": "2024-01-01T00:00:00.000Z",
      "profile": {
        "avatar": "https://cdn.jibonflow.com/avatars/user123.jpg",
        "dateOfBirth": "1990-01-01",
        "gender": "male",
        "address": {
          "street": "123 Main St",
          "city": "Dhaka",
          "division": "Dhaka",
          "postalCode": "1000"
        }
      }
    }
  },
  "message": "Profile retrieved successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

---

### 📧 Email Verification

#### Verify Email
Verifies a user's email address using a verification token.

**GET** `/auth/verify-email/:token`

**Parameters:**
- `token`: Email verification token from verification email

**Success Response (200):**
```json
{
  "success": true,
  "data": {
    "isEmailVerified": true
  },
  "message": "Email verified successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

**Status:** ⚠️ Not yet implemented (returns 501)

---

### 🔑 Password Management

#### Forgot Password
Initiates the password reset process by sending a reset email.

**POST** `/auth/forgot-password`

**Request Body:**
```json
{
  "email": "patient@example.com"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": null,
  "message": "Password reset email sent",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

**Status:** ⚠️ Not yet implemented (returns 501)

---

#### Reset Password
Resets user password using a valid reset token.

**POST** `/auth/reset-password`

**Request Body:**
```json
{
  "token": "password-reset-token",
  "password": "NewSecurePass123!"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": null,
  "message": "Password reset successful",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

**Status:** ⚠️ Not yet implemented (returns 501)

---

#### Change Password
Changes password for an authenticated user.

**POST** `/auth/change-password`
**Auth Required:** Yes

**Request Body:**
```json
{
  "currentPassword": "OldSecurePass123!",
  "newPassword": "NewSecurePass123!"
}
```

**Success Response (200):**
```json
{
  "success": true,
  "data": null,
  "message": "Password changed successfully",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "requestId": "uuid-v4"
}
```

**Status:** ⚠️ Not yet implemented (returns 501)

---

## Multi-Factor Authentication (MFA)

For MFA endpoints, see [MFA Service Documentation](./mfa-service.md).

---

## Error Codes

| Code | Description |
|------|-------------|
| `VALIDATION_ERROR` | Request validation failed |
| `INVALID_CREDENTIALS` | Invalid email or password |
| `USER_NOT_FOUND` | User does not exist |
| `USER_ALREADY_EXISTS` | Email already registered |
| `EMAIL_NOT_VERIFIED` | Email verification required |
| `MFA_REQUIRED` | MFA token required |
| `INVALID_MFA_TOKEN` | Invalid MFA code |
| `TOKEN_EXPIRED` | JWT token has expired |
| `INVALID_TOKEN` | Invalid or malformed token |
| `SESSION_EXPIRED` | User session has expired |
| `RATE_LIMIT_EXCEEDED` | Too many requests |
| `ACCOUNT_LOCKED` | Account temporarily locked |

---

## Security Features

### Password Requirements
- Minimum 12 characters
- Must contain: uppercase, lowercase, number, special character
- Cannot contain common passwords or user information

### Session Management
- JWT access tokens (1 hour expiry)
- Refresh tokens (30 days expiry)
- Automatic session invalidation on logout
- Device tracking and suspicious activity detection

### Rate Limiting
- Login: 5 attempts per 15 minutes per IP
- Registration: 3 attempts per hour per IP
- Password reset: 3 attempts per hour per email

### Security Headers
All responses include security headers:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `X-XSS-Protection: 1; mode=block`
- `Strict-Transport-Security: max-age=31536000`

---

## Development Notes

### Not Yet Implemented
- Email verification (`/verify-email/:token`)
- Password reset flow (`/forgot-password`, `/reset-password`)
- Password change (`/change-password`)

### TODO Items
- Implement email verification service
- Add password reset functionality  
- Implement password change with current password verification
- Add account lockout after failed attempts
- Implement device management and session controls
- Add audit logging for security events

---

## Testing

### Test User Accounts
Development environment includes test accounts:

```json
{
  "patient": {
    "email": "patient@test.com",
    "password": "TestPassword123!"
  },
  "provider": {
    "email": "doctor@test.com", 
    "password": "TestPassword123!"
  },
  "pharmacy": {
    "email": "pharmacy@test.com",
    "password": "TestPassword123!"
  }
}
```

### cURL Examples

**Register:**
```bash
curl -X POST http://localhost:4000/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "TestPassword123!",
    "firstName": "Test",
    "lastName": "User",
    "userType": "patient"
  }'
```

**Login:**
```bash
curl -X POST http://localhost:4000/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "TestPassword123!"
  }'
```

**Get Profile:**
```bash
curl -X GET http://localhost:4000/auth/profile \
  -H "Authorization: Bearer <access_token>"
```