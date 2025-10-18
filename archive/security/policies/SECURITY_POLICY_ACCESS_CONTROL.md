# Access Control Policy
## JibonFlow Healthcare Platform

**Document Version**: 1.0  
**Effective Date**: October 17, 2025  
**Classification**: CONFIDENTIAL - Healthcare Security Policy  
**Review Cycle**: Annually or upon material changes

---

## 1. EXECUTIVE SUMMARY

This Access Control Policy establishes the mechanisms and procedures for managing user authentication, authorization, and accountability within the JibonFlow healthcare platform. The policy aligns with HIPAA Security Rule § 164.312(a)(2) and implements technical safeguards for access management.

### Policy Objectives
- Ensure only authorized personnel access healthcare data
- Maintain audit trails of all data access
- Prevent unauthorized modifications to data
- Enable rapid revocation of access upon termination
- Support HIPAA § 164.312(a)(2)(i) requirements (unique user identification)

---

## 2. AUTHENTICATION FRAMEWORK

### 2.1 Identification & Authentication (HIPAA § 164.312(a)(2)(i))

#### Unique User Identification
- **Requirement**: Every user must have unique, non-reusable identifier
- **Format**: Email address or enterprise ID
- **Never Shared**: Accounts cannot be shared between users
- **Tracking**: All actions traceable to individual user
- **Compliance**: Mandatory for HIPAA compliance

#### Authentication Methods

##### Method 1: Email + Password (Patient Portal)
- Primary for patient access
- Meets minimum security requirements
- Password policy: 12+ characters, complexity required
- Account lockout after 5 failed attempts
- Session timeout: 30 minutes of inactivity

##### Method 2: Email + Password + MFA (Provider/Admin)
- Mandatory for all healthcare professionals
- Mandatory for administrative access
- MFA required upon login every 24 hours
- Session timeout: 15 minutes of inactivity

##### Method 3: Single Sign-On (SSO) via Organization
- Enterprise authentication via SAML/OAuth2
- Organization-managed identity provider
- MFA enforced at identity provider level
- Reduces credential management burden
- Audit trail maintained in both systems

##### Method 4: Temporary Authentication (Emergency)
- Emergency access tokens (1-hour validity)
- Requires emergency access request approval
- Dual approval required (manager + security officer)
- Audit logged with specific reason
- Email notification to account owner

### 2.2 Password Management

#### Password Policy for All Users

| Policy Element | Requirement |
|---|---|
| Minimum Length | 12 characters |
| Complexity | Must include: uppercase, lowercase, number, symbol |
| Maximum Age | 90 days |
| Minimum Age | 1 day (prevent rapid cycling) |
| History | Last 12 passwords remembered, cannot reuse |
| Dictionary Check | No dictionary words or variations |
| Lockout | 5 failed attempts = 30-minute lockout |
| Unlock | Automatic after 30 minutes or admin unlock |
| Reset | Requires identity verification (email link) |
| Transmission | Never via email or chat, only secure reset links |

#### Password Change Process
```
User Initiates Reset
    ↓
Email Verification Link Sent (valid 24 hours)
    ↓
User Clicks Link, Sets New Password
    ↓
Old Session Terminated
    ↓
User Re-authenticates with New Password
    ↓
Audit Log Entry: "Password changed by user"
```

#### Compromised Password Response
1. Immediately expire password
2. Force password reset on next login
3. Terminate all active sessions
4. Notify user of password change
5. Review account access logs for suspicious activity
6. Monitor account for 48 hours

### 2.3 Multi-Factor Authentication (MFA)

#### MFA Requirement Matrix

| User Type | MFA Required | MFA Method |
|---|---|---|
| Patient | Optional | TOTP/Email |
| Pharmacist | Yes | TOTP/Hardware Key |
| Prescriber | Yes | TOTP/Hardware Key |
| Healthcare Administrator | Yes | TOTP/Hardware Key |
| Security Personnel | Yes | Hardware Security Key Only |
| System Administrator | Yes | Hardware Security Key Only |

#### TOTP (Time-Based One-Time Password)
- Algorithm: HMAC-SHA1 (RFC 4226)
- Time window: 30 seconds
- Codes: 6-digit numerical
- Backup codes: 10 generated per enrollment, single-use
- Re-enrollment: Required if device lost

#### Hardware Security Keys
- Standards: FIDO2/WebAuthn
- Devices: YubiKey 5 series or equivalent
- Backup: 2 hardware keys per person (one main, one backup)
- Registration: Performed in secure environment
- Replacement: Immediate issuance upon loss

