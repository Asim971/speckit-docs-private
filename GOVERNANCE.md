---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - governance-agent
  - documentation-agent-v2
related_services:
  - prompt_system/governance/policies.json
  - src/core/governance/governance-service.ts
verification_notes: >
  Confirmed policy definitions and enforcement code exist; governance audit
  outputs are not stored in this repository.
---

🟡 **Partial** — Core governance controls are in place, but audit evidence is yet
to be attached.

# Governance & Policy Management

## Overview

SpecKit implements comprehensive governance controls to ensure compliance, security, and quality standards throughout the development lifecycle. The governance system provides policy-driven execution with audit trails and automated enforcement.

## Governance Architecture

### Core Components

- **Policy Engine**: Evaluates requests against defined policies
- **Audit Logger**: Records all governance decisions and violations
- **Compliance Checker**: Validates current state against policies
- **Remediation Engine**: Provides automated fixes for policy violations

### Policy Structure

Policies are defined in JSON format with schema validation:

```json
{
  "version": "1.0.0",
  "policies": [
    {
      "id": "security-secrets",
      "name": "Secret Detection",
      "category": "security",
      "severity": "critical",
      "description": "Prevent accidental commit of secrets",
      "rules": [
        {
          "type": "pattern",
          "pattern": "(?i)(api_key|secret|token|password)\\s*[:=]\\s*['\"][^'\"]+['\"]",
          "action": "block",
          "message": "Potential secret detected in code"
        }
      ]
    }
  ]
}
```

## Policy Categories

### Security Policies

- **Secret Detection**: Prevents accidental exposure of API keys, tokens, passwords
- **Dependency Scanning**: Validates third-party package security
- **Access Control**: Enforces proper authentication and authorization

### Quality Policies

- **Code Standards**: Enforces coding conventions and style guides
- **Test Coverage**: Requires minimum test coverage thresholds
- **Documentation**: Ensures proper documentation standards

### Compliance Policies

- **License Compliance**: Validates open source license compatibility
- **Data Privacy**: Enforces data handling and privacy requirements
- **Regulatory Requirements**: Domain-specific compliance rules

## Governance Commands

### Policy Validation

Check current compliance status:

```bash
npm run governance:check
npm run governance:check -- --policy security
npm run governance:check -- --verbose
```

### Secret Scanning

Scan workspace for potential secrets:

```bash
npm run scan:secrets
npm run scan:secrets -- --exclude node_modules
```

### Audit Logging

View governance audit logs:

```bash
# View recent audit entries
tail -f logs/governance-audit.log

# Search for specific violations
grep "VIOLATION" logs/governance-audit.log
```

## Bootstrap Integration

Governance checks are automatically run during bootstrap:

```bash
npm run bootstrap  # Includes governance validation
```

Bootstrap will:

1. Load governance policies from `prompt_system/governance/policies.json`
2. Validate current workspace state
3. Log compliance status
4. Block execution on critical violations (configurable)

## Policy Configuration

### Policy Files

Policies are stored in `prompt_system/governance/`:

```text
prompt_system/governance/
├── policies.json          # Main policy definitions
├── policies.md            # Policy documentation
├── policies.schema.json   # JSON schema for validation
└── audit/                 # Audit log storage
```

### Configuration Options

Configure governance behavior in `package.json`:

```json
{
  "speckit": {
    "governance": {
      "policies_path": "prompt_system/governance/policies.json",
      "audit_log": "logs/governance-audit.log",
      "auto_enforce": true,
      "fail_on_warnings": false
    }
  }
}
```

## Custom Policies

### Creating Custom Policies

1. Define policy in JSON format
2. Validate against schema
3. Add to policies.json
4. Test with governance checker

Example custom policy:

```json
{
  "id": "custom-code-quality",
  "name": "Custom Code Quality",
  "category": "quality",
  "severity": "medium",
  "rules": [
    {
      "type": "file_pattern",
      "pattern": "*.ts",
      "condition": "not_contains",
      "value": "console.log",
      "action": "warn",
      "message": "Avoid console.log in production code"
    }
  ]
}
```

### Policy Testing

Test custom policies:

```bash
npm run governance:check -- --policy custom-code-quality --dry-run
```

## Audit & Compliance Reporting

### Audit Reports

Generate compliance reports:

```bash
npm run governance:check -- --report json > compliance-report.json
```

### Continuous Monitoring

Integrate with CI/CD for continuous compliance:

```yaml
# .github/workflows/compliance.yml
name: Compliance Check
on: [push, pull_request]
jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: "18"
      - run: npm ci
      - run: npm run governance:check
      - run: npm run scan:secrets
```

## Troubleshooting

### Common Issues

**Policy Validation Failures:**

- Check policy JSON syntax
- Verify file paths in rules
- Review error messages for specific violations

**False Positives:**

- Adjust policy patterns
- Add exclusions for known safe cases
- Update policy severity levels

**Performance Issues:**

- Optimize complex regex patterns
- Limit scan scopes
- Schedule checks appropriately

### Debug Mode

Enable debug logging:

````bash
DEBUG=governance:* npm run governance:check
```</content>
<parameter name="filePath">d:\Asim's Prompt\agent\docs\GOVERNANCE.md
````
