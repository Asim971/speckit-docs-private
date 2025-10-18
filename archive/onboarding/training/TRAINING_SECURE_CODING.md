# Secure Coding Practices Training
## For JibonFlow Development Team

**Document Version**: 1.0  
**Effective Date**: October 17, 2025  
**Classification**: INTERNAL - Developer Training  
**Duration**: 90 minutes  
**Audience**: All developers, engineers, QA engineers

---

## 1. INTRODUCTION

### Why Secure Coding Matters

**Software Security Issues Lead To**:
- Data breaches exposing patient records
- Unauthorized access to systems
- System crashes and downtime
- Regulatory fines and legal liability
- Loss of customer trust
- Reputation damage

**Statistics**:
- 70% of breaches involve code vulnerabilities
- Average breach costs $4.5 million
- Most common: SQL injection, weak authentication, unencrypted data
- Many vulnerabilities preventable with secure coding

### Learning Objectives

After this training, you will understand:
- ✅ Common vulnerabilities in healthcare applications
- ✅ Secure coding principles and best practices
- ✅ How to prevent SQL injection, XSS, CSRF attacks
- ✅ Proper authentication and authorization implementation
- ✅ Encryption implementation in code
- ✅ Secure logging without exposing secrets
- ✅ Input validation and sanitization
- ✅ Dependency scanning and updates

---

## 2. OWASP TOP 10 VULNERABILITIES

### Overview

The OWASP (Open Web Application Security Project) Top 10 lists the most critical security risks. Every developer should understand these.

### Top 10 List

#### 1. Broken Access Control
**What It Is**: Users can access data/functions they shouldn't

**Healthcare Impact**: Patient can view another patient's records

**Example (VULNERABLE)**:
```typescript
// VULNERABLE: No authorization check
app.get('/api/patients/:id', (req, res) => {
  // User can request ANY patient ID
  const patient = db.getPatient(req.params.id);
  res.json(patient);
});
```

**Example (SECURE)**:
```typescript
// SECURE: Verify user has access
app.get('/api/patients/:id', authenticateUser, (req, res) => {
  const patientId = req.params.id;
  const userId = req.user.id;
  
  // Check if user can access this patient
  if (!hasAccess(userId, patientId)) {
    res.status(403).send('Access denied');
    return;
  }
  
  const patient = db.getPatient(patientId);
  res.json(patient);
});
```

---

#### 2. Cryptographic Failures
**What It Is**: Weak encryption or unencrypted sensitive data

**Healthcare Impact**: Patient health records readable without password

**Example (VULNERABLE)**:
```javascript
// VULNERABLE: Storing PHI unencrypted
const AsyncStorage = require('@react-native-async-storage/async-storage');

async function saveMedication(medication) {
  // Stored in clear text!
  await AsyncStorage.setItem('currentMed', JSON.stringify(medication));
}

// Later - anyone can read it
const data = await AsyncStorage.getItem('currentMed');
console.log(data); // { drug: "Metformin", dosage: "500mg" }
```

**Example (SECURE)**:
```typescript
// SECURE: Using encrypted storage
import EncryptedStorage from 'react-native-encrypted-storage';
import crypto from 'crypto';

async function saveMedication(medication: Medication) {
  // Encrypt with AES-256-GCM
  const encryptionKey = await getEncryptionKey(); // From secure key store
  
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', encryptionKey, iv);
  
  let encrypted = cipher.update(JSON.stringify(medication), 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  // Store encrypted data
  await EncryptedStorage.setItem('currentMed', 
    iv.toString('hex') + ':' + encrypted + ':' + authTag.toString('hex')
  );
}

async function getMedication(): Promise<Medication> {
  const encryptionKey = await getEncryptionKey();
  const encrypted = await EncryptedStorage.getItem('currentMed');
  
  if (!encrypted) return null;
  
  const [iv, data, authTag] = encrypted.split(':');
  const decipher = crypto.createDecipheriv('aes-256-gcm', encryptionKey, Buffer.from(iv, 'hex'));
  decipher.setAuthTag(Buffer.from(authTag, 'hex'));
  
  let decrypted = decipher.update(data, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return JSON.parse(decrypted);
}
```

---

#### 3. Injection (SQL Injection, NoSQL Injection, etc.)
**What It Is**: Untrusted input interpreted as code