#### Email OTP (One-Time Passcode)
- Used as backup MFA method only
- 6-digit code, valid for 10 minutes
- Rate limited: 3 attempts per code
- Sent to verified email address
- Recovery: SMS or hardware key

#### MFA Enforcement
- Enforced immediately upon login
- Bypass disabled in all environments
- Session requires MFA re-verification every 24 hours
- MFA removal requires manager approval + security approval

---

## 3. AUTHORIZATION FRAMEWORK

### 3.1 Role-Based Access Control (RBAC)

#### Predefined Roles

##### Patient Role
```
Permissions:
  - View own profile
  - View own prescriptions
  - Submit refill requests
  - View refill status
  - Upload clinical documents
  - Manage account settings
  - View medication history (6 years)

Restrictions:
  - Cannot see other patients
  - Cannot modify prescriptions
  - Cannot access provider notes
  - Cannot export bulk data
```

##### Pharmacist Role
```
Permissions:
  - View assigned prescriptions
  - Approve/deny refill requests
  - Update prescription status
  - Add clinical notes
  - View patient allergy information
  - Generate refill reports
  - Access audit logs (own actions)

Restrictions:
  - Cannot modify patient demographics
  - Cannot delete prescriptions
  - Cannot access non-assigned patients
  - Cannot access provider-only notes
  - Cannot change access controls
```

##### Prescriber Role
```
Permissions:
  - View assigned patients
  - Issue new prescriptions
  - View refill requests
  - Approve/deny refill requests
  - Update clinical information
  - View audit logs (own patients)
  - Generate clinical reports

Restrictions:
  - Cannot access patient financial data
  - Cannot delete audit logs
  - Cannot modify user accounts
  - Cannot access non-assigned patients
```

##### Healthcare Administrator Role
```
Permissions:
  - All patient data (read/write)
  - User account management
  - Organization settings
  - Access control configuration
  - Limited audit log access
  - Report generation
  - Backup management

Restrictions:
  - Cannot access encryption keys
  - Cannot change security policies
  - Cannot access compliance officer logs
  - Cannot modify incident response procedures
```

##### Security Officer Role
```
Permissions:
  - Full audit log access
  - Compliance reporting
  - Security incident investigation
  - Access control audit
  - Encryption key audit (HSM only)
  - Vulnerability assessment results
  - Security policy enforcement

Restrictions:
  - Read-only access (no modifications)
  - Cannot access patient clinical data
  - Cannot modify user accounts
```

##### System Administrator Role
```
Permissions:
  - System configuration
  - Infrastructure management
  - Database administration
  - Encryption key management (HSM)
  - Backup/restore operations
  - Network configuration
  - Incident response execution

Restrictions:
  - Cannot access patient data
  - Cannot modify audit logs
  - Cannot change security policies
  - Cannot approve users for high-privilege roles
```

#### Role Assignment Process
```
Manager Requests Access
    ↓
Security Approval Review (within 24 hours)
    ↓
Role Assigned if Approved
    ↓
MFA Enforcement Activated
    ↓
Audit Log: "User assigned role [X]"
    ↓
User Notified of Role Assignment
```

### 3.2 Attribute-Based Access Control (ABAC)

#### Attributes Evaluated

##### User Attributes
- `employment_status`: active, terminated, leave_of_absence
- `department`: pharmacy, medical, administration, it
- `job_title`: pharmacist, technician, prescriber, admin
- `tenure_level`: junior (< 1 year), intermediate (1-3 years), senior (> 3 years)
- `clearance_level`: basic, intermediate, advanced
- `training_status`: trained (within 12 months), untrained

##### Context Attributes
- `access_time`: 06:00-22:00 (business hours only)
- `access_location`: office_ip, vpn_ip, public_ip (restricted)
- `device_type`: corporate_device, personal_device (mfa required)
- `device_compliance`: compliant, non_compliant
- `network_security`: internal_network, external_network

##### Resource Attributes
- `data_classification`: public, internal, confidential, phi
- `pii_type`: patient_health_data, payment_info, contact_info
- `access_requires`: audit_logging, encryption_verification, mfa
- `geographic_restriction`: us_only, no_restriction

#### ABAC Rules

##### Rule 1: Sensitive PHI Access
```
IF user.role IN [pharmacist, prescriber]
  AND user.training_status == 'trained'
  AND resource.classification == 'phi'
  AND context.access_time IN [06:00-22:00]
  AND context.network_security == 'internal_network'
THEN ALLOW
  WITH audit_logging = true
  AND session_timeout = 15 minutes
ELSE DENY
```

