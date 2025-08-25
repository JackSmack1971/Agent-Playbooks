---
trigger: ["*.py", "model_decision"]
description: Comprehensive LangGraph ruleset for building stateful, persistent workflows with robust memory management. Covers PostgreSQL/Redis integrations, checkpointing, error handling, and production deployment patterns.
globs: ["*.py", "main.py", "workflow.py", "graph.py", "agent.py"]
---

# LangGraph Stateful Workflow Rules

## Core Architecture and Setup

### Package Installation and Dependencies
- Always install with specific backends: `pip install -U langgraph langgraph-checkpoint-postgres langgraph-checkpoint-redis "psycopg[binary,pool]"`
- For TypeScript: `npm install @langchain/langgraph @langchain/langgraph-checkpoint-postgres`
- Import required modules explicitly:
```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.redis import RedisSaver
from langgraph.store.postgres import PostgresStore  
from langgraph.store.redis import RedisStore
```

### State Schema Definition
- Use `TypedDict`, `Pydantic` models, or `dataclasses` in Python; `Zod` schemas in TypeScript
- Always include type annotations for all state fields
- Define state reducers for complex state updates using `Annotated[list, add_messages]`
- Example:
```python
from typing_extensions import TypedDict, Annotated
from langchain_core.messages import AnyMessage
from langgraph.graph import add_messages

class WorkflowState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    user_id: str
    summary: str = ""
    step_count: int = 0
```

### Graph Construction Patterns
- Always use `StateGraph(StateClass)` constructor with explicit state schema
- Add nodes with `builder.add_node("node_name", node_function)`
- Define entry point with `builder.add_edge(START, "entry_node")`
- Use conditional edges for branching logic: `builder.add_conditional_edges("source", condition_func)`
- Compile with persistence: `graph = builder.compile(checkpointer=checkpointer, store=store)`

## Memory Management and Persistence

### Checkpointer Configuration
- **Development**: Use `InMemorySaver()` for testing only
- **Production**: Always use persistent backends (PostgreSQL/Redis)
- Setup pattern for PostgreSQL:
```python
DB_URI = "postgresql://user:password@host:port/dbname?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # Run once for initial setup
    graph = builder.compile(checkpointer=checkpointer)
```

### Store Integration for Long-term Memory
- Use `BaseStore` implementations for user-specific memory across threads
- Namespace memories with user/session identifiers: `namespace = ("memories", user_id)`
- Search existing memories: `memories = store.search(namespace, query=query_string)`
- Store new memories: `store.put(namespace, str(uuid.uuid4()), {"data": memory_content})`
- Always use both checkpointer and store in production:
```python
graph = builder.compile(checkpointer=checkpointer, store=store)
```

### Thread and Session Management
- Always provide `thread_id` in configuration for state persistence
- Use `user_id` for cross-thread memory sharing
- Configuration pattern:
```python
config = {
    "configurable": {
        "thread_id": "unique_thread_id",
        "user_id": "user_identifier"
    }
}
```

## Database Integration Best Practices

### PostgreSQL Configuration
- Use connection pooling with `psycopg[binary,pool]`
- Set proper connection limits in URI: `?pool_size=20&max_overflow=30`
- Always handle connection context properly:
```python
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    # Use checkpointer here
    pass
```

### Redis Configuration  
- Configure Redis with appropriate eviction policies: `maxmemory-policy allkeys-lru`
- Set TTL for session data to prevent memory bloat
- Use Redis for short-term state, PostgreSQL for durable data
- Connection pattern:
```python
with RedisSaver.from_conn_string("redis://localhost:6379") as checkpointer:
    store.setup()
    checkpointer.setup()
```

### Async Operations
- Use async variants for high-concurrency applications:
- `AsyncPostgresSaver`, `AsyncRedisSaver`, `AsyncPostgresStore`, `AsyncRedisStore`
- Pattern:
```python
async with AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer:
    async def call_model(state, config, *, store):
        memories = await store.asearch(namespace, query=query)
        await store.aput(namespace, key, data)
```

## Node Implementation Patterns

### Function Node Structure
- Accept state as first parameter, config as second (if needed)
- Use `store: BaseStore` parameter injection for memory access
- Return dictionary with state updates only
- Example:
```python
def call_model(
    state: MessagesState,
    config: RunnableConfig,
    *,
    store: BaseStore,
):
    # Process state
    response = model.invoke(state["messages"])
    return {"messages": [response]}
```

### Error Handling in Nodes
- Wrap node logic in try-catch blocks
- Return partial state updates on recoverable errors
- Log errors with context for debugging
- Use state versioning for rollback capability

### Memory Integration in Nodes
- Extract user context from config: `user_id = config["configurable"]["user_id"]`
- Search relevant memories before processing
- Store new memories conditionally based on content
- Pattern:
```python
def memory_aware_node(state, config, *, store):
    user_id = config["configurable"]["user_id"]
    namespace = ("memories", user_id)
    
    # Retrieve relevant memories
    memories = store.search(namespace, query=str(state["messages"][-1].content))
    context = "\n".join([d.value["data"] for d in memories])
    
    # Process with context
    system_msg = f"Context: {context}"
    response = model.invoke([{"role": "system", "content": system_msg}] + state["messages"])
    
    # Store new memories if needed
    if "remember" in state["messages"][-1].content.lower():
        store.put(namespace, str(uuid.uuid4()), {"data": "extracted_memory"})
    
    return {"messages": [response]}
```

