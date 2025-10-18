---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - synchronization-agent
  - documentation-agent-v2
related_services:
  - src/sync
  - .speckit/state
verification_notes: >
  The synchronization modules exist in `src/sync/`; recent checksum validation
  outputs are not recorded.
---

🟡 **Partial** — Synchronization code is present, but operational reports need to
be attached.

# Prompt Synchronization Engine

## Overview

The Prompt Synchronization Engine ensures that all agents remain synchronized regardless of workflow entry point, interruptions, or restarts. It uses JSON-based state management with checksums for verification.

SpecKit now externalizes agent prompts through the manifest at `prompts/manifest.json`. Pair each workflow execution with the manifest version (for example by storing it in project metadata) so you can spot drift when prompts are updated or imported from another workspace. Run `npm run validate:prompts` before starting a workflow to ensure manifest metadata and template files are ready for synchronization.

## Architecture

### Components

1. **StateManager** - Manages JSON state files for each phase
2. **WorkflowCoordinator** - Orchestrates agent execution and phase transitions
3. **PromptExecutor** - Executes agent prompts with context
4. **SyncEngine** - Ensures state consistency across restarts

### State Directory Structure

```text
.speckit/
└── state/
  ├── project.json          # Overall project state
  ├── specification.json    # Specification agent output
  ├── architecture.json     # Architecture agent output
  ├── scaffolding.json      # Scaffolding agent output
  ├── development.json      # Development agent output
  ├── testing.json          # Testing agent output
  ├── deployment.json       # Deployment agent output
  ├── documentation.json    # Documentation agent output
  └── snapshots/            # State snapshots for rollback
  ├── specification-1234567890.json
  ├── architecture-1234567890.json
  ├── scaffolding-1234567890.json
  ├── development-1234567890.json
  ├── testing-1234567890.json
  ├── deployment-1234567890.json
  └── documentation-1234567890.json
```

Snapshots are timestamped ({phase}-{epoch}.json) so each phase can be restored independently. For example, `specification-1234567890.json` preserves the most recent specification agent output, while `architecture-1234567890.json` captures the paired architecture plan. Downstream phases such as `scaffolding-1234567890.json`, `testing-1234567890.json`, and `documentation-1234567890.json` follow the same naming pattern for consistent rollback across the workflow.

### Snapshot Utilities

Regenerate the sample snapshots with fresh timestamps using the CLI helper:

```powershell
npm run generate-snapshots
```

Pass `--partial-testing` to capture an interim failure scenario where the testing agent blocks deployment until defects are resolved:

```powershell
npm run generate-snapshots -- --partial-testing
```

Use `--suffix`, `--phases`, and `--force` flags to customize output or overwrite existing examples. Snapshots default to `.speckit/state/snapshots` but the target directory can be overridden with `--output` when needed.

## State File Formats

### project.json

```json
{
  "projectId": "string",
  "createdAt": "ISO-8601",
  "updatedAt": "ISO-8601",
  "currentPhase": "specification|architecture|scaffolding|development|testing|deployment|documentation",
  "phases": {
    "specification": {
      "status": "not-started|in-progress|completed|failed",
      "checksum": "sha256-hash",
      "startedAt": "ISO-8601",
      "completedAt": "ISO-8601"
    },
    "architecture": {
      "status": "not-started|in-progress|completed|failed",
      "checksum": "sha256-hash",
      "startedAt": "ISO-8601",
      "completedAt": "ISO-8601"
    },
    "scaffolding": {
      "status": "not-started|in-progress|completed|failed",
      "checksum": "sha256-hash",
      "startedAt": "ISO-8601",
      "completedAt": "ISO-8601"
    },
    "development": { /* ... */ },
    "testing": { /* ... */ },
    "deployment": { /* ... */ },
    "documentation": { /* ... */ }
  }
}
```

### Phase State Files (scaffolding.json, development.json, etc.)

```json
{
  "agentId": "string",
  "status": "success|failed|partial",
  "timestamp": "ISO-8601",
  "artifacts": {
    /* Phase-specific artifacts */
  },
  "metadata": {
    /* Phase-specific metadata */
  },
  "nextAgent": "string|null",
  "syncState": {
    "phase": "string",
    "completed": boolean,
    "checksum": "sha256-hash",
    /* Phase-specific sync data */
  }
}
```

## Synchronization Protocol

### 1. Workflow Initialization

```typescript
// Check for existing state
const currentPhase = await stateManager.getCurrentPhase();

if (currentPhase) {
  // Resume from interruption
  await coordinator.resumeFromPhase(currentPhase);
} else {
  // Start new workflow
  await coordinator.start(prdData);
}
```

### 2. Phase Execution

