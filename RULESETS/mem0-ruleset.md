---
trigger: glob
description: Comprehensive rules for implementing Mem0's validated multi-level memory system with user/agent/session scoping and contradiction resolution
globs: ['**/mem0/**', '**/*mem0*', '**/memory/**', '**/memories/**']
---

# Mem0 Multi-Level Memory System Rules

## Platform Selection and Initialization

### Choose Deployment Strategy Based on Requirements
- **Hosted Platform**: Use `MemoryClient` for production applications requiring scalability and managed infrastructure
- **Open Source**: Use `Memory` class for self-hosted deployments requiring custom configurations and data control
- **Never mix**: Don't switch between hosted and OSS clients in the same application without clear migration strategy

```python
# Hosted Platform (Recommended for production)
from mem0 import MemoryClient
client = MemoryClient(api_key="your-api-key")

# Open Source (For custom deployments)
from mem0 import Memory
config = {
    "vector_store": {"provider": "qdrant", "config": {"host": "localhost", "port": 6333}},
    "llm": {"provider": "openai", "config": {"api_key": "key", "model": "gpt-4"}},
    "embedder": {"provider": "openai", "config": {"api_key": "key", "model": "text-embedding-3-small"}}
}
memory = Memory.from_config(config)
```

### API Key Management
- **Store securely**: Never hardcode API keys in source code
- **Environment variables**: Use `MEM0_API_KEY` environment variable for hosted platform
- **Validation**: Always validate API key presence before initialization
- **Rotation**: Implement API key rotation strategy for production

### Version Specification
- **Explicit versions**: Always specify API version in configuration (`"version": "v1.1"`)
- **Breaking changes**: Review migration guides when updating versions
- **Compatibility**: Test thoroughly when upgrading between major versions

## Memory Architecture and Data Flow

### Understand the Three-Phase Pipeline
- **Extraction Phase**: Converts conversations into candidate memories using LLM fact extraction
- **Update Phase**: Resolves conflicts through ADD/UPDATE/DELETE/NOOP operations
- **Retrieval Phase**: Searches and filters memories using vector similarity and metadata

### Memory Types and Scoping
- **User memories**: Personal preferences, history, and characteristics (`user_id`)
- **Agent memories**: Agent-specific knowledge and behavior patterns (`agent_id`)
- **Session memories**: Temporary conversation context and state (`run_id`)
- **Global memories**: Shared knowledge across users (no scoping parameters)

```python
# Multi-level scoping example
memory.add(
    "User prefers vegetarian food and is allergic to nuts",
    user_id="alice",
    agent_id="nutrition-assistant", 
    run_id="consultation-001",
    metadata={"category": "dietary_preferences", "confidence": 0.9}
)
```

### Memory Lifecycle Management
- **Creation**: Extract meaningful facts from conversation context
- **Validation**: Check for duplicates and contradictions before storage
- **Consolidation**: Merge related memories to reduce redundancy
- **Expiration**: Set expiration dates for time-sensitive information
- **Deletion**: Remove outdated or incorrect memories proactively

## Memory Operations Best Practices

### Adding Memories
- **Context richness**: Provide conversation context for better fact extraction
- **Consistent scoping**: Always use consistent `user_id`, `agent_id`, `run_id` patterns
- **Metadata enrichment**: Include relevant metadata for categorization and filtering
- **Batch operations**: Use batch adding for multiple related memories

```python
# Rich context addition
messages = [
    {"role": "user", "content": "I'm vegetarian and allergic to peanuts"},
    {"role": "assistant", "content": "I'll remember your dietary restrictions"}
]
result = client.add(messages, user_id="alice", metadata={"category": "diet"})
```

### Searching Memories
- **Semantic queries**: Use natural language queries for vector similarity search
- **Scope filtering**: Combine semantic search with scope parameters for precision
- **Metadata filtering**: Leverage categories and metadata for exact matches
- **Limit results**: Set appropriate limits to control response size and latency

```python
# Multi-dimensional search
results = client.search(
    "What are my dietary restrictions?",
    user_id="alice",
    categories=["diet", "health"],
    metadata={"confidence": {"gte": 0.8}},
    limit=10
)
```