## Execution and Streaming

### Graph Invocation Patterns
- Use `graph.stream()` for real-time processing
- Use `graph.invoke()` for single-shot execution
- Always provide configuration with thread_id
- Stream mode options: `"values"`, `"updates"`, `"debug"`

### State Management During Execution
- Access current state: `graph.get_state(config)`
- Get state history: `graph.get_state_history(config)`
- Update state manually: `graph.update_state(config, values)`

### Concurrent Execution Handling
- Use unique thread_ids for parallel workflows
- Implement proper locking for shared resources
- Handle connection pool exhaustion gracefully

## Advanced Patterns

### Message Summarization for Long Conversations
- Implement conversation summarization to manage context limits
- Use `RemoveMessage` to delete old messages while preserving summary
- Pattern:
```python
def summarize_conversation(state: State):
    summary = state.get("summary", "")
    if summary:
        summary_prompt = f"Previous summary: {summary}\nExtend summary with new messages:"
    else:
        summary_prompt = "Create summary of conversation:"
    
    messages = state["messages"] + [HumanMessage(content=summary_prompt)]
    response = model.invoke(messages)
    
    # Delete old messages, keep recent ones
    delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
    return {"summary": response.content, "messages": delete_messages}
```

### Tool Integration with State Access
- Use `InjectedState` for tools that need graph state
- Pattern:
```python
from langgraph.prebuilt import InjectedState

@tool
def state_aware_tool(
    query: str,
    state: Annotated[CustomState, InjectedState]
) -> str:
    user_id = state.get("user_id", "unknown")
    return f"Processing {query} for user {user_id}"
```

### Subgraph Composition
- Use shared state schemas for parent-subgraph communication
- Compile subgraphs with same checkpointer for state consistency
- Pass relevant state portions to subgraphs

## Error Handling and Debugging

### Common Error Patterns and Solutions
- **`KeyError`/`AttributeError` in nodes**: Validate state schema and field access
- **Connection exhaustion**: Configure proper connection pooling
- **`psycopg2.OperationalError`**: Check DB URI, credentials, and network connectivity
- **Stale Redis data**: Implement proper cache invalidation and TTL
- **State inconsistency**: Use atomic transactions and state versioning

### Debugging Strategies
- Enable debug streaming: `stream_mode="debug"`
- Use state snapshots for inspection: `graph.get_state(config)`
- Log state changes at node boundaries
- Implement health checks for database connections

### State Validation
- Validate state schema at node entry/exit points
- Use type hints and runtime validation
- Implement state invariants and constraints

## Performance Optimization

### Database Performance
- Add indexes for frequently queried fields
- Use connection pooling: `QueuePool` for SQLAlchemy
- Monitor query performance and optimize slow operations
- Implement proper transaction boundaries

### Memory Optimization
- Set appropriate TTL for Redis keys
- Use efficient data structures (Hashes vs multiple keys)
- Implement state pruning for long-running workflows
- Monitor memory usage patterns

### Execution Optimization
- Use async operations for I/O-bound workflows
- Implement parallel node execution where possible
- Cache frequently accessed data
- Profile and optimize hot paths

## Production Deployment

### Environment Configuration
- Set `REDIS_URI` for streaming and temporary storage
- Set `DATABASE_URI` for persistent state and memory
- Configure `LANGGRAPH_CLOUD_LICENSE_KEY` if using enterprise features
- Set `LANGSMITH_ENDPOINT` for self-hosted tracing

### Security Best Practices
- Enable PostgreSQL/Redis authentication
- Use TLS encryption for database connections
- Implement proper access controls and VPC restrictions
- Rotate credentials regularly
- Validate all inputs and sanitize user data

### Monitoring and Observability
- Track state size growth and memory usage
- Monitor database connection pool utilization
- Set up alerting for error rates and latency spikes
- Use distributed tracing for complex workflows
- Implement health checks for all dependencies

### Scaling Considerations
- Use Redis for ephemeral data, PostgreSQL for durability
- Implement horizontal scaling with proper session affinity
- Consider sharding strategies for high-volume applications
- Plan for database backup and disaster recovery

## Known Issues and Mitigations

### Schema Evolution
- Plan migration strategies for state schema changes
- Use versioning for backward compatibility
- Test migrations on non-production data first

### Memory Leaks
- Regularly audit and clean up unused state
- Implement garbage collection for abandoned threads
- Monitor long-term memory growth patterns

### Race Conditions
- Use atomic operations for shared state updates
- Implement proper locking mechanisms
- Design idempotent operations where possible

### Testing Strategies
- Use `InMemorySaver` for unit tests
- Test with realistic data volumes
- Implement integration tests with actual databases
- Test failure scenarios and recovery mechanisms