```typescript
// Lock state for atomic operation
await stateManager.lockState(phase);

try {
  // Create snapshot before execution
  const snapshotId = await stateManager.createSnapshot('project');
  
  // Execute agent
  const output = await executeAgent(phase, input);
  
  // Validate output
  if (!validateOutput(phase, output)) {
    throw new Error('Invalid output');
  }
  
  // Save state with checksum
  const checksum = stateManager.calculateChecksum(output);
  output.syncState.checksum = checksum;
  await stateManager.writeState(phase, output);
  
  // Update project state
  await stateManager.updateProjectState(phase, 'completed');
} finally {
  // Always unlock
  await stateManager.unlockState(phase);
}
```

### 3. State Verification

```typescript
// Verify state integrity
const isValid = await stateManager.verifyChecksum(phase, expectedChecksum);

if (!isValid) {
  // Restore from snapshot
  await stateManager.restoreSnapshot(snapshotId);
}
```

### 4. Idempotent Operations

All operations are designed to be idempotent:

- Re-running a completed phase will skip it
- Checksums prevent duplicate work
- Lock files prevent concurrent modifications
- Snapshots enable rollback

## Usage Examples

### Starting a New Workflow

```typescript
import { WorkflowCoordinator } from '@sync/workflow-coordinator';

const coordinator = new WorkflowCoordinator('/path/to/project', 'project-123');
await coordinator.initialize();

// Process PRD and start workflow
const prdData = await processPRD('path/to/prd.md');
await coordinator.start(prdData);
```

### Resuming After Interruption

```typescript
// System automatically detects interruption
const coordinator = new WorkflowCoordinator('/path/to/project', 'project-123');
await coordinator.initialize();

// Resumes from last checkpoint
await coordinator.start({}); // PRD data loaded from state
```

### Manual Phase Execution

```typescript
// Execute specific phase
const scaffoldingInput = {
  projectType: 'web-app',
  techStack: { language: 'TypeScript', framework: 'React' },
  // ...
};

await coordinator.executePhase('scaffolding', scaffoldingInput);
```

### Rollback to Previous Phase

```typescript
// Rollback to development phase
await coordinator.rollback('development');

// This resets testing, deployment, and documentation phases
```

## Event System

The coordinator emits events for monitoring and integration:

```typescript
coordinator.on('execute-agent', ({ phase, input }) => {
  console.log(`Executing ${phase} agent`);
});

coordinator.on('phase-completed', ({ phase, output }) => {
  console.log(`${phase} completed successfully`);
});

coordinator.on('phase-failed', ({ phase, error }) => {
  console.error(`${phase} failed:`, error);
});

coordinator.on('workflow-completed', () => {
  console.log('Workflow completed!');
});
```

## Error Handling

### State Lock Timeout

```typescript
// Check for stale locks
const isLocked = await stateManager.isLocked(phase);

if (isLocked) {
  // Check lock age
  const lockPath = path.join(stateDir, `${phase}.lock`);
  const stats = await fs.stat(lockPath);
  const age = Date.now() - stats.mtimeMs;
  
  if (age > 300000) { // 5 minutes
    // Force unlock stale lock
    await stateManager.unlockState(phase);
  }
}
```

### Corrupted State Recovery

```typescript
// Validate state integrity
const state = await stateManager.readState(phase);

if (!stateManager.validateState(state, requiredFields)) {
  // Restore from latest snapshot
  const snapshots = await fs.readdir(snapshotsDir);
  const latest = snapshots.sort().reverse()[0];
  await stateManager.restoreSnapshot(latest);
}
```

## Integration with GitHub Actions

### Workflow Trigger

```yaml
- name: Start Workflow
  run: |
    node dist/cli.js workflow start \
      --prd prd/project.md \
      --project-id ${{ github.run_id }}
```

### Resume from Checkpoint

```yaml
- name: Resume Workflow
  run: |
    node dist/cli.js workflow resume \
      --project-id ${{ github.run_id }}
```

### Phase-Specific Execution

```yaml
- name: Execute Development Phase
  run: |
    node dist/cli.js workflow execute \
      --phase development \
      --project-id ${{ github.run_id }}
```

## Best Practices

1. **Always Lock State**: Use locks for all write operations
2. **Create Snapshots**: Before each phase execution
3. **Verify Checksums**: After reading state files
4. **Handle Interruptions**: Design for graceful resume
5. **Emit Events**: For monitoring and debugging
6. **Validate Output**: Before saving state
7. **Clean Up Locks**: Always unlock in finally blocks
8. **Test Idempotency**: Ensure operations can be repeated

## Troubleshooting

### Workflow Stuck

```bash
# Check current state
node dist/cli.js workflow status

# Force unlock if stuck
node dist/cli.js workflow unlock --phase development

# Reset phase and restart
node dist/cli.js workflow reset --phase development
```

### State Corruption

```bash
# List snapshots
node dist/cli.js workflow snapshots

# Restore from snapshot
node dist/cli.js workflow restore --snapshot scaffolding-1234567890
```

### Checksum Mismatch

```bash
# Verify all checksums
node dist/cli.js workflow verify

# Recalculate checksums
node dist/cli.js workflow recalculate-checksums
```