**Healthcare Impact**: Attacker can extract all patient records

**Example (VULNERABLE - SQL Injection)**:
```javascript
// VULNERABLE: String concatenation with user input
const username = req.body.username; // Could be: ' OR '1'='1
const query = `SELECT * FROM users WHERE username = '${username}'`;
// Becomes: SELECT * FROM users WHERE username = '' OR '1'='1'
// Returns ALL users!
db.query(query);
```

**Example (SECURE - Parameterized Queries)**:
```typescript
// SECURE: Parameterized query
import { Pool } from 'pg';

const pool = new Pool();
const username = req.body.username; // User input

// Use parameterized query ($1 placeholder)
const query = 'SELECT * FROM users WHERE username = $1';
const result = await pool.query(query, [username]);

// User input cannot be interpreted as code
```

**Example (NoSQL Injection)**:
```javascript
// VULNERABLE: MongoDB query with unsanitized input
const userId = req.body.id;
const user = await User.findOne({ _id: userId }); // Could be: { $gt: '' }

// SECURE: Type validation
const mongoose = require('mongoose');
const userId = mongoose.Types.ObjectId(req.body.id); // Throws if invalid
const user = await User.findOne({ _id: userId });
```

---

#### 4. Insecure Design
**What It Is**: Missing security controls in architecture

**Healthcare Impact**: System has no multi-factor authentication

**Key Principles**:
- Design security in from start, don't add later
- Threat modeling
- Secure defaults
- Fail securely
- Defense in depth (multiple layers)

**Secure Design Example**:
```
ARCHITECTURE WITH SECURITY:

Patient → TLS/HTTPS → API Gateway → Auth Service
                           ↓              ↓
                        Firewall    Multi-Factor Auth
                           ↓              ↓
                      Rate Limiting  Token Validation
                           ↓              ↓
                      Logging & Audit   Access Control
                           ↓              ↓
                      Database (Encrypted)
                           ↓
                      Audit Trail
```

---

#### 5. Security Misconfiguration
**What It Is**: Systems not securely configured

**Healthcare Impact**: Debug endpoints left enabled, exposing patient data

**Common Misconfigurations**:
- ✗ Debug mode enabled in production
- ✗ Default credentials not changed
- ✗ Unnecessary services enabled
- ✗ Security headers not set
- ✗ Directory listing enabled
- ✗ Error messages expose system details

**Example (SECURE)**:
```typescript
// Secure Express.js configuration
import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

const app = express();

// Set security headers
app.use(helmet());

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // Limit each IP to 100 requests per windowMs
});
app.use('/api/', limiter);

// Hide X-Powered-By header
app.disable('x-powered-by');

// Only JSON content type accepted
app.use(express.json({ limit: '1mb' }));

// CORS - restrict to known origins
import cors from 'cors';
app.use(cors({
  origin: ['https://jibonflow.com', 'https://app.jibonflow.com'],
  credentials: true
}));

// Never expose stack traces
app.use((err, req, res, next) => {
  console.error('Error:', err); // Log internally
  
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({ error: 'Internal server error' });
  } else {
    res.status(500).json({ error: err.message }); // Dev only
  }
});

export default app;
```

---

#### 6. Vulnerable and Outdated Components
**What It Is**: Using libraries with known security vulnerabilities

**Healthcare Impact**: Patient data exposed via compromised npm package

**Prevention**:
```bash
# Check for vulnerabilities
npm audit

# Update packages
npm update

# View security vulnerabilities
npm audit --json

# Automated scanning
npm ci --ignore-scripts  # Install from package-lock.json only
```

**Example Package.json Scanning**:
```json
{
  "name": "jibonflow",
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.0.3",
    "pg": "^8.8.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "eslint": "^8.40.0"
  }
}
```

**In CI/CD**:
```yaml
# GitHub Actions workflow
name: Security Audit
on: [push, pull_request]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm audit --audit-level=moderate
```

---

#### 7. Identification & Authentication Failures
**What It Is**: Weak password policies, no MFA, session fixation

**Healthcare Impact**: Attacker gains access to patient accounts

