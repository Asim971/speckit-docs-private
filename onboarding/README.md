---
status: "🚧 Planned"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
    - onboarding-agent
    - documentation-agent-v2
related_services:
    - scripts/bootstrap-smoke.mjs
    - src/agent-builder
verification_notes: >
    Legacy onboarding steps reference repositories that are not part of this workspace;
    verification with current teams is still pending.
---

🚧 **Planned** — Onboarding workflows require validation with current teams before
adoption.

# Onboarding Overview

## Scope
- Provide a safe landing page for new contributors while archived materials are
    reviewed.
- Highlight actionable work needed to align onboarding with the present SpecKit
    workspace.

## Evidence Reviewed
- `scripts/bootstrap-smoke.mjs` executes the quick bootstrap referenced during
    onboarding conversations.
- `src/agent-builder/` and related orchestrator modules define the runtime that
    new contributors must understand.

## Verification Gaps
1. External repository references (frontend, microservices, infrastructure) were
     not located in this workspace and require updated access instructions.
2. Training assets now archived under `docs/archive/onboarding/training/` have
     not been refreshed or verified for compliance accuracy.
3. No mentorship roster or escalation pathways are captured within the repo; add
     governance-backed ownership metadata.

## Archive Index
- `docs/archive/onboarding/SETUP_SUMMARY.md`
- `docs/archive/onboarding/training/TRAINING_SECURE_CODING.md`
- `docs/archive/onboarding/training/TRAINING_HIPAA_COMPLIANCE.md`
- `docs/archive/onboarding/training/TRAINING_INCIDENT_RESPONSE_PROCEDURES.md`
- `docs/archive/onboarding/training/TRAINING_KEY_MANAGEMENT_PROCEDURES.md`
- `docs/archive/onboarding/training/TRAINING_SECURITY_TESTING_METHODOLOGY.md`

> The archived modules remain available for historical context and must be
> revalidated before reuse.

## Next Actions
- Identify onboarding owners and capture their contact information in repository
    metadata.
- Draft a pared-down 7-day onboarding plan grounded purely in assets maintained
    within this workspace.
- Promote this document to 🟡 or ✅ once cross-team dependencies are resolved and
    evidence of completion is recorded.

