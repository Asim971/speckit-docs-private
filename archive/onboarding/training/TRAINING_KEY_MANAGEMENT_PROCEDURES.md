# Security Training: Key Management Procedures
## Protecting Cryptographic Keys in JibonFlow

**Duration**: 45 minutes  
**Audience**: System Operators, DevOps Engineers, Database Administrators  
**Level**: Intermediate - Advanced  
**Completion Certificate**: Yes  
**Assessment**: 8-question scenario-based quiz (75% pass required)  

---

## 📋 Table of Contents

1. [Learning Objectives](#learning-objectives)
2. [Why Key Management Matters](#why-key-management-matters)
3. [Key Lifecycle Overview](#key-lifecycle-overview)
4. [Key Generation Process](#key-generation-process)
5. [Key Storage & Protection](#key-storage--protection)
6. [Key Access Control](#key-access-control)
7. [Key Rotation Procedures](#key-rotation-procedures)
8. [Emergency Access Procedures](#emergency-access-procedures)
9. [Key Compromise Response](#key-compromise-response)
10. [Audit & Compliance](#audit--compliance)
11. [Real-World Scenarios](#real-world-scenarios)
12. [Knowledge Assessment](#knowledge-assessment)

---

## 1. Learning Objectives

By completing this training, you will be able to:

- ✅ Understand the full lifecycle of cryptographic keys in JibonFlow
- ✅ Follow proper key generation procedures aligned with NIST standards
- ✅ Implement secure key storage using HSM and AWS Secrets Manager
- ✅ Execute zero-downtime key rotation procedures
- ✅ Respond appropriately to suspected key compromise
- ✅ Access emergency recovery procedures
- ✅ Monitor and audit key usage for compliance
- ✅ Apply key management best practices in daily operations

---

## 2. Why Key Management Matters

### The Critical Path

Cryptographic keys are the foundation of data protection in JibonFlow:

```
Patient Data → Encrypted with Keys → Protected PHI → Secure Storage
   ↓
If keys are compromised → ALL encrypted data becomes readable
   ↓
HIPAA breach → Notifications → Legal liability → Organization risk
```

### Statistics

- **40%** of healthcare data breaches involve compromised credentials/keys
- **Mean Time to Detect**: 287 days (healthcare industry average)
- **Cost per Breach Record**: $429 (2024 data)
- **HIPAA Penalty**: Up to $1.5M per violation

### Your Role

As a key operator, you control:
- ✅ When keys are generated
- ✅ Where keys are stored
- ✅ Who can access keys
- ✅ How keys are rotated
- ✅ What happens in emergencies

**Your actions directly impact the security of 100,000+ patient records.**

---

## 3. Key Lifecycle Overview

### The Complete Journey

```
┌─────────────────────────────────────────────────────────────┐
│                    KEY LIFECYCLE PHASES                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. GENERATION        2. STORAGE         3. USE            │
│  ↓                     ↓                  ↓                 │
│  NIST SP 800-133      HSM + Secrets      Application       │
│  Approved RNG         Manager            Encryption        │
│                                                             │
│  4. ROTATION          5. COMPROMISE      6. RETIREMENT    │
│  ↓                     ↓                  ↓                │
│  90-day or event      Detection          Destruction       │
│  Zero-downtime        Immediate response 7-year archive   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Phase Transitions

| Phase | Timeline | Responsibility | Approval |
|-------|----------|-----------------|----------|
| **Generation** | <1 hour | Security team | CISO |
| **Storage** | <30 min | DevOps | Security team |
| **Use** | Ongoing | Applications | Audit logs |
| **Rotation** | Scheduled 90 days | DevOps + Sec ops | Change management |
| **Compromise** | Immediate | CISO + DevOps | Executive team |
| **Retirement** | After rotation | DevOps | Audit signed-off |

---

## 4. Key Generation Process

### When Keys Are Generated

1. **Initial Application Setup** (one-time)
   - Master key for encryption at rest
   - Session keys for telemedicine E2EE

2. **Scheduled Rotation** (every 90 days)
   - New data encryption key
   - Old key retained for decryption

3. **Emergency Situations** (unscheduled)
   - Suspected compromise
   - Vendor breach affecting key material
   - Cryptographic algorithm deprecated

### Generation Procedure (Step-by-Step)

**Prerequisites**:
- 2 authorized operators present
- Hardware Security Module (HSM) accessible
- NIST SP 800-133 cryptographic RNG available
- Audit logging enabled

**Steps**:

```bash
# Step 1: Verify HSM is operational
aws kms describe-key --key-id jibonflow-master-key

# Step 2: Request key generation with dual approval
# Initiator creates CSR (Certificate Signing Request)
openssl genrsa -out master-key.pem 2048

# Step 3: Approve request (separate operator)
# Must be different person with "key-management:approve" role
# Documented in audit log automatically

# Step 4: Generate key material in HSM
# - Uses FIPS 140-2 Level 3 HSM
# - Cryptographic RNG (entropy source verified)
# - Key escrow: 50% threshold secret sharing

# Step 5: Import into AWS KMS
aws kms import-key-material \
  --key-id jibonflow-master-key \
  --import-token <token> \
  --encrypted-key-material <material>

# Step 6: Verify generation in logs
aws logs filter-log-events \
  --log-group-name /aws/kms/jibonflow \
  --filter-pattern "GenerateKey SUCCESS"

# Step 7: Store key fingerprint in audit database
# Fingerprint: SHA256(key material) - never store full key
```

### Generation Checklist

- [ ] HSM operational and time-synchronized
- [ ] Both operators present and identified
- [ ] Entropy source verified (>128 bits collected)
- [ ] Key generation parameters logged
- [ ] Approval documented with signatures
- [ ] Key fingerprint stored in audit database
- [ ] Notification sent to CISO
- [ ] Retention policy applied (7-year archive)

---

## 5. Key Storage & Protection

### Storage Hierarchy

```
PROTECTION LEVEL 1 (Highest Security)
    ↓
Master Key → Stored in FIPS 140-2 Level 3 HSM
             - Never leaves hardware unencrypted
             - Access via specialized commands only
             - Physical security: Data center locks + surveillance
             - Availability: 99.9% SLA
    ↓
PROTECTION LEVEL 2 (High Security)
    ↓
Data Keys → Stored in AWS Secrets Manager
            - Encrypted at rest with master key
            - Encrypted in transit (TLS 1.2+)
            - Access via IAM roles only
            - Audit log: every access recorded
    ↓
PROTECTION LEVEL 3 (Application Level)
    ↓
Session Keys → Stored in application memory
               - Cleared after session ends
               - Never written to disk
               - Rotated per connection (TLS)
    ↓
```

### Storage Locations

**DO STORE HERE** ✅:
- AWS KMS (master keys)
- AWS Secrets Manager (data keys, encrypted)
- Environment variables (only in deployment pipelines)
- HashiCorp Vault (if using on-premises)

**NEVER STORE HERE** ❌:
- Source code repositories
- Configuration files (unencrypted)
- Logs or monitoring output
- Email or chat systems
- Personal devices or USB drives
- Public cloud storage (S3 without encryption)
- Git commit history

### Storage Verification

```typescript
// VULNERABLE - Never do this
const masterKey = "hfH2x9kL3pQ4vW..."; // DON'T store in code
fs.writeFileSync('.env', `KEY=${masterKey}`); // DON'T store in files

// SECURE - Do this instead
const masterKey = await secretsManager.getSecret('jibonflow/master-key');
// ✅ Encrypted in transit (TLS)
// ✅ Retrieved at runtime only
// ✅ Never persisted to disk
// ✅ Access logged automatically
```

---

## 6. Key Access Control

### Access Levels

| Role | Generate | Store | Access | Rotate | Destroy | Audit |
|------|----------|-------|--------|--------|---------|-------|
| **CISO** | ✅ Approve | ✅ Monitor | Limited | ✅ Approve | ✅ Approve | ✅ Full |
| **DevOps** | ✅ Execute | ✅ Manage | Limited | ✅ Execute | Limited | ✅ Full |
| **Operators** | — | Limited | Yes | Limited | — | Limited |
| **Developers** | — | — | Via SDK | — | — | — |
| **Auditors** | — | — | — | — | — | ✅ Full |

### Access Control Implementation

**IAM Policy Example**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowKeyOperations",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/devops-key-manager"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:GenerateDataKey",
        "kms:CreateGrant"
      ],
      "Resource": "arn:aws:kms:us-east-1:ACCOUNT:key/*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        },
        "IpAddress": {
          "aws:SourceIp": ["10.0.0.0/8"]  // Internal network only
        }
      }
    }
  ]
}
```

### Verification Commands

```bash
# Check who accessed keys in last 24 hours
aws logs filter-log-events \
  --log-group-name /aws/kms/jibonflow \
  --start-time $(date -d '24 hours ago' +%s)000 \
  --filter-pattern "DescribeKey OR Decrypt OR GenerateDataKey"

# Verify access restrictions are in place
aws kms get-key-policy \
  --key-id jibonflow-master-key \
  --policy-name default

# Check IAM role permissions
aws iam get-role-policy \
  --role-name devops-key-manager \
  --policy-name kms-key-management
```

---

## 7. Key Rotation Procedures

### Rotation Frequency

- **Scheduled Rotation**: Every 90 days
- **Event-based Rotation**: Upon detection of compromise
- **Vendor-triggered Rotation**: When vendor is breached
- **Algorithm Deprecation**: When cryptographic standards change

### Zero-Downtime Rotation Process

**Phase 1: Preparation (30 minutes before)**

```bash
# Step 1: Create new key
NEW_KEY_ID=$(aws kms create-key --description "JibonFlow rotation $(date)" --query 'KeyMetadata.KeyId' --output text)

# Step 2: Test new key with non-critical data
TEST_DATA="test-encryption-verification"
ENCRYPTED=$(aws kms encrypt --key-id $NEW_KEY_ID --plaintext $TEST_DATA --query CiphertextBlob --output text)
DECRYPTED=$(aws kms decrypt --ciphertext-blob $ENCRYPTED --query Plaintext --output text | base64 -d)

# Verify
[ "$DECRYPTED" = "$TEST_DATA" ] && echo "✅ Key verification successful"

# Step 3: Update configuration (not deployed yet)
echo $NEW_KEY_ID > /tmp/new-key-id.txt
```

**Phase 2: Deployment (synchronized)**

```bash
# Step 1: Update application configuration
# - All instances get new key ID
# - Still using old key for decryption
# - Transparent to users

# Step 2: Switch to dual-encryption mode
# For new data:   Use new key (encrypt)
# For old data:   Use old key (decrypt)
# Gradual migration as data is re-encrypted

# Step 3: Monitor transition
# Watch error logs for decryption failures
# Verify no user impact

# Step 4: Confirm health metrics
# - Response times normal
# - Error rate unchanged
# - All services healthy
```

**Phase 3: Finalization (24 hours later)**

```bash
# Step 1: Archive old key (keep for compliance)
aws kms disable-key --key-id old-key-id

# Step 2: Verify all data successfully rotated
# Query database: Count records encrypted with old key
SELECT COUNT(*) FROM encrypted_data WHERE key_version = 'old';

# If count = 0, all data rotated successfully
# If count > 0, force re-encryption for remaining records

# Step 3: Generate audit report
# - Rotation completion timestamp
# - Records affected: N
# - Decryption errors: 0
# - User impact: None

# Step 4: Sign-off by CISO
```

### Rotation Checklist

- [ ] Scheduled rotation window identified
- [ ] New key tested with test data
- [ ] Configuration updated (not deployed)
- [ ] Old key backed up
- [ ] Deployment window communicated
- [ ] Health monitoring enabled
- [ ] Gradual migration monitored
- [ ] All data verified re-encrypted
- [ ] Old key archived
- [ ] Audit report generated
- [ ] CISO sign-off obtained
- [ ] Compliance documentation updated

---

## 8. Emergency Access Procedures

### When to Use Emergency Access

**DO use emergency access for**:
- Production incident requiring immediate key access
- Data recovery after system failure
- Compliance investigation requiring decryption
- Disaster recovery procedures

**DO NOT use emergency access for**:
- Regular operational tasks (use standard procedures)
- Circumventing approval workflows
- Personal curiosity or testing

### Emergency Access Process

**Step 1: Trigger**

```bash
# Request emergency access (anyone can initiate)
aws secretsmanager request-secret-version \
  --secret-id jibonflow/emergency-key \
  --reason "EMERGENCY: System down, patient data inaccessible" \
  --tags "[{\"Key\": \"incident-id\", \"Value\": \"INC-2025-001\"}]"

# Automated: Creates support ticket + alerts CISO
# Timeline: CISO notified within 2 minutes
```

**Step 2: Dual Authorization (Required)**

```
Two authorized personnel must approve:

Approver 1: CISO or delegate
├─ Verifies incident necessity
├─ Confirms identity of requester
└─ Signs approval with timestamp

Approver 2: Security Officer
├─ Reviews reason provided
├─ Confirms CISO approval
└─ Signs approval with timestamp

Both approvals required → Access granted
```

**Step 3: Access Grant**

```bash
# After dual authorization
aws secretsmanager get-secret-value \
  --secret-id jibonflow/emergency-key \
  --version-id emergency-version

# Automatic actions:
# ✅ Session logging enabled
# ✅ Screen recording (if terminal session)
# ✅ Notification to security team
# ✅ Timer started: 30-minute session max
```

**Step 4: Activity Monitoring**

```bash
# Every action logged in real-time
# - Command executed
# - Data accessed
# - Timestamp: precise to millisecond
# - User identity: confirmed via MFA

# Real-time alerts if:
# - Unusual commands detected
# - Accessing data outside incident scope
# - Exceeding 30-minute limit
```

**Step 5: Session Termination**

```bash
# Session automatically ends after 30 minutes
# Manual termination: Any approval authority can revoke

# Post-session actions:
# 1. Session recording stored in secure vault
# 2. Incident timeline created
# 3. Audit summary generated
# 4. CISO review required
# 5. Access revoked immediately
```

### Emergency Access Checklist

- [ ] Genuine emergency confirmed (patient safety impact)
- [ ] CISO approval obtained within 5 minutes
- [ ] Security Officer approval obtained
- [ ] Emergency reason documented with evidence
- [ ] Incident ticket created and linked
- [ ] Session limit acknowledged (30 min max)
- [ ] Actions during session monitored
- [ ] Session recording saved
- [ ] Post-incident review scheduled
- [ ] Findings documented

---

## 9. Key Compromise Response

### Detection: Signs of Compromise

**System Alerts**:
- ⚠️ Unusual key access patterns (>2x normal rate)
- ⚠️ Access from unexpected IP addresses
- ⚠️ Multiple failed decryption attempts
- ⚠️ HSM tamper detection alerts

**Manual Detection**:
- Unauthorized access to KMS console
- Unexplained increase in AWS KMS costs
- Keys missing from inventory
- Audit log gaps

### Immediate Actions (0-15 minutes)

```bash
# STEP 1: DECLARE EMERGENCY
# Call incident commander immediately
# Page CISO + Security team
echo "KEY COMPROMISE SUSPECTED - INITIATING RESPONSE" | mail -s "🚨 SECURITY INCIDENT 🚨" ciso@jibonflow.com

# STEP 2: ISOLATE AFFECTED KEY
# Immediately disable key in KMS
aws kms disable-key --key-id <compromised-key-id>

# STEP 3: STOP BLEEDING
# Prevent further unauthorized use
# - All services immediately switch to backup key
# - Verify key is disabled (should fail decrypt operations)
aws kms describe-key --key-id <compromised-key-id> | grep "KeyState"
# Expected output: "KeyState": "Disabled"

# STEP 4: VERIFY ISOLATION
# Test that old key is truly disabled
ENCRYPTED=$(aws kms encrypt --key-id <compromised-key-id> --plaintext "test" 2>&1)
[[ $ENCRYPTED == *"DisabledException"* ]] && echo "✅ Key successfully disabled"
```

### Investigation Phase (15 minutes - 2 hours)

```bash
# STEP 1: Review access logs
aws logs filter-log-events \
  --log-group-name /aws/kms/jibonflow \
  --filter-pattern "[time, request_id, event_name, ...] && event_name = Decrypt" \
  --start-time $(date -d '7 days ago' +%s)000 > /tmp/key-access.log

# STEP 2: Identify suspicious access
# Look for:
# - Accesses from unknown IPs
# - Accesses by unknown IAM roles
# - Excessive decryption attempts
# - Access outside normal hours

# STEP 3: Determine compromise scope
# - When was key compromised? (first suspicious access)
# - How much data was accessed?
# - Which records affected?
COMPROMISE_START="2025-10-17T14:30:00Z"
aws logs filter-log-events \
  --log-group-name /aws/kms/jibonflow \
  --filter-pattern "[time >= $COMPROMISE_START] && Decrypt" > /tmp/affected-accesses.log

# STEP 4: Calculate breach impact
# Count patient records accessed between compromise start and key disable
SELECT COUNT(DISTINCT patient_id) FROM audit_log 
WHERE event_time >= '$COMPROMISE_START' AND event = 'Decrypt'
```

### Response Phase (2 hours onward)

**1. Containment** (Already done: key disabled)

**2. Recovery**:
```bash
# Generate new key
NEW_KEY=$(aws kms create-key --description "Emergency rotation post-compromise")

# Re-encrypt all data with new key
# (May take several hours for large datasets)
for record in $(database_query_all_encrypted_records); do
  # Decrypt with disabled key (if accessible via backup)
  plaintext = decrypt($record, old_key_with_backup_access)
  
  # Re-encrypt with new key
  new_ciphertext = encrypt($plaintext, NEW_KEY)
  
  # Update database
  update_record($record, new_ciphertext, NEW_KEY)
done

# Verify re-encryption complete
SELECT COUNT(*) FROM encrypted_data WHERE key_version != 'new-key'
# Expected result: 0
```

**3. Notification**:
```bash
# HIPAA Breach Notification Rule (if PHI accessed)
# Timeline: 60 days maximum

# Notify affected individuals
# - Email with breach letter
# - Offer free credit monitoring (1 year)
# - Provide breach summary

# Notify regulatory agencies
# - HHS Office for Civil Rights (if >500 residents)
# - State Attorney General
# - Provide detailed breach report

# Media notification (if >500 residents)
# - Press release
# - Coordinate with legal/PR team
```

### Post-Incident Review

```bash
# STEP 1: Root cause analysis
# - How was key compromised?
# - Was it our fault or vendor fault?
# - Could it have been prevented?

# STEP 2: Mitigation steps
# - Additional monitoring enabled?
# - Access controls tightened?
# - Training needed?

# STEP 3: Documentation
# - Incident report saved
# - Timeline of events
# - Lessons learned documented
# - Process improvements identified

# STEP 4: Sign-off
# - CISO approval
# - Legal review (for breach notification)
# - Auditor notification (for compliance records)
```

---

## 10. Audit & Compliance

### Audit Logging (Automatic)

Every key operation is logged automatically:

```json
{
  "timestamp": "2025-10-17T14:45:30.123Z",
  "event_type": "Decrypt",
  "key_id": "jibonflow-master-key",
  "principal": {
    "arn": "arn:aws:iam::123456:role/application-runtime",
    "user_id": "AIDC4EXAMPLE7M6XAMPLE"
  },
  "source_ip": "10.0.1.42",
  "request_id": "a1b2c3d4e5f6g7h8",
  "status": "Success",
  "records_decrypted": 1,
  "duration_ms": 45
}
```

### Compliance Checks

**Quarterly Audit**:
```bash
# 1. Verify key lifecycle adherence
aws kms list-keys | while read key_id; do
  CREATION=$(aws kms describe-key --key-id $key_id | jq .KeyMetadata.CreationDate)
  ROTATION=$(aws kms get-key-rotation-status --key-id $key_id | jq .KeyRotationEnabled)
  
  echo "$key_id created: $CREATION, rotation enabled: $ROTATION"
done

# 2. Verify access controls
aws iam list-role-policies --role-name devops-key-manager

# 3. Verify compliance with retention policy
aws logs filter-log-events \
  --log-group-name /aws/kms/jibonflow \
  --start-time $(date -d '90 days ago' +%s)000 | \
  jq '.events | length'
# Expected: >10,000 events (normal operations)

# 4. Generate compliance report
# - All keys rotated within 90 days?
# - All access logged?
# - All compromise incidents handled?
# - HIPAA compliant?
```

---

## 11. Real-World Scenarios

### Scenario 1: Scheduled 90-Day Rotation

**Situation**: Your rotation window is Friday 2am-4am

```
Current Time: Thursday 6pm
Task: Execute 90-day key rotation

✅ Correct Approach:
1. Prepare environment (verify backups exist)
2. Schedule downtime window in change management
3. 30 minutes before: Generate new key + test
4. At 2am: Deploy new key configuration
5. Monitor: Watch error logs for decryption failures
6. After 24 hours: Verify all data rotated, archive old key
7. Sign-off: Get CISO approval

❌ Wrong Approach:
- Rotating during business hours without communication
- Not testing new key first
- Not monitoring during transition
- Proceeding without CISO sign-off
```

### Scenario 2: Suspected Compromise

**Situation**: You notice someone accessed the HSM from an unknown IP

```
✅ Correct Approach (Immediate - within 5 minutes):
1. Alert CISO immediately (page if after hours)
2. Disable compromised key: aws kms disable-key
3. Verify disabled: test decrypt operation should fail
4. Call incident response team
5. Secure compromised HSM: Lock physical access
6. Preserve evidence: Don't delete any logs
7. Begin investigation: Review access patterns
8. Communicate: Notify security team, begin breach assessment

❌ Wrong Approach:
- Waiting to tell anyone "until you're sure"
- Rotating key manually (too slow, might make it worse)
- Deleting logs to "clean up"
- Assuming it's a false alarm and ignoring
```

### Scenario 3: Emergency Data Recovery

**Situation**: Database crashed. Need old key to recover encrypted patient data.

```
✅ Correct Approach:
1. Create incident ticket with patient safety impact
2. Request emergency access from CISO
3. Two approvers required (CISO + Security Officer)
4. Access granted: 30-minute session maximum
5. Decrypt data needed for recovery
6. Re-encrypt with current key
7. Restore database
8. Session ends automatically (or manual revocation)
9. Post-incident review within 24 hours

❌ Wrong Approach:
- Accessing key without creating incident ticket
- Not documenting why you need access
- Proceeding without dual approval
- Taking longer than 30 minutes (session will be revoked)
- Not having post-incident review
```

---

## 12. Knowledge Assessment

**Instructions**: Answer 8 questions based on scenarios. **75% pass required (6/8 correct)**

---

### Question 1: Key Generation Frequency
**Which statement is TRUE about key generation in JibonFlow?**

A) Generate new master key every month  
B) Generate new data key every 90 days OR when compromise suspected  
C) Generate new session key for every encrypted connection ✅  
D) All of the above  

**Explanation**:
- Master key: Kept for years (rarely generated)
- Data key: Rotated every 90 days (B and C are true)
- Session key: Generated fresh per TLS connection (C is true)
- **Correct answer: C** (all three types use different frequencies)

---

### Question 2: Emergency Access Scenario
**You detect the master key might be compromised. What's your FIRST action?**

A) Change the password to the HSM  
B) Notify CISO immediately (page if after hours) ✅  
C) Disable the key immediately without telling anyone  
D) Wait and see if more evidence appears  

**Explanation**:
- Every second counts in a compromise
- CISO needs to declare incident immediately
- Disabling key alone isn't enough - incident needs management attention
- **Correct answer: B** (Communication is critical first step)

---

### Question 3: Key Storage Locations
**Which of these is a VALID location to store cryptographic keys?**

A) In application source code as constants  
B) In AWS Secrets Manager (encrypted) ✅  
C) As environment variables in .env files (committed to git)  
D) In customer-facing documentation  