**Example (VULNERABLE)**:
```typescript
// VULNERABLE: Weak authentication
app.post('/login', (req, res) => {
  const { username, password } = req.body;
  
  // No password hashing
  const user = db.query(`SELECT * FROM users WHERE username = ? AND password = ?`, 
                        [username, password]);
  
  if (user) {
    // Session cookie without HttpOnly
    res.cookie('session', user.id);
    res.send('Logged in');
  }
});
```

**Example (SECURE)**:
```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import rateLimit from 'express-rate-limit';

// Rate limit login attempts
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5 // 5 attempts per 15 minutes
});

app.post('/login', loginLimiter, async (req, res) => {
  const { username, password } = req.body;
  
  // Validate input
  if (!username || !password) {
    res.status(400).json({ error: 'Missing credentials' });
    return;
  }
  
  try {
    // Get user from DB
    const user = await db.query('SELECT * FROM users WHERE username = $1', [username]);
    
    if (!user) {
      // Don't reveal whether username exists
      res.status(401).json({ error: 'Invalid credentials' });
      return;
    }
    
    // Compare password with hash
    const validPassword = await bcrypt.compare(password, user.passwordHash);
    
    if (!validPassword) {
      res.status(401).json({ error: 'Invalid credentials' });
      return;
    }
    
    // Check if MFA enabled
    if (user.mfaEnabled) {
      // Send MFA challenge
      res.status(200).json({ requiresMFA: true, sessionId: generateTempToken() });
      return;
    }
    
    // Generate JWT token
    const token = jwt.sign(
      { userId: user.id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: '15m', algorithm: 'HS256' }
    );
    
    // Set secure cookie
    res.cookie('token', token, {
      httpOnly: true,      // Prevent JavaScript access
      secure: true,        // Only HTTPS
      sameSite: 'strict',  // CSRF protection
      maxAge: 15 * 60 * 1000 // 15 minutes
    });
    
    res.json({ success: true });
    
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Server error' });
  }
});
```

---

#### 8. Software and Data Integrity Failures
**What It Is**: Insecure CI/CD, compromised dependencies, untrusted updates

**Prevention**:
- ✅ Code review for all changes
- ✅ Automated testing in CI/CD
- ✅ Dependency pinning (exact versions)
- ✅ Secure download verification (signatures, hashes)
- ✅ Encrypted artifact storage

---

#### 9. Logging and Monitoring Failures
**What It Is**: Insufficient logging, not detecting attacks

**Healthcare Impact**: Breach goes undetected for months

**Example (VULNERABLE)**:
```javascript
// VULNERABLE: No logging
app.get('/api/patients/:id', (req, res) => {
  const patient = db.getPatient(req.params.id);
  res.json(patient);
  // No audit trail - who accessed what?
});
```

**Example (SECURE)**:
```typescript
import winston from 'winston';

// Configure logging
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  defaultMeta: { service: 'jibonflow-api' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

app.get('/api/patients/:id', authenticateUser, (req, res) => {
  const patientId = req.params.id;
  const userId = req.user.id;
  
  try {
    // Check authorization
    if (!hasAccess(userId, patientId)) {
      // Log unauthorized attempt
      logger.warn('Unauthorized access attempt', {
        userId,
        patientId,
        ip: req.ip,
        timestamp: new Date().toISOString()
      });
      res.status(403).json({ error: 'Access denied' });
      return;
    }
    
    const patient = db.getPatient(patientId);
    
    // Log successful access to PHI
    logger.info('PHI accessed', {
      userId,
      patientId,
      action: 'read',
      dataFields: Object.keys(patient),
      timestamp: new Date().toISOString()
    });
    
    res.json(patient);
    
  } catch (error) {
    logger.error('Patient retrieval error', {
      userId,
      patientId,
      error: error.message,
      stack: error.stack
    });
    res.status(500).json({ error: 'Server error' });
  }
});
```

---

#### 10. Server-Side Request Forgery (SSRF)
**What It Is**: Attacker makes server request to unauthorized internal services

**Example (VULNERABLE)**:
```javascript
// VULNERABLE: Server fetches arbitrary URL
const url = req.query.imageUrl;
const image = await fetch(url); // Could fetch http://internal-database:5432
```

