# Key Management Policy
## JibonFlow Healthcare Platform

**Document Version**: 1.0  
**Effective Date**: October 17, 2025  
**Classification**: CONFIDENTIAL - Healthcare Security Policy  
**Review Cycle**: Annually or upon material changes

---

## 1. EXECUTIVE SUMMARY

This Key Management Policy establishes procedures for generating, protecting, rotating, and retiring cryptographic keys used to encrypt PHI and sensitive data in the JibonFlow platform. The policy implements HIPAA Security Rule § 164.312(a)(2)(iv) (Encryption and Decryption) and NIST cryptographic standards.

### Key Objectives
- Ensure strong, cryptographically sound key generation
- Protect keys from unauthorized access
- Enable secure key rotation
- Maintain key integrity
- Support regulatory compliance

---

## 2. KEY CLASSIFICATION & TYPES

### 2.1 Key Classification

#### Master Keys (Level 1)
- **Purpose**: Encrypt/decrypt other keys (key encryption keys)
- **Classification**: Critical
- **Storage**: Hardware Security Module (HSM) only
- **Access**: System administrators only
- **Rotation**: Every 90 days or upon suspected compromise

#### Data Encryption Keys (Level 2)
- **Purpose**: Encrypt/decrypt PHI and sensitive data
- **Classification**: Critical
- **Storage**: AWS Secrets Manager or encrypted in application
- **Access**: Service account or specific applications
- **Rotation**: Every 90 days or upon data sensitivity change

#### Session Keys (Level 3)
- **Purpose**: Temporary keys for individual sessions
- **Classification**: High
- **Storage**: Memory during session only
- **Access**: Single session only
- **Rotation**: Per session (new key for each session)

#### API Keys (Level 4)
- **Purpose**: Authenticate API requests
- **Classification**: High
- **Storage**: AWS Secrets Manager
- **Access**: Specific services only
- **Rotation**: Every 60 days or if exposed

### 2.2 Key Types by Algorithm

#### Symmetric Encryption Keys
- **Algorithm**: AES-256-GCM
- **Size**: 256 bits (32 bytes)
- **Purpose**: Fast encryption of large data volumes
- **Use Case**: Database encryption, file encryption, at-rest encryption

#### Asymmetric Encryption Keys (RSA)
- **Algorithm**: RSA with 2048-bit minimum (4096-bit preferred)
- **Purpose**: Secure key exchange, digital signatures
- **Public Key**: Can be shared
- **Private Key**: Protected in HSM
- **Use Case**: TLS certificates, API authentication

#### HMAC Keys
- **Algorithm**: HMAC-SHA256
- **Size**: 256 bits minimum
- **Purpose**: Data integrity verification
- **Use Case**: Message authentication codes, tamper detection

#### Session Keys (DTLS-SRTP)
- **Algorithm**: AES-128-GCM for SRTP
- **Size**: 128 bits
- **Purpose**: Real-time communication encryption
- **Use Case**: VoIP/telemedicine E2EE sessions

---

## 3. KEY GENERATION

### 3.1 Generation Requirements

#### Standards
- **NIST Compliance**: All keys must meet NIST SP 800-133 guidelines
- **Randomness**: Cryptographically secure random number generator (CSPRNG)
- **Entropy**: Minimum 256 bits of entropy
- **Generator**: Hardware-based RNG preferred (HSM)
- **Verification**: Generated keys validated for entropy

#### Generation Environment
- **Location**: Secure, isolated environment
- **Network**: Air-gapped during initial generation
- **Authorization**: Dual approval required
- **Documentation**: Generation logged with timestamp
- **Verification**: Independent verification of output

#### Generation Procedures

##### Master Key Generation
```
Procedure: Generate new master key in HSM

Step 1: Request Authorization
  - Security officer initiates key generation request
  - Manager approves (documented)
  - Reason and purpose documented

Step 2: Prepare HSM
  - Verify HSM operational and healthy
  - Clear HSM session state
  - Prepare backup procedures

Step 3: Generate Key
  - HSM generates 256-bit random key
  - Key never exported from HSM
  - Cryptographic hash generated
  - Generation logged with timestamp

Step 4: Verification
  - Independent verification performed
  - Hash compared to records
  - Key properties verified
  - Backup taken in HSM

Step 5: Documentation
  - Certificate generated
  - Key attributes recorded
  - Purpose documented
  - Expiration date set
  - Backup verified
```

