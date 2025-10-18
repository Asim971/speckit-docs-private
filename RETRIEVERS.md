# Vector Storage & Retrieval Adapters

## Overview

SpecKit supports multiple vector storage backends for semantic search and retrieval-augmented generation (RAG). The adapter system provides a unified interface for document storage, similarity search, and health monitoring across different vector databases.

## Supported Adapters

### Chroma (Local Development)

**Best for:** Development, testing, and small-scale deployments

```json
{
  "adapters": [
    {
      "id": "chroma-local",
      "provider": "chroma",
      "type": "vector",
      "connection": {
        "host": "http://localhost:8000",
        "collection": "speckit-docs"
      }
    }
  ]
}
```

**Installation:**

```bash
npm install chroma-js
```

### Pinecone (Cloud Production)

**Best for:** Production workloads with high availability and scalability

```json
{
  "adapters": [
    {
      "id": "pinecone-prod",
      "provider": "pinecone",
      "type": "vector",
      "connection": {
        "environment": "us-east-1-aws",
        "project_id": "speckit-prod",
        "index": "speckit-docs"
      },
      "credentials": {
        "api_key_env": "PINECONE_API_KEY"
      }
    }
  ]
}
```

**Installation:**

```bash
npm install @pinecone-database/pinecone
```

### DynamoDB/Redis (Hybrid)

**Best for:** Cost-effective hybrid storage with metadata in DynamoDB and embeddings in Redis

```json
{
  "adapters": [
    {
      "id": "dynamo-redis",
      "provider": "dynamo-redis",
      "type": "hybrid",
      "connection": {
        "dynamodb_table": "speckit-docs",
        "redis_url": "redis://localhost:6379"
      }
    }
  ]
}
```

**Installation:**

```bash
npm install @aws-sdk/client-dynamodb ioredis
```

## Configuration

### Retriever Configuration File

Configure adapters in `config/retrievers.config.json`:

```json
{
  "$schema": "./schemas/retriever-adapters.schema.json",
  "version": "1.0.0",
  "defaults": {
    "primaryAdapterId": "chroma-local",
    "fallbackAdapterId": "pinecone-prod",
    "strategy": "semantic",
    "defaultTopK": 12,
    "autoRerank": true
  },
  "adapters": [
    {
      "id": "chroma-local",
      "name": "Chroma Development",
      "provider": "chroma",
      "type": "vector",
      "enabled": true,
      "connection": {
        "host": "http://localhost:8000"
      },
      "capabilities": {
        "maxResults": 100,
        "supportsFilters": true,
        "supportsBatch": true
      }
    }
  ]
}
```

### Adapter Interface

All adapters implement the `RetrieverAdapter` interface:

```typescript
interface RetrieverAdapter {
  initialize(config: RetrieverConfig): Promise<void>;
  addDocuments(documents: VectorDocument[]): Promise<void>;
  search(query: string, options: SearchOptions): Promise<SearchResult[]>;
  deleteDocuments(ids: string[]): Promise<void>;
  healthCheck(): Promise<{ healthy: boolean; message?: string }>;
  close(): Promise<void>;
}
```

## Usage in Prompts

### Declaring Retrieval Requirements

Prompts can declare required retrieval backends in their manifests:

```json
{
  "prompts": [
    {
      "id": "architecture-agent",
      "name": "Architecture Agent",
      "retrieval": {
        "required": true,
        "backends": ["vector", "keyword"],
        "min_results": 5,
        "max_results": 20
      }
    }
  ]
}
```

### Runtime Retrieval

Agents access retrieval through the orchestrator:

```typescript
// Search for relevant documents
const results = await orchestrator.searchDocuments(query, {
  backend: "chroma-local",
  limit: 10,
  filter: { category: "architecture" },
});

// Add new documents to the index
await orchestrator.addDocuments(documents, {
  backend: "pinecone-prod",
  collection: "project-docs",
});
```

## Document Format

### VectorDocument Structure

```typescript
interface VectorDocument {
  id: string;
  content: string;
  metadata?: Record<string, any>;
  embedding?: number[]; // Optional, computed if not provided
}
```

### Search Results

```typescript
interface SearchResult {
  document: {
    id: string;
    content: string;
    metadata?: Record<string, any>;
  };
  score: number; // Similarity score
}
```

## Health Monitoring

### Health Checks

Monitor adapter health:

```bash
# Check all adapters
npm run bootstrap  # Includes health checks

# Manual health check
npm run telemetry:report -- --category retrievers
```

### Health Status

Health checks return:

- **healthy**: Boolean status
- **message**: Optional diagnostic information
- **latency**: Response time metrics
- **error_rate**: Recent error percentages

## Performance Optimization

### Indexing Strategies

- **Batch Processing**: Use batch operations for large document sets
- **Incremental Updates**: Update indexes incrementally rather than full rebuilds
- **Metadata Filtering**: Use metadata filters to narrow search scope

### Caching

Enable caching for improved performance:

```json
{
  "storage": {
    "cache_enabled": true,
    "cache_ttl": 3600000,
    "cache_strategy": "lru"
  }
}
```

### Connection Pooling

Configure connection pooling for high-throughput scenarios:

```json
{
  "adapters": [
    {
      "connection": {
        "pool_size": 10,
        "max_idle_time": 30000,
        "acquire_timeout": 60000
      }
    }
  ]
}
```

## Migration & Backup

### Data Migration

Migrate data between adapters:

```bash
# Export from source adapter
npm run retrievers:export -- --adapter chroma-local --output data.json

# Import to target adapter
npm run retrievers:import -- --adapter pinecone-prod --input data.json
```

### Backup Strategies

- **Regular Snapshots**: Schedule regular index snapshots
- **Cross-Region Replication**: Replicate indexes across regions
- **Versioned Backups**: Maintain versioned backups for rollback

## Troubleshooting

### Common Issues

**Connection Failures:**

- Verify endpoint URLs and credentials
- Check network connectivity and firewalls
- Validate authentication tokens

**Search Quality Issues:**

- Review embedding model compatibility
- Check document preprocessing
- Validate similarity metrics

**Performance Problems:**

- Monitor query latency and throughput
- Optimize batch sizes and concurrency
- Consider index partitioning

### Debug Commands

```bash
# View adapter status
npm run telemetry:dashboard -- --category retrievers

# Test search functionality
npm run retrievers:test -- --adapter chroma-local --query "test query"

# Validate configuration
npm run validate:retrievers
```

## Security Considerations

### Access Control

- Use environment variables for credentials
- Implement proper authentication
- Restrict network access to vector databases

### Data Encryption

- Encrypt data at rest and in transit
- Use secure connections (HTTPS, TLS)
- Implement proper key management

### Audit Logging

- Log all retrieval operations
- Monitor for unusual access patterns
- Implement retention policies for logs
