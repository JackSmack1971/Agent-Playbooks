---
trigger: ["model_decision"]
description: Comprehensive rules for developing Model Context Protocol (MCP) servers and clients using the Python MCP SDK. Covers FastMCP framework, low-level server APIs, authentication, deployment, and best practices.
globs: ["**/*mcp*.py", "**/server*.py", "**/client*.py", "**/fastmcp*.py"]
version: "0.1.0"
last_updated: "2025-01-01"
---

# Python MCP SDK Rules

## Installation & Setup

- **ALWAYS** install MCP with CLI tools: `uv add "mcp[cli]"` or `pip install "mcp[cli]"`
- **ALWAYS** use virtual environments for dependency isolation: `python -m venv .venv` or `uv venv`
- **RECOMMENDED** use `uv` for faster package management and dependency resolution
- **REQUIRED** Python 3.8+ for compatibility with MCP SDK

```bash
# Install with uv (recommended)
uv add "mcp[cli]"

# Or with pip
pip install "mcp[cli]"
```

## Configuration & Initialization

- **STORE** sensitive credentials in environment variables, never in code
- **USE** `.env` files for local development configuration
- **SET** `FASTMCP_DEBUG=false` in production environments
- **CONFIGURE** log levels appropriately: `FASTMCP_LOG_LEVEL=INFO` (production) or `DEBUG` (development)

## Core Concepts / API Usage

### FastMCP Framework (Recommended Approach)

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

## Security & Permissions

- **NEVER** hard-code credentials in source code
- **ALWAYS** validate and sanitize external input
- **USE** HTTPS in production deployments
- **IMPLEMENT** proper rate limiting for public endpoints
- **STORE** tokens securely with encryption at rest
- **VALIDATE** OAuth scopes for each protected resource

## Performance & Scalability

- **USE** `async`/`await` for all I/O operations
- **IMPLEMENT** connection pooling for database/HTTP clients
- **SET** appropriate timeouts using float values for precision
- **BATCH** operations when possible to reduce overhead

## Error Handling & Troubleshooting

- **IMPLEMENT** proper exception management with logging
- **USE** context for progress reporting and error communication
- **VALIDATE** input parameters before processing
- **HANDLE** network timeouts and connection errors gracefully

```python
import logging
from mcp.server.fastmcp import Context

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

## Testing

- **CREATE** test server instances for unit testing
- **MOCK** external dependencies and network calls
- **VALIDATE** tool schemas and resource patterns
- **TEST** error conditions and edge cases

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

## Deployment & Production Patterns

- **USE** stateless HTTP for cloud deployment and horizontal scaling
- **IMPLEMENT** health check endpoints for load balancers
- **CONFIGURE** proper timeouts and connection limits
- **MONITOR** server performance and error rates

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

## Known Issues & Mitigations

- **ENOENT Errors on macOS**: Rebuild virtual environment after Homebrew upgrades
- **Connection Timeout Issues**: Use float timeouts for precise control
- **Memory Leaks in Long-Running Servers**: Close connections properly in lifespan context
- **OAuth Token Validation**: Implement proper token verification with scope checking

## Version Compatibility Notes

- **Current version tested**: Python MCP SDK 0.1.0
- **Python compatibility**: Requires Python 3.8+
- **Breaking changes**: Major API changes between versions - test thoroughly when upgrading
- **FastMCP updates**: Regular updates for new features and bug fixes

## References

- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [Python MCP SDK Documentation](https://github.com/modelcontextprotocol/python-sdk)
- [FastMCP Framework Guide](https://fastmcp.io/)
- [MCP Community Resources](https://modelcontextprotocol.io/community)