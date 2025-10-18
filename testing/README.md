---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - testing-agent-v1
  - documentation-agent-v2
related_services:
  - package.json
  - __tests__/
verification_notes: >
  Confirmed Jest-based unit tests exist for orchestrator modules; end-to-end and
  performance evidence is not present in this repository.
---

🟡 **Partial** — Unit and integration scaffolding is verified, broader QA
artifacts remain outstanding.

# Testing Overview

## Scope
- Capture the current state of automated testing within the SpecKit orchestrator
  workspace.
- Provide guidance on the verification backlog required to restore comprehensive
  QA coverage.

## Evidence Reviewed
- Active Jest suites located under `__tests__/` covering `prompt-builder`,
  `governance-service`, and `workspace-scanner` behaviour.
- `package.json` exposes runnable scripts including `test`, `test:agents`, and
  `test:governance` for policy compliance.

## Verification Gaps
1. No evidence of end-to-end or UI automation within this repository; integrate
   Playwright or equivalent suites before promising coverage to stakeholders.
2. Performance and load testing directories referenced in prior docs are absent;
   capture new traces or remove unsupported claims.
3. CI workflow results have not been attached to `logs/` or `evidence/` to prove
   governance gate execution.

## Archive Index
- `docs/archive/testing/TESTING_AGENT_V1_*`
- `docs/archive/testing/VERIFICATION_CHECKLIST_FRONTEND_AGENT_V1.md`

> Archived QA artefacts chronicle the previous agent-led test planning. Reuse
> them only after refreshing evidence under the verification-first workflow.

## Actionable Next Steps
- Execute `npm test` and `npm run test:agents` during each release cut and store
  the output under `logs/testing/`.
- Add traceable coverage reports generated via `npm run test:coverage` and track
  thresholds in documentation.
