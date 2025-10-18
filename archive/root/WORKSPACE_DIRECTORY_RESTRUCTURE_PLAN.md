# Workspace Directory Restructure Plan

## Objective
- Provide the Workspace Architecture Agent with a structured blueprint to declutter the repository
- Align directory ownership with functional areas (frontend apps, backend services, shared packages, orchestration assets)
- Preserve auditability mandated by governance policies while improving discoverability for engineering teams

## Current Observations
- `docs/` mixes active runbooks with historical phase reports at the same hierarchy level
- `Jira_Management/jibonflow/apps/` and `Jira_Management/jibonflow/services/` contain numerous agent-generated summaries that should move alongside implementation source
- Root-level deployment and orchestration status files (e.g., `DEPLOYMENT_*`, `ORCHESTRATOR_*`) crowd the repository entry point
- Shared configuration and prompt assets reside in disparate locations without an index (e.g., `prompt_system`, `.speckit/state`)

## Recommended Target Structure
1. **Project Top Level**
   - `docs/` (active knowledge base)
   - `docs/archive/` (historical reports)
   - `orchestration/` (move orchestrator + agent lifecycle outputs from scattered Markdown files)
   - `delivery/` (deployment manifests, execution summaries)
2. **JibonFlow Monorepo (`Jira_Management/jibonflow/`)
   - `apps/` (Next.js apps) — ensure each app has `README` with implementation badge
   - `services/` (Node services) — co-locate architecture summaries under `services/<name>/docs/`
   - `packages/` (shared libraries)
   - `ops/` (Docker, CI scripts currently mixed into services)
3. **Governance & Policies**
   - Centralize policies under `governance/` with symbolic links from legacy locations to maintain tooling compatibility
   - Introduce `governance/README.md` summarizing enforcement commands
4. **Generated/Agent Artifacts**
   - Create `generated/agents/` for activation banners, phase summaries, and dashboards currently spread across root
   - Link to the canonical status dashboard from `README.md`

## Action Plan (Workspace Architecture Agent)
1. **Discovery Phase**
   - Run workspace scanner to map current directories and detect references (`npm run workspace:scan` once registered)
   - Identify hard-coded paths within scripts/tests that would break if directories move
2. **Proposal Phase**
   - Produce a phased migration plan prioritizing non-breaking archival moves (e.g., relocating Markdown reports)
   - Validate proposed structure against `AGENT_LIFECYCLE.md` registration requirements and `GOVERNANCE.md` policy constraints
3. **Execution Phase**
   - Implement symbolic links or update config references where necessary to keep CI pipelines functional
   - Coordinate with Documentation Agent to ensure doc links remain accurate after moves
4. **Validation Phase**
   - Run `npm run governance:check` and relevant test suites to confirm no regressions
   - Update `orchestrator_meta.json` with completion details and handoff to repository maintainers

## Risks & Mitigations
- **Broken References**: Mitigate via comprehensive search for moved paths and update in CI scripts/configs
- **Governance Drift**: Maintain audit logs of every relocation under `logs/governance/`
- **Agent Registry Impact**: Re-run `npm run agents:register` post-move to refresh agent manifests referencing relocated prompts

## Success Metrics
- Root directory reduces document clutter by ≥70%
- All critical knowledge assets accessible within three clicks from repo root `README`
- Governance and bootstrap commands succeed without path adjustments
- Updated orchestrator metadata reflects phase completion with validation evidence
