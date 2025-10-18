---
status: "🟡 Partial"
owner: "Documentation Agent v2"
last_verified: "2025-10-18"
related_agents:
  - telemetry-agent
  - documentation-agent-v2
related_services:
  - npm run telemetry:dashboard
  - logs/
verification_notes: >
  Telemetry commands are available, yet no recent reports are checked into the
  repository.
---

🟡 **Partial** — Telemetry tooling exists, but validated dashboards are pending.

# Telemetry & Observability

## Overview

SpecKit provides comprehensive telemetry and observability capabilities to monitor agent performance, track system health, and debug issues. The telemetry system captures structured spans, events, and metrics throughout the development lifecycle.

## Telemetry Architecture

### Core Components

- **Span Tracker**: Captures request lifecycle with timing and metadata
- **Event Logger**: Records discrete events and state changes
- **Metrics Collector**: Aggregates performance and health metrics
- **Dashboard Interface**: Provides real-time monitoring and reporting

### Data Flow

```text
Agent Request → Span Creation → Event Logging → Metrics Aggregation → Dashboard Display
```

## Span Tracking

### Span Structure

Spans capture complete request lifecycles:

```json
{
  "traceId": "abc123",
  "spanId": "def456",
  "parentSpanId": "parent123",
  "name": "agent-execution",
  "startTime": "2025-10-10T11:30:00.000Z",
  "endTime": "2025-10-10T11:30:05.123Z",
  "duration": 5123,
  "attributes": {
    "agent.id": "development-agent",
    "agent.type": "development",
    "workflow.id": "full-development-cycle",
    "input.tokens": 150,
    "output.tokens": 300
  },
  "events": [
    {
      "name": "prompt_loaded",
      "timestamp": "2025-10-10T11:30:00.500Z",
      "attributes": { "template": "code-generation" }
    }
  ]
}
```

### Automatic Span Creation

Spans are automatically created for:

- Agent executions
- Workflow steps
- Bootstrap operations
- Validation checks
- Retrieval operations

## Event Logging

### Event Types

The system logs various event types:

- **Agent Events**: Registration, execution, errors
- **Workflow Events**: Start, completion, failures
- **System Events**: Bootstrap, validation, health checks
- **User Events**: CLI commands, configuration changes

### Event Structure

```json
{
  "timestamp": "2025-10-10T11:30:00.000Z",
  "level": "info",
  "event": "agent_execution_completed",
  "context": {
    "agentId": "development-agent",
    "workflowId": "full-development-cycle",
    "duration": 5123,
    "success": true
  },
  "metadata": {
    "traceId": "abc123",
    "userId": "user123",
    "sessionId": "session456"
  }
}
```

## Telemetry Commands

### Dashboard

Interactive telemetry dashboard:

```bash
npm run telemetry:dashboard
npm run telemetry:dashboard -- --category agents
npm run telemetry:dashboard -- --time-range 1h
```

### Report Generation

Generate detailed reports:

```bash
# Summary report
npm run telemetry:report -- --summary

# Agent-specific report
npm run telemetry:report -- --agent development-agent

# Time-filtered report
npm run telemetry:report -- --since "2025-10-10T00:00:00Z"

# JSON output for integration
npm run telemetry:report -- --format json > telemetry.json
```

### Handoff Status

Track agent transitions and handoffs:

```bash
npm run handoff:status
npm run handoff:status -- --agent development-agent
npm run handoff:status -- --workflow full-development-cycle
```

## Configuration

### Telemetry Settings

Configure telemetry in `package.json`:

```json
{
  "speckit": {
    "observability": {
      "telemetry_enabled": true,
      "tracing_backend": "console",
      "metrics_interval": 30000,
      "log_level": "info",
      "span_retention_days": 30,
      "event_retention_days": 90
    }
  }
}
```

### Storage Locations

Telemetry data is stored in:

```text
logs/
├── telemetry-spans.log     # Structured span data
├── telemetry-events.log    # Event logs
└── telemetry-metrics.log   # Aggregated metrics

.speckit/state/
├── telemetry/              # Processed telemetry data
└── handoff-status.json     # Current handoff state
```

