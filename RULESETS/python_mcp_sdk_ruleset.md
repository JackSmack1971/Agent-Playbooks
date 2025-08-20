---
trigger: model_decision
description: Comprehensive rules for developing Model Context Protocol (MCP) servers and clients using the Python MCP SDK. Covers FastMCP framework, low-level server APIs, authentication, deployment, and best practices.
globs: 
  - "**/*mcp*.py"
  - "**/server*.py" 
  - "**/client*.py"
  - "**/fastmcp*.py"
---

# Python MCP SDK Rules

## Installation and Environment Setup

### Package Installation
- **ALWAYS** install MCP with CLI tools: `uv add "mcp[cli]"` or `pip install "mcp[cli]"`
- **ALWAYS** use virtual environments for dependency isolation: `python -m venv .venv` or `uv venv`
- **RECOMMENDED** use `uv` for faster package management and dependency resolution
- **REQUIRED** Python 3.8+ for compatibility with MCP SDK

### Environment Configuration
- **STORE** sensitive credentials in environment variables, never in code
- **USE** `.env` files for local development configuration
- **SET** `FASTMCP_DEBUG=false` in production environments
- **CONFIGURE** log levels appropriately: `FASTMCP_LOG_LEVEL=INFO` (production) or `DEBUG` (development)

## FastMCP Framework (Recommended Approach)

### Server Creation and Configuration
```python
from mcp.server.fastmcp import FastMCP

# Basic server creation
mcp = FastMCP("My Server Name")

# Production configuration
mcp = FastMCP(
    "Production Server",
    host="0.0.0.0",  # Allow external connections
    port=8000,
    debug=False,     # Disable debug in production
    log_level="INFO"
)
```

### Tool Definition Best Practices
```python
@mcp.tool()
def calculate_sum(a: int, b: int) -> int:
    """Add two numbers together."""  # REQUIRED: Descriptive docstring
    return a + b

# Async tools for I/O operations
@mcp.tool()
async def fetch_data(url: str, ctx: Context) -> str:
    """Fetch data with progress reporting."""
    await ctx.info(f"Fetching data from {url}")
    # Implementation here
    return result
```

### Resource Definition Patterns
```python
# Static resources
@mcp.resource("config://settings")
def get_settings() -> str:
    """Return application configuration."""
    return json.dumps({"theme": "dark", "version": "1.0"})

# Dynamic resources with parameters
@mcp.resource("file://documents/{name}")
def read_document(name: str) -> str:
    """Read document by name with validation."""
    # ALWAYS validate input parameters
    if not name.replace("_", "").replace("-", "").isalnum():
        raise ValueError("Invalid document name")
    return f"Content of {name}"
```

### Context and Progress Reporting
```python
from mcp.server.fastmcp import Context

@mcp.tool()
async def long_running_task(task_name: str, ctx: Context, steps: int = 5) -> str:
    """Execute task with proper progress reporting."""
    await ctx.info(f"Starting: {task_name}")
    
    for i in range(steps):
        progress = (i + 1) / steps
        await ctx.report_progress(
            progress=progress,
            total=1.0,
            message=f"Step {i + 1}/{steps}"
        )
        await ctx.debug(f"Completed step {i + 1}")
    
    return f"Task '{task_name}' completed"
```

## Low-Level Server API

### Server Initialization and Lifespan Management
```python
import asyncio
from mcp.server.lowlevel import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio

# Create server with lifespan management
server = Server("example-server")

@asynccontextmanager
async def server_lifespan(_server: Server):
    """Manage server startup and shutdown."""
    # Initialize resources
    db = await Database.connect()
    try:
        yield {"db": db}
    finally:
        # Cleanup
        await db.disconnect()

server = Server("example-server", lifespan=server_lifespan)
```

### Handler Registration Patterns
```python
@server.list_tools()
async def handle_list_tools() -> list[types.Tool]:
    """List available tools with proper schema definition."""
    return [
        types.Tool(
            name="query_db",
            description="Query the database",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "SQL query"}
                },
                "required": ["query"]
            },
            # NEW: Output schema for structured results
            outputSchema={
                "type": "object",
                "properties": {
                    "rows": {"type": "array", "description": "Query results"},
                    "count": {"type": "number", "description": "Row count"}
                }
            }
        )
    ]

@server.call_tool()
async def handle_tool_call(name: str, arguments: dict) -> dict:
    """Handle tool calls with structured output."""
    if name == "query_db":
        # Access lifespan context
        ctx = server.request_context
        db = ctx.lifespan_context["db"]
        
        # Return structured data matching output schema
        return {
            "rows": [{"id": 1, "name": "example"}],
            "count": 1
        }
    
    raise ValueError(f"Unknown tool: {name}")
```

## Client Development

