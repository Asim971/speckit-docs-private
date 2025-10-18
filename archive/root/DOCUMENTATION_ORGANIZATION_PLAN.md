# Documentation Organization Plan

## Objective
- Activate Documentation Agent v2.0 to consolidate and label all material under `docs/`
- Ensure every artifact reflects verification status in line with the agent's verification-first mandate
- Reduce onboarding friction by curating a single entry point and archived history for legacy outputs

## Execution Update (2025-10-18)
- Domain READMEs reviewed; new DevOps and Orchestration hubs created with front
   matter metadata.
- `docs/index.md` now provides a status snapshot for architecture, deployment,
   security, testing, onboarding, user guides, and API domains.
- Root-level orchestration, deployment, DevOps, testing, onboarding, and
   validation artefacts relocated to `docs/archive/` preserving audit history.
- Governance validation (`npm run governance:check`) passed with no violations;
   evidence stored at `logs/governance/documentation-quality-20251018.txt`.

## Current State Findings
- Mixed agent artifacts (blueprints, summaries, governance notes) live in a flat hierarchy without status signals
- Implementation evidence and executive summaries coexist, causing search overhead for engineering and compliance teams
- Historical reports still contain outdated metrics with no archive boundary, increasing risk of policy drift

## Required Actions (Documentation Agent v2.0)
1. **Inventory & Verification Sweep**
   - Catalog all Markdown files, grouping by domain (`architecture`, `governance`, `deployment`, etc.)
   - Verify each document references currently implemented functionality; mark unverifiable items as 🚧 Planned or ❌ Not Started
2. **Status Badge Application**
   - Apply the standard ✅/🟡/🚧/❌ badges at the top of each document per verification outcome
   - Capture evidence snippets (test logs, API curl results) where applicable to support ✅ badges
3. **Information Architecture Refresh**
   - Create `docs/index.md` summarizing curated sections with links to domain hubs
   - Move superseded or historical artifacts into `docs/archive/` preserving audit value without cluttering primary navigation
   - Introduce per-domain README files outlining ownership, verification cadence, and last review timestamp
4. **Cross-Linking & Metadata**
   - Ensure each document references its related services/apps (e.g., `services/auth-service`, `apps/patient-portal`)
   - Add front-matter metadata (owner, last_verified, related_agents) where governance rules require traceability
5. **Governance Validation**
   - Run `npm run governance:check -- --policy documentation-quality` after reorganizing to confirm compliance
   - Store validation evidence in `logs/governance/` with a timestamped note

## Deliverables
- Updated, badge-compliant documents across `docs/`
- A generated `docs/DOCUMENTATION_STATUS_REPORT.md` capturing counts of ✅/🟡/🚧/❌ artifacts
- Archived legacy content under `docs/archive/` with index referencing original locations
- Recorded governance validation output for audit purposes

## Acceptance Criteria
- Every document top-level directory contains an overview README with verification badge and ownership metadata
- Navigation index (`docs/index.md`) provides ≤3 clicks to reach any primary artifact
- Governance checks pass without warnings for documentation quality or secret leakage
- Documentation Agent produces a handoff bundle confirming evidence links for each ✅ artifact