##### Data Encryption Key Generation
```
Procedure: Generate new data encryption key

Step 1: Trigger Event
  - Periodic rotation (90 days)
  - New data category
  - Key compromise suspected
  - Regulatory requirement

Step 2: Generate Key
  - AWS KMS or HSM invoked
  - 256-bit AES key generated
  - Wrapped with master key
  - Hash generated

Step 3: Deploy Key
  - Key stored in Secrets Manager
  - Version tracking enabled
  - Alias created
  - Access permissions set

Step 4: Activate Key
  - New key marked as active
  - Old key marked as legacy
  - System updated to use new key
  - Audit logged

Step 5: Archive Old Key
  - Old key stored in key archive
  - Retention metadata set (7 years)
  - Access restricted
  - No further use permitted
```

##### Session Key Generation
```
Procedure: Generate session-specific key

Step 1: Session Initiated
  - User logs in or session starts
  - Session ID generated
  - Context stored (user, permissions)

Step 2: Session Key Generated
  - CSPRNG generates 128-bit key (SRTP) or 256-bit (general)
  - Key associated with session ID
  - Never logged or stored persistently
  - Expiration set (session lifetime)

Step 3: Session Key Provisioning
  - Key sent to client via TLS channel
  - Client uses key for session
  - Server maintains key only in memory
  - Key expires with session

Step 4: Session Key Cleanup
  - Upon logout: Key destroyed
  - Upon expiration: Key securely cleared
  - No key recovery possible
  - Session entry removed
```

---

## 4. KEY PROTECTION

### 4.1 Master Key Protection

#### Hardware Security Module (HSM)
- **Requirement**: Masters keys MUST be stored in FIPS 140-2 Level 3 HSM
- **Provider**: AWS CloudHSM or equivalent
- **Backup**: Replicated across geographic regions
- **Access**: Network access only via secure connection
- **Logging**: All access logged and monitored

#### HSM Configuration
```yaml
HSM Cluster:
  - Primary Region: us-east-1 (Virginia)
  - Backup Region: us-west-2 (Oregon)
  - Replication: Real-time across regions
  - Failover: Automatic upon primary failure
  - Capacity: Minimum 2 HSM appliances per region

Network Security:
  - VPC isolation: Private subnets only
  - Security Groups: Restricted ports
  - Network ACLs: Whitelist-based access
  - Monitoring: All access logged
  - Encryption: All traffic TLS 1.2 minimum

Access Control:
  - IAM roles: Limited to service accounts
  - MFA: Required for manual HSM access
  - Approval: Dual control for sensitive ops
  - Audit logging: All operations logged
```

#### HSM Backup & Disaster Recovery
```
Backup Schedule:
  - Frequency: Daily snapshots
  - Retention: 30 days
  - Off-site: Replicated to backup region
  - Testing: Monthly restore test
  - Verification: Cryptographic hash verification

Disaster Recovery:
  - RTO (Recovery Time Objective): 1 hour
  - RPO (Recovery Point Objective): 1 hour
  - Backup activation: Automated failover
  - Testing: Quarterly full recovery drill
  - Documentation: Recovery procedures current
```

### 4.2 Data Encryption Key Protection

#### Storage Methods

##### AWS Secrets Manager
- **Usage**: Most data encryption keys
- **Encryption**: Keys encrypted at rest with master key
- **Access**: IAM-based access control
- **Rotation**: Automatic rotation enabled
- **Retention**: Keys retained per policy

**Configuration**:
```json
{
  "Name": "phi-encryption-key-prod",
  "Description": "Primary encryption key for patient data",
  "KmsKeyId": "arn:aws:kms:us-east-1:xxx:key/xxx",
  "RotationEnabled": true,
  "RotationRules": {
    "AutomaticallyAfterDays": 90
  },
  "Tags": [
    {"Key": "Environment", "Value": "production"},
    {"Key": "Classification", "Value": "critical"},
    {"Key": "Compliance", "Value": "HIPAA"}
  ]
}
```

##### Encrypted Local Storage
- **Location**: Application configuration
- **Encryption**: Encrypted with master key
- **Access**: Application process only
- **Decryption**: On-demand at runtime
- **Audit**: Every use logged

##### Database Encryption
- **Method**: Transparent data encryption (TDE)
- **Key Storage**: AWS KMS
- **Scope**: All tables with PHI
- **Verification**: Encryption status monitored
- **Rotation**: Automated per KMS policy

### 4.3 Application-Level Key Protection

#### In-Memory Protection
```typescript
// Key loaded into memory - use secure practices
import * as crypto from 'crypto';

// GOOD: Use secure buffers
const keyBuffer = Buffer.from(keyString, 'base64');
const cipher = crypto.createCipheriv('aes-256-gcm', keyBuffer, iv);

// GOOD: Clear key from memory after use
crypto.randomFillSync(keyBuffer);
keyBuffer.fill(0);

// BAD: Keep key as plain string
const key = keyString; // High risk
```

#### Key Zeroization
- **Process**: After use, keys cleared from memory with zeros
- **Implementation**: crypto libraries' built-in methods
- **Timing**: Immediate after cryptographic operation
- **Verification**: Random verification of cleared memory

### 4.4 Key Access Control

#### Access Policies

| Role | Master Key | Data Key | Session Key |
|---|---|---|---|
| Application | View only (for encryption) | Yes | Yes |
| System Admin | Yes (with MFA) | Emergency only | No |
| DBA | No | Emergency only | No |
| Security Officer | Audit only | Audit only | Audit only |
| CISO | Emergency only | Emergency only | No |

#### Principle of Least Privilege
- Each service gets only keys it needs
- Different keys for different purposes
- Separate keys for different environments (prod, staging, dev)
- Key access revoked immediately when no longer needed

---

## 5. KEY ROTATION

### 5.1 Rotation Schedule

#### Automatic Rotation Schedule
```
Key Type                    Rotation Interval   Trigger
Master Key                  Every 90 days       Time-based
Data Encryption Key         Every 90 days       Time-based
Session Key                 Per session         Event-based
API Key                     Every 60 days       Time-based or compromised
Database Encryption Key     Every 90 days       Time-based
TLS Certificate Key         Every 1 year        Time-based
Backup Encryption Key       Every 180 days      Time-based
```

#### Manual Rotation Triggers
- Key compromise suspected
- Unauthorized access detected
- Employee with key access leaves
- Change in key usage requirements
- Regulatory requirement
- Security vulnerability discovered

### 5.2 Rotation Procedures

#### Zero-Downtime Key Rotation
```
Phase 1: Generate New Key
  - New key generated in HSM/KMS
  - Dual verification performed
  - Key wrapped with master key
  - Versioning enabled

Phase 2: Parallel Operations
  - New key marked as "active"
  - Old key marked as "legacy"
  - System accepts both for decryption
  - Encryption uses new key only
  - No service disruption

Phase 3: Data Re-encryption
  - Background process starts
  - Scans all PHI encrypted with old key
  - Re-encrypts with new key
  - Verification checksum compared
  - Audit logged

Phase 4: Migration Complete
  - Re-encryption 100% complete
  - Old key removed from active service
  - Old key archived
  - Monitoring for 7 days
  - Success reported

Phase 5: Cleanup
  - Old key retained in archive (7 years)
  - Access restricted to audit only
  - Metadata recorded
  - Rotation cycle documented
```

### 5.3 Rotation Monitoring

#### Metrics Tracked
```
Metric                      Target
Rotation Completion Time    < 4 hours
Data Re-encryption Time     < 24 hours
Verification Success Rate   100%
Audit Entry Completion      100%
Zero Data Loss              100%
Zero Service Disruption     100%
Rollback Capability         Verified
```

---

## 6. KEY COMPROMISE & EMERGENCY RESPONSE

### 6.1 Compromise Detection

#### Detection Indicators
- Unauthorized key access attempt
- Key exported from HSM (not permitted)
- Key used outside normal parameters
- Suspicious activity on key-using system
- Audit log inconsistencies
- Backup key corruption

#### Verification Process
```
Suspected Compromise
    ↓
Verify Integrity (cryptographic check)
    ↓
IF Compromise Confirmed
    ↓
IMMEDIATE ACTIONS:
  - Stop all encryption with compromised key
  - Revoke key from all systems
  - Terminate active sessions
  - Notify stakeholders
  
INVESTIGATION:
  - Determine exposure scope
  - Identify affected data
  - Review access logs
  - Determine impact duration
  
REMEDIATION:
  - Generate emergency key
  - Re-encrypt all data
  - Restore from backup if necessary
  - Implement additional controls
  
NOTIFICATION:
  - Legal/compliance notified
  - Affected parties identified
  - Breach notification if applicable
```

### 6.2 Emergency Access Procedures

#### Emergency Key Access Protocol
```
Condition: Master key emergency access needed

Step 1: Emergency Authorization
  - Two executives required to approve:
    * CISO
    * CEO or General Counsel
  - Reason documented (e.g., system recovery)
  - Approval recorded in audit log

Step 2: Access Preparation
  - Isolated secure room prepared
  - Two security officers assigned
  - Dual control verification enabled
  - Recording enabled (audio/video)

Step 3: Access Execution
  - First officer logs in (MFA required)
  - Second officer enables dual control
  - Both enter separate credentials
  - Key operation executed
  - All actions logged

Step 4: Access Conclusion
  - Both officers verify completion
  - HSM session terminated
  - Audit report generated
  - Recording preserved
  - Immediate executive notification
```

### 6.3 Post-Compromise Actions

#### Data Exposure Assessment
- Determine data encryption timeline
- Identify compromised key duration
- Estimate records potentially exposed
- Assess attack sophistication
- Determine likely attacker

#### Re-encryption Campaign
```
Phase 1: Generate new master key (immediate)
Phase 2: Generate new data keys (within 1 hour)
Phase 3: Prepare re-encryption process (within 4 hours)
Phase 4: Execute re-encryption (within 24 hours)
Phase 5: Verification (within 48 hours)
Phase 6: Archive compromised key (within 72 hours)
```

#### Legal & Regulatory Notification
- Internal breach notification (within 24 hours)
- Legal review of obligations (within 48 hours)
- Affected party notification (within 60 days)
- Regulatory authority notification (within 60 days)
- Public disclosure if required (within 60 days)

---

## 7. KEY LIFECYCLE TRACKING

### 7.1 Key Inventory

#### Master Key Inventory
```json
{
  "KeyId": "key-master-001",
  "Name": "Production Master Key",
  "Algorithm": "AES-256-GCM",
  "GeneratedDate": "2025-10-17",
  "ExpirationDate": "2026-01-15",
  "Status": "active",
  "Storage": "AWS CloudHSM",
  "Owner": "System Owner",
  "Purpose": "Encrypts all data encryption keys",
  "RotationSchedule": "Every 90 days",
  "BackupStatus": "Verified",
  "AuditLog": ["approval", "generation", "backup"],
  "Version": 1
}
```

#### Data Key Inventory
```json
{
  "KeyId": "key-data-phi-001",
  "Name": "PHI Encryption Key (Prod)",
  "Algorithm": "AES-256-GCM",
  "GeneratedDate": "2025-10-17",
  "ExpirationDate": "2026-01-15",
  "Status": "active",
  "Storage": "AWS Secrets Manager",
  "EncryptedWith": "key-master-001",
  "Owner": "Database Team",
  "Purpose": "Encrypt patient health records in database",
  "RotationSchedule": "Every 90 days",
  "LastRotated": "2025-10-17",
  "NextRotation": "2026-01-15",
  "Version": 1
}
```

### 7.2 Key Metadata Tracking

#### Mandatory Metadata
- Key ID (unique identifier)
- Key name (descriptive)
- Algorithm and parameters
- Key size (bits)
- Generation date and time
- Expiration date
- Status (active, legacy, retired)
- Owner and custodian
- Purpose and usage
- Storage location
- Protection method
- Rotation schedule
- Audit trail

### 7.3 Key Audit & Verification

#### Monthly Key Verification
```
Verification Task              Check
Master key accessibility      Can HSM create new keys? ✓
Data key integrity            Cryptographic hash match? ✓
Rotation status               Rotation scheduled/executed? ✓
Access logs review            Unauthorized access? ✓
Backup verification           Backups current and restorable? ✓
Compliance status             Policy compliant? ✓
```

#### Quarterly Key Audit
- Full key inventory audit
- Unauthorized keys discovered?
- Orphaned keys identified?
- Access control verification
- Rotation schedule compliance
- Backup status verification

#### Annual Key Lifecycle Review
- Compliance certification
- Technology updates evaluated
- Policy effectiveness assessed
- Incidents reviewed
- Lessons learned incorporated
- Forward-looking improvements planned

---

## 8. KEY DESTRUCTION

### 8.1 End-of-Life Key Destruction

#### Retirement Process
```
Step 1: Deprecation Notice
  - Key marked for retirement
  - 30-day notice period
  - New key generated and activated
  - System transitioned to new key

Step 2: Re-encryption Campaign
  - All data encrypted with old key re-encrypted
  - Verification complete
  - No data uses old key

Step 3: Key Destruction Approval
  - CISO approves destruction
  - Legal confirms retention period complete
  - Final audit log review
  - Destruction authorized

Step 4: Key Destruction
  - HSM key destroyed permanently
  - Backup copy destroyed
  - Destruction logged
  - Certificate of destruction generated
  - Audit trail preserved

Step 5: Documentation
  - Key destruction documented
  - Timeline recorded
  - Approver identified
  - Reason documented
  - Archive updated
```

#### Retention Before Destruction
```
Key Type                    Retention Period    Reason
Data Encryption Key         7 years             Audit/forensics
Master Key                  10 years            Regulatory
Session Key                 0 years             Destroyed immediately
API Key (after rotation)    2 years             Audit trail
Compromised Key             10 years            Legal evidence
```

### 8.2 Physical Key Destruction

#### For Hardware Stored Keys
- Secure transport to destruction facility
- Witnessed destruction
- Destruction certificate
- Photographic evidence
- Audit documented
- Certificate archived (7 years)

---

## 9. COMPLIANCE & AUDITING

### 9.1 Regulatory Compliance

#### HIPAA § 164.312(a)(2)(iv)
- ✓ Encryption and decryption standards met
- ✓ Strong algorithms used (AES-256-GCM)
- ✓ Key management procedures documented
- ✓ Access controls implemented
- ✓ Audit trails maintained

#### NIST SP 800-133 Compliance
- ✓ Key generation entropy verified
- ✓ Key storage protection implemented
- ✓ Key rotation schedule established
- ✓ Key destruction procedures defined

### 9.2 Audit Testing

#### Monthly Security Tests
- Key generation verification
- Rotation completion verification
- Access control testing
- Encryption/decryption verification
- Audit log completeness

#### Quarterly Compliance Tests
- Full access control audit
- Key inventory reconciliation
- Backup restoration test
- Disaster recovery test
- Policy compliance verification

#### Annual Independent Audit
- Third-party key management audit
- Compliance certification
- Recommendations for improvement
- Penetration testing of key systems
- Policy effectiveness review

---

## 10. CONTACT & APPROVAL

### Document Owner
- **Title**: Chief Information Security Officer
- **Email**: ciso@jibonflow.com

### Key Custodians
| Role | Name | Contact |
|---|---|---|
| HSM Administrator | [Name] | [Email/Phone] |
| Secrets Manager Admin | [Name] | [Email/Phone] |
| Database Admin | [Name] | [Email/Phone] |
| Security Officer | [Name] | [Email/Phone] |

### Approvals Required
- **Chief Information Officer**: [Approval]
- **Chief Information Security Officer**: [Approval]
- **General Counsel**: [Approval]

### Last Updated
October 17, 2025

### Next Review Date
October 17, 2026

---

## 11. APPENDICES

### Appendix A: HSM Configuration Guide
[See separate technical documentation]

### Appendix B: Key Rotation Playbook
[See separate operational procedures]

### Appendix C: Emergency Access Request Form
[See separate form document]

### Appendix D: Key Backup and Recovery Procedures
[See separate technical documentation]

---

**Generated by Security Remediation Agent v1.0**  
**Phase 2B: Security Remediation & Compliance Hardening**  
**Generated**: 2025-10-17
