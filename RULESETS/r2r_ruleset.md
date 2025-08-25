---
trigger: glob
description: Comprehensive coding rules for R2R (RAG to Riches) production-ready RAG system with advanced search and knowledge graph capabilities
globs: 
  - "**/*.py"
  - "**/*.js"
  - "**/*.ts"
  - "**/*.toml"
  - "**/r2r*"
  - "**/docker-compose*.yaml"
  - "**/requirements*.txt"
  - "**/package*.json"
---

# R2R (RAG to Riches) Production RAG Ruleset

## Installation and Setup

### Quick Installation
- Install R2R via pip: `pip install r2r`
- For JavaScript: `npm install r2r-js`
- ALWAYS set required environment variables before starting:
  ```bash
  export OPENAI_API_KEY=sk-...
  # or for local models
  export OLLAMA_API_BASE=http://localhost:11434
  ```

### Deployment Modes
- **Light Mode**: Use `python -m r2r.serve` for development/prototyping
- **Full Mode**: Use Docker Compose for production with persistent services
- **Docker Command**: `r2r serve --docker` for containerized deployment
- **Production Setup**: Clone repository and use `compose.full.yaml` for complete infrastructure

### Environment Configuration
- Use `.env` files or TOML configuration for environment-specific settings
- Set `R2R_CONFIG_NAME=full` for OpenAI integration
- Set `R2R_CONFIG_NAME=full_ollama` for local model deployment
- Custom configs: Use `R2R_CONFIG_PATH` to point to custom TOML files

## Core Client Initialization

### Python Client Setup
```python
from r2r import R2RClient

# Basic initialization
client = R2RClient(base_url="http://localhost:7272")

# With API key for SciPhi Cloud
import os
os.environ["R2R_API_KEY"] = "your-api-key"
client = R2RClient()
```

### JavaScript Client Setup
```javascript
const { r2rClient } = require('r2r-js');
const client = new r2rClient("http://localhost:7272");
```

### Health Check Pattern
- ALWAYS verify connection with health check:
  ```python
  health_response = client.health()
  assert health_response["status"] == "ok"
  ```

## Document Management

### Document Ingestion Best Practices
- **Single Document**: Use `client.documents.create(file_path="/path/to/file")`
- **Batch Processing**: For multiple files, iterate with individual calls
- **Metadata Enhancement**: Always include relevant metadata for better retrieval
- **File Types Supported**: txt, pdf, json, png, mp3, docx, xlsx, and 40+ formats via Unstructured integration

### Document Operations
```python
# Ingest with metadata
ingestion_response = client.documents.create(
    file_path="/path/to/document.pdf",
    metadata={"title": "Document Title", "category": "research"}
)

# List documents
documents = client.documents.list()

# Update document
client.documents.update(document_id, metadata=new_metadata)
```

### File Processing Patterns
- Use temporary files for downloaded content before ingestion
- Clean up temporary files after successful ingestion
- Handle ingestion errors gracefully with try-catch blocks
- Monitor ingestion logs for vector count and processing status

## Search and Retrieval

### Search Configuration
- **Basic Search**: `client.retrieval.search(query="search terms")`
- **Hybrid Search**: Enable both semantic and keyword search for optimal results
- **Search Limits**: Configure appropriate limits (default 10, production 25-50)
- **Filters**: Apply document-level or metadata filters for scoped searches

### RAG Implementation Patterns
```python
# Basic RAG
response = client.retrieval.rag(
    query="What is the nature of consciousness?",
    rag_generation_config={
        "model": "gpt-4o-mini",
        "temperature": 0.0,
        "max_tokens_to_sample": 2048
    },
    search_settings={
        "use_hybrid_search": True,
        "limit": 25
    }
)

# Streaming RAG for real-time applications
streaming_response = client.retrieval.rag(
    query="Complex analysis question",
    rag_generation_config={
        "stream": True,
        "model": "anthropic/claude-3.5-sonnet"
    }
)
```

### Advanced Retrieval Features
- **Knowledge Graph Search**: Enable `use_kg_search=True` for relationship-aware retrieval
- **Vector Search**: Use `use_vector_search=True` for semantic similarity
- **Reciprocal Rank Fusion**: Combines multiple retrieval methods for better relevance