**Example (SECURE)**:
```typescript
import { URL } from 'url';

app.get('/api/proxy-image', async (req, res) => {
  const urlString = req.query.imageUrl;
  
  try {
    // Validate URL
    const url = new URL(urlString);
    
    // Whitelist allowed domains only
    const allowedDomains = ['cdn.jibonflow.com', 'images.jibonflow.com'];
    if (!allowedDomains.includes(url.hostname)) {
      res.status(400).json({ error: 'Domain not allowed' });
      return;
    }
    
    // Prevent internal IP addresses
    const ipv4Regex = /^(\d{1,3}\.){3}\d{1,3}$/;
    if (ipv4Regex.test(url.hostname) || url.hostname === 'localhost') {
      res.status(400).json({ error: 'Internal URLs not allowed' });
      return;
    }
    
    // Fetch only from whitelisted domain
    const response = await fetch(url.toString(), {
      timeout: 5000 // 5 second timeout
    });
    
    res.set('Content-Type', response.headers.get('content-type'));
    response.body.pipe(res);
    
  } catch (error) {
    res.status(400).json({ error: 'Invalid URL' });
  }
});
```

---

## 3. INPUT VALIDATION & SANITIZATION

### Core Principle
**Never trust user input. Always validate and sanitize.**

### Input Validation Examples

#### String Input Validation
```typescript
// Validate email format
function isValidEmail(email: string): boolean {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email) && email.length <= 254;
}

// Validate phone number
function isValidPhone(phone: string): boolean {
  const phoneRegex = /^(\+1)?[\s.-]?\(?(\d{3})\)?[\s.-]?(\d{3})[\s.-]?(\d{4})$/;
  return phoneRegex.test(phone);
}

// Validate date
function isValidDate(dateString: string): boolean {
  const date = new Date(dateString);
  return date instanceof Date && !isNaN(date.getTime());
}

// Usage
app.post('/api/patients', (req, res) => {
  const { email, phone, dob } = req.body;
  
  if (!isValidEmail(email)) {
    res.status(400).json({ error: 'Invalid email' });
    return;
  }
  
  if (!isValidPhone(phone)) {
    res.status(400).json({ error: 'Invalid phone' });
    return;
  }
  
  if (!isValidDate(dob)) {
    res.status(400).json({ error: 'Invalid date' });
    return;
  }
  
  // Proceed with validated data
});
```

#### Type Validation (TypeScript)
```typescript
interface Patient {
  id: string;
  firstName: string;
  lastName: string;
  dateOfBirth: Date;
  medications: string[];
}

// Zod for runtime validation
import { z } from 'zod';

const PatientSchema = z.object({
  id: z.string().uuid(),
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  dateOfBirth: z.date(),
  medications: z.array(z.string()).max(50)
});

app.post('/api/patients', (req, res) => {
  try {
    const validatedPatient = PatientSchema.parse(req.body);
    // Use validatedPatient - guaranteed to match schema
  } catch (error) {
    res.status(400).json({ error: 'Invalid patient data' });
  }
});
```

#### XSS Prevention (Output Encoding)
```typescript
// VULNERABLE: XSS Attack
app.get('/user/:name', (req, res) => {
  const name = req.params.name; // Could be: <script>alert('XSS')</script>
  res.send(`<h1>Welcome ${name}</h1>`); // Script executes!
});

// SECURE: HTML Encoding
import escapeHtml from 'escape-html';

app.get('/user/:name', (req, res) => {
  const name = escapeHtml(req.params.name);
  res.send(`<h1>Welcome ${name}</h1>`);
  // Output: <h1>Welcome &lt;script&gt;alert('XSS')&lt;/script&gt;</h1>
});

// SECURE: Use templating engine with auto-escaping
app.set('view engine', 'ejs'); // EJS auto-escapes by default
app.get('/user/:name', (req, res) => {
  res.render('welcome', { name: req.params.name });
  // In template: <%= name %> (auto-escaped)
});
```

---

## 4. AUTHENTICATION & AUTHORIZATION

### Secure Password Storage

#### How TO Encrypt Passwords

```typescript
import bcrypt from 'bcrypt';

// When user creates account
async function registerUser(username: string, password: string) {
  // Hash password with salt (bcrypt does both)
  const hashedPassword = await bcrypt.hash(password, 10);
  
  // Store hashedPassword in database (NOT original password)
  await db.insert('users', {
    username,
    passwordHash: hashedPassword
  });
}

// When user logs in
async function validatePassword(inputPassword: string, storedHash: string) {
  // Compare input with stored hash
  const isValid = await bcrypt.compare(inputPassword, storedHash);
  return isValid;
}
```

