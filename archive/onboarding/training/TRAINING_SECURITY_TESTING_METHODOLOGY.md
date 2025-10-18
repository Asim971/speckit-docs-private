# Security Training: Security Testing Methodology
## Comprehensive Security Validation for JibonFlow

**Duration**: 55 minutes  
**Audience**: QA Engineers, Security Testers, DevOps, Developers (advanced)  
**Level**: Advanced  
**Completion Certificate**: Yes  
**Assessment**: 9-question hands-on quiz (75% pass required)  

---

## 📋 Table of Contents

1. [Learning Objectives](#learning-objectives)
2. [Security Testing Overview](#security-testing-overview)
3. [Test Planning & Scope](#test-planning--scope)
4. [Dependency Scanning (npm audit)](#dependency-scanning-npm-audit)
5. [Static Application Security Testing (SAST)](#static-application-security-testing-sast)
6. [Dynamic Application Security Testing (DAST)](#dynamic-application-security-testing-dast)
7. [Container & Infrastructure Scanning](#container--infrastructure-scanning)
8. [Secrets Detection](#secrets-detection)
9. [Penetration Testing](#penetration-testing)
10. [Compliance Validation](#compliance-validation)
11. [Test Report Generation](#test-report-generation)
12. [Knowledge Assessment](#knowledge-assessment)

---

## 1. Learning Objectives

By completing this training, you will be able to:

- ✅ Understand the complete security testing lifecycle
- ✅ Execute automated dependency scanning (OWASP Top 10)
- ✅ Run static code analysis for common vulnerabilities
- ✅ Perform dynamic testing of running applications
- ✅ Scan container images for vulnerabilities
- ✅ Detect exposed secrets in codebase and configs
- ✅ Execute manual penetration testing scenarios
- ✅ Validate compliance with HIPAA requirements
- ✅ Generate security test reports
- ✅ Remediate vulnerabilities following priority

---

## 2. Security Testing Overview

### Why Security Testing Matters

```
Development Cycle:
Commit → Build → Test → Deploy
         ↓
Security testing should happen:
✅ At every commit (secrets scan)
✅ During build (dependencies, SAST)
✅ Before deployment (DAST, compliance)
✅ In production (monitoring, periodic tests)
```

### Testing Levels

| Level | When | Tool | False Positives | Time |
|-------|------|------|-----------------|------|
| **Dependencies** | Build | npm audit | Low | <5 min |
| **SAST** | Build | SonarQube | Medium | 5-15 min |
| **Container** | Build | Trivy | Low | 2-5 min |
| **Secrets** | Commit | git-secrets | Very low | <1 min |
| **DAST** | Staging | OWASP ZAP | High | 30-60 min |
| **Penetration** | Pre-deploy | Manual | N/A | 40 hours |

### The Security Testing Pyramid

```
                     ▲
                    /│\
                   / │ \
                  /  │  \  Penetration Testing (Manual)
                 /   │   \  High cost, high confidence
                /    │    \
               ├─────┼─────┤
              /      │      \
             /       │       \  DAST (Automated)
            /        │        \ Medium cost, medium confidence
           ├─────────┼─────────┤
          /          │          \
         /           │           \
        /            │            \  SAST + Container + Dependencies (Fast)
       /             │             \ Low cost, catches 80% of issues
      /              │              \
     ╱_______________│_______________╲
             Build & Commit

TEST PYRAMID PRINCIPLE:
- Automate as much as possible at bottom (cheap, fast)
- Use expensive tests for validation at top
- Balance coverage with execution time
```

---

## 3. Test Planning & Scope

### Pre-Testing Checklist

```
BEFORE RUNNING ANY SECURITY TESTS:

✅ Stakeholder Approval
   - Inform team that testing will occur
   - Identify test window (if DAST - need staging env)
   - Approve scope (all services? public APIs only?)

✅ Environment Preparation
   - Testing environment matches production
   - No live patient data in test environment
   - Database snapshots are non-PHI sanitized data
   - All monitoring/logging functional

✅ Tool Calibration
   - Tools updated to latest version
   - Rules/policies configured
   - False positive baseline established
   - Integration with CI/CD pipeline validated

✅ Team Readiness
   - Tester credentials ready
   - Test accounts created (not real users)
   - Vulnerability assessment criteria defined
   - Remediation process communicated
```

### Scope Definition

**Include in Testing** ✅:
- All application code (frontend, backend, APIs)
- Third-party dependencies (npm packages)
- Container images and deployment configs
- Database schemas and access controls
- Configuration files (GitOps definitions)
- Encryption implementations

**Exclude from Testing** ❌:
- Third-party services (not our responsibility)
- Production environment (non-destructive testing only)
- Real patient data (use sanitized test data)
- Other teams' systems (coordinate testing)
- Devices outside managed infrastructure

### Test Criteria & Acceptance

```json
{
  "CRITICAL": {
    "severity": "🔴 Critical",
    "rule": "Must be fixed before deployment",
    "examples": ["SQL Injection", "Authentication bypass", "PHI exposure"]
  },
  "HIGH": {
    "severity": "🟠 High",
    "rule": "Must have remediation plan",
    "examples": ["Missing encryption", "Weak password policy"]
  },
  "MEDIUM": {
    "severity": "🟡 Medium",
    "rule": "Should be fixed in next sprint",
    "examples": ["Missing HTTP headers", "Outdated library"]
  },
  "LOW": {
    "severity": "🟢 Low",
    "rule": "Track for improvements",
    "examples": ["Missing comments", "Style issues"]
  }
}
```

---

## 4. Dependency Scanning (npm audit)

### When & Why

**WHEN**: Every commit to main branch  
**WHY**: 40% of vulnerabilities come from dependencies  

### Step-by-Step Execution

**Step 1: Update npm to latest**

```bash
npm install -g npm@latest
npm --version  # Should be 8.0+
```

**Step 2: Run audit**

```bash
# Full project scan
cd /home/asim/Apps/Asim\'s_New_Projects/SpecKit/Jira_Management/jibonflow
npm audit

# Expected output:
# found 5 vulnerabilities (2 high, 3 moderate)
#   run `npm audit fix` to fix 3 of them
#   1 vulnerability requires manual review

# Output format:
# Package: package-name
# Severity: critical|high|moderate|low
# Issue: description
# Fixed version: X.Y.Z
# Introduced by: your-app@A.B.C > dependency > ...
```

**Step 3: Analyze results**

```bash
# Generate detailed JSON report
npm audit --json > /tmp/npm-audit.json

# Parse with jq for analysis
jq '.metadata.vulnerabilities' /tmp/npm-audit.json
# Output:
# {
#   "critical": 0,
#   "high": 2,
#   "moderate": 3,
#   "low": 0
# }
```

### Interpreting Results

**Semantic Versioning Reminder**:
```
Version: 1.2.3
        │ │ └─ Patch (1.2.3 → 1.2.4) - bugfixes, safe
        │ └─── Minor (1.2.0 → 1.3.0) - new features, usually safe
        └───── Major (1.0.0 → 2.0.0) - breaking changes, requires review
```

**Decision Matrix**:

```
Vulnerability Severity?
├─ 🔴 CRITICAL
│  ├─ Upgrade? → YES (even with breaking changes)
│  ├─ Update immediately
│  └─ Block deployment if not fixed
│
├─ 🟠 HIGH
│  ├─ Fixable? → YES? Upgrade in next sprint
│  │           NO? → Request security exception + monitoring
│  └─ Monitor for exploitation
│
├─ 🟡 MEDIUM
│  ├─ Can wait until next minor release
│  ├─ Track in backlog
│  └─ Schedule for next quarter
│
└─ 🟢 LOW
   └─ Informational, update when convenient
```

### Practical Example

```bash
# Scenario: Found vulnerability in 'lodash' package
npm audit
# vulnerability: Prototype Pollution in lodash
# severity: high
# affected versions: <4.17.21
# fixed version: 4.17.21

# Check current version
npm ls lodash
# lodash@4.17.20

# Option 1: Direct upgrade
npm install lodash@4.17.21
npm audit  # Re-run to verify fixed

# Option 2: Update package-lock.json
npm audit fix
# Auto-updates to 4.17.21

# Option 3: Test before committing
npm test  # Verify nothing broke
npm run lint  # Check for regressions
git diff package.json  # Review change

# Commit with message
git add package.json package-lock.json
git commit -m "fix: upgrade lodash to 4.17.21 (security)"
```

---

## 5. Static Application Security Testing (SAST)

### What Is SAST?

SAST analyzes source code without running it, looking for patterns of insecure code.

### Common Vulnerabilities Detected

```
SQL Injection Pattern:
❌ BAD: database.query(`SELECT * FROM users WHERE id = ${req.params.id}`)
✅ GOOD: database.query('SELECT * FROM users WHERE id = ?', [req.params.id])

XSS (Cross-Site Scripting):
❌ BAD: <div>{userInput}</div>  // Direct HTML injection
✅ GOOD: <div>{escapeHtml(userInput)}</div>

Hardcoded Secrets:
❌ BAD: const apiKey = 'sk-1234567890abcdef'
✅ GOOD: const apiKey = process.env.API_KEY

Missing Authentication:
❌ BAD: app.get('/api/patients', (req, res) => {...})
✅ GOOD: app.get('/api/patients', authenticateUser, (req, res) => {...})

Weak Cryptography:
❌ BAD: crypto.createCipher('aes-256', password)
✅ GOOD: crypto.createCipheriv('aes-256-gcm', key, iv)
```

### Running SAST with SonarQube

**Installation** (Docker):

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:latest
# Access at: http://localhost:9000
```

**Scanning Project**:

```bash
# Install SonarQube scanner
npm install -g sonarqube-scanner

# Run scan
sonarqube-scanner \
  -Dsonar.projectKey=jibonflow \
  -Dsonar.sources=src \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.login=<token>
```

**Interpreting Results**:

```
SonarQube Dashboard shows:
├─ Security Hotspots: 23
│  ├─ CRITICAL: 0
│  ├─ HIGH: 2
│  ├─ MEDIUM: 8
│  └─ LOW: 13
│
├─ Code Smells: 156
│  ├─ Maintainability concerns (not security)
│  └─ Can be addressed in future refactoring
│
└─ Coverage: 78%
   ├─ Lines of code analyzed
   └─ 22% untested = potential risk areas
```

### Fix Workflow

```bash
# Step 1: Review flagged code
# Open SonarQube issue: "SQL Injection Risk in user.service.ts"
# Line 45: database.query(`SELECT FROM users WHERE id = ${id}`)

# Step 2: Understand context
# - Is this really vulnerable? (false positives exist)
# - What's the actual risk?
# - Can I reproduce it?

# Step 3: Fix the issue
# Change line 45 to use parameterized query
// BEFORE:
database.query(`SELECT * FROM users WHERE id = ${req.body.id}`)

// AFTER:
database.query('SELECT * FROM users WHERE id = ?', [req.body.id])

# Step 4: Re-scan
sonarqube-scanner -Dsonar.projectKey=jibonflow ...
# Issue should be resolved

# Step 5: Test thoroughly
npm test
npm run e2e
# All tests should pass
```

---

## 6. Dynamic Application Security Testing (DAST)

### What Is DAST?

DAST tests running applications by sending requests and analyzing responses, like an attacker would.

### Common DAST Checks

```
✅ SQL Injection Testing
   Sends: admin' OR '1'='1
   Expected: Application rejects or parameterizes query
   
✅ XSS Testing
   Sends: <script>alert('xss')</script>
   Expected: Browser blocks, or content escaped

✅ Authentication Testing
   Sends: Request without auth token
   Expected: 401 Unauthorized response

✅ Authorization Testing
   Sends: Request to /api/admin/users as patient user
   Expected: 403 Forbidden response

✅ HTTPS/TLS Testing
   Checks: SSL certificate validity, TLS 1.2+
   Expected: Secure connection established
```

### DAST with OWASP ZAP

**Installation**:

```bash
# Docker
docker pull owasp/zap2docker-stable

# Run in daemon mode
docker run -d \
  -p 8080:8080 \
  -p 8090:8090 \
  --name zap \
  owasp/zap2docker-stable \
  zap.sh -config api.disablekey=true -daemon -port 8080
```

**Scanning Staging Environment**:

```bash
# Step 1: Authenticate to application
# DAST needs to test authenticated endpoints too

# Step 2: Define scope
# Target: http://staging-api.jibonflow.local:3001
# Exclude: /health, /metrics (not relevant for security)

# Step 3: Run baseline scan
curl "http://localhost:8090/JSON/spider/action/scan/?url=http://staging-api.jibonflow.local:3001"

# Step 4: Wait for completion and get results
curl "http://localhost:8090/JSON/report/action/report/?title=jibonflow-dast"

# Output: HTML report with findings
# - Vulnerabilities found
# - Confidence level (high/medium/low)
# - OWASP Top 10 mapping
# - Remediation advice
```

---

## 7. Container & Infrastructure Scanning

### Why Container Scanning?

Docker images often include:
- Outdated libraries (vulnerable versions)
- Unnecessary packages (attack surface)
- Exposed secrets in image layers
- Missing security patches

### Scanning with Trivy

**Installation**:

```bash
# Homebrew
brew install aquasecurity/trivy/trivy

# Docker
docker run aquasec/trivy image [image-name]
```

**Scanning JibonFlow Container**:

```bash
# Build image
docker build -t jibonflow:latest .

# Scan for vulnerabilities
trivy image jibonflow:latest

# Expected output:
# jibonflow:latest (debian 11.0)
# =======================
#
# Total: 23 vulnerabilities
# CRITICAL: 2, HIGH: 5, MEDIUM: 16, LOW: 0

# Detailed view:
trivy image --severity CRITICAL,HIGH jibonflow:latest

# JSON output for CI/CD
trivy image -f json -o /tmp/trivy-report.json jibonflow:latest
```

### Interpreting & Fixing

**Typical Result**:
```
CRITICAL: CVE-2023-12345
├─ Package: openssl
├─ Installed version: 1.1.1g
├─ Fixed version: 1.1.1w
└─ Description: Buffer overflow in cryptographic operations

FIX:
1. Update base image: FROM ubuntu:22.04  (includes openssl 1.1.1w)
2. Rebuild: docker build -t jibonflow:latest .
3. Scan again: trivy image jibonflow:latest
```

---

## 8. Secrets Detection

### Why Secrets Scanning?

Developers accidentally commit:
- API keys
- Database passwords
- SSH private keys
- OAuth tokens
- Certificate keys

### Automated Detection Tools

**Tool Options**:
- git-secrets (local)
- TruffleHog (comprehensive, slow)
- GitGuardian (cloud service)
- Gitleaks (GitHub native)

### git-secrets Setup

```bash
# Installation
brew install git-secrets
# or: git clone https://github.com/awslabs/git-secrets && cd git-secrets && make install

# Configuration
cd /path/to/jibonflow-repo
git secrets --install

# Add patterns to detect
git secrets --register-aws
git secrets --add-provider -- cat ~/.ssh/id_rsa

# Test it
echo 'AKIAIOSFODNN7EXAMPLE' > test.txt
git add test.txt
git commit -m "test"
# Error: Found AWS credentials!

# Remove test file
git rm test.txt
git commit --amend
```

### Pre-commit Hook Integration

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/awslabs/git-secrets
    rev: master
    hooks:
      - id: git-secrets
        stages: [commit]
```

### Scanning Existing Repository

```bash
# Find any secrets already committed
truffleHog filesystem /path/to/jibonflow --json > /tmp/secrets-found.json

# If secrets found:
# 1. Regenerate compromised secrets (rotate keys)
# 2. Remove from git history:
git filter-branch --tree-filter 'rm -f vulnerable-file' -- --all
# 3. Force push (dangerous! coordinate with team)
git push origin --force --all
```

---

## 9. Penetration Testing

### Manual Security Testing

After automated tools, manual penetration testing validates:
- Business logic vulnerabilities
- Complex attack chains
- Social engineering risks

### OWASP Top 10 Manual Tests

**1. SQL Injection**

```typescript
// Test vector
Endpoint: GET /api/patients?name=Robert'; DROP TABLE patients;--
Expected response: Error or empty results
Unexpected: Table deleted (vulnerability)
```

**2. Cross-Site Scripting (XSS)**

```html
<!-- Test vector -->
Input: <img src=x onerror=alert('xss')>
Expected: Escaped in output
Unexpected: JavaScript executes (vulnerability)
```

**3. Broken Authentication**

```bash
# Test vectors
curl http://api.jibonflow.local:3001/api/admin/users
# No auth header

Expected: 401 Unauthorized
Unexpected: Returns patient list (vulnerability)
```

**4. Sensitive Data Exposure**

```bash
# Test vectors
curl -i http://api.jibonflow.local:3001/api/patients
# Check headers

Expected:
- ✅ HTTPS connection
- ✅ Strict-Transport-Security header
- ✅ No sensitive data in responses

Unexpected:
- ❌ HTTP (not HTTPS)
- ❌ Password visible in response
- ❌ Medical details in plain text
```

**5. Broken Access Control**

```bash
# Test: Can patient access other patient's data?
# Login as Patient A
TOKEN_A=$(login_and_get_token "patient.a@example.com")

# Try to access Patient B's data
curl -H "Authorization: Bearer $TOKEN_A" \
  http://api.jibonflow.local:3001/api/patients/patient-b-id/records

Expected: 403 Forbidden
Unexpected: Returns Patient B's records (vulnerability)
```

---

## 10. Compliance Validation

### HIPAA § 164.312 Automated Tests

```typescript
// Test 1: Encryption at Rest
test('PHI is encrypted with AES-256-GCM', () => {
  const database = connect();
  const record = database.query('SELECT * FROM patients LIMIT 1');
  
  // Record should be encrypted in database
  expect(record.encrypted).toBe(true);
  expect(record.encryption_algorithm).toContain('AES-256');
});

// Test 2: Encryption in Transit
test('API enforces HTTPS/TLS 1.2+', () => {
  const response = http.get('http://api.jibonflow.local:3001/api/patients');
  expect(response.status).toBe(301); // Redirect to HTTPS
  
  // Verify TLS version
  const https_response = https.get('https://api.jibonflow.local:3001/api/patients');
  expect(https_response.headers['strict-transport-security']).toBeDefined();
});

// Test 3: Access Control
test('Access requires valid authentication', () => {
  const response = http.get('https://api.jibonflow.local:3001/api/patients');
  expect(response.status).toBe(401); // Unauthorized
});

// Test 4: Audit Logging
test('All PHI access is logged', () => {
  const record = database.query('SELECT * FROM patients LIMIT 1');
  const audit = auditLog.query('SELECT * FROM access_log WHERE record_id = ?', [record.id]);
  
  expect(audit.length).toBeGreaterThan(0);
  expect(audit[0].user_id).toBeDefined();
  expect(audit[0].timestamp).toBeDefined();
});

// Test 5: Data Minimization
test('APIs do not return unnecessary PHI', () => {
  const response = api.get('/api/patients/demographics');
  expect(response.body).toHaveProperty('name');
  expect(response.body).not.toHaveProperty('social_security_number');
  expect(response.body).not.toHaveProperty('payment_info');
});
```

---

## 11. Test Report Generation

### Report Structure

```
SECURITY TEST REPORT
JibonFlow Healthcare Prescription Refill Portal
Test Date: October 17, 2025
Test Environment: Staging (production-like)
Tester: Security QA Team

═══════════════════════════════════════════

EXECUTIVE SUMMARY

Overall Risk: 🟠 MEDIUM (2 High, 4 Medium, 8 Low)
Deployment Recommendation: ⚠️ CONDITIONAL PASS
  - 2 High severity issues must be fixed before production
  - 4 Medium issues can be addressed in next sprint

───────────────────────────────────────────

FINDINGS DETAIL

[1] SQL Injection in Patient Search (HIGH)
────────────────────────────────────────
Severity: 🟠 HIGH
URL: GET /api/patients/search
Parameter: name
Issue: User input not parameterized in database query
Impact: Attacker could read entire patient database

Test Case:
  Input: name='; DROP TABLE patients;--
  Expected: Error or no results
  Actual: Potential SQL execution
  
Remediation:
  Use parameterized queries:
  db.query('SELECT * FROM patients WHERE name = ?', [name])
  
Timeline: MUST FIX before deployment
Assigned: Backend Team Lead
Due: October 19, 2025

───────────────────────────────────────────

[2] Missing HSTS Header (MEDIUM)
────────────────────────────────
Severity: 🟡 MEDIUM
Location: Response headers
Issue: HTTP Strict-Transport-Security header not present
Impact: Browser could downgrade to HTTP, exposing transmission

Remediation:
  Add to API: 
  app.use((req, res, next) => {
    res.setHeader('Strict-Transport-Security', 'max-age=31536000');
    next();
  });
  
Timeline: Fix in next sprint
Assigned: DevOps Team
Due: October 30, 2025

───────────────────────────────────────────

TEST COVERAGE SUMMARY

Security Testing Type    Coverage    Pass/Fail
───────────────────────────────────────────
Dependency Scanning      100%        ✅ PASS (0 critical)
SAST (Code Analysis)     87%         ⚠️ CONDITIONAL (2 issues)
DAST (Dynamic Testing)   92%         ✅ PASS
Container Scanning       100%        ✅ PASS
Secrets Detection        100%        ✅ PASS
Penetration Testing      80%         ⚠️ CONDITIONAL (1 issue)
HIPAA Compliance         95%         ⚠️ CONDITIONAL (1 issue)

───────────────────────────────────────────

VULNERABILITY DISTRIBUTION

        10 │
          │
          │     ██
        5 │     ██ ██
          │     ██ ██ ██
          │     ██ ██ ██
        0 │_____██_██_██_
               Low Med High Critical

Low: 8 issues    (non-blocking)
Medium: 4 issues (next sprint)
High: 2 issues   (must fix)
Critical: 0      (none)

───────────────────────────────────────────

REMEDIATION TRACKING

Issue                          Status      Owner           Due Date
────────────────────────────────────────────────────────
SQL Injection Fix              🔴 Open     Backend Lead    Oct 19
HSTS Header Addition           🟡 In-Prog  DevOps Lead     Oct 30
Missing Rate Limiting          🟡 In-Prog  Security Eng    Oct 25
Outdated Library Update        ✅ Fixed    Backend Lead    Oct 18

───────────────────────────────────────────

RECOMMENDATIONS

IMMEDIATE (Before Production):
1. ✅ Fix SQL Injection vulnerability
2. ✅ Add missing authentication on 2 endpoints
3. ✅ Validate encryption key management

SHORT-TERM (Next 30 days):
1. Add HTTP security headers (HSTS, CSP, X-Frame-Options)
2. Update 2 outdated npm packages
3. Implement rate limiting on login endpoint

LONG-TERM (Next Quarter):
1. Implement WAF (Web Application Firewall)
2. Expand code coverage from 87% to 95%
3. Conduct threat modeling session
4. Schedule quarterly penetration testing

───────────────────────────────────────────

COMPLIANCE STATUS

HIPAA § 164.312(a)(2)(i) - Unique User ID      ✅ PASS
HIPAA § 164.312(a)(2)(ii) - Emergency Access   ✅ PASS
HIPAA § 164.312(a)(2)(iv) - Encryption         ⚠️ CONDITIONAL
HIPAA § 164.312(b) - Audit Controls            ✅ PASS
HIPAA § 164.312(e)(1) - Transmission Security  ✅ PASS

Overall HIPAA Assessment: ⚠️ COMPLIANT PENDING HIGH FIXES

───────────────────────────────────────────

APPROVAL

Approved for Deployment: ⚠️ WITH CONDITIONS
  Condition 1: SQL Injection MUST be fixed
  Condition 2: All High severity issues resolved
  Condition 3: Re-scan confirms fixes

[ ] CTO Approval
[ ] Security Lead Approval
[ ] QA Lead Approval
[ ] Legal Review (if breach risk)

```

---

## 12. Knowledge Assessment

**Instructions**: Answer 9 questions about security testing. **75% pass required (7/9 correct)**

---

### Question 1: Test Priority
**Which test should be run FIRST in CI/CD pipeline?**

A) Penetration testing (most thorough)  
B) Secrets detection (fastest, blocks if secrets found) ✅  
C) Penetration testing (catches everything)  
D) DAST on production (most realistic)  

**Explanation**:
- Secrets detection: <1 second
- Dependencies: <5 minutes
- SAST: 10-15 minutes
- DAST: 30+ minutes
- Penetration: 40+ hours
- **Correct answer: B** (Fail fast on secrets)

---

### Question 2: Vulnerability Severity Decision
**Found: Library with MEDIUM severity vulnerability, no fix available yet. What do you do?**

A) Immediately remove library (production integrity risk)  
B) Replace with alternative library (might have other issues)  
C) Track in backlog for next sprint ✅  
D) Wait until patch is available (could wait forever)  

**Explanation**:
- MEDIUM: Monitor, but can deploy with awareness
- Severity: does it impact us? (maybe not)
- **Correct answer: C** (Balance security with delivery)

---

### Question 3: SAST False Positive
**SAST flags: "Potential SQL Injection" on parameterized query code. What's most likely?**

A) Real vulnerability (SAST is always right)  
B) False positive from rule overfitting ✅  
C) Testers made a mistake  
D) Database driver has vulnerability  