### Updating and Deleting Memories
- **Conflict resolution**: Let Mem0's built-in conflict resolution handle contradictions
- **Explicit updates**: Use direct memory updates for known corrections
- **Soft deletion**: Consider marking memories as invalid rather than hard deletion
- **Audit trail**: Maintain history of memory changes for debugging

```python
# Update specific memory
client.update(memory_id="uuid", text="Updated dietary preference")

# Delete with confirmation
client.delete(memory_id="uuid")
```

## Multi-Level Scoping Strategies

### User-Level Scoping
- **Personal identity**: Store user preferences, characteristics, and history
- **Cross-session persistence**: Maintain user context across different interactions
- **Privacy boundaries**: Ensure user memories don't leak to other users
- **Progressive learning**: Build comprehensive user profiles over time

### Agent-Level Scoping
- **Agent behavior**: Store agent-specific knowledge and interaction patterns
- **Role specialization**: Maintain separate memories for different agent personas
- **Shared agent knowledge**: Allow multiple users to benefit from agent learning
- **Agent evolution**: Track agent capability improvements and customizations

### Session-Level Scoping
- **Conversation context**: Store temporary context for current interaction
- **Task continuity**: Maintain state across multi-turn conversations
- **Session isolation**: Prevent session memories from affecting other sessions
- **Cleanup strategy**: Define session memory retention and cleanup policies

```python
# Scoping strategy examples
# User preference (persistent across sessions)
memory.add("Prefers morning meetings", user_id="alice")

# Agent learning (shared across users for this agent)
memory.add("Technical questions require code examples", agent_id="coding-assistant")

# Session context (temporary)
memory.add("Currently working on Python project", user_id="alice", run_id="session-123")
```

## Contradiction Resolution and Validation

### Conflict Detection
- **Automatic resolution**: Trust Mem0's built-in conflict resolution for most cases
- **Manual review**: Implement manual review workflows for critical contradictions
- **Confidence scoring**: Use metadata confidence scores to guide resolution
- **Version control**: Maintain memory versions for audit and rollback

### Memory Validation
- **Fact checking**: Validate critical facts against authoritative sources
- **Consistency checks**: Ensure memory consistency within user profiles
- **Duplicate detection**: Monitor for and merge semantically similar memories
- **Quality scoring**: Implement memory quality metrics and cleanup processes

### Update Strategies
- **ADD**: Allow new memories that don't conflict with existing ones
- **UPDATE**: Modify existing memories with new information
- **DELETE**: Remove contradictory or outdated memories
- **NOOP**: Skip updates when new information doesn't add value

```python
# Contradiction handling example
# Old memory: "User likes spicy food"
# New information: "User can't eat spicy food due to medical condition"
# Mem0 will automatically UPDATE or DELETE the old memory

result = memory.add(
    "Cannot eat spicy food due to medical condition",
    user_id="alice",
    metadata={"category": "health", "confidence": 1.0, "source": "medical_record"}
)
```

## Graph-Based Memory Features

### Entity and Relationship Modeling
- **Entity extraction**: Identify key entities (people, places, concepts) in memories
- **Relationship mapping**: Define relationships between entities for complex reasoning
- **Graph traversal**: Use graph queries for multi-hop reasoning and discovery
- **Schema consistency**: Maintain consistent entity and relationship schemas

### Graph Memory Operations
- **Triple storage**: Store memories as subject-predicate-object triples
- **Subgraph retrieval**: Query specific subgraphs for targeted information
- **Relationship inference**: Leverage graph structure for implicit knowledge discovery
- **Graph updates**: Handle entity and relationship updates gracefully

```python
# Graph-based memory with entities and relationships
config_with_graph = {
    "graph_store": {
        "provider": "neo4j",
        "config": {
            "url": "neo4j+s://your-instance",
            "username": "neo4j", 
            "password": "password"
        }
    }
}
# Mem0 will extract entities and relationships automatically
```

## Performance Optimization

