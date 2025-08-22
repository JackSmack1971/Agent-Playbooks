---
trigger: glob
description: Comprehensive ruleset for SlowAPI middleware - a rate limiter for Starlette and FastAPI applications
globs: ["**/slowapi/**", "**/*slowapi*", "**/middleware/*rate*", "**/rate_limit*"]
---

# SlowAPI Middleware Rules

## Installation and Dependencies

- **CRITICAL**: Pin SlowAPI version in production environments (latest stable: ~0.1.9)
- **REQUIRED**: Ensure compatible versions: `slowapi>=0.1.5`, `fastapi>=0.65.0`, `starlette>=0.25.0`
- **VERSION COMPATIBILITY**: Recent Starlette version conflicts (2024-2025) require careful dependency management
- Install with: `pip install slowapi`
- For Redis backend: `pip install slowapi[redis]`

```python
# requirements.txt example
slowapi==0.1.9
fastapi>=0.115.8,<1.0
starlette>=0.36,<1.0
redis>=4.3.0  # if using Redis backend
```

## Basic Setup and Configuration

- **MANDATORY**: Import required components from slowapi and slowapi.errors
- **REQUIRED**: Initialize Limiter with a key_func (typically get_remote_address)
- **ESSENTIAL**: Add limiter to app.state for access across routes
- **CRITICAL**: Register RateLimitExceeded exception handler

```python
from fastapi import FastAPI, Request, Response
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

## Middleware Integration

- **PREFERRED**: Use SlowAPIASGIMiddleware for better performance and async support
- **LEGACY**: SlowAPIMiddleware (WSGI) available but less performant
- **WARNING**: Starlette may deprecate BaseHTTPMiddleware in future versions
- Add middleware using app.add_middleware() method

```python
from slowapi.middleware import SlowAPIMiddleware, SlowAPIASGIMiddleware

# Preferred ASGI middleware
app.add_middleware(SlowAPIASGIMiddleware)

# Legacy WSGI middleware (not recommended for new projects)
# app.add_middleware(SlowAPIMiddleware)
```

## Request Parameter Requirements

- **CRITICAL ERROR**: Missing request parameter causes silent failure of rate limiting
- **MANDATORY**: All decorated endpoints MUST include `request: Request` parameter
- **BREAKING**: Endpoints without request parameter will not be rate limited

```python
# CORRECT - request parameter included
@limiter.limit("5/minute")
async def correct_endpoint(request: Request):
    return {"message": "ok"}

# INCORRECT - will not be rate limited
@limiter.limit("5/minute")  # This will silently fail
async def broken_endpoint():
    return {"message": "not limited"}
```

## Decorator Order Requirements

- **CRITICAL**: Route decorator MUST come before limit decorator
- **BREAKING**: Wrong order causes routing failures
- **RULE**: @app.get() or @router.get() first, then @limiter.limit()

```python
# CORRECT order
@app.get("/test")
@limiter.limit("2/minute")
async def correct_order(request: Request):
    return "works"

# INCORRECT order - will break
@limiter.limit("2/minute")
@app.get("/test")  # This order fails
async def wrong_order(request: Request):
    return "broken"
```

## Rate Limit Patterns

- **FLEXIBLE**: Support for various time units: /second, /minute, /hour, /day
- **COMPLEX**: Multiple limits can be combined with semicolons
- **GRANULAR**: Supports fractional limits (e.g., "1/30seconds")

```python
# Simple limits
@limiter.limit("100/minute")
@limiter.limit("5/second")
@limiter.limit("1000/hour")

# Multiple combined limits
@limiter.limit("5/minute;100/hour;1000/day")

# Fractional periods
@limiter.limit("1/30seconds")
```

## Key Functions and Identification

- **DEFAULT**: get_remote_address for IP-based limiting
- **CUSTOM**: Define custom key functions for user-based or session-based limiting
- **PROXY**: Handle X-Forwarded-For headers for applications behind proxies

```python
from slowapi.util import get_remote_address, get_ipaddr

# IP-based limiting (default)
limiter = Limiter(key_func=get_remote_address)

# Custom user-based limiting
def get_user_id(request: Request):
    return request.headers.get("X-User-ID", "anonymous")

limiter = Limiter(key_func=get_user_id)

# Multiple factor identification
def composite_key(request: Request):
    user_id = request.headers.get("X-User-ID", "")
    ip = get_remote_address(request)
    return f"{user_id}:{ip}"
```

## Storage Backends

- **DEFAULT**: In-memory storage (development only)
- **PRODUCTION**: Redis recommended for distributed systems
- **ALTERNATIVES**: Memcached supported
- **URI FORMAT**: Use connection strings for external backends

```python
# Redis backend (recommended for production)
limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379/0"
)

# Redis with auth
limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://user:password@host:port/db"
)

# Memcached backend
limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="memcached://localhost:11211"
)
```

## Key Style Configuration

- **URL STYLE**: Different URL parameters create separate rate limits
- **ENDPOINT STYLE**: Routes with parameters share the same rate limit
- **DEFAULT**: "endpoint" style recommended for most use cases

```python
# URL-based keys (separate limits for /user/1, /user/2, etc.)
limiter = Limiter(
    key_func=get_remote_address,
    key_style="url"
)