**Explanation**:
- A: Source code is public → keys exposed
- B: Encrypted, access controlled, audit logged ✓
- C: Git history is permanent → keys exposed
- D: Documentation is shared → keys exposed
- **Correct answer: B** (Only AWS Secrets Manager is secure)

---

### Question 4: Zero-Downtime Rotation
**During a key rotation, what should happen to the OLD encryption key?**

A) Immediately delete it  
B) Store it securely for 90 days, then delete  
C) Rotate it and keep the old key disabled for 7 years (compliance) ✅  
D) Use it for new encryptions until rotation completes  

**Explanation**:
- A: Too aggressive - existing data can't be decrypted
- B: 90 days insufficient for compliance
- C: HIPAA requires data retention for 7 years ✓
- D: Defeats purpose of rotation (risk management)
- **Correct answer: C** (Balance security with compliance)

---

### Question 5: Access Control Decision
**A developer asks for direct access to decrypt patient data with the master key. What do you do?**

A) Grant access - they need it for their job  
B) Deny access - provide API/SDK instead ✅  
C) Ask CISO permission (no consistent policy)  
D) Give them emergency access for 24 hours  

**Explanation**:
- A: Violates principle of least privilege
- B: Developers use SDK, don't need raw key ✓
- C: Policy should exist (principle of least privilege)
- D: Emergency access is for emergencies only
- **Correct answer: B** (Principle of least privilege)

