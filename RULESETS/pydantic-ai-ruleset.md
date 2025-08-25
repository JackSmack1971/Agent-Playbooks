---
trigger: glob
description: Comprehensive development rules for PydanticAI framework covering agent design, tools, output types, performance optimization, and production best practices
globs: ["**/*.py", "**/requirements.txt", "**/pyproject.toml", "**/Dockerfile"]
---

# PydanticAI Rules

## Installation and Package Management

- **Use slim installation**: Install `pydantic-ai-slim` instead of full `pydantic-ai` for production to avoid unnecessary dependencies
```bash
pip install "pydantic-ai-slim[openai,anthropic]"
```

- **Install model-specific dependencies**: Only include optional groups for models you actually use
```bash
# Good: Only install what you need
pip install "pydantic-ai-slim[openai,logfire]"
# Avoid: Installing all dependencies
pip install "pydantic-ai[examples]"
```

- **Pin dependency versions**: Always specify version constraints in production environments
```python
# requirements.txt
pydantic-ai-slim[openai]==0.0.49
openai>=1.35.0,<2.0.0
```

## Agent Configuration and Design

- **Define typed dependencies**: Always specify `deps_type` for type safety and better IDE support
```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class AppDependencies:
    db_conn: DatabaseConnection
    api_key: str

agent = Agent(
    'openai:gpt-4o',
    deps_type=AppDependencies,  # Always specify this
    system_prompt="You are a helpful assistant."
)
```

- **Use dynamic system prompts for context**: Leverage `@agent.system_prompt` for runtime-dependent instructions
```python
@agent.system_prompt
def dynamic_prompt(ctx: RunContext[AppDependencies]) -> str:
    user_info = get_user_context(ctx.deps.db_conn)
    return f"User context: {user_info.name}, Role: {user_info.role}"
```

- **Combine static and dynamic prompts**: Use both for comprehensive context
```python
agent = Agent(
    'openai:gpt-4o',
    system_prompt="Base behavior: Be helpful and concise.",  # Static
)

@agent.system_prompt  # Dynamic addition
def add_context(ctx: RunContext[str]) -> str:
    return f"Current user: {ctx.deps}"
```

- **Make agents reusable**: Create agents once and reuse throughout application lifecycle
```python
# Good: Module-level agent creation
chat_agent = Agent('openai:gpt-4o', system_prompt="...")

# Avoid: Creating agents in request handlers
def handle_request():
    agent = Agent('openai:gpt-4o')  # Don't do this
```

## Tool Registration and Implementation

- **Use appropriate tool decorators**: Choose between `@agent.tool` and `@agent.tool_plain` based on context needs
```python
@agent.tool  # Use when you need RunContext
async def database_query(ctx: RunContext[AppDependencies], query: str) -> str:
    return await ctx.deps.db_conn.execute(query)

@agent.tool_plain  # Use for stateless functions
def calculate_sum(a: int, b: int) -> int:
    return a + b
```

- **Add comprehensive tool docstrings**: LLMs use docstrings as tool descriptions
```python
@agent.tool
async def search_products(ctx: RunContext[AppDependencies], 
                         category: str, 
                         max_price: float) -> list[str]:
    """Search for products in the database.
    
    Args:
        category: Product category to search in (electronics, clothing, etc.)
        max_price: Maximum price filter in USD
        
    Returns:
        List of product names matching the criteria
    """
    return await ctx.deps.db_conn.search_products(category, max_price)
```

- **Handle tool errors gracefully**: Use ModelRetry for recoverable errors
```python
from pydantic_ai import ModelRetry

@agent.tool
async def external_api_call(ctx: RunContext[AppDependencies], query: str) -> str:
    try:
        response = await ctx.deps.http_client.get(f"/api/{query}")
        response.raise_for_status()
        return response.text
    except httpx.HTTPStatusError as e:
        if e.response.status_code >= 500:
            raise ModelRetry(f"Service temporarily unavailable: {e}")
        raise  # Re-raise client errors
```

- **Use Tool class for advanced configuration**: When you need more control over tool definitions
```python
from pydantic_ai.tools import Tool

def custom_tool(data: str) -> str:
    return process_data(data)

agent = Agent(
    'openai:gpt-4o',
    tools=[
        Tool(custom_tool, takes_ctx=False, name="data_processor"),
    ]
)
```

## Output Types and Validation

- **Always define structured output types**: Use Pydantic models for consistent, validated responses
```python
from pydantic import BaseModel, Field

class CustomerSupportResponse(BaseModel):
    response: str = Field(description="Response to customer")
    priority: Literal['low', 'medium', 'high'] = Field(description="Issue priority")
    requires_escalation: bool = Field(description="Whether to escalate to human")

agent = Agent(
    'openai:gpt-4o',
    output_type=CustomerSupportResponse,
)
```

