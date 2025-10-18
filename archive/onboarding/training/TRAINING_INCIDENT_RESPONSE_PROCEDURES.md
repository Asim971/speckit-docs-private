# Security Training: Incident Response Procedures
## Responding to Security Events in JibonFlow

**Duration**: 50 minutes  
**Audience**: All Security Personnel, DevOps, IT Managers, Incident Commanders  
**Level**: Intermediate  
**Completion Certificate**: Yes  
**Assessment**: 7-question scenario-based quiz (75% pass required)  

---

## 📋 Table of Contents

1. [Learning Objectives](#learning-objectives)
2. [Why Incident Response Matters](#why-incident-response-matters)
3. [Incident Classification](#incident-classification)
4. [Incident Response Phases](#incident-response-phases)
5. [Detection & Initial Response](#detection--initial-response)
6. [Investigation & Analysis](#investigation--analysis)
7. [Containment & Eradication](#containment--eradication)
8. [HIPAA Breach Notification](#hipaa-breach-notification)
9. [Recovery & Restoration](#recovery--restoration)
10. [Post-Incident Review](#post-incident-review)
11. [Tabletop Exercises](#tabletop-exercises)
12. [Knowledge Assessment](#knowledge-assessment)

---

## 1. Learning Objectives

By completing this training, you will be able to:

- ✅ Identify and classify security incidents by severity level
- ✅ Execute the 4-phase incident response cycle
- ✅ Perform initial triage and containment
- ✅ Investigate incidents using forensic techniques
- ✅ Determine if a breach occurred (HIPAA definition)
- ✅ Execute HIPAA-compliant breach notification procedures
- ✅ Lead team communication during incidents
- ✅ Recover systems to operational state
- ✅ Conduct post-incident reviews and improvement planning

---

## 2. Why Incident Response Matters

### The Numbers

- **87%** of healthcare organizations experienced a security incident in 2024
- **Mean Time to Detect (MTTD)**: 287 days (healthcare average)
- **Your MTTD Goal**: <1 hour (via monitoring and alerts)
- **HIPAA Breach Notification**: 60 days maximum
- **Cost of Delay**: $10K+ per hour for active breach

### The Impact Cycle

```
┌──────────────────────────────────────────┐
│  Security Incident Detected              │
│  (Patient data potentially compromised)  │
├──────────────────────────────────────────┤
│                                          │
│  Phase 1: CONTAINMENT (0-30 min)         │
│  ├─ Stop the bleeding                   │
│  ├─ Isolate affected systems            │
│  └─ Prevent further compromise          │
│                                          │
│  Phase 2: INVESTIGATION (30 min - 24 hr)│
│  ├─ Determine what happened             │
│  ├─ Assess scope of breach              │
│  └─ Calculate patient impact            │
│                                          │
│  Phase 3: ERADICATION (1 - 7 days)      │
│  ├─ Remove attacker access              │
│  ├─ Patch vulnerabilities               │
│  └─ Harden defenses                     │
│                                          │
│  Phase 4: RECOVERY (1 - 30 days)        │
│  ├─ Restore systems safely              │
│  ├─ Validate security controls          │
│  └─ Return to production                │
│                                          │
│  HIPAA BREACH NOTIFICATION: 60 days max │
│  ├─ Notify affected individuals         │
│  ├─ Report to regulatory agencies       │
│  └─ Media notification if >500 people   │
│                                          │
└──────────────────────────────────────────┘
```

### Your Role

Your actions during an incident determine:
- ✅ Speed of containment (security impact)
- ✅ Scope of investigation (breach size)
- ✅ Legal exposure (HIPAA penalties up to $1.5M)
- ✅ Patient trust (reputation impact)
- ✅ Timeline recovery (business continuity)

**You are a first responder in our security incident response.**

---

## 3. Incident Classification

### Severity Levels

| Level | Name | Examples | Response Time | HIPAA Impact |
|-------|------|----------|----------------|-------------|
| **🔴 Critical** | System Breach | Active attacker access, encrypted data stolen | 5 minutes | Likely breach notification needed |
| **🟠 High** | Data Exposure | Unencrypted data accessed, credentials exposed | 15 minutes | Breach assessment required |
| **🟡 Medium** | Security Event | Failed login attempts, unusual access patterns | 1 hour | Monitor, may not be breach |
| **🟢 Low** | Info Gathering | Port scans, vulnerability scans, bot activity | 4 hours | No breach risk |

### Classification Decision Tree

```
Incident Detected
│
├─ Is patient data (PHI) involved?
│  ├─ NO → Likely Low-Medium severity
│  │
│  └─ YES:
│     ├─ Is attacker CURRENTLY active?
│     │  ├─ YES → 🔴 CRITICAL (Immediate containment)
│     │  └─ NO → Continue analysis
│     │
│     ├─ Was PHI data accessed (not encrypted)?
│     │  ├─ YES → 🟠 HIGH (Breach notification likely)
│     │  └─ NO → 🟡 MEDIUM (Assess containment)
│     │
│     └─ Is encryption in place & keys safe?
│        ├─ YES → Risk reduced significantly
│        └─ NO → Assume breach until proven otherwise
```

### Examples by Severity

**🔴 CRITICAL Examples**:
- Attacker accessing production database directly
- Credentials stolen and used for unauthorized access
- Ransomware encrypting patient data
- Insider threat exfiltrating PHI to external device

**🟠 HIGH Examples**:
- Unauthorized user access to patient portal (but no data accessed)
- Exposed credentials in git repository (not yet used)
- Unencrypted backup file found outside data center
- Vendor breached and our data may be at risk

**🟡 MEDIUM Examples**:
- Unusual login from unfamiliar location (employee traveling)
- Brute force attempt detected by WAF (blocked)
- Employee accessing patient data outside normal role
- Quarterly security scan finds low-risk vulnerabilities

**🟢 LOW Examples**:
- Port scan from external IP (standard bot activity)
- Vulnerability disclosed but patch available
- Typosquatter domain registered
- Phishing email (not opened)

---

## 4. Incident Response Phases

### The Four Phases

```
┌─────────────────────────────────────────────────────┐
│              INCIDENT RESPONSE CYCLE                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│  PHASE 1: CONTAINMENT (Stop it now)                │
│  Goal: Prevent further damage                      │
│  Timeline: 0-30 minutes for CRITICAL              │
│  Success: Attacker access removed                  │
│                                                     │
│  PHASE 2: INVESTIGATION (Find out what happened)  │
│  Goal: Understand scope and impact                │
│  Timeline: 30 min to 24 hours                      │
│  Success: Root cause identified                    │
│                                                     │
│  PHASE 3: ERADICATION (Remove vulnerability)      │
│  Goal: Fix the underlying problem                 │
│  Timeline: 1-7 days                               │
│  Success: Vulnerability patched, attacker locked  │
│                                                     │
│  PHASE 4: RECOVERY (Get back to normal)           │
│  Goal: Restore systems safely                     │
│  Timeline: 1-30 days                              │
│  Success: Production operational, controls tested │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 5. Detection & Initial Response

### Detection Methods

**Automated Detection** (Most common):
- ✅ SIEM alerts (failed login attempts, unusual data access)
- ✅ Intrusion detection system (IDS) triggers
- ✅ Application monitoring (sudden error spike)
- ✅ Security scanning (vulnerability detected)
- ✅ Endpoint protection (malware detected)

**Manual Detection**:
- Employee reports suspicious activity
- Manager notices unusual account behavior
- Customer reports unauthorized access
- Vendor notifies us they've been breached

### Initial Response Checklist (First 5 minutes)

```
CRITICAL INCIDENT DETECTED
│
├─ ✅ Step 1: VERIFY IT'S REAL (not false alarm)
│  └─ Check: Is the alert confirmed? Have others verified?
│
├─ ✅ Step 2: DECLARE INCIDENT
│  └─ Page: Incident Commander + Security Lead
│  └─ Note: The incident is DECLARED at this point
│
├─ ✅ Step 3: INITIAL TRIAGE
│  └─ Is PHI involved? (determines notification risk)
│  └─ Is attacker currently active? (determines urgency)
│  └─ How many systems affected?
│
├─ ✅ Step 4: ACTIVATE INCIDENT TEAM
│  └─ Incident Commander (leads response)
│  └─ Technical Lead (system expertise)
│  └─ Security Investigator (forensics)
│  └─ Communications (internal/external)
│
├─ ✅ Step 5: GATHER EVIDENCE
│  └─ Take screenshots of alerts
│  └─ Note system clock time (for logs correlation)
│  └─ Preserve evidence (don't reboot affected systems yet)
│  └─ Start incident timeline
│
└─ ✅ Step 6: BEGIN CONTAINMENT
   └─ See next section for containment steps
```

### Evidence Preservation

**CRITICAL**: Preserve evidence before taking action!

```bash
# On affected system (before rebooting):

# Step 1: Capture network connections
netstat -anb > /tmp/incident/netstat-$(date +%s).txt

# Step 2: Capture running processes
ps -aux > /tmp/incident/processes-$(date +%s).txt

# Step 3: Capture open files
lsof > /tmp/incident/lsof-$(date +%s).txt

# Step 4: Check for persistence mechanisms
crontab -l > /tmp/incident/crontab-$(date +%s).txt
cat /etc/cron.d/* > /tmp/incident/system-crons-$(date +%s).txt

# Step 5: Capture memory (if available)
dd if=/dev/mem of=/tmp/incident/memory-dump-$(date +%s).bin

# Step 6: Secure evidence
tar czf /tmp/incident-evidence-$(date +%s).tar.gz /tmp/incident/
# Copy to secure, segregated storage immediately
```

---

## 6. Investigation & Analysis

### Investigation Process

**Step 1: Establish Timeline**

```
Timeline: When did what happen?

2025-10-17 14:00:00 - Alert: 50 failed login attempts from IP 203.0.113.42
2025-10-17 14:02:15 - Successful login from same IP as admin_user@corp
2025-10-17 14:03:00 - Database query: SELECT * FROM patient_records LIMIT 1000
2025-10-17 14:04:30 - Firewall blocked outbound connection to 198.51.100.7:443
2025-10-17 14:05:00 - Alert: Unusual amount of data access (10x normal)

Analysis:
- Attacker brute-forced admin credentials
- Successfully authenticated
- Queried patient data for export
- Attempted to exfiltrate data (firewall blocked)
- Incident duration: ~5 minutes
```

**Step 2: Determine Scope of Access**

```sql
-- What data was accessed?
SELECT 
  timestamp,
  user_id,
  action,
  table_name,
  rows_accessed
FROM audit_log
WHERE 
  user_id = 'admin_user' AND
  timestamp BETWEEN '2025-10-17 14:00:00' AND '2025-10-17 14:06:00'
ORDER BY timestamp;

-- Results might show:
-- 14:03:00 - Patient records table: 1,000 rows accessed
-- 14:03:15 - Prescription data table: 2,500 rows accessed
-- 14:03:45 - Billing data table: 800 rows accessed

-- Calculate unique patients affected:
SELECT COUNT(DISTINCT patient_id) 
FROM accessed_records 
WHERE access_timestamp BETWEEN '2025-10-17 14:00:00' AND '2025-10-17 14:06:00';
-- Result: 2,847 unique patients potentially affected
```

**Step 3: Determine if It's a BREACH**

HIPAA defines a breach as:
- Unauthorized acquisition, access, use, or disclosure
- That compromises the security or privacy of PHI
- AND it was not prevented or mitigated

```
Breach Determination Logic:

Was PHI involved?
├─ NO → Not a breach, no notification needed
│
└─ YES:
   ├─ Was it encrypted?
   │  ├─ YES (AES-256 with secure key) → Not a breach
   │  │  (Even if accessed, unreadable to attacker)
   │  │
   │  └─ NO (unencrypted):
   │     ├─ Did attacker actually access it?
   │     │  ├─ NO (blocked before access) → Not a breach
   │     │  │
   │     │  └─ YES (successfully accessed):
   │     │     └─ 🔴 BREACH CONFIRMED
   │     │        → Notification required within 60 days
   │     │        → Identify all affected individuals
   │     │        → Prepare breach letter
```

---

## 7. Containment & Eradication

### Immediate Containment (Phase 1)

**For Critical Severity**:

```bash
# STEP 1: REVOKE COMPROMISED CREDENTIALS
# Immediately invalidate the credentials used in attack
aws iam update-access-key-status \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive \
  --user-name admin_user

# STEP 2: REVOKE ACTIVE SESSIONS
# Force all sessions of compromised user to log out
aws cognito-idp admin-user-global-sign-out \
  --user-pool-id us-east-1_1234567890 \
  --username admin_user@corp

# STEP 3: ISOLATE AFFECTED SYSTEMS
# Network isolation to prevent lateral movement
aws ec2 modify-instance-attribute \
  --instance-id i-0abcd1234efgh5678 \
  --groups sg-isolated-incident

# STEP 4: VERIFY ISOLATION
# Test that no more data is being accessed
# Monitor logs for further unauthorized access
# Check: Are new attack attempts still occurring?

# STEP 5: PRESERVE EVIDENCE
# See "Evidence Preservation" section above
```

### Eradication (Phase 3)

**Understanding Root Cause**:

```
Attack Vector: Brute force on admin account

Root Causes to Address:
1. Weak password policy
   └─ Fix: Enforce 12+ character passwords + complexity
   
2. No MFA on admin account
   └─ Fix: Require MFA for all privileged accounts
   
3. No rate limiting on login
   └─ Fix: Implement WAF rule: 5 failed attempts → 15 min lockout
   
4. Password not changed in 2 years
   └─ Fix: Implement 90-day password rotation policy
   
5. No alert on multiple failed attempts
   └─ Fix: Add SIEM alert: >3 failed attempts = page security
```

**Eradication Steps**:

```bash
# STEP 1: Change ALL admin credentials
aws iam update-access-key-status --access-key-id OLD_KEY --status Inactive
aws iam create-access-key --user-name admin_user
# New credentials generated and rotated

# STEP 2: Enable MFA on all privileged accounts
aws iam enable-mfa-device \
  --user-name admin_user \
  --serial-number arn:aws:iam::ACCOUNT:mfa/admin_user \
  --authentication-code1 123456 \
  --authentication-code2 789012

# STEP 3: Deploy WAF rate limiting rule
# Add to API Gateway / CloudFront:
# Block after 5 failed login attempts for 15 minutes

# STEP 4: Apply patches
# All vulnerabilities exploited in attack should be patched
# Verify patches deployed to all systems

# STEP 5: Verify defenses
# Test that attack vector no longer works
# Attempt brute force → should be blocked within 30 seconds
```

---

## 8. HIPAA Breach Notification

### Determining if Notification Required

**Rule**: If PHI was accessed and is unencrypted → MUST notify

**The 60-Day Timeline**:

```
Day 0: Breach Confirmed (end of investigation)
  ↓
Days 1-30: Prepare Breach Letter
  ├─ What happened (description of incident)
  ├─ When it happened (specific dates/times)
  ├─ What PHI was affected (types of data)
  ├─ What JibonFlow is doing (remediation steps)
  ├─ What patients should do (credit monitoring)
  └─ Contact info for questions
  
Days 20-35: Notify Individuals
  ├─ Mail breach letter to last known address
  ├─ Email if email on file
  ├─ Public notice in newspaper (if >500 residents)
  └─ Toll-free number for questions: 1-800-XXX-0911
  
Days 35-50: Notify Regulatory Agencies
  ├─ HHS Office for Civil Rights (always)
  ├─ State Attorney General (if >500 residents of that state)
  ├─ Media notification (if >500 residents nationally)
  └─ Provide detailed breach report
  
Day 60: Deadline (final notification must be sent)

PENALTY FOR LATE NOTIFICATION:
- Additional civil penalties
- Loss of patient trust
- Potential HHS enforcement action
```

### Breach Letter Template

```
JIBONFLOW HEALTHCARE: NOTICE OF SECURITY BREACH

Date: [date of letter]
To: [Patient Name] at [address]

This letter is to inform you of a security incident involving 
your personal health information (PHI) stored at JibonFlow 
Healthcare.

WHAT HAPPENED
On [date], we discovered that unauthorized individuals gained 
access to our systems through [description of attack vector].

We conducted a thorough investigation and determined that your 
personal health information may have been accessed during the 
period of [date range].

TYPES OF INFORMATION AFFECTED
- Name
- Date of birth
- Medical record number
- Prescription information (specific medications accessed)

WHAT WE'VE DONE
1. Immediately revoked unauthorized access
2. Investigated the full scope of the incident
3. Enhanced security controls to prevent recurrence
4. Notified law enforcement (case #[number])

WHAT YOU SHOULD DO
- Monitor credit reports: [free service details]
- Watch for identity theft signs
- Call us with questions: 1-800-XXX-0911
- No cost to you for monitoring services

COMPENSATION
We are offering 2 years of free credit monitoring and 
identity theft protection services through [vendor name].

If you need assistance, please contact:
JibonFlow Breach Response Team
1-800-XXX-0911
breach-response@jibonflow.com

Sincerely,
[CEO Name]
Chief Executive Officer
JibonFlow Healthcare

---

This notice is being provided in accordance with 
45 CFR § 164.404 (HIPAA Breach Notification Rule)
```

### Notification Logistics

```bash
# Step 1: Generate mailing list
SELECT name, address, email 
FROM patients 
WHERE patient_id IN (affected_patient_ids)
ORDER BY zip_code;

# Step 2: Print letters (certified mail)
# - Print on official letterhead
# - Sign each letter
# - Use certified mail (proof of delivery)
# - Cost: ~$5-10 per letter (can be $200K+ for large breaches)

# Step 3: Send email notification (if available)
# Send same day as postal mail
# Include toll-free number
# Don't include full details (email not secure)

# Step 4: Media notification (if >500 residents)
# Write press release
# Send to major outlets
# Coordinate with legal/PR team

# Step 5: Report to HHS OCR
# Online form at: www.hhs.gov/ocr/privacy/hipaa/breachnotification
# Include:
#  - Number of residents affected
#  - Date of discovery
#  - Description of incident
#  - Remediation steps
```

---

## 9. Recovery & Restoration

### Safe Restoration Process

**Step 1: Verify Cleanliness**

```bash
# Before bringing systems back online:

# Check 1: Are there lingering backdoors?
# Search for suspicious cron jobs, services, users
grep -r "^[^:]*:[^:]*:0:" /etc/passwd  # Hidden root accounts

# Check 2: Are patches applied?
apt list --upgradable  # Show any pending security updates

# Check 3: Is malware present?
rkhunter --check --quiet --skip-keypress

# Check 4: Are old credentials still usable?
# Attempt login with old compromised credentials
# Should FAIL (successfully revoked)
```

**Step 2: Restore from Known-Good Backup**

```bash
# Create new systems from scratch (not patching compromised systems)

# Step 1: Take down compromised systems
aws ec2 terminate-instances --instance-ids i-0abcd1234efgh5678

# Step 2: Spin up new instances from clean image
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --security-groups sg-clean-security

# Step 3: Apply latest security patches
sudo apt update && sudo apt upgrade -y

# Step 4: Restore data from uncompromised backup
# (Backup should be verified uncompromised before restoration)
aws s3 cp s3://jibonflow-backups/encrypted-backup.tar.gz .
tar xzf encrypted-backup.tar.gz
# Restore database, files, etc.

# Step 5: Validate restored data
# Checksums match known-good values?
# Patient count matches expected?
# Recent data intact?
```

**Step 3: Operational Validation**

```bash
# Test 1: Can patients log in?
# Test with real patient credentials (staging environment)
# Verify MFA works
# Verify portal functions normally

# Test 2: Can providers access data?
# Test with provider credentials
# Verify prescriptions visible
# Verify audit logging captures access

# Test 3: Can admins manage systems?
# Test with admin credentials
# Verify monitoring dashboards update
# Verify backups execute successfully

# Test 4: Are security controls active?
# Attempt to access system without MFA → should fail
# Attempt database bypass → should fail
# Attempt to escalate privileges → should fail

# PROCEED ONLY IF ALL TESTS PASS
```

**Step 4: Return to Production**

```
Communication Timeline:

T-4 hours: Notify staff of planned restoration
T-2 hours: Send customer notification (maintenance window)
T-0: Begin restoration (small window, <1 hour ideal)
T+5 min: Systems back online, monitoring activated
T+30 min: Validation complete, health check green
T+1 hour: All-clear signal, normal operations resume

Communication: "Due to security maintenance, services were 
offline from X to Y. We've completed necessary updates and 
services are now fully operational. No patient data was impacted. 
Thank you for your patience."
```

---

## 10. Post-Incident Review

### Timing

- **Day 1**: Preliminary review (while fresh)
- **Day 7**: Detailed root cause analysis
- **Day 30**: Management presentation + action items

### Review Meeting Checklist

**Attendees** (Mandatory):
- Incident Commander (led response)
- Technical Lead (system knowledge)
- Security Investigator (forensics)
- CISO (oversight)
- Legal (compliance implications)
- Customer Service (patient communication impact)

**Agenda**:

```
1. INCIDENT SUMMARY (10 min)
   - Date/time of incident
   - Duration (how long was it active?)
   - Severity classification
   - Key statistics (how many records, what impact)
   
2. TIMELINE REVIEW (15 min)
   - When was it first detected?
   - How long to detect (MTTD)?
   - Actions taken and when
   - Key decision points
   
3. ROOT CAUSE ANALYSIS (20 min)
   - Why did this happen?
   - What was the attack vector?
   - What vulnerabilities were exploited?
   - Were there any warnings we missed?
   
4. IMPACT ASSESSMENT (10 min)
   - Patient records affected: N
   - Data compromised: X
   - Revenue/reputation impact: $Y
   - Regulatory reporting required: Yes/No
   
5. RESPONSE EFFECTIVENESS (15 min)
   - What went well in response?
   - What could be improved?
   - Was our incident plan adequate?
   - Did we have right skills/tools?
   
6. IMPROVEMENT ACTION ITEMS (20 min)
   - Technical fixes (patches, architecture changes)
   - Process improvements (procedures, automation)
   - Training needs (skills gaps identified)
   - Tooling improvements (better visibility/response)
   
7. SIGN-OFF (5 min)
   - CISO approval of action items
   - Assignment of owners
   - Due dates for completion
```

### Post-Incident Report Template

```
CONFIDENTIAL - ATTORNEY-WORK PRODUCT

INCIDENT POST-MORTEM REPORT
Incident ID: INC-2025-001
Date: October 17, 2025

EXECUTIVE SUMMARY
On [date], JibonFlow discovered and responded to a security 
incident involving unauthorized access to patient systems. 
The incident was contained within [X minutes], and no patient 
harm resulted. This report details findings and improvements.

INCIDENT DETAILS
├─ Detection Time: [date/time]
├─ Containment Time: [date/time]
├─ Total Duration: [hours]
├─ Affected Records: N
└─ Regulatory Impact: Yes/No

FINDINGS
1. Root Cause: Weak password on administrative account
2. Contributing Factors:
   - No rate limiting on login attempts
   - No MFA on privileged accounts
   - Infrequent password rotation
3. Detection Method: SIEM alert on failed login spike

IMPACT
- Patient Records Affected: 2,847
- Data Types: Names, DOBs, Prescriptions
- Business Impact: 1 hour of degraded service
- Financial Impact: $50K (incident response costs)

IMPROVEMENTS IMPLEMENTED
1. ✅ MFA now required for all admin accounts (immediate)
2. ✅ WAF rate limiting deployed (immediate)
3. ✅ Password policy updated (within 30 days)
4. ✅ SIEM alert tuning (within 7 days)
5. ✅ Incident response drill scheduled (within 60 days)

LESSONS LEARNED
- Our incident response team performed well under pressure
- Automation gaps made investigation slower than ideal
- Good: automated evidence collection worked perfectly
- Opportunity: implement real-time compliance checking

APPROVAL
[CISO Signature] _________________ Date: _______
[CEO Signature] _________________ Date: _______

Classification: Internal Use Only
Retention: 7 years (per HIPAA)
```

---

## 11. Tabletop Exercises

### Purpose

Tabletop exercises prepare teams for real incidents by simulating them in a low-stakes environment.

### Exercise Format

**Time**: 2 hours  
**Frequency**: Quarterly  
**Attendees**: All incident response team members  

### Sample Scenario: Ransomware Attack

**Scenario Setup**:

```
DATE/TIME: October 17, 2025, 2:30 PM

Your SIEM shows:
- 50 systems infected with ransomware
- Ransom note on patient portal: "Your data encrypted"
- Email system down
- Backups appear encrypted too

In the "simulation," this is hypothetical. 
In reality, this would be CRITICAL.

YOUR TASK: Walk through incident response steps as if this 
were real. Discuss what you'd do.
```

**Facilitated Discussion**:

```
FACILITATOR ASKS:

"It's 2:35 PM. Systems are still being encrypted as we speak. 
What's your FIRST action right now?"

Expected answer: "Page incident commander and isolate network 
segment immediately."

If answer is slow or wrong: "We just lost another 30 systems 
in the last 30 seconds. Time matters."

Continue with:
- What evidence would you preserve?
- How would you determine scope?
- What would you communicate to patients?
- How would you determine this is a breach?
- What's your recovery plan?
```

---

## 12. Knowledge Assessment

**Instructions**: Answer 7 questions about incident response scenarios. **75% pass required (6/7 correct)**

---

### Question 1: Incident Classification
**A user gets a phishing email with a malware attachment. The attachment was downloaded but antivirus blocked execution. What severity?**

A) 🔴 Critical (users are immediately at risk)  
B) 🟠 High (malware was present)  
C) 🟡 Medium (attachment was received, but blocked) ✅  
D) 🟢 Low (nothing bad happened)  

**Explanation**:
- Phishing arrived (security event)
- Malware downloaded (concerning)
- Antivirus blocked execution (defense worked)
- Result: Medium severity (investigate delivery, improve awareness)
- **Correct answer: C**

---

### Question 2: HIPAA Breach Definition
**Which scenario IS a HIPAA breach that requires notification?**

A) Database accessed by unauthorized user, but data was encrypted  
B) Unencrypted file found in trash directory outside data center ✅  
C) Brute force attack on login (failed after 5 attempts, blocked)  
D) Vulnerability disclosed (no evidence of exploitation)  

**Explanation**:
- A: Encrypted = not readable even if accessed
- B: Unencrypted PHI outside protection = BREACH ✓
- C: Attack blocked = no unauthorized access
- D: Vulnerability ≠ breach unless exploited
- **Correct answer: B**

---

### Question 3: Containment Timeline
**A CRITICAL incident is declared at 2:00 PM. When should initial containment be COMPLETE?**

A) Within 5 minutes (2:05 PM) ✅  
B) Within 30 minutes (2:30 PM)  
C) Within 1 hour (3:00 PM)  
D) By end of shift (5:00 PM)  

**Explanation**:
- Every second counts with active breach
- Revoke credentials: <2 minutes
- Isolate systems: <3 minutes
- Total containment: <5 minutes for critical
- Investigation takes longer, but "stop the bleeding" is immediate
- **Correct answer: A**

---

### Question 4: Evidence Preservation
**What's the BEST way to investigate a compromised server?**

A) Reboot the server immediately (restart clears malware)  
B) Capture memory, connections, processes BEFORE rebooting ✅  
C) Take a screenshot of the desktop  
D) Delete suspicious files to "clean it up"  

**Explanation**:
- A: Rebooting loses volatile memory evidence
- B: Capture data → analyze offline ✓
- C: Screenshot insufficient; need detailed evidence
- D: Destroying evidence = destroying forensic capability
- **Correct answer: B**

---

### Question 5: Breach Notification Timeline
**If a breach is confirmed on Monday, when is the 60-day HIPAA deadline?**

A) Thursday of the same week  
B) 60 calendar days later (around December 16) ✅  
C) 60 business days later (around January 28)  
D) Patients can be notified whenever convenient  

**Explanation**:
- HIPAA specifies "60 days" = calendar days
- Including weekends, holidays
- No exceptions for holidays
- **Correct answer: B**

---

### Question 6: Eradication vs. Recovery
**What's the difference between eradication and recovery?**

A) No real difference - they're the same thing  
B) Eradication = remove attacker; Recovery = restore systems ✅  
C) Eradication = fix bugs; Recovery = hire new staff  
D) They happen at the same time  

**Explanation**:
- Eradication: Remove attacker, patch vulnerability, harden defenses
- Recovery: Restore systems from clean backups, validate they work
- Both needed for complete response
- **Correct answer: B**

---

### Question 7: Post-Incident Meeting
**A post-incident review meeting is scheduled. Who MUST attend?**

A) Only the IT department  
B) Incident Commander, Technical Lead, Security, CISO, Legal ✅  
C) Only the CISO  
D) Everyone in the company  

**Explanation**:
- A: Too narrow - need security + legal perspective
- B: Essential roles represented ✓
- C: Too narrow - need technical details
- D: Too broad - need focused meeting
- **Correct answer: B**

---

## Assessment Summary

**Pass Criteria**: 6/7 questions correct (85%)

**Your Results**:
- [ ] Submitted and waiting for grading
- [ ] (To be filled by training administrator)

---

## Emergency Contact Information

**Immediate Incident**: Call Security Ops  
- **Phone**: 1-800-XXX-0911 (24/7 emergency line)
- **Email**: security-incident@jibonflow.com
- **Slack**: #security-incident

**Incident Commander on Call**:
- Check PagerDuty for current on-call
- Page immediately for any incident

**After Business Hours**:
- Emergency hotline has priority
- On-call engineer will respond within 5 minutes

---

## Additional Resources

- **HIPAA Breach Notification Rule**: 45 CFR §§ 164.400-414
- **Incident Response Plan**: `/generated/SECURITY_POLICY_INCIDENT_RESPONSE.md`
- **Breach Letter Templates**: Included in policy document
- **NIST Incident Response Guide**: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf

---

## Completion Certificate

Upon passing this assessment (6/7 correct), you receive:

**CERTIFICATE OF COMPLETION**

This certifies that _________________________ has completed

**"Incident Response Procedures Training"**

Training Code: IRP-101-2025  
Completion Date: _____________  
Expiration: 12 months  
Re-certification Required: October 2026  

For security role: All Security Personnel, DevOps, IT Management

---

## Questions?

Contact: security-training@jibonflow.com  
Expected Response Time: <24 hours

---

**Training Generated**: October 17, 2025  
**Version**: 1.0  
**Status**: Production Ready ✅

---

*This training is mandatory for all incident response team members. Certificate must be renewed annually. Incident response drills held quarterly.*