##### Rule 2: Out-of-Hours Emergency Access
```
IF user.role IN [pharmacist, prescriber]
  AND context.access_time NOT IN [06:00-22:00]
  AND resource.classification == 'phi'
THEN REQUIRE
  - Dual approval (manager + security)
  - Additional MFA
  - Reason documented
  - Enhanced audit logging
THEN ALLOW for 1 hour
```

##### Rule 3: Device Compliance Check
```
IF resource.access_requires == 'encryption_verification'
  AND device_compliance == 'non_compliant'
THEN DENY with error message
  "Device must be compliant with security policy"
```

### 3.3 Data-Level Access Control

#### Organization-Scoped Access
- Users can only access data for their organization
- Cross-organization access denied by default
- Requires explicit cross-org permission + audit logging
- Used only for multi-tenant healthcare networks

#### Department-Scoped Access
- Pharmacy staff cannot access prescriber-only notes
- Administrative staff cannot see clinical details
- Finance cannot access medical information
- Exception: Compliance officers (read-only audit)

#### Patient-Scoped Access
- Patients access only their own records
- Providers access only assigned patients
- Pharmacists access only patients with prescriptions
- Staff limited to shift-assigned responsibilities

---

## 4. SESSION MANAGEMENT

### 4.1 Session Initialization
- Created upon successful authentication
- Unique session ID (cryptographic randomness)
- User ID, role, permissions encoded
- MFA status tracked
- Login timestamp and method recorded

### 4.2 Session Security
```
Session Component     Security Requirement
Token Format          JWT (RS256 algorithm)
Token Expiration      15 minutes (admin), 30 minutes (staff), 8 hours (patients)
Refresh Token         7-day validity, single-use
Token Storage         httpOnly cookies (web), EncryptedStorage (mobile)
CSRF Protection       Same-site cookie policy (Strict)
Transport             HTTPS/TLS 1.2+
```

### 4.3 Session Termination
- Automatic: Upon expiration
- Automatic: Upon role change
- Automatic: Upon logout request
- Automatic: Upon access policy violation
- Forced: Upon administrative termination
- Forced: Upon user account suspension

### 4.4 Session Audit Logging
Every session event logged:
- Session creation (user, IP, device, method)
- Token refresh attempts
- Session timeout
- Explicit logout
- Force termination (reason)
- Sensitive operations within session

---

## 5. ACCOUNTABILITY & MONITORING

### 5.1 Audit Logging (HIPAA § 164.312(b))

#### Mandatory Logged Events
- All authentication attempts (success/failure)
- All authorization decisions (allow/deny)
- PHI access (create, read, update, delete)
- Permission changes
- Role assignment/revocation
- User account modifications
- Session creation/termination
- Failed access attempts
- Credential changes

#### Audit Log Contents
```json
{
  "timestamp": "2025-10-17T14:35:22.123Z",
  "event_type": "phi_access",
  "event_action": "read",
  "user_id": "user123",
  "user_role": "pharmacist",
  "resource_type": "prescription",
  "resource_id": "RX-2025-001",
  "patient_id": "PAT-001",
  "result": "success|denied",
  "denial_reason": "[if denied]",
  "ip_address": "192.168.1.100",
  "user_agent": "[browser info]",
  "session_id": "sess-abc123",
  "mfa_used": true,
  "data_fields_accessed": ["medication", "dosage"],
  "data_fields_modified": "[if applicable]"
}
```

#### Audit Log Immutability
- Logs stored in append-only database
- No deletion capability (retention only)
- Cryptographic integrity verification
- Tamper-evident with hash chains
- Replicated to off-site storage

### 5.2 Real-Time Monitoring

#### Anomaly Detection
```
Alert Triggers:
  - Multiple failed login attempts (> 5 in 10 minutes)
  - Access outside business hours
  - Access from new IP address
  - Bulk data export
  - Role-inappropriate access
  - Access to archived/inactive records
  - Simultaneous session from different locations
```

#### Investigation Protocol
1. Alert triggered → Automatic logging enhanced
2. Investigation initiated within 15 minutes
3. User contacted for verification
4. Suspicious session terminated if unauthorized
5. Report generated within 4 hours
6. Follow-up documentation

### 5.3 Periodic Reviews

#### Monthly Access Review
- Verify all active users still require access
- Identify unused accounts (disable after 60 days)
- Check for role mismatches
- Review admin account usage
- Verify MFA enrollment