#### NEVER Do This
```javascript
// VULNERABLE: Storing plain text password
db.insert('users', { username, password });

// VULNERABLE: Weak hashing (MD5, SHA1)
const hash = require('crypto').createHash('md5').update(password).digest('hex');

// VULNERABLE: Using same salt for all passwords
const salt = 'staticSalt123';
const hash = sha256(password + salt);
```

### Token-Based Authentication

```typescript
import jwt from 'jsonwebtoken';

// Generate token after login
function generateToken(userId: string, userRole: string): string {
  const token = jwt.sign(
    { userId, role: userRole },
    process.env.JWT_SECRET,
    {
      expiresIn: '15m',  // Short expiration
      algorithm: 'HS256'
    }
  );
  return token;
}

// Verify token on each protected request
function verifyToken(token: string): { userId: string; role: string } | null {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    return decoded;
  } catch {
    return null;
  }
}

// Middleware for protected routes
function authenticateRequest(req, res, next) {
  const token = req.cookies.token || req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    res.status(401).json({ error: 'No token' });
    return;
  }
  
  const user = verifyToken(token);
  if (!user) {
    res.status(401).json({ error: 'Invalid token' });
    return;
  }
  
  req.user = user;
  next();
}

// Usage
app.get('/api/patients/:id', authenticateRequest, (req, res) => {
  // User is authenticated
  console.log(req.user.userId);
});
```

### Authorization: Role-Based Access Control

```typescript
// Define permissions by role
const permissions = {
  patient: ['read:own-records', 'submit:refill-request'],
  pharmacist: ['read:prescriptions', 'approve:refills', 'read:patient-history'],
  prescriber: ['write:prescriptions', 'read:patients', 'approve:refills'],
  admin: ['read:all', 'write:all', 'delete:all', 'manage:users']
};

// Check permission
function hasPermission(userRole: string, requiredPermission: string): boolean {
  return permissions[userRole]?.includes(requiredPermission) ?? false;
}

// Middleware
function authorize(requiredPermission: string) {
  return (req, res, next) => {
    if (!hasPermission(req.user.role, requiredPermission)) {
      res.status(403).json({ error: 'Insufficient permissions' });
      return;
    }
    next();
  };
}

// Usage
app.delete('/api/users/:id', authorize('manage:users'), (req, res) => {
  // Only admins can delete users
  db.deleteUser(req.params.id);
  res.json({ success: true });
});
```

---

## 5. DATA ENCRYPTION

### Encrypting Sensitive Data in Code

#### Database Encryption
```typescript
import crypto from 'crypto';

// Encrypt before saving to database
function encryptPHI(data: string, encryptionKey: Buffer): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', encryptionKey, iv);
  
  let encrypted = cipher.update(data, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  // Return IV + encrypted data + auth tag
  return iv.toString('hex') + ':' + encrypted + ':' + authTag.toString('hex');
}

// Decrypt when retrieving from database
function decryptPHI(encrypted: string, encryptionKey: Buffer): string {
  const [ivHex, encryptedHex, authTagHex] = encrypted.split(':');
  
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  
  const decipher = crypto.createDecipheriv('aes-256-gcm', encryptionKey, iv);
  decipher.setAuthTag(authTag);
  
  let decrypted = decipher.update(encryptedHex, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}

// Usage
const encryptionKey = await getEncryptionKeyFromSecureStore();

// Save encrypted
const encryptedMedication = encryptPHI(JSON.stringify(medication), encryptionKey);
await db.saveMedication({ 
  patientId, 
  encryptedData: encryptedMedication 
});

// Retrieve and decrypt
const record = await db.getMedication(patientId);
const medication = JSON.parse(decryptPHI(record.encryptedData, encryptionKey));
```

---

## 6. SECURE LOGGING

### What TO Log
```typescript
logger.info('Security Event', {
  eventType: 'PHI_ACCESS',
  userId: req.user.id,
  patientId: patientData.id,
  action: 'READ',
  timestamp: new Date().toISOString(),
  ipAddress: req.ip
});
```