### Token Efficiency
- **Context optimization**: Provide relevant context without overwhelming the LLM
- **Memory summarization**: Use summary mechanisms for long conversation histories
- **Batch processing**: Group related operations to reduce API calls
- **Caching strategies**: Implement memory caching for frequently accessed data

### Latency Optimization
- **Async operations**: Use asynchronous memory operations where possible
- **Connection pooling**: Maintain persistent connections to vector stores
- **Result pagination**: Use pagination for large memory retrievals
- **Pre-computation**: Pre-compute frequently used memory aggregations

### Scaling Strategies
- **Vector store selection**: Choose appropriate vector stores for scale requirements
- **Sharding strategies**: Implement memory sharding for large user bases
- **Read replicas**: Use read replicas for high-read scenarios
- **Memory pruning**: Implement automated pruning of old or low-value memories

```python
# Performance optimization example
# Use pagination for large retrievals
results = client.get_all(
    user_id="alice", 
    page=1, 
    page_size=50,
    categories=["recent_activity"]
)

# Async operations (if supported)
await client.add_async(memory_data, user_id="alice")
```

## Integration Patterns

### Framework Integration
- **LangChain**: Use Mem0 as memory backend for LangChain agents
- **LlamaIndex**: Integrate with LlamaIndex for enhanced RAG capabilities
- **CrewAI**: Implement team memory for multi-agent systems
- **AutoGen**: Provide persistent memory for conversational agents

### Common Use Cases
- **Customer support**: Maintain customer interaction history and preferences
- **Personal assistants**: Build comprehensive user profiles and preferences
- **Educational systems**: Track learning progress and personalized content
- **Healthcare**: Maintain patient history and care preferences

### API Patterns
- **RESTful integration**: Use REST APIs for web service integration
- **Webhook handlers**: Implement webhooks for real-time memory updates
- **Batch processing**: Design batch jobs for bulk memory operations
- **Event-driven updates**: React to application events with memory updates

```python
# LangChain integration pattern
from langchain.memory import ConversationBufferWindowMemory
from mem0 import MemoryClient

class Mem0LangChainMemory:
    def __init__(self, user_id):
        self.client = MemoryClient()
        self.user_id = user_id
    
    def save_context(self, inputs, outputs):
        conversation = f"User: {inputs}\nAssistant: {outputs}"
        self.client.add(conversation, user_id=self.user_id)
```

## Security and Compliance

### Data Privacy
- **Data isolation**: Ensure complete isolation between different users' memories
- **Encryption**: Use encryption for sensitive memory data at rest and in transit
- **Access controls**: Implement fine-grained access controls for memory operations
- **Data retention**: Define and enforce data retention policies

### Compliance Requirements
- **GDPR compliance**: Implement right to deletion and data portability
- **HIPAA compliance**: Ensure healthcare data handling meets HIPAA requirements
- **Audit logging**: Maintain comprehensive audit logs for all memory operations
- **Data sovereignty**: Respect data residency requirements for international users

### Security Best Practices
- **API key security**: Rotate API keys regularly and monitor for unauthorized usage
- **Input sanitization**: Sanitize all inputs to prevent injection attacks
- **Rate limiting**: Implement rate limiting to prevent abuse
- **Monitoring**: Monitor for unusual patterns in memory access and modification

```python
# Security-conscious memory operation
import hashlib

def secure_memory_add(content, user_id, client):
    # Hash sensitive identifiers
    hashed_user_id = hashlib.sha256(user_id.encode()).hexdigest()
    
    # Sanitize content
    sanitized_content = sanitize_input(content)
    
    # Add with security metadata
    return client.add(
        sanitized_content,
        user_id=hashed_user_id,
        metadata={"security_level": "high", "source": "verified"}
    )
```

## Error Handling and Resilience

### Common Error Patterns
- **API rate limiting**: Handle rate limit responses with exponential backoff
- **Network failures**: Implement retry logic with circuit breakers
- **Invalid configurations**: Validate configurations before initialization
- **Memory conflicts**: Handle conflict resolution failures gracefully