### STDIO Client Pattern
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def run_client():
    """Standard client connection pattern."""
    server_params = StdioServerParameters(
        command="uv",
        args=["run", "server", "fastmcp_quickstart", "stdio"],
        env={"UV_INDEX": os.environ.get("UV_INDEX", "")}
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize connection
            await session.initialize()
            
            # List and use resources
            resources = await session.list_resources()
            if resources.resources:
                content = await session.read_resource(resources.resources[0].uri)
                
            # Call tools
            tools = await session.list_tools()
            if tools.tools:
                result = await session.call_tool("tool_name", {"param": "value"})
```

### HTTP Client with Authentication
```python
from mcp.client.streamable_http import streamablehttp_client
from mcp.client.auth import OAuthClientProvider

# OAuth authentication setup
oauth_auth = OAuthClientProvider(
    server_url="http://localhost:8001",
    client_metadata=OAuthClientMetadata(
        client_name="My MCP Client",
        redirect_uris=[AnyUrl("http://localhost:3000/callback")],
        grant_types=["authorization_code", "refresh_token"],
        response_types=["code"],
        scope="user"
    ),
    storage=TokenStorage(),  # Implement secure token storage
    redirect_handler=handle_redirect,
    callback_handler=handle_callback
)

async with streamablehttp_client("http://localhost:8001/mcp", auth=oauth_auth) as (read, write, _):
    async with ClientSession(read, write) as session:
        await session.initialize()
        # Use authenticated session
```

## Transport Configuration

### Transport Selection Rules
- **STDIO**: Default for local development and CLI tools
- **Streamable HTTP**: REQUIRED for cloud deployment and remote access
- **SSE**: Legacy transport, use only for backward compatibility

### Streamable HTTP Configuration
```python
# For cloud deployment
mcp = FastMCP("CloudServer", stateless_http=True)

# Custom endpoint mounting
from starlette.applications import Starlette
from starlette.routing import Mount

app = Starlette(routes=[
    Mount("/mcp", app=mcp.streamable_http_app()),
    Mount("/health", app=health_check_app())
])

# Multiple server mounting
github_mcp = FastMCP("GitHub API")
github_mcp.settings.mount_path = "/github"

app = Starlette(routes=[
    Mount("/github", app=github_mcp.sse_app()),
    Mount("/search", app=search_mcp.sse_app("/search"))
])
```

## Authentication and Security

### OAuth 2.1 Resource Server Setup
```python
from mcp.server.auth.provider import TokenVerifier, AccessToken
from mcp.server.auth.settings import AuthSettings

class ProductionTokenVerifier(TokenVerifier):
    """Production-grade token verification."""
    
    async def verify_token(self, token: str) -> AccessToken | None:
        """Verify OAuth token with proper validation."""
        # REQUIRED: Implement actual token validation
        # - Verify token signature
        # - Check expiration
        # - Validate scopes
        # - Rate limiting checks
        pass

# Server with authentication
mcp = FastMCP(
    "Secure Weather Service",
    token_verifier=ProductionTokenVerifier(),
    auth=AuthSettings(
        issuer_url=AnyHttpUrl("https://auth.example.com"),
        resource_server_url=AnyHttpUrl("http://localhost:3001"),
        required_scopes=["user", "data.read"]
    )
)
```

### Security Best Practices
- **NEVER** hard-code credentials in source code
- **ALWAYS** validate and sanitize external input
- **USE** HTTPS in production deployments
- **IMPLEMENT** proper rate limiting for public endpoints
- **STORE** tokens securely with encryption at rest
- **VALIDATE** OAuth scopes for each protected resource

## Development and Testing

### Development Workflow
```bash
# Install dependencies with development tools
uv sync --frozen --all-extras --dev

# Run in development mode with hot reload
uv run mcp dev server.py

# Add dependencies for development
uv run mcp dev server.py --with pandas --with numpy

# Install server for Claude Desktop
uv run mcp install server.py --name "My Analytics Server"
```

### Testing Patterns
```python
import pytest
from mcp.server.fastmcp import FastMCP

@pytest.fixture
async def test_server():
    """Create test server instance."""
    mcp = FastMCP("Test Server", debug=True)
    
    @mcp.tool()
    def test_tool(input_data: str) -> str:
        return f"Processed: {input_data}"
    
    return mcp

@pytest.mark.asyncio
async def test_tool_functionality(test_server):
    """Test tool execution."""
    # Test implementation here
    pass
```

### Pre-commit Configuration
```bash
# Install pre-commit hooks
uv tool install pre-commit --with pre-commit-uv --force-reinstall

# Run formatting and linting
uv run ruff format .
uv run ruff check . --fix
uv run pyright  # Type checking
```

## Error Handling and Logging

### Exception Management
```python
import logging

logger = logging.getLogger(__name__)

@mcp.tool()
async def safe_operation(data: str, ctx: Context) -> str:
    """Tool with proper error handling."""
    try:
        # Operation logic
        result = process_data(data)
        await ctx.info(f"Successfully processed {len(data)} characters")
        return result
    
    except ValueError as e:
        logger.exception("Invalid input data")
        await ctx.error(f"Invalid input: {str(e)}")
        raise
    
    except (ConnectionError, TimeoutError) as e:
        logger.exception("Network operation failed")
        await ctx.warning("Retrying operation...")
        # Implement retry logic
        raise
    
    except Exception as e:
        logger.exception("Unexpected error in safe_operation")
        await ctx.error("Internal server error")
        raise
```

### Progress and Status Reporting
```python
@mcp.tool()
async def batch_processor(items: list[str], ctx: Context) -> dict:
    """Process items with detailed progress reporting."""
    total_items = len(items)
    processed = []
    failed = []
    
    for i, item in enumerate(items):
        try:
            # Report progress
            await ctx.report_progress(
                progress=(i + 1) / total_items,
                total=1.0,
                message=f"Processing item {i + 1}/{total_items}"
            )
            
            result = await process_item(item)
            processed.append(result)
            await ctx.debug(f"Successfully processed item {i + 1}")
            
        except Exception as e:
            await ctx.warning(f"Failed to process item {i + 1}: {str(e)}")
            failed.append({"item": item, "error": str(e)})
    
    return {
        "processed_count": len(processed),
        "failed_count": len(failed),
        "results": processed,
        "failures": failed
    }
```

## Performance Optimization

### Async Best Practices
- **USE** `async`/`await` for all I/O operations
- **IMPLEMENT** connection pooling for database/HTTP clients
- **SET** appropriate timeouts using float values for precision
- **BATCH** operations when possible to reduce overhead

### Memory Management
```python
@mcp.tool()
async def large_data_processor(data_source: str, ctx: Context) -> str:
    """Process large datasets efficiently."""
    # Stream processing for large datasets
    total_processed = 0
    
    async for batch in stream_data_batches(data_source, batch_size=1000):
        # Process in chunks to manage memory
        processed_batch = await process_batch(batch)
        total_processed += len(processed_batch)
        
        # Report progress periodically
        if total_processed % 5000 == 0:
            await ctx.info(f"Processed {total_processed} records")
    
    return f"Successfully processed {total_processed} records"
```

## Known Issues and Mitigations

### ENOENT Errors on macOS
- **CAUSE**: Python path issues after Homebrew upgrades
- **FIX**: Rebuild virtual environment and reinstall dependencies
```bash
# Remove old virtual environment
rm -rf .venv

# Create new virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Reinstall dependencies
pip install "mcp[cli]"
```

### Connection Timeout Issues
- **USE** float timeouts for precise control: `timeout_ms=5.5`
- **IMPLEMENT** exponential backoff for retries
- **MONITOR** connection health with heartbeat messages

### Memory Leaks in Long-Running Servers
- **CLOSE** database connections properly in lifespan context
- **CLEAR** cached data periodically
- **USE** weak references for circular dependencies

## Deployment Considerations

### Production Deployment Checklist
- [ ] **Environment**: Set `debug=False` and appropriate log levels
- [ ] **Security**: Enable HTTPS, validate OAuth tokens, sanitize inputs
- [ ] **Monitoring**: Implement health checks and error tracking
- [ ] **Performance**: Configure connection pooling and timeouts
- [ ] **Scaling**: Use stateless HTTP for horizontal scaling

### Cloud Platform Configuration
```python
# For serverless/cloud deployment
mcp = FastMCP(
    "Production API",
    stateless_http=True,  # Required for serverless
    json_response=True,   # Disable SSE for some platforms
    host="0.0.0.0",
    port=int(os.environ.get("PORT", 8000))
)

# Health check endpoint
@mcp.tool()
def health_check() -> dict:
    """Health check for load balancers."""
    return {"status": "healthy", "timestamp": datetime.now().isoformat()}
```

## Version Compatibility and Migration

### Current Protocol Version: 2025-06-18
- **REMOVED**: JSON-RPC batching (deprecated from 2025-03-26)
- **NEW**: Structured tool outputs with schema validation
- **ENHANCED**: OAuth security requirements
- **ADDED**: Audio content support

### Migration Guidelines
- **UPDATE** to latest SDK version: Check GitHub releases
- **REMOVE** any JSON-RPC batching code
- **IMPLEMENT** structured output schemas for tools
- **UPGRADE** OAuth implementation to meet security requirements
- **TEST** thoroughly after protocol version updates

### Backward Compatibility
- **MAINTAIN** support for clients on older protocol versions when possible
- **DOCUMENT** breaking changes in release notes
- **PROVIDE** migration scripts for major version updates