## Authentication and User Management

### User Authentication Pattern
```python
# Register new user
client.register("user@example.com", "secure_password")

# Verify email (if enabled)
client.verify_email("user@example.com", "verification_code")

# Login
client.login("user@example.com", "secure_password")
```

### Access Control
- Authenticated users automatically have access restricted to their documents
- Use collections for document organization and access control
- Implement user-specific document filtering in production applications

## Configuration Management

### TOML Configuration Structure
```toml
[database]
provider = "postgres"
user = "postgres"
password = "your_password"
host = "localhost"
port = 5432
db_name = "r2r"

[embedding]
provider = "openai"
model = "text-embedding-3-small"
dimension = 1536

[llm]
provider = "openai"
model = "gpt-4o-mini"
temperature = 0.0
```

### Configuration Best Practices
- Use environment variables for sensitive data (API keys, passwords)
- Create custom TOML files for different environments (dev, staging, prod)
- Override configurations at runtime for dynamic behavior
- Validate configurations before deployment

## GraphRAG and Knowledge Graphs

### Knowledge Graph Configuration
- Enable automatic entity and relationship extraction during ingestion
- Configure knowledge graph providers (Neo4j, PostgreSQL with graph extensions)
- Use graph-based retrieval for complex relationship queries
- Implement community detection for improved context clustering

### GraphRAG Implementation
```python
# Enable knowledge graph features
response = client.retrieval.rag(
    query="How do these concepts relate?",
    rag_generation_config={
        "model": "gpt-4o",
        "temperature": 0.1
    },
    search_settings={
        "use_kg_search": True,
        "use_hybrid_search": True,
        "kg_generation_config": {
            "max_knowledge_triples": 100
        }
    }
)
```

### Graph Processing Patterns
- Batch process large documents for graph construction
- Monitor graph construction metrics (entities, relationships, processing time)
- Implement graph partitioning for large-scale deployments
- Cache frequently accessed graph patterns

## Agentic RAG and Deep Research

### Deep Research Agent Usage
```python
# Multi-step reasoning with web integration
response = client.retrieval.agent(
    message={
        "role": "user", 
        "content": "Analyze market implications of recent AI developments"
    },
    rag_generation_config={
        "model": "anthropic/claude-3.5-sonnet",
        "extended_thinking": True,
        "thinking_budget": 4096,
        "temperature": 0.7,
        "max_tokens_to_sample": 8192
    }
)
```

### Agent Tool Configuration
- Configure web search tools with `SERPER_API_KEY`
- Enable web scraping with `FIRECRAWL_API_KEY`
- Monitor agent execution for performance and cost optimization
- Implement tool usage policies for production environments

## Production Deployment Patterns

### Docker Configuration
```yaml
# docker-compose.yml
version: '3.8'
services:
  r2r:
    image: sciphi/r2r:latest
    environment:
      - R2R_CONFIG_NAME=full
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    ports:
      - "7272:7272"
    depends_on:
      - postgres
      - redis
```

### Environment Variables Management
- Use secure secret management for API keys
- Separate configurations for different deployment stages
- Implement proper environment validation before startup
- Monitor configuration drift in production

### Scaling Considerations
- Implement horizontal scaling for multiple R2R instances
- Use load balancers for high-availability deployments
- Configure database connection pooling for concurrent users
- Monitor memory usage for large document processing

## Error Handling and Monitoring

### Common Error Patterns
- **Database Connection Issues**: Verify PostgreSQL configuration and network connectivity
- **API Key Errors**: Validate environment variables and key permissions
- **Ingestion Failures**: Check file formats and processing limits
- **Authentication Errors**: Verify user credentials and session management

### Monitoring Best Practices
```python
import logging

# Configure logging for R2R operations
logging.basicConfig(level=logging.INFO)

try:
    response = client.retrieval.rag(query="test query")
    logging.info(f"RAG successful: {len(response['results']['completion'])} characters")
except Exception as e:
    logging.error(f"RAG failed: {str(e)}")
    # Implement fallback strategy
```