---

### Question 6: Audit Log Analysis
**You're checking audit logs and see a decrypt operation at 3am from IP 203.0.113.42. What do you do?**

A) Ignore it - probably automated maintenance  
B) Review recent access patterns and alert CISO if suspicious ✅  
C) Immediately disable the key without investigation  
D) Assume it's a false positive from monitoring software  

**Explanation**:
- A: Could be compromise - don't assume
- B: Investigate context - could be batch job or real compromise ✓
- C: Premature - get more facts first
- D: Too risky - needs investigation
- **Correct answer: B** (Investigate before reacting)

---

### Question 7: HIPAA Compliance Statement
**HIPAA § 164.312(a)(2)(iv) requires what regarding keys?**

A) Encryption of PHI in transit and at rest ✅  
B) Keys rotated exactly every 90 days (not 89, not 91)  
C) Master key destruction after key rotation  
D) All key operations announced publicly  

**Explanation**:
- A: HIPAA encryption requirement ✓
- B: "At least every 90 days" - flexibility needed
- C: Archive, not destroy (7-year retention)
- D: Confidentiality required, not public
- **Correct answer: A** (Encryption requirement)

---

### Question 8: Post-Compromise Timeline
**A key compromise is detected and fixed at 2pm Monday. When should the post-incident review be completed?**

