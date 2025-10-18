---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - agent-lifecycle
  - documentation-agent-v2
related_services:
  - .speckit/state/agents/registry.json
  - npm run agents:eval
verification_notes: >
  Registry data is present; evaluation history and retraining outputs have not
  been captured under logs/ or evidence/.
---

🟡 **Partial** — Lifecycle processes are defined, but recent evaluation evidence
is pending.

# Agent Lifecycle Management

## Overview

SpecKit provides comprehensive lifecycle management for AI agents, including registration, evaluation, retraining, and metrics tracking. The agent lifecycle system ensures agents remain effective and up-to-date throughout their operational lifetime.

## Agent Registration

### Automatic Registration

Agents are automatically registered during workspace bootstrap:

```bash
npm run bootstrap
```

This process:

- Scans the `prompt_system/agents/` directory for agent definitions
- Validates agent manifests against the schema
- Registers agents with the orchestrator
- Updates the agent registry at `.speckit/state/agents/registry.json`

### Manual Registration

For manual registration or updates:

```bash
npm run agents:register
```

## Agent Evaluation

### Running Evaluations

Evaluate agent performance using the evaluation suite:

```bash
npm run agents:eval
npm run agents:eval -- --agent development-agent
npm run agents:eval -- --category accuracy
```

### Evaluation Metrics

The system tracks comprehensive metrics:

- **Accuracy**: Correctness of agent outputs
- **Latency**: Response time performance
- **Reliability**: Consistency of results
- **Relevance**: Appropriateness for given contexts
- **Safety**: Adherence to governance policies

### Metrics Storage

Evaluation results are persisted in `.speckit/state/agents/`:

```text
.speckit/state/agents/
├── registry.json          # Agent registry
├── development-agent.json # Individual agent metrics
├── testing-agent.json
└── eval-history.json      # Historical evaluation data
```

## Agent Retraining

### Triggering Retraining

Retraining is triggered based on:

- Performance degradation below thresholds
- New training data availability
- Manual retraining commands

```bash
npm run agents:retrain
npm run agents:retrain -- --agent development-agent
npm run agents:retrain -- --force
```

### Retraining Process

1. **Data Collection**: Gather new training examples
2. **Model Update**: Update agent models with new data
3. **Validation**: Run evaluation suite on updated models
4. **Deployment**: Deploy updated models if validation passes

## Agent Health Monitoring

### Health Checks

Monitor agent health through telemetry:

```bash
npm run telemetry:dashboard -- --category agents
npm run telemetry:report -- --agent development-agent
```

### Automated Health Checks

The system automatically:

- Monitors agent performance metrics
- Triggers retraining when thresholds are breached
- Logs health status in telemetry streams
- Provides alerts for degraded performance

## Agent Configuration

### Agent Manifest Structure

Each agent is defined by a manifest:

```json
{
  "id": "development-agent",
  "name": "Development Agent",
  "type": "development",
  "version": "1.0.0",
  "capabilities": ["code-generation", "refactoring"],
  "evaluation": {
    "accuracy_threshold": 0.85,
    "latency_budget_ms": 5000
  },
  "retraining": {
    "trigger_threshold": 0.8,
    "min_samples": 100
  }
}
```

### Configuration Validation

Validate agent configurations:

```bash
npm run validate:prompts -- --agents-only
```

## Troubleshooting

### Common Issues

**Agent Not Registering:**

- Check manifest syntax
- Verify prompt templates exist
- Ensure unique agent IDs

**Poor Evaluation Scores:**

- Review training data quality
- Check for prompt drift
- Validate evaluation criteria

**Retraining Failures:**

- Verify training data availability
- Check model compatibility
- Review error logs

### Debug Commands

````bash
# View agent registry
cat .speckit/state/agents/registry.json

# Check agent metrics
npm run telemetry:report -- --agent <agent-id> --format json

# Validate agent manifests
npm run validate:prompts
```</content>
<parameter name="filePath">d:\Asim's Prompt\agent\docs\AGENT_LIFECYCLE.md
````