### Performance Metrics
- Track ingestion success rates and processing times
- Monitor search latency and retrieval quality
- Measure RAG response generation time and token usage
- Alert on database connection failures and API rate limits

## Security and Data Privacy

### Data Protection Patterns
- Implement document-level access controls
- Use HTTPS for all client-server communication
- Encrypt sensitive data in configuration files
- Regular security updates for dependencies

### Privacy Considerations
- Implement data retention policies for user documents
- Provide data deletion capabilities for GDPR compliance
- Monitor and log access patterns for audit trails
- Use secure authentication methods (JWT, OAuth)

## Performance Optimization

### Caching Strategies
- Implement vector embedding caching for repeated queries
- Cache frequent search results with appropriate TTL
- Use Redis for session and temporary data storage
- Implement intelligent cache invalidation for updated documents

### Resource Management
```python
# Optimize batch processing
def batch_ingest_documents(file_paths, batch_size=10):
    for i in range(0, len(file_paths), batch_size):
        batch = file_paths[i:i + batch_size]
        for file_path in batch:
            try:
                client.documents.create(file_path=file_path)
                time.sleep(0.1)  # Rate limiting
            except Exception as e:
                logging.error(f"Failed to ingest {file_path}: {e}")
```

### Query Optimization
- Use appropriate search limits based on use case
- Implement query preprocessing for better results
- Cache expensive graph traversals
- Monitor and optimize vector database performance

## Integration Patterns

### Web Application Integration
```javascript
// React/Next.js integration pattern
async function performRAG(query) {
    try {
        const response = await client.rag({
            query: query,
            rag_generation_config: {
                model: "gpt-4o-mini",
                temperature: 0.1,
                stream: false
            }
        });
        return response.results.completion;
    } catch (error) {
        console.error('RAG failed:', error);
        throw error;
    }
}
```

### API Integration
- Use official SDKs for reliable integration
- Implement proper error handling and retries
- Handle streaming responses appropriately
- Validate API responses before processing

## Known Issues and Mitigations

### Database Configuration Issues
- **Problem**: Configuration not passed from .env or TOML files
- **Solution**: Verify file paths and environment variable precedence
- **Mitigation**: Use explicit configuration validation during startup

### Docker Network Issues
- **Problem**: Database connectivity in containerized environments
- **Solution**: Ensure proper Docker network configuration
- **Mitigation**: Use `docker network connect` for custom network setups

### Memory Management
- **Problem**: Large document processing causing memory issues
- **Solution**: Implement chunking and batch processing strategies
- **Mitigation**: Monitor memory usage and set appropriate limits

### Scaling Limitations
- **Problem**: Single-instance bottlenecks in high-traffic scenarios
- **Solution**: Implement horizontal scaling with load balancing
- **Mitigation**: Use database read replicas and caching layers

## Testing and Validation

### Unit Testing Patterns
```python
import pytest

def test_document_ingestion():
    # Test successful document ingestion
    response = client.documents.create(file_path="test_document.txt")
    assert response["results"]["processed_documents"] > 0

def test_rag_functionality():
    # Test RAG response quality
    response = client.retrieval.rag(query="test query")
    assert len(response["results"]["completion"]) > 0
    assert "results" in response
```

### Integration Testing
- Test complete ingestion-to-retrieval workflows
- Validate authentication and authorization flows
- Test error handling and recovery mechanisms
- Performance testing under load conditions

### Production Validation
- Health check endpoints for monitoring
- Gradual rollout strategies for configuration changes
- A/B testing for RAG quality improvements
- Continuous monitoring of key metrics

## Version Compatibility and Updates

### Supported Versions
- Python: 3.8+
- Node.js: 16+ for JavaScript SDK
- PostgreSQL: 12+ for database backend
- Docker: 20.10+ for containerized deployments

### Update Strategies
- Review changelog for breaking changes before updates
- Test updates in staging environment first
- Backup database before major version upgrades
- Monitor system behavior after updates

### Migration Patterns
- Plan for data migration when changing database schemas
- Update configuration files for new features
- Test SDK compatibility with new R2R versions
- Implement rollback procedures for failed updates