**Explanation**:
- Parameterized queries are safe (binding prevents injection)
- SAST has false positives (overly aggressive rules)
- Need manual review of context
- **Correct answer: B** (Rule likely too broad)

---

### Question 4: Container Scanning
**Trivy found OLD_PACKAGE v1.0 in Docker image. Patch v1.1 exists. What do you do?**

A) Wait for v2.0 (major version safer)  
B) Upgrade to v1.1, rebuild, rescan ✅  
C) Ignore (it's in a container, not user-facing)  
D) Remove package entirely (too risky)  

**Explanation**:
- Patch versions are generally safe (1.0 → 1.1)
- Vulnerability could be exploited in container
- Must rebuild to incorporate patch
- **Correct answer: B** (Timely patch management)

---

### Question 5: Secrets Detection
**Your CI/CD caught a secret that was accidentally committed. You:**

A) Ignore it (it's in git history, attackers won't find it)  
B) Delete the file and commit again (hides it)  
C) Regenerate the secret + use git-filter-branch to remove ✅  
D) Flag as false positive and skip  

**Explanation**:
- Git history is permanent (git clone gets all history)
- Attackers can find secrets in git history (common tactic)
- Must regenerate: old secret is burned
- Must remove: rewrite history
- **Correct answer: C** (Comprehensive remediation)