### Monitoring and Observability
- **Performance metrics**: Track memory operation latency and success rates
- **Error tracking**: Monitor and alert on memory operation failures
- **Usage analytics**: Analyze memory usage patterns for optimization
- **Health checks**: Implement health checks for memory system components

### Disaster Recovery
- **Backup strategies**: Implement regular backups of critical memories
- **Failover mechanisms**: Design failover strategies for high availability
- **Data recovery**: Plan for memory data recovery scenarios
- **Business continuity**: Ensure memory operations don't block critical workflows

```python
import time
import random
from typing import Optional

def resilient_memory_operation(client, operation_func, *args, **kwargs):
    max_retries = 3
    base_delay = 1
    
    for attempt in range(max_retries):
        try:
            return operation_func(*args, **kwargs)
        except RateLimitError:
            if attempt < max_retries - 1:
                delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
                time.sleep(delay)
            else:
                raise
        except NetworkError as e:
            if attempt < max_retries - 1:
                time.sleep(base_delay)
            else:
                logger.error(f"Memory operation failed after {max_retries} attempts: {e}")
                raise
```

## Known Issues and Mitigations

### Vector Store Limitations
- **Qdrant**: Requires explicit embedding dimensions configuration
- **Chroma**: May have performance issues with large datasets
- **Elasticsearch**: Requires proper index configuration for optimal performance
- **In-memory**: Not suitable for production due to data persistence issues

### LLM Provider Issues
- **OpenAI**: Rate limiting can affect memory processing performance
- **Local models**: May have quality issues with fact extraction and conflict resolution
- **Azure OpenAI**: Requires specific endpoint configuration patterns
- **Anthropic**: Different prompt formats may affect memory quality

### Configuration Anti-patterns
- **Mixed environments**: Don't mix development and production configurations
- **Hardcoded values**: Avoid hardcoding configuration values in application code
- **Default configurations**: Don't rely on default configurations for production
- **Version mismatches**: Ensure consistent versions across all components

```python
# Configuration validation pattern
def validate_mem0_config(config):
    required_keys = ["vector_store", "llm", "embedder"]
    for key in required_keys:
        if key not in config:
            raise ConfigurationError(f"Missing required configuration key: {key}")
    
    # Validate vector store configuration
    if config["vector_store"]["provider"] == "qdrant":
        if "embedding_model_dims" not in config["vector_store"]["config"]:
            raise ConfigurationError("Qdrant requires embedding_model_dims configuration")
    
    return True
```

## Testing and Quality Assurance

### Unit Testing
- **Memory operations**: Test all CRUD operations with various inputs
- **Scoping logic**: Verify memory isolation across different scopes
- **Configuration handling**: Test configuration validation and error handling
- **Mock external services**: Use mocks for vector stores and LLMs in tests

### Integration Testing
- **End-to-end workflows**: Test complete memory workflows from addition to retrieval
- **Cross-service interactions**: Test integration with external systems
- **Performance testing**: Validate performance under expected loads
- **Failure scenarios**: Test behavior under various failure conditions

### Memory Quality Testing
- **Fact extraction accuracy**: Validate quality of extracted facts
- **Conflict resolution effectiveness**: Test contradiction handling
- **Search relevance**: Measure search result quality and relevance
- **Memory consistency**: Verify memory consistency across operations

```python
# Testing example
import pytest
from mem0 import Memory

@pytest.fixture
def memory_client():
    config = {
        "vector_store": {"provider": "memory"},  # In-memory for testing
        "llm": {"provider": "openai", "config": {"model": "gpt-4o-mini"}}
    }
    return Memory.from_config(config)

def test_memory_scoping(memory_client):
    # Add memories with different scopes
    memory_client.add("User likes coffee", user_id="alice")
    memory_client.add("Agent specializes in beverages", agent_id="barista")
    
    # Test scope isolation
    user_memories = memory_client.get_all(user_id="alice")
    agent_memories = memory_client.get_all(agent_id="barista")
    
    assert len(user_memories) == 1
    assert len(agent_memories) == 1
    assert user_memories[0]["memory"] != agent_memories[0]["memory"]
```