### What NOT TO Log
```typescript
// VULNERABLE: Never log secrets
logger.info('Authenticating', {
  username: user.username,
  password: req.body.password,  // ❌ NEVER LOG PASSWORDS
  apiKey: process.env.API_KEY   // ❌ NEVER LOG SECRETS
});

// SECURE: Log only what's needed for audit
logger.info('Authentication Attempt', {
  username: user.username,
  success: true,
  timestamp: new Date().toISOString()
  // Password not included
});
```

### Sanitizing Log Output

```typescript
function sanitizeForLogging(obj: any): any {
  const sensitiveFields = ['password', 'apiKey', 'secret', 'token'];
  
  if (typeof obj !== 'object') return obj;
  
  const sanitized = Array.isArray(obj) ? [...obj] : { ...obj };
  
  for (const key in sanitized) {
    if (sensitiveFields.some(field => key.toLowerCase().includes(field))) {
      sanitized[key] = '[REDACTED]';
    } else if (typeof sanitized[key] === 'object') {
      sanitized[key] = sanitizeForLogging(sanitized[key]);
    }
  }
  
  return sanitized;
}

// Usage
logger.info('User data', sanitizeForLogging(userData));
// Output: { password: '[REDACTED]', apiKey: '[REDACTED]', username: 'john' }
```

---

## 7. DEPENDENCY SCANNING

### npm Audit Commands

```bash
# Check for vulnerabilities
npm audit

# See detailed report
npm audit --json

# Fix automatically
npm audit fix

# Fix specific package
npm update [package-name]

# Force install audited version
npm ci

# Check without fixing
npm audit --audit-level=moderate
```

### Continuous Scanning in CI/CD

```yaml
# GitHub Actions
name: Security Audit
on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm audit --audit-level=moderate
```

### Dependency Pinning

```json
{
  "dependencies": {
    "express": "4.18.2",  // Exact version (preferred for prod)
    "lodash": "~4.17.21"  // Patch updates allowed
  },
  "devDependencies": {
    "typescript": "^5.0.0" // Minor/patch updates allowed for dev
  }
}
```

---

## 8. CODE REVIEW CHECKLIST

When reviewing code, check for:

- ✅ Input validation on all user inputs
- ✅ Authorization checks before data access
- ✅ Passwords hashed (never stored plainly)
- ✅ Sensitive data encrypted
- ✅ No hardcoded secrets or credentials
- ✅ Parameterized queries (no string concatenation)
- ✅ Proper error handling (no stack trace exposure)
- ✅ Audit logging for sensitive operations
- ✅ HTTPS/TLS for all network communication
- ✅ Rate limiting on public endpoints
- ✅ CSRF tokens on form submissions
- ✅ Security headers set correctly
- ✅ Dependencies up to date
- ✅ No console.log() with sensitive data

---

## 9. SECURE CODING PRACTICES SUMMARY

### The Golden Rules

1. **Validate Input** - Always validate and sanitize user input
2. **Encrypt Data** - PHI must be encrypted at rest and in transit
3. **Authenticate Users** - Verify user identity (strong passwords/MFA)
4. **Authorize Access** - Verify user has permission for the action
5. **Handle Errors** - Don't expose sensitive info in error messages
6. **Log Auditably** - Log sensitive actions, but never log secrets
7. **Use HTTPS** - All traffic encrypted
8. **Keep Updated** - Patch dependencies and systems promptly
9. **Review Code** - Security review before deployment
10. **Test Security** - Include security tests in automated testing

---

## 10. CONCLUSION & NEXT STEPS

### Key Takeaways
✅ Most security issues are preventable through secure coding  
✅ Input validation is the foundation of security  
✅ Encryption protects PHI and sensitive data  
✅ Logging creates accountability and enables incident detection  
✅ Security is everyone's responsibility  

### Resources
- OWASP Top 10: https://owasp.org/Top10/
- Node.js Security: https://nodejs.org/en/docs/guides/security/
- TypeScript Security: https://cheatsheetseries.owasp.org/

### Next Training
- Mobile security (React Native, iOS, Android)
- API security in depth
- Penetration testing and security testing

---

**Generated by Security Remediation Agent v1.0**  
**Phase 2B: Security Remediation & Compliance Hardening**  
**Generated**: 2025-10-17