## Metrics & KPIs

### Agent Metrics

- **Execution Time**: Average and p95 response times
- **Success Rate**: Percentage of successful executions
- **Error Rate**: Frequency of failures by type
- **Throughput**: Requests per minute/hour

### Workflow Metrics

- **Completion Rate**: Percentage of workflows finishing successfully
- **Stage Duration**: Time spent in each workflow stage
- **Bottleneck Identification**: Longest-running stages
- **Retry Frequency**: How often stages need retries

### System Metrics

- **Memory Usage**: Heap and process memory consumption
- **CPU Utilization**: Core and overall CPU usage
- **Disk I/O**: Read/write operations for state management
- **Network I/O**: External API calls and data transfer

## Alerting & Monitoring

### Threshold Alerts

Configure alerts for metric thresholds:

```json
{
  "alerts": {
    "agent_latency_p95": {
      "threshold": 10000,
      "operator": ">",
      "severity": "warning"
    },
    "error_rate": {
      "threshold": 0.05,
      "operator": ">",
      "severity": "error"
    }
  }
}
```

### Health Checks

Automated health monitoring:

```bash
# Run health checks
npm run health:check

# Include in CI/CD
npm run health:check -- --fail-on-warnings
```

## Integration & Export

### External Systems

Export telemetry to external monitoring systems:

```bash
# Export to Datadog
npm run telemetry:export -- --format datadog --destination datadog.json

# Export to Prometheus
npm run telemetry:export -- --format prometheus --endpoint http://prometheus:9090

# Export to Elasticsearch
npm run telemetry:export -- --format elasticsearch --index speckit-telemetry
```

### API Access

Programmatic access to telemetry data:

```typescript
import { TelemetryService } from "./src/utils/telemetry.js";

// Query spans
const spans = await TelemetryService.querySpans({
  agentId: "development-agent",
  timeRange: { start: "2025-10-10T00:00:00Z", end: "2025-10-10T23:59:59Z" },
});

// Get metrics
const metrics = await TelemetryService.getMetrics({
  category: "agents",
  period: "1h",
});
```

## Privacy & Security

### Data Protection

- **PII Filtering**: Automatically filter sensitive data from logs
- **Retention Policies**: Configurable data retention periods
- **Access Control**: Role-based access to telemetry data
- **Encryption**: Encrypt telemetry data at rest and in transit

### Compliance

- **GDPR Compliance**: Data minimization and user consent
- **Audit Trails**: Complete audit logs for compliance
- **Data Export**: Allow users to export their telemetry data
- **Anonymization**: Optional data anonymization features

## Troubleshooting

### Common Issues

**Missing Telemetry Data:**

- Check telemetry configuration
- Verify log file permissions
- Ensure sufficient disk space

**Performance Impact:**

- Adjust metrics collection intervals
- Enable sampling for high-volume scenarios
- Use external telemetry backends

**Data Quality Issues:**

- Validate span completeness
- Check for dropped events
- Monitor data parsing errors

### Debug Mode

Enable detailed telemetry logging:

```bash
DEBUG=telemetry:* npm run telemetry:dashboard
SPECKIT_TELEMETRY_LEVEL=debug npm run bootstrap
```

## Best Practices

### Implementation Guidelines

1. **Consistent Naming**: Use consistent span and event names
2. **Rich Metadata**: Include relevant context in attributes
3. **Error Handling**: Properly capture and log errors
4. **Performance**: Minimize telemetry overhead

### Monitoring Strategy

1. **Define KPIs**: Establish clear success metrics
2. **Set Baselines**: Understand normal operating ranges
3. **Alert Configuration**: Set up meaningful alerts
4. **Regular Review**: Periodically review and adjust monitoring

### Dashboard Usage

1. **Real-time Monitoring**: Use dashboard for immediate insights
2. **Trend Analysis**: Review historical data for patterns
3. **Incident Response**: Use telemetry for debugging issues
4. **Capacity Planning**: Monitor resource usage trends