---

### Question 6: DAST Scope
**Running DAST on production, targeting GET /health endpoint. Should you do this?**

A) Yes, /health is non-destructive  
B) Yes, production is most realistic  
C) No, DAST should test staging not production ✅  
D) Maybe, if monitoring is enabled  

**Explanation**:
- DAST sends attack payloads
- Could trigger false alerts in production
- Staging environment is purpose-built for testing
- Production is for validated, tested code only
- **Correct answer: C** (Test on staging first)

---

### Question 7: HIPAA Compliance Test
**Test finds patient SSN visible in API response. Severity?**

A) 🟢 Low (just metadata)  
B) 🟡 Medium (should be removed)  
C) 🟠 High (sensitive data exposed)  
D) 🔴 Critical (definite HIPAA violation) ✅  

**Explanation**:
- SSN is Protected Health Information (PHI)
- HIPAA requires data minimization
- Exposing SSN violates compliance
- Could require breach notification
- **Correct answer: D** (Critical compliance issue)

---

### Question 8: Test Automation
**What percentage of security tests should be automated?**

A) 20% (manual is more thorough)  
B) 50% (balance of automation and manual)  
C) 80-90% (automate what's repeatable) ✅  
D) 100% (manual adds no value)  

**Explanation**:
- Automated: dependencies, SAST, container scanning (repeatable)
- Manual: complex business logic, social engineering (unique)
- Use pyramid: wide automated base, manual validation top
- **Correct answer: C** (Practical balance)