# Endpoint-based keys (shared limits for all /user/{id} calls)
limiter = Limiter(
    key_func=get_remote_address,
    key_style="endpoint"  # default
)
```

## Global Default Limits

- **CONVENIENT**: Apply rate limits to all routes automatically
- **OVERRIDE**: Individual route limits override global defaults
- **EXEMPT**: Use @limiter.exempt to exclude specific routes

```python
# Global limits for all routes
limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["100/minute", "1000/hour"]
)

# Exempt specific routes from global limits
@app.get("/unlimited")
@limiter.exempt
async def unlimited_endpoint(request: Request):
    return "No limits here"
```

## Custom Cost Per Request

- **ADVANCED**: Define variable costs based on request characteristics
- **USE CASES**: Rate limit by request size, complexity, or resource usage
- **FUNCTION**: Cost function receives request object and returns integer

```python
# Cost based on request size
def request_size_cost(request: Request) -> int:
    content_length = request.headers.get("content-length", "0")
    return max(1, int(content_length) // 1024)  # Cost per KB

@app.post("/upload")
@limiter.limit("1000/minute", cost=request_size_cost)
async def upload(request: Request):
    return "Upload processed"

# Cost based on query complexity
def query_cost(request: Request) -> int:
    query_params = len(request.query_params)
    return max(1, query_params)

@app.get("/search")
@limiter.limit("100/minute", cost=query_cost)
async def search(request: Request):
    return "Search results"
```

## Response Handling

- **AUTOMATIC**: Default 429 status for rate limit exceeded
- **HEADERS**: Rate limit information automatically added to responses
- **CUSTOM**: Override default exception handler for custom error responses

```python
# Custom rate limit exceeded handler
async def custom_rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={
            "error": "Rate limit exceeded",
            "retry_after": exc.retry_after,
            "limit": str(exc.detail)
        }
    )

app.add_exception_handler(RateLimitExceeded, custom_rate_limit_handler)
```

## Response Object Requirements

- **EXPLICIT**: Pass response object when returning non-Response data
- **HEADERS**: Allows SlowAPI to add rate limit headers
- **PATTERN**: Include response parameter for dict/model returns

```python
# When returning Response objects directly
@limiter.limit("5/minute")
async def direct_response(request: Request):
    return Response("Direct response content")

# When returning dict/model data - include response parameter
@limiter.limit("5/minute")
async def json_response(request: Request, response: Response):
    return {"key": "value"}  # SlowAPI can add headers to response
```

## Disabling Rate Limiting

- **TESTING**: Disable rate limiting during development/testing
- **GLOBAL**: Set enabled=False during initialization
- **RUNTIME**: Modify limiter.enabled property dynamically

```python
# Disable during initialization
limiter = Limiter(key_func=get_remote_address, enabled=False)

# Disable at runtime
limiter.enabled = False

# Conditional enabling based on environment
import os
limiter = Limiter(
    key_func=get_remote_address,
    enabled=os.getenv("ENABLE_RATE_LIMITING", "true").lower() == "true"
)
```

## Known Issues and Mitigations

### Starlette Version Conflicts (2024-2025)
- **ISSUE**: Dependency conflicts between FastAPI and Starlette versions
- **SOLUTION**: Use compatible version ranges in requirements
- **MONITORING**: Check GitHub issues for version compatibility updates

### Silent Failures
- **ISSUE**: Missing request parameter causes silent rate limiting failure
- **DETECTION**: Verify rate limiting works in tests
- **SOLUTION**: Always include request: Request in endpoint signatures

### Middleware Ordering
- **ISSUE**: Multiple middleware additions can cause unexpected behavior
- **SOLUTION**: Add SlowAPI middleware once, preferably early in middleware stack
- **BEST PRACTICE**: Use middleware parameter during app initialization when possible

### Alpha Quality Warning
- **STATUS**: SlowAPI is marked as alpha quality software
- **PRECAUTION**: Pin exact versions in production
- **TESTING**: Thoroughly test rate limiting behavior before deployment

## Testing and Debugging

- **VERIFICATION**: Test rate limiting behavior with automated tests
- **MONITORING**: Log rate limit headers for debugging
- **TOOLS**: Use tools like curl or httpx to verify rate limiting

```python
# Test example
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_rate_limiting():
    async with AsyncClient(app=app, base_url="http://test") as client:
        # First request should succeed
        response = await client.get("/limited-endpoint")
        assert response.status_code == 200
        
        # Subsequent requests should be rate limited
        for _ in range(10):
            response = await client.get("/limited-endpoint")
        assert response.status_code == 429
```

## Security Considerations

- **BYPASS**: Be aware of potential bypasses through IP spoofing
- **PROXY**: Handle proxy headers appropriately for accurate client identification
- **DDOS**: Rate limiting helps but isn't complete DDoS protection
- **REDIS**: Secure Redis instances when used as storage backend

## Performance Optimization

- **ASGI**: Prefer SlowAPIASGIMiddleware over WSGI version
- **REDIS**: Use Redis backend for better performance in distributed systems
- **CONNECTION POOLING**: Configure Redis connection pooling for high traffic
- **MONITORING**: Monitor rate limiter performance impact

## Production Deployment

- **BACKEND**: Always use external storage (Redis/Memcached) in production
- **MONITORING**: Monitor rate limit metrics and false positives
- **SCALING**: Consider rate limiter state in horizontal scaling scenarios
- **BACKUP**: Plan for rate limiter backend availability

```python
# Production-ready configuration
limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://redis-cluster:6379/0",
    default_limits=["1000/minute", "10000/hour"],
    key_style="endpoint"
)
```