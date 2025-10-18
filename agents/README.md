---
status: "🚧 Planned"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - documentation-agent-v2
  - orchestrator-agent
related_services:
  - .speckit/state/agents/registry.json
  - src/core/agent-orchestrator.ts
verification_notes: >
  The active agent registry is up to date, but supporting prompt refinements and
  operational runbooks remain archived pending review.
---

🚧 **Planned** — Agent prompt refinements require validation before being
restored to the primary documentation set.

# Agent Documentation Overview

## Scope
- Maintain visibility into archived agent prompt design documents.
- Track verification tasks required before reintroducing prompt guidance into the
  active workflow.

## Evidence Reviewed
- Verified the live agent registry stored in `.speckit/state/agents/registry.json`.
- Confirmed orchestration logic in `src/core/agent-orchestrator.ts` governs agent
  activation and sequencing.

## Archive Index
- `docs/archive/agents/AGENT_ACTIVATION_REGISTRY_DEPLOYMENT.md`
- `docs/archive/agents/AGENT_PROMPT_IMPROVEMENTS_2025-10-14.md`
- `docs/archive/agents/DEVELOPMENT_AGENT_V2_CRITICAL_TASK_DASHBOARD_FIX.md`
- `docs/archive/agents/FRONTEND_AGENT_V1_GENERATION_REPORT.md`
- `docs/archive/agents/OPTIMIZED_JIBONFLOW_BOOTSTRAP_PROMPT.md`

> Both documents contain legacy improvements that need alignment with current
> workflow manifests before reintegration.

## Next Actions
- Re-run prompt evaluations via `npm run agents:eval` and capture results under
  `logs/agents/`.
- Update prompt templates in `prompts/agents/` once evaluations succeed and
  governance checks pass.
- Promote this document to ✅ after validated prompt updates are merged.
