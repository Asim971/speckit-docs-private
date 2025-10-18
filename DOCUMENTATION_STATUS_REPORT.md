---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - documentation-agent-v2
related_services:
  - docs/index.md
verification_notes: >
  Status counts are derived from the domain overview READMEs created or
  refreshed on 2025-10-18. Evidence attachments are pending for promotions to
  ✅.
---

🟡 **Partial** — Documentation inventory is tracked; several domains still need
verification artefacts before reaching ✅.

# Documentation Status Report

## Badge Counts
- ✅ Working: 0
- 🟡 Partial: 3
- 🚧 Planned: 6
- ❌ Not Started: 1

> Counts reflect the current domain-summary pages listed in `docs/index.md`.

## Domain Breakdown
| Domain | Status | Notes |
| --- | --- | --- |
| Architecture | 🚧 | Blueprints archived; awaiting regenerated diagrams |
| Deployment | 🚧 | Needs validated runbooks and rollback evidence |
| DevOps | 🚧 | CI/CD pipelines require live execution logs |
| Orchestration | 🟡 | Implementation verified; telemetry attachments missing |
| Security & Governance | 🟡 | Policies confirmed without audit exports |
| Testing | 🟡 | Unit tests pass; E2E coverage outstanding |
| Agents | 🚧 | Prompt evaluations pending rerun |
| Onboarding | 🚧 | External dependencies need confirmation |
| User Guides | ❌ | No verified end-user manuals available |
| API | 🚧 | Endpoint behaviour unverified in current environment |

## Next Review
- Recompute status counts after each domain moves to ✅.
- Attach supporting evidence (logs, test reports) to promote sections from 🚧 or
  🟡.
- Log governance check outputs under `logs/governance/` for each update cycle.
