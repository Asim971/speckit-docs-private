---
status: "🚧 Planned"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - devops-agent-v2
  - documentation-agent-v2
related_services:
  - scripts/
  - package.json
verification_notes: >
  CI/CD scripts and configuration references exist, but no recent pipeline
  evidence has been attached. DevOps documentation is pending regeneration under
  the verification-first policy.
---

🚧 **Planned** — DevOps runbooks need refreshed evidence before they can be
presented as verified operational guidance.

# DevOps Overview

## Scope
- Track the status of automation, CI, and infrastructure-as-code workflows that
  underpin SpecKit deployments.
- Direct readers to the archived DevOps artefacts consolidated under
  `docs/archive/devops/`.
- Define the validation work required to reactivate DevOps documentation.

## Current Signals
- `package.json` exposes scripts such as `npm run governance:check` and
  `npm run scan:secrets` that enforce baseline quality gates.
- The `scripts/` directory contains deployment and validation utilities awaiting
  fresh execution logs.

## Verification Gaps
1. Provide proof of successful CI runs (GitHub Actions or equivalent) covering
   linting, testing, and deployment stages.
2. Attach infrastructure provisioning manifests (CDK/Terraform) together with
   applied-state evidence.
3. Record rollback or disaster-recovery exercises under `logs/devops/`.

## Archive Index
The following historical DevOps artefacts are preserved for context:

- `docs/archive/devops/DEVOPS_AGENT_*`
- `docs/archive/devops/DEVOPS_PHASE2A_FINAL_REPORT.md`

> These files document prior automation planning but were produced before the
> verification-first policy. Treat them as reference material only.

## Next Steps
- Execute the DevOps agent workflow to regenerate validated runbooks.
- Capture CI build URLs and store summaries under `logs/devops/`.
- Promote this README to 🟡 once at least one complete pipeline execution is
  documented with accompanying evidence.