- Promote this file to ✅ once E2E, performance, and compliance evidence are
  published alongside automated quality gates.
  // Step 2: Enter phone
  await page.fill('input[name="phone"]', '+8801712345678');
  await page.click('button:has-text("Next")');
  
  // Step 3: Verify OTP
  await page.fill('input[name="otp"]', '123456');
  await page.click('button:has-text("Verify")');
  
  // Assertion
  await expect(page.locator('h1')).toContainText('Welcome, Jane!');
});
```

**E2E Best Practices:**
- Run E2E tests in CI against ephemeral staging environments.
- Use data fixtures to ensure reproducibility.
- Limit E2E suite runtime to <10 minutes to avoid slowing down deployments.

## 6. Security Testing

### Secrets Scanning

```bash
npm run scan:secrets
```
Detects hardcoded credentials in source files, environment variable files, and configuration JSON. Blocks CI if violations are found.

### Authentication & Authorization Tests

Validate:
- JWT token expiration and refresh logic
- Role-based access control (RBAC) enforcement
- Multi-factor authentication (MFA) flows
- Session timeout behavior (15-minute idle)

Example test:

```typescript
describe('Auth Service', () => {
  it('rejects expired tokens', async () => {
    const expiredToken = jwt.sign({ userId: 123 }, SECRET, { expiresIn: '-1h' });
    
    const response = await request(app)
      .get('/api/protected')
      .set('Authorization', `Bearer ${expiredToken}`);
    
    expect(response.status).toBe(401);
    expect(response.body.error).toBe('Token expired');
  });
});
```

### Encryption Validation

Ensure:
- PHI at rest is encrypted using AES-256.
- Telemedicine video streams use Agora E2EE.
- Database connection strings leverage TLS 1.2+.

Compliance test:

```typescript
describe('PHI Encryption', () => {
  it('encrypts patient records before storage', async () => {
    const plaintext = { name: 'Jane Doe', dob: '1990-01-01' };
    const encrypted = await encryptPHI(plaintext);
    
    expect(encrypted).not.toContain('Jane Doe');
    expect(encrypted).toMatch(/^[A-Za-z0-9+/=]+$/); // Base64 pattern
  });
});
```

## 7. Performance Testing

### Load Testing with k6

Install k6 and run performance scripts:

```bash
k6 run scripts/load-test-auth.js
```

Example k6 script:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 50 },   // Ramp to 50 users
    { duration: '3m', target: 100 },  // Hold 100 users
    { duration: '1m', target: 0 }     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
  }
};

export default function () {
  const res = http.post('https://api-staging.jibonflow.com/auth/login', {
    phone: '+8801712345678',
    otp: '123456'
  });
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

**Performance Thresholds:**
- Auth service: p95 latency <500ms
- Telemedicine service: p95 latency <1s
- Database queries: p99 <200ms

### Benchmarking SpecKit Orchestrator

```bash
npm run benchmark
npm run benchmark:agents
npm run benchmark:retrievers
npm run benchmark:report
```

Produces reports in `generated/benchmark-report.json` measuring agent execution time, retriever query latency, and memory usage.

## 8. Compliance and Governance Tests

### Policy Evaluation

```bash
npm run governance:check
npm run governance:validate
```

Tests verify:
- All policies have valid expiration dates.
- Required controls are documented.
- Evidence artifacts exist at specified paths.
- Audit logs capture sensitive operations.

Example governance test:

```typescript
describe('Governance Policies', () => {
  it('flags expired policies as violations', () => {
    const policySet: GovernancePolicySet = {
      version: 'test-suite',
      generatedAt: new Date().toISOString(),
      reviewCadenceDays: 30,
      policies: [
        {
          id: 'policy-test-002',
          name: 'Expired policy',
          category: 'operations',
          enforced: true,
          expiresAt: formatDate(-1), // Yesterday
          // ... controls and evidence
        }
      ]
    };

    const report = GovernanceService.evaluateCompliance(policySet);
    expect(report.status).toBe('failed');
    expect(report.violations.some(v => v.type === 'expired')).toBe(true);
  });
});
```

### Audit Logging Verification

Confirm that:
- User authentication attempts are logged.
- PHI access events are recorded with timestamps and actor IDs.
- Failed authorization attempts trigger alerts.

Test example:

```typescript
describe('Audit Logging', () => {
  it('logs PHI access events', async () => {
    await accessPatientRecord(patientId, userId);
    
    const logs = await fetchAuditLogs({ eventType: 'PHI_ACCESS' });
    expect(logs).toContainEqual(
      expect.objectContaining({
        eventType: 'PHI_ACCESS',
        patientId,
        userId,
        timestamp: expect.any(String)
      })
    );
  });
});
```

## 9. Test Data Management

### Fixtures

Store reusable test data in `__tests__/fixtures/`:

```
__tests__/fixtures/
├── patient-profiles.json
├── provider-schedules.json
├── telemedicine-sessions.json
└── governance-policies.json
```

Load fixtures in tests:

```typescript
import patientFixtures from './fixtures/patient-profiles.json';

describe('Patient Service', () => {
  it('retrieves patient profile', async () => {
    const patient = patientFixtures[0];
    const result = await patientService.getById(patient.id);
    expect(result.name).toBe(patient.name);
  });
});
```

### Mocking and Stubs

Use Jest mocks for external services:

```typescript
jest.mock('../src/services/sms.service', () => ({
  sendOTP: jest.fn().mockResolvedValue({ success: true })
}));
```

Avoid mocking internal modules; prefer dependency injection and interface-based testing.

## 10. Continuous Integration

The CI pipeline (`.github/workflows/ci.yml`) runs on every push and PR:

1. **Lint**: `npm run lint` - ESLint with TypeScript rules
2. **Type Check**: `npm run type-check` - Verify TypeScript compilation
3. **Unit Tests**: `npm run test:unit` - Fast, isolated tests
4. **Integration Tests**: `npm run test:integration` - Service-to-service validation
5. **Coverage Report**: `npm run test:coverage` - Enforce 80% threshold
6. **Governance**: `npm run governance:check` - Policy compliance
7. **Security Scan**: `npm run scan:secrets` - Credential leak detection

### Coverage Thresholds

Configured in `jest.config.ts`:

```typescript
coverageThreshold: {
  global: {
    branches: 80,
    functions: 80,
    lines: 80,
    statements: 80
  }
}
```

PRs that drop coverage below 80% are flagged for review.

## 11. Test Environment Setup

### Local Testing

1. **Install dependencies:**
   ```bash
   npm install
   ```

2. **Set environment variables:**
   - Copy `.env.example` to `.env.test`
   - Use mock credentials for third-party services (Agora, AWS, etc.)

3. **Start local services:**
   - PostgreSQL: `docker-compose up -d postgres`
   - Redis: `docker-compose up -d redis`
   - Mock API server: `npm run mock:api`

4. **Run tests:**
   ```bash
   npm test
   ```

### CI Environment

GitHub Actions provisions:
- Node.js 20.x
- PostgreSQL 15 (service container)
- Redis 7 (service container)
- Environment secrets from repository settings

Service containers are ephemeral and destroyed after CI completes.

## 12. Manual QA Procedures

### Pre-Production Checklist

Before deploying to production:

- [ ] All CI checks pass
- [ ] Test coverage ≥80%
- [ ] Security scan shows no high/critical issues
- [ ] Governance policies are current (not expired)
- [ ] Manual smoke test on staging environment
- [ ] Load test confirms performance thresholds
- [ ] HIPAA compliance review complete
- [ ] Release notes drafted
- [ ] Rollback plan documented

### Staging Environment Testing

Access staging at `https://staging.jibonflow.com`:

1. **Patient onboarding flow:** Register new account, verify OTP, complete profile.
2. **Telemedicine session:** Schedule consultation, join video call, verify E2EE.
3. **Medicine ordering:** Browse catalog, add to cart, complete payment, track delivery.
4. **Provider console:** Log in, review patient queue, access medical records.
5. **Pharmacy portal:** Accept prescription, update stock, confirm fulfillment.

Document any issues in Jira under the `QA` project.

## 13. Debugging Test Failures

### Common Issues

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `Cannot find module` | Missing dependency or incorrect import path | Run `npm install`, verify import syntax |
| `Timeout of 5000ms exceeded` | Async operation not awaited | Add `await`, increase timeout in test config |
| `Jest did not exit one second after` | Open handles (DB connections, timers) | Call `afterAll(() => cleanupConnections())` |
| `Expected 200, received 401` | Mock auth token expired | Regenerate test token with valid expiration |

### Verbose Test Output

```bash
npm test -- --verbose --no-coverage
```

### Debugging a Single Test

```bash
npm test -- --testNamePattern="marks compliant policy sets as passed"
```

### Inspecting Coverage Gaps

```bash
npm run test:coverage
open coverage/lcov-report/index.html
```

Red highlights indicate uncovered lines.

## 14. QA Roles and Responsibilities

- **Developers**: Write unit tests alongside feature code; aim for 100% coverage on new modules.
- **QA Engineers**: Author E2E test suites; perform manual exploratory testing on staging.
- **Security Team**: Conduct penetration testing; validate encryption and RBAC.
- **Compliance Officer**: Review audit logs; ensure governance policies are enforced.
- **DevOps**: Maintain CI/CD pipelines; monitor test execution metrics.

## 15. Resources and References

- **Jest Documentation**: [https://jestjs.io/docs/getting-started](https://jestjs.io/docs/getting-started)
- **Playwright Docs**: [https://playwright.dev](https://playwright.dev)
- **Detox Testing**: [https://wix.github.io/Detox/](https://wix.github.io/Detox/)
- **k6 Load Testing**: [https://k6.io/docs/](https://k6.io/docs/)
- **JibonFlow Testing Standards**: `docs/testing/standards.md` (internal)
- **HIPAA Testing Guidelines**: `docs/compliance/hipaa-testing.md` (internal)

## 16. Continuous Improvement

- **Retrospectives**: Hold monthly QA retrospectives to identify flaky tests and tooling gaps.
- **Test Automation Metrics**: Track test execution time, failure rate, and coverage trends.
- **Shift-Left Training**: Conduct workshops on TDD (Test-Driven Development) and BDD (Behavior-Driven Development).
- **Tooling Upgrades**: Evaluate new testing frameworks and integrate if ROI is positive.

For questions or suggestions, contact the QA guild lead in `#jibonflow-qa` on Slack.
