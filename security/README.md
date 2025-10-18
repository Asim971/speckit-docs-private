---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - governance-agent
  - security-agent
  - documentation-agent-v2
related_services:
  - prompt_system/governance/policies.json
  - src/core/governance/governance-service.ts
verification_notes: >
  Governance policies are present, but referenced evidence (e.g. threat models,
  audit exports) could not be located under docs/ or logs/.
---

🟡 **Partial** — Policy scaffolding is verified, yet supporting control evidence
is still pending.

# Security Overview

## Scope
- Summarize available governance artefacts relevant to security posture.
- Provide a checklist for elevating security documentation to ✅ status.

## Evidence Reviewed
- `prompt_system/governance/policies.json` lists enforced policies (security,
  compliance, operations) with evidence expectations.
- `src/core/governance/governance-service.ts` enforces policy validation logic
  across agent workflows.

## Verification Gaps
1. Missing threat-model and audit log artefacts referenced by the policies file
   (e.g. `docs/security/threat-model.md`).
2. No recent security assessment or penetration test summaries under `logs/` or
   `evidence/`.
3. Secrets rotation evidence and incident response drill reports are absent.

## Archive Index
- `docs/archive/security/AUDIT_LOGGING_SPEC.md`
- `docs/archive/security/E2EE_ARCHITECTURE_SPEC.md`
- `docs/archive/security/ENCRYPTED_STORAGE_SCHEMA_SPEC.md`
- `docs/archive/security/hipaa-compliance.md`
- `docs/archive/security/policies/` (legacy policy drafts)

> These artefacts are preserved for reference but require validation before they
> inform operational decisions.

## Next Actions
- Recreate threat modeling outputs and link them under `docs/security/`.
- Capture governance validation logs in `logs/governance/` for the next review
  cycle.
- Promote this document to ✅ after evidence links are in place and audits pass.