#### Quarterly Audit Review
- Random sample of 100+ access logs reviewed
- Verify appropriateness of access
- Check for policy violations
- Identify training needs
- Compliance assessment

#### Annual Comprehensive Review
- Full audit trail analysis
- Access trend analysis
- Policy effectiveness evaluation
- Technology platform review
- Penetration testing results

---

## 6. ACCESS TERMINATION

### 6.1 User Offboarding Process

#### Upon Termination Notice
- Schedule access termination meeting (exit interview)
- Document business data in shared locations
- Identify any critical access dependencies
- Create succession plan if needed

#### On Last Day
```
08:00 - Disable account (immediate termination)
       - Revoke all passwords
       - Revoke all MFA devices
       - Disable SSO integration
       - Terminate all active sessions

09:00 - Collect equipment
       - Laptops, phones, USB keys
       - Hardware security keys
       - Facility badges
       - Document collection

10:00 - Conduct exit interview
       - Review confidentiality obligations
       - Explain final paycheck status
       - Explain COBRA/benefits
       - Legal confirmation of non-disclosure

11:00 - Audit access removal
       - Verify account disabled
       - Confirm session termination
       - Check for residual access
       - Generate termination audit report
```

#### Post-Termination (Days 1-30)
- Daily monitoring of account for unauthorized access
- Weekly audit log review for terminated user
- Verify no cached credentials accessible
- Search logs for suspicious activity

### 6.2 Temporary Access Suspension
- Approved by manager and security
- Reason documented
- Duration specified (max 90 days)
- All sessions immediately terminated
- User notified of suspension reason
- Automatic re-enable after duration or manual reactivation

---

## 7. PRIVILEGED ACCESS MANAGEMENT (PAM)

### 7.1 Administrative Access Controls

#### Elevated Privileges
- Defined with specific, limited scope
- Require additional MFA factor
- Audit logging mandatory for all admin actions
- Time-limited sessions (30 minutes maximum)
- Dual control for sensitive operations

#### Admin Action Audit Trail
```
Admin Action                    Dual Control Required    Audit Logging
Database user creation          Yes (2 approvals)        Full log + before/after
Encryption key modification     Yes (2 approvals)        Full log + HSM log
Policy change                   Yes (security officer)   Full change history
User role elevation             Yes (manager)            Full log
System configuration change     Yes (manager)            Full log + rollback plan
```

### 7.2 Just-In-Time Access (JIT)

#### Request-Based Access
```
Admin Requests Elevated Access
    ↓
Request Logged (reason, duration, scope)
    ↓
Automatic Approval if
  - Pre-approved request category
  - Within normal hours
  - Reasonable duration (< 4 hours)
ELSE Manual Approval by Security
    ↓
Access Provisioned (time-limited)
    ↓
Session Monitoring (real-time alerts)
    ↓
Access Auto-revoked at expiration
    ↓
Session Recording Reviewed
```

---

## 8. POLICY ENFORCEMENT

### 8.1 Monitoring and Compliance
- Automated access control testing (monthly)
- Penetration testing of authentication (quarterly)
- Compliance audit (annual independent)
- Policy adherence spot-checks
- Incident investigations

### 8.2 Violations and Discipline
- Minor violations: Written warning + retraining
- Major violations: Suspension + investigation
- Severe violations (intentional breach): Termination + legal action
- Unauthorized access: Report to legal and law enforcement

### 8.3 Continuous Improvement
- Quarterly policy review
- Technology updates evaluated
- Industry best practices incorporated
- Incident lessons learned applied
- User feedback solicited

---

## 9. CONTACT AND APPROVAL

### Document Owner
- **Title**: Director of Security & Compliance
- **Email**: compliance@jibonflow.com

### Approvals Required
- **Chief Information Security Officer**: [Signature/Approval]
- **Chief Medical Officer**: [Signature/Approval]
- **General Counsel**: [Signature/Approval]

### Last Updated
October 17, 2025

### Next Review Date
October 17, 2026

---

## 10. APPENDICES

### Appendix A: Role Modification Request Form
[See separate document]

### Appendix B: Emergency Access Request Procedure
[See separate document]

### Appendix C: Account Recovery Procedures
[See separate document]

### Appendix D: SAML/SSO Integration Specifications
[See separate document]

---

**Generated by Security Remediation Agent v1.0**  
**Phase 2B: Security Remediation & Compliance Hardening**  
**Generated**: 2025-10-17
