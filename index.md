---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - documentation-agent-v2
  - governance-agent
related_services:
  - docs/
  - .speckit/state/metadata.json
  - scripts/
verification_notes: >
  Root-level documentation was inventoried and relocated into domain-specific
  folders. Archive boundaries are now enforced, but domain evidence still
  requires refresh before ✅ promotion.
---

🟡 **Partial** — Navigation is updated and archives are organized; several
sections remain pending verification evidence.

# Documentation Directory Index

## Status Snapshot
| Domain | Status | Primary Reference | Last Verified | Notes |
| --- | --- | --- | --- | --- |
| Architecture | 🚧 | [docs/architecture/README.md](architecture/README.md) | 2025-10-18 | Blueprints archived pending revalidation |
| Deployment | 🚧 | [docs/deployment/README.md](deployment/README.md) | 2025-10-18 | Awaiting verified runbooks and rollback proof |
| DevOps | 🚧 | [docs/devops/README.md](devops/README.md) | 2025-10-18 | Pipelines need CI evidence before publication |
| Orchestration | 🟡 | [docs/orchestration/README.md](orchestration/README.md) | 2025-10-18 | Code paths verified; telemetry attachments pending |
| Security & Governance | 🟡 | [docs/security/README.md](security/README.md) | 2025-10-18 | Policies verified without audit artefacts |
| Testing | 🟡 | [docs/testing/README.md](testing/README.md) | 2025-10-18 | Unit tests verified; E2E coverage outstanding |
| Agents | 🚧 | [docs/agents/README.md](agents/README.md) | 2025-10-18 | Prompt updates require new evaluations |
| Onboarding | 🚧 | [docs/onboarding/README.md](onboarding/README.md) | 2025-10-18 | External dependencies need confirmation |
| User Guides | ❌ | [docs/user-guides/README.md](user-guides/README.md) | 2025-10-18 | No verified user-facing guides available |
| API | 🚧 | [docs/api/README.md](api/README.md) | 2025-10-18 | Endpoint contracts await live validation |

Refer to [`docs/DOCUMENTATION_STATUS_REPORT.md`](DOCUMENTATION_STATUS_REPORT.md)
for aggregated badge counts and review notes.

## Archive Overview
- All historical deployment artefacts now live under
  `docs/archive/deployment/`.
- DevOps planning records moved to `docs/archive/devops/`.
- Orchestrator and phase activation reports consolidated in
  `docs/archive/orchestration/`.
- Testing agent outputs and verification checklists reside in
  `docs/archive/testing/`.
- Validation logs relocated to `docs/archive/validation/` for audit-only use.

> Legacy documents retain their original filenames for traceability but are no
> longer part of the verified navigation tree.

## Next Actions
- Regenerate evidence for 🚧 and ❌ domains, storing logs under the relevant
  `logs/` subdirectories.
- Promote sections once verification artefacts (tests, telemetry, runbooks) are
  attached and governance checks pass.
- Execute `npm run governance:check -- --policy documentation-quality` after
  each documentation update to enforce compliance.
