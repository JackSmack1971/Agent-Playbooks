---
trigger: glob
description: Comprehensive technical ruleset for mem0-ai memory layer implementation covering installation, configuration, operations, and best practices
globs: 
  - "**/*.py"
  - "**/requirements.txt"
  - "**/package.json"
  - "**/*.yaml"
  - "**/*.yml"
  - "**/*.env*"
  - "**/Dockerfile*"
  - "**/docker-compose*.yml"
---

# Mem0-AI Rules

## Installation and Dependencies

### Python Environment Requirements
- **ALWAYS** use Python 3.8 or newer for full compatibility with Mem0
- **ALWAYS** install using: `pip install mem0ai` (not `mem0` or other variants)
- **RECOMMEND** using virtual environments (`venv`, `conda`, `poetry`) to avoid dependency conflicts
- **VERIFY** installation by importing: `from mem0 import Memory` (for OSS) or `from mem0 import MemoryClient` (for Platform)

### Node.js Installation
- **USE** `npm install mem0ai` for Node.js/TypeScript projects
- **ENSURE** Node.js version 16+ for compatibility

## Configuration and Initialization

### Platform vs Open Source Selection
- **Platform (Hosted)**: Use `MemoryClient` with API key for cloud-hosted memory
- **Open Source (Self-hosted)**: Use `Memory` with local configuration for self-hosted deployments
- **NEVER** mix platform and OSS configurations in the same application

### Platform Configuration Pattern
```python
from mem0 import MemoryClient

# ALWAYS include error handling for API key validation
client = MemoryClient(api_key="your-api-key")
```

### Open Source Configuration Pattern
```python
from mem0 import Memory

config = {
    "vector_store": {
        "provider": "qdrant",  # Use specific provider names
        "config": {
            "host": "localhost",
            "port": 6333,
            "collection_name": "memories"
        }
    },
    "llm": {
        "provider": "openai",
        "config": {
            "api_key": "your-api-key",
            "model": "gpt-4"
        }
    },
    "embedder": {
        "provider": "openai", 
        "config": {
            "api_key": "your-api-key",
            "model": "text-embedding-3-small"
        }
    }
}

memory = Memory.from_config(config)
```

### Configuration Version Management
- **ALWAYS** specify `"version": "v1.1"` in configuration for latest features
- **INCLUDE** version in config to ensure consistent behavior across deployments

## Vector Store Configuration

### Provider Name Accuracy
- **CRITICAL**: Use `"chromadb"` NOT `"chroma"` for ChromaDB vector store
- **VALIDATE** provider names against official documentation before deployment
- **SUPPORTED PROVIDERS**: `qdrant`, `chromadb`, `pinecone`, `weaviate`, `elasticsearch`, `pgvector`, `azure_ai_search`

### ChromaDB Configuration
```python
"vector_store": {
    "provider": "chromadb",  # CORRECT name
    "config": {
        "collection_name": "memories",
        "path": "./chroma_db",  # Local path for persistence
        "embedding_model_dims": 1536  # Match embedder dimensions
    }
}
```

### Qdrant Configuration  
```python
"vector_store": {
    "provider": "qdrant",
    "config": {
        "host": "localhost",
        "port": 6333,
        "collection_name": "memories",
        "embedding_model_dims": 1536  # ALWAYS match embedder model dimensions
    }
}
```

### Dimension Matching Rule
- **CRITICAL**: Vector store `embedding_model_dims` MUST match embedder model output dimensions
- **OpenAI text-embedding-3-small**: 1536 dimensions
- **OpenAI text-embedding-ada-002**: 1536 dimensions
- **Ollama nomic-embed-text**: 768 dimensions

## Memory Operations

### Adding Memories
```python
# Platform approach with conversation context
result = client.add([
    {"role": "user", "content": "I love Italian food"},
    {"role": "assistant", "content": "Noted your preference for Italian cuisine"}
], user_id="user_123")

# OSS approach with direct text
result = memory.add("User prefers Italian cuisine", user_id="user_123")
```

### Adding with Metadata and Categories
```python
# ALWAYS include relevant metadata for filtering
result = client.add(
    [{"role": "user", "content": "I'm allergic to nuts"}],
    user_id="user_123",
    metadata={"category": "health", "severity": "high"},
    categories=["health", "allergies"]
)
```

### Searching Memories
```python
# Semantic search with user context
results = client.search("What foods does the user like?", user_id="user_123")

# ALWAYS handle empty results
if results:
    for memory in results:
        print(f"Memory: {memory['memory']}, Score: {memory.get('score', 'N/A')}")
else:
    print("No relevant memories found")
```

### Advanced Filtering (Platform v2)
```python
# Complex filter example with AND/OR logic
filters = {
    "AND": [
        {"user_id": "user_123"},
        {"created_at": {"gte": "2024-01-01", "lte": "2024-12-31"}},
        {"categories": {"contains": "food_preferences"}}
    ]
}

memories = client.get_all(version="v2", filters=filters, page=1, page_size=50)
```

### Memory Management
```python
# Get all memories with pagination
all_memories = client.get_all(user_id="user_123", page=1, page_size=100)

# Update specific memory
client.update(memory_id="memory-uuid", data="Updated memory content")

# Delete memory
client.delete(memory_id="memory-uuid")

# Delete all memories for user (USE WITH CAUTION)
client.delete_all(user_id="user_123")
```

## LLM and Embedder Configuration

### OpenAI Configuration
```python
"llm": {
    "provider": "openai",
    "config": {
        "api_key": "sk-...",
        "model": "gpt-4o-mini",  # Recommended for cost efficiency
        "temperature": 0.1,      # Low temperature for consistent memory extraction
        "max_tokens": 1500
    }
},
"embedder": {
    "provider": "openai", 
    "config": {
        "api_key": "sk-...",
        "model": "text-embedding-3-small"  # Recommended for balance of performance/cost
    }
}
```

### Ollama Local Configuration
```python
"llm": {
    "provider": "ollama",
    "config": {
        "model": "llama3.1:8b",
        "ollama_base_url": "http://localhost:11434",
        "temperature": 0
    }
},
"embedder": {
    "provider": "ollama",
    "config": {
        "model": "nomic-embed-text:latest",
        "ollama_base_url": "http://localhost:11434"
    }
}
```

### Azure OpenAI Configuration
```python
"llm": {
    "provider": "azure_openai",
    "config": {
        "api_key": "your-key",
        "azure_endpoint": "https://your-resource.openai.azure.com/",
        "api_version": "2024-10-21",
        "model": "gpt-4o-mini"
    }
},
"embedder": {
    "provider": "azure_openai",
    "config": {
        "api_key": "your-key", 
        "azure_endpoint": "https://your-resource.openai.azure.com/",
        "api_version": "2024-10-21",
        "model": "text-embedding-3-small"
    }
}
```

## Error Handling and Validation

### Configuration Validation
```python
def validate_mem0_config(config):
    """Validate Mem0 configuration before initialization"""
    required_sections = ["vector_store", "llm", "embedder"]
    
    for section in required_sections:
        if section not in config:
            raise ValueError(f"Missing required configuration section: {section}")
        
        if "provider" not in config[section]:
            raise ValueError(f"Missing provider in {section} configuration")
    
    # Validate vector store provider name
    valid_providers = ["qdrant", "chromadb", "pinecone", "weaviate", "elasticsearch", "pgvector"]
    vs_provider = config["vector_store"]["provider"]
    if vs_provider not in valid_providers:
        raise ValueError(f"Invalid vector store provider: {vs_provider}")
    
    return True

# ALWAYS validate before initializing
validate_mem0_config(config)
memory = Memory.from_config(config)
```

### API Error Handling
```python
from mem0 import MemoryClient
import logging

try:
    client = MemoryClient(api_key="your-key")
    result = client.add("Memory content", user_id="user_123")
except Exception as e:
    logging.error(f"Mem0 operation failed: {str(e)}")
    # IMPLEMENT appropriate fallback behavior
    handle_memory_failure(e)
```

### Memory Operation Validation
```python
def safe_memory_add(client, content, user_id, **kwargs):
    """Safely add memory with validation"""
    if not content or not content.strip():
        raise ValueError("Memory content cannot be empty")
    
    if not user_id or not user_id.strip():
        raise ValueError("User ID is required")
    
    try:
        return client.add(content, user_id=user_id, **kwargs)
    except Exception as e:
        logging.error(f"Failed to add memory for user {user_id}: {e}")
        return None
```

## Environment and Security

### Environment Variable Management
```bash
# REQUIRED environment variables
MEM0_API_KEY=your-mem0-api-key           # For platform usage
OPENAI_API_KEY=your-openai-key           # For OpenAI LLM/embedder
AZURE_OPENAI_API_KEY=your-azure-key      # For Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://...        # For Azure OpenAI
```

### API Key Security
- **NEVER** hardcode API keys in source code
- **ALWAYS** use environment variables or secure vaults
- **IMPLEMENT** API key rotation strategies for production
- **VALIDATE** API keys before application startup

### Data Privacy Considerations
```python
# For sensitive data, consider local deployment
config = {
    "vector_store": {"provider": "chromadb", "config": {"path": "./local_db"}},
    "llm": {"provider": "ollama", "config": {"model": "llama3.1:8b"}},
    "embedder": {"provider": "ollama", "config": {"model": "nomic-embed-text"}}
}
```

## Known Issues and Solutions

### ChromaDB Provider Name Issue
- **PROBLEM**: Documentation may show `"chroma"` but correct name is `"chromadb"`
- **SOLUTION**: Always use `"chromadb"` as provider name
- **ERROR**: `ValueError: Unsupported VectorStore provider: chroma`

### Embedding Dimension Mismatch
- **PROBLEM**: Vector store dimensions don't match embedder output
- **SOLUTION**: Explicitly set `embedding_model_dims` in vector store config
- **VERIFICATION**: Check embedder model specifications for correct dimensions

### Memory Persistence Issues
- **PROBLEM**: Memories not persisting between sessions
- **SOLUTION**: Ensure proper path configuration for local vector stores
- **CHECK**: File permissions and disk space for database files

### Rate Limiting and Costs
- **MONITOR**: API usage for hosted LLM/embedder services
- **IMPLEMENT**: Request batching and caching strategies
- **OPTIMIZE**: Use efficient models (gpt-4o-mini vs gpt-4, text-embedding-3-small vs ada-002)

## Production Best Practices

### Performance Optimization
```python
# Use connection pooling for high-throughput applications
config = {
    "vector_store": {
        "provider": "qdrant",
        "config": {
            "host": "localhost",
            "port": 6333,
            "timeout": 30,      # Adjust based on network latency
            "retry_count": 3    # Enable retries for reliability
        }
    }
}
```

### Memory Lifecycle Management
```python
def cleanup_old_memories(client, user_id, days_threshold=90):
    """Remove memories older than threshold"""
    cutoff_date = datetime.now() - timedelta(days=days_threshold)
    
    # Use date filtering to identify old memories
    filters = {
        "AND": [
            {"user_id": user_id},
            {"created_at": {"lte": cutoff_date.isoformat()}}
        ]
    }
    
    old_memories = client.get_all(version="v2", filters=filters)
    for memory in old_memories.get("results", []):
        client.delete(memory_id=memory["id"])
```

### Monitoring and Observability
```python
import time
import logging

def monitored_memory_operation(client, operation_func, *args, **kwargs):
    """Wrapper for monitoring memory operations"""
    start_time = time.time()
    operation_name = operation_func.__name__
    
    try:
        result = operation_func(*args, **kwargs)
        duration = time.time() - start_time
        logging.info(f"Memory operation {operation_name} completed in {duration:.2f}s")
        return result
    except Exception as e:
        duration = time.time() - start_time
        logging.error(f"Memory operation {operation_name} failed after {duration:.2f}s: {e}")
        raise
```

### Deployment Patterns

#### Docker Deployment
```dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

# COPY application code
COPY . .

# EXPOSE appropriate ports for vector store services
EXPOSE 6333

# SET environment variables securely
ENV MEM0_API_KEY=""
ENV OPENAI_API_KEY=""

CMD ["python", "main.py"]
```

#### Environment-Specific Configuration
```python
import os

def get_mem0_config(environment="development"):
    """Get environment-specific Mem0 configuration"""
    base_config = {
        "version": "v1.1",
        "llm": {
            "provider": "openai",
            "config": {
                "api_key": os.getenv("OPENAI_API_KEY"),
                "model": "gpt-4o-mini"
            }
        }
    }
    
    if environment == "production":
        # Production: Use managed vector store
        base_config["vector_store"] = {
            "provider": "pinecone",
            "config": {
                "api_key": os.getenv("PINECONE_API_KEY"),
                "index_name": "mem0-prod"
            }
        }
    else:
        # Development: Use local vector store
        base_config["vector_store"] = {
            "provider": "chromadb",
            "config": {
                "collection_name": "mem0-dev",
                "path": "./chroma_db"
            }
        }
    
    return base_config
```

## Testing and Quality Assurance

### Unit Testing Memory Operations
```python
import pytest
from mem0 import Memory

@pytest.fixture
def test_memory():
    """Create test memory instance"""
    config = {
        "vector_store": {
            "provider": "chromadb",
            "config": {"collection_name": "test", "path": ":memory:"}
        },
        "llm": {"provider": "openai", "config": {"api_key": "test"}},
        "embedder": {"provider": "openai", "config": {"api_key": "test"}}
    }
    return Memory.from_config(config)

def test_memory_add_and_search(test_memory):
    """Test basic memory operations"""
    # Add memory
    result = test_memory.add("User likes pizza", user_id="test_user")
    assert result is not None
    
    # Search memory
    results = test_memory.search("food preferences", user_id="test_user")
    assert len(results) > 0
    assert "pizza" in results[0]["memory"].lower()
```

### Integration Testing
```python
def test_end_to_end_memory_workflow():
    """Test complete memory workflow"""
    client = MemoryClient(api_key=os.getenv("MEM0_TEST_API_KEY"))
    
    test_user = f"test_user_{int(time.time())}"
    
    try:
        # Add multiple memories
        memories = [
            "User is vegetarian",
            "User lives in San Francisco", 
            "User works as a software engineer"
        ]
        
        for memory in memories:
            client.add(memory, user_id=test_user)
        
        # Test search functionality
        food_results = client.search("dietary preferences", user_id=test_user)
        location_results = client.search("where does user live", user_id=test_user)
        
        assert len(food_results) > 0
        assert len(location_results) > 0
        
    finally:
        # Cleanup test data
        client.delete_all(user_id=test_user)
```