A) Immediately (same day)  
B) Within 24 hours ✅  
C) Within 1 week (less urgent)  
D) Within 30 days (HIPAA requirement)  

**Explanation**:
- A: Investigators need time to gather data
- B: Insight while fresh, CISO availability ✓
- C: Too slow - team context fades
- D: Could be legal requirement, but internal review faster
- **Correct answer: B** (24 hours optimal for root cause)

---

## Assessment Summary

**Pass Criteria**: 6/8 questions correct (75%)

**Your Results**:
- [ ] Submitted and waiting for grading
- [ ] (To be filled by training administrator)

---

## Quick Reference: Key Operations

### Common Commands

```bash
# List all keys
aws kms list-keys

# Create new key
aws kms create-key --description "Description here"

# Disable key
aws kms disable-key --key-id <key-id>

# Enable key
aws kms enable-key --key-id <key-id>

# Check key status
aws kms describe-key --key-id <key-id>

# View key policy
aws kms get-key-policy --key-id <key-id> --policy-name default

# Request emergency access
aws secretsmanager request-secret-version \
  --secret-id jibonflow/emergency-key \
  --reason "EMERGENCY: <description>"
```

### Emergency Contacts

- **CISO**: ciso@jibonflow.com | Ext: 1001
- **Security Ops**: security-ops@jibonflow.com | Emergency: 1-800-XXX-0911
- **DevOps Lead**: devops-lead@jibonflow.com | Ext: 1234
- **On-Call (after hours)**: Check PagerDuty for on-call engineer

---

## Completion Certificate

Upon passing this assessment (6/8 correct), you receive:

**CERTIFICATE OF COMPLETION**

This certifies that _________________________ has completed

**"Key Management Procedures Training"**

Training Code: KMP-101-2025  
Completion Date: _____________  
Expiration: 12 months  
Re-certification Required: October 2026  

For security role: System Operators, DevOps, Database Administrators

---

## Additional Resources

- **NIST SP 800-133**: Recommendation for Key Derivation Methods
- **HIPAA § 164.312(a)(2)(iv)**: Encryption and Decryption
- **AWS KMS Documentation**: https://docs.aws.amazon.com/kms/
- **Key Compromise Response Plan**: `/generated/SECURITY_POLICY_KEY_MANAGEMENT.md`

---

## Questions?

Contact: security-training@jibonflow.com  
Expected Response Time: <24 hours

---

**Training Generated**: October 17, 2025  
**Version**: 1.0  
**Status**: Production Ready ✅

---

*This training is mandatory for all personnel with key management responsibilities. Certificate must be renewed annually. Questions on compliance? Contact your CISO.*