---

### Question 9: Report Recommendation
**Report shows 2 High, 5 Medium, 12 Low issues. Deploy recommendation?**

A) ✅ Deploy immediately (12 low = acceptable)  
B) ⚠️ Deploy conditionally, high issues must be fixed ✅  
C) ❌ Don't deploy (anything flagged is risky)  
D) ⏳ Wait 6 months (more time to fix)  

**Explanation**:
- HIGH: Blocking deployment
- MEDIUM: Can be addressed in next sprint
- LOW: Non-blocking
- Business decision: 2 High = too risky
- **Correct answer: B** (Conditional deployment)

---

## Assessment Summary

**Pass Criteria**: 7/9 questions correct (75%)

**Your Results**:
- [ ] Submitted and waiting for grading
- [ ] (To be filled by training administrator)

---

## Quick Reference: Testing Commands

```bash
# Dependency Scanning
npm audit
npm audit fix
npm audit --json > report.json

# SAST
sonarqube-scanner -Dsonar.projectKey=jibonflow

# Container Scanning
trivy image jibonflow:latest
trivy image --severity CRITICAL,HIGH jibonflow:latest

# Secrets Detection
git secrets --install
git secrets --add-provider -- cat ~/.ssh/id_rsa

# DAST
docker run -p 8080:8080 owasp/zap2docker-stable

# Compliance Testing
npm test -- --testNamePattern="HIPAA"
```

---

## Additional Resources

- **OWASP Top 10**: https://owasp.org/Top10/
- **NIST SP 800-53**: Security and Privacy Controls for Federal Systems
- **HIPAA Security Rule**: 45 CFR Part 164
- **CWE/CVSS**: https://cwe.mitre.org/

---

## Completion Certificate

Upon passing this assessment (7/9 correct), you receive:

**CERTIFICATE OF COMPLETION**

This certifies that _________________________ has completed

**"Security Testing Methodology Training"**

Training Code: STM-101-2025  
Completion Date: _____________  
Expiration: 12 months  
Re-certification Required: October 2026  

For security role: QA Engineers, Security Testers, DevOps

---

## Questions?

Contact: security-training@jibonflow.com  
Expected Response Time: <24 hours

---

**Training Generated**: October 17, 2025  
**Version**: 1.0  
**Status**: Production Ready ✅

---

*This training is strongly recommended for all personnel involved in security testing and validation. Certification demonstrates competency in modern security testing methodology.*