- **Use Union types for multiple output formats**: When agents can return different response types
```python
from typing import Union

class SuccessResponse(BaseModel):
    result: str
    confidence: float

class ErrorResponse(BaseModel):
    error: str
    suggestions: list[str]

agent = Agent(
    'openai:gpt-4o',
    output_type=Union[SuccessResponse, ErrorResponse],
)
```

- **Implement output validators**: Add validation logic for complex business rules
```python
@agent.output_validator
async def validate_response(ctx: RunContext[AppDependencies], 
                          output: CustomerSupportResponse) -> CustomerSupportResponse:
    if output.requires_escalation and output.priority == 'low':
        raise ModelRetry("Cannot escalate low priority issues")
    return output
```

- **Choose appropriate output modes**: Use NativeOutput for better model support when available
```python
# Prefer NativeOutput for supported models
from pydantic_ai import NativeOutput

agent = Agent(
    'openai:gpt-4o',  # Supports structured output
    output_type=NativeOutput(
        CustomerSupportResponse,
        name='customer_support_response',
        description='Structured customer support response'
    ),
)
```

## Async and Streaming Patterns

- **Handle async environments properly**: Use nest_asyncio in Jupyter/Colab environments
```python
# In notebooks only
import nest_asyncio
nest_asyncio.apply()

# Then use async methods normally
result = await agent.run("Hello world", deps=dependencies)
```

- **Use streaming for long responses**: Implement streaming for better user experience
```python
async def stream_response(prompt: str):
    async with agent.run_stream(prompt, deps=deps) as response:
        async for chunk in response.stream_text():
            print(chunk, end='', flush=True)
    
    # Get final result
    final_result = await response.get_result()
    return final_result.output
```

- **Implement proper async error handling**: Handle streaming errors appropriately
```python
async def safe_stream_response(prompt: str):
    try:
        async with agent.run_stream(prompt, deps=deps) as response:
            chunks = []
            async for chunk in response.stream_text():
                chunks.append(chunk)
                yield chunk
    except Exception as e:
        logger.error(f"Streaming error: {e}")
        yield f"Error: {str(e)}"
```

## Performance Optimization

- **Use run_sync for synchronous contexts**: When you don't need async capabilities
```python
# In synchronous code
result = agent.run_sync(prompt, deps=deps)

# Don't use asyncio.run() unnecessarily
# result = asyncio.run(agent.run(prompt, deps=deps))  # Avoid
```

- **Optimize model settings**: Configure temperature, max_tokens for your use case
```python
from pydantic_ai.models import ModelSettings

agent = Agent(
    'openai:gpt-4o',
    settings=ModelSettings(
        temperature=0.1,  # Lower for deterministic responses
        max_tokens=500,   # Limit for cost control
    )
)
```

- **Use model_construct for trusted data**: Skip validation when data is guaranteed to be valid
```python
# In performance-critical paths with trusted data
response = CustomerSupportResponse.model_construct(
    response="Thank you for contacting us",
    priority="medium",
    requires_escalation=False
)
```

- **Implement connection pooling**: Reuse HTTP connections for external tools
```python
@dataclass
class AppDependencies:
    http_client: httpx.AsyncClient  # Reuse connection pool

# Create once at application startup
async def create_dependencies():
    client = httpx.AsyncClient(timeout=30.0, limits=httpx.Limits(max_connections=10))
    return AppDependencies(http_client=client)
```

## Error Handling and Retries

- **Configure retries for transient failures**: Use tenacity for robust retry logic
```python
# Install: pip install "pydantic-ai-slim[retries]"
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10)
)
async def resilient_agent_run(prompt: str, deps: AppDependencies):
    return await agent.run(prompt, deps=deps)
```

- **Handle validation errors systematically**: Catch and log Pydantic validation errors
```python
from pydantic import ValidationError

async def safe_agent_run(prompt: str):
    try:
        result = await agent.run(prompt, deps=deps)
        return result.output
    except ValidationError as e:
        logger.error(f"Output validation failed: {e}")
        return None
    except Exception as e:
        logger.error(f"Agent execution failed: {e}")
        raise
```

- **Use ModelRetry for recoverable errors**: Allow the model to retry with feedback
```python
from pydantic_ai import ModelRetry

@agent.tool
async def database_operation(ctx: RunContext[AppDependencies], query: str) -> str:
    try:
        return await ctx.deps.db_conn.execute(query)
    except DatabaseError as e:
        if "connection" in str(e).lower():
            raise ModelRetry(f"Database connection issue: {e}")
        raise  # Non-recoverable error
```

## Production Configuration

- **Set API keys via environment variables**: Never hardcode credentials
```python
import os

# Set in environment
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="cl-..."

# Use in code
agent = Agent('openai:gpt-4o')  # Uses OPENAI_API_KEY automatically
```

