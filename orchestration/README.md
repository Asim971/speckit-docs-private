---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - orchestration-agent
  - documentation-agent-v2
  - governance-agent
related_services:
  - src/core/agent-orchestrator.ts
  - src/core/orchestrator-config.ts
  - .speckit/state/orchestration/
verification_notes: >
  Confirmed the orchestrator implementation and workflow registry exist in the
  repository. Execution evidence for the latest phase runs remains stored
  outside source control, so real-time telemetry has not been attached.
---

🟡 **Partial** — Orchestrator code paths are verified, yet runtime evidence and
telemetry attachments are pending for a ✅ promotion.

# Orchestrator Overview

## Scope
- Summarise the SpecKit agent orchestration engine and its governance hooks.
- Provide a curated entry point to historical activation logs now archived under
  `docs/archive/orchestration/`.
- Track the backlog of evidence required before reinstating end-to-end workflow
  runbooks.

## Verified Assets
- `src/core/agent-orchestrator.ts` implements the event-driven agent pipeline
  used across specification through deployment phases.
- `src/core/orchestrator-config.ts` contains workflow routing rules validated by
  existing unit tests under `__tests__/`.
- `.speckit/state/orchestration/` captures the most recent workflow session
  snapshots, including `session_orch_20251018_002.json`.

## Evidence Gaps
1. Attach the latest orchestrator execution traces (from `logs/` or hosted
   telemetry) demonstrating successful multi-agent handoffs.
2. Include governance validation output proving policy compliance gates fire at
   each phase boundary.
3. Capture retry behaviour for failed agent responses to document resilience
   guarantees.

## Archive Index
The following historical artefacts have been relocated for reference only:

- `docs/archive/orchestration/ORCHESTRATOR_*`
- `docs/archive/orchestration/PHASE2*` and `PHASE_*` activation summaries
- `docs/archive/orchestration/ORCHESTRATION_DECISION_FINAL.txt`

> Archived files pre-date the verification-first mandate. Do not treat them as
> authoritative runbooks without fresh evidence.

## Next Steps
- Re-run orchestration smoke tests and store output under `logs/orchestration/`.
- Generate a ✅ handoff bundle once telemetry and governance proofs are attached.
- Update this README to ✅ when a full workflow replay (spec → deployment) is
  documented with evidence links.
