---
status: "🚧 Planned"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - development-agent-v2
  - documentation-agent-v2
related_services:
  - src/core/agent-orchestrator.ts
  - src/core/prompt-builder.ts
verification_notes: >
  Endpoint behaviour and schemas have not been validated against a running
  environment; legacy references are retained in the archive for audit only.
---

🚧 **Planned** — API references remain archived until live service contracts are
validated.

# API Overview

## Scope
- Provide a discoverable entry point to API assets pending verification.
- Track the outstanding work required before endpoint documentation is republished.

## Evidence Reviewed
- Confirmed presence of historical specs under `docs/archive/api/`, including
  `refill-service.openapi.json` and service-specific markdown files.
- Observed orchestration hooks in `src/core/prompt-builder.ts` that assemble API
  prompts for downstream agents.

## Verification Gaps
1. Launch service containers (see `scripts/` deployment utilities) and capture
   real `curl` traces for every documented endpoint.
2. Regenerate OpenAPI contracts from source using the latest service handlers in
   `src/` and compare with archived specs.
3. Add automated contract tests under `__tests__/` validating error payloads,
   pagination, and authentication flows.

## Archive Index
- `docs/archive/api/API_DOCUMENTATION.md`
- `docs/archive/api/auth-service.md`
- `docs/archive/api/error-reference.md`
- `docs/archive/api/telemedicine-service.md`
- `docs/archive/api/typescript-types.md`
- `docs/archive/api/OPENAPI_ENHANCEMENTS_GUIDE.md`
- `docs/archive/api/refill-service.openapi.json`

> Regenerate and re-verify each artifact before linking it back into the active
> documentation set.

## Next Actions
- Assign an API steward to coordinate contract tests with the development team.
- Publish updated Postman collections or Insomnia exports after successful
  verification.
- Promote this file to ✅ status once live endpoints and schemas have matching
  evidence snapshots.