- **Implement comprehensive logging**: Use Logfire for observability
```python
import logfire

# Configure at application startup
logfire.configure()
logfire.instrument_pydantic_ai()

# Logs are automatically captured
result = await agent.run(prompt, deps=deps)
```

- **Configure custom HTTP clients**: For monitoring and authentication
```python
import httpx
from pydantic_ai.providers.openai import OpenAIProvider

# Custom client with authentication and monitoring
http_client = httpx.AsyncClient(
    headers={"User-Agent": "MyApp/1.0"},
    timeout=30.0,
)

provider = OpenAIProvider(
    api_key=os.getenv("OPENAI_API_KEY"),
    http_client=http_client
)

agent = Agent(
    OpenAIModel('gpt-4o', provider=provider)
)
```

- **Use fallback models**: Implement model failover for high availability
```python
from pydantic_ai.models import FallbackModel

primary_model = OpenAIModel('gpt-4o')
fallback_model = AnthropicModel('claude-3-5-sonnet-latest')

agent = Agent(FallbackModel(primary_model, fallback_model))
```

## Testing and Development

- **Use TestModel for unit tests**: Mock model responses in tests
```python
from pydantic_ai.models.test import TestModel

def test_agent_response():
    test_model = TestModel()
    agent = Agent(test_model)
    
    result = agent.run_sync("Test prompt")
    # TestModel returns mock responses
    assert result.output is not None
```

- **Test with examples**: Use the examples package for development
```bash
# Install examples
pip install "pydantic-ai[examples]"

# Run examples
python -m pydantic_ai_examples.pydantic_model
```

- **Copy examples for customization**: Modify examples for your use case
```bash
python -m pydantic_ai_examples --copy-to examples/
```

## Known Issues and Mitigations

- **Jupyter/Colab event loop conflicts**: Use nest_asyncio before running agents
```python
# Required in Jupyter/Colab
import nest_asyncio
nest_asyncio.apply()

# Then run normally
result = await agent.run(prompt, deps=deps)
```

- **Model-specific limitations**: Some features don't work with all models
```python
# tool_choice='required' not supported by all models
# Check model capabilities before using advanced features
try:
    result = await agent.run(prompt, deps=deps)
except Exception as e:
    if "tool_choice" in str(e):
        logger.warning("Model doesn't support required tool choice")
        # Fallback to standard execution
```

- **Large response handling**: Stream responses for large outputs
```python
# For large responses, always use streaming
if expected_large_response:
    async with agent.run_stream(prompt, deps=deps) as response:
        async for chunk in response.stream_text():
            process_chunk(chunk)
```

- **Memory management with long conversations**: Clear context periodically
```python
# For long-running conversations, manage message history
class ConversationManager:
    def __init__(self, max_messages: int = 50):
        self.messages = []
        self.max_messages = max_messages
    
    def add_message(self, message: str):
        self.messages.append(message)
        if len(self.messages) > self.max_messages:
            self.messages = self.messages[-self.max_messages//2:]  # Keep recent half
```

## Integration Patterns

- **FastAPI integration**: Use AG-UI protocol for web applications
```python
from fastapi import FastAPI
from pydantic_ai.ag_ui import run_ag_ui

app = FastAPI()

@app.post("/chat")
async def chat_endpoint(request: Request):
    run_input = RunAgentInput.model_validate(await request.json())
    event_stream = run_ag_ui(agent, run_input)
    return StreamingResponse(event_stream, media_type="text/event-stream")
```

- **Multi-agent systems**: Use agent delegation for complex workflows
```python
# Coordinator agent
coordinator = Agent('openai:gpt-4o', system_prompt="Route requests to appropriate specialists")

# Specialist agents
data_agent = Agent('openai:gpt-4o', system_prompt="Data analysis specialist")
support_agent = Agent('openai:gpt-4o', system_prompt="Customer support specialist")

@coordinator.tool
async def delegate_to_data_agent(ctx: RunContext[AppDependencies], query: str) -> str:
    result = await data_agent.run(query, deps=ctx.deps, usage=ctx.usage)
    return result.output

@coordinator.tool
async def delegate_to_support_agent(ctx: RunContext[AppDependencies], query: str) -> str:
    result = await support_agent.run(query, deps=ctx.deps, usage=ctx.usage)
    return result.output
```

- **MCP (Model Context Protocol) integration**: Use MCP servers for extended functionality
```python
from pydantic_ai.mcp import MCPServerStdio

# Connect to MCP server
server = MCPServerStdio('python', args=['mcp_server.py'])
agent = Agent('claude-3-5-haiku-latest', toolsets=[server])

# Agent can now use MCP tools
result = await agent.run("Execute Python: print('Hello, World!')")
```
