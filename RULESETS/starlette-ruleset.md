---
trigger: model_decision
description: Comprehensive ruleset for Starlette ASGI framework development covering application structure, routing, middleware, async patterns, WebSockets, testing, and production best practices
globs: ["**/*.py"]
---

# Starlette Framework Rules

## Application Structure and Configuration

- Use `Starlette` application class for full framework features with routing, middleware, and exception handling
- Enable debug mode only in development: `app = Starlette(debug=True, routes=routes)` for development, `debug=False` for production
- Use `Config` and `Secret` types for environment-based configuration management with `.env` files
- Structure applications with separate modules: `settings.py` for config, `routes.py` for routing, `middleware.py` for custom middleware
- Always define routes as a list and pass to `Starlette(routes=routes)` constructor rather than using decorators for better maintainability

```python
from starlette.applications import Starlette
from starlette.config import Config, Secret
from starlette.routing import Route

config = Config(".env")
DEBUG = config('DEBUG', cast=bool, default=False)
SECRET_KEY = config('SECRET_KEY', cast=Secret)

async def homepage(request):
    return JSONResponse({'hello': 'world'})

routes = [Route('/', endpoint=homepage)]
app = Starlette(debug=DEBUG, routes=routes)
```

## Routing and URL Patterns

- Define routes using explicit `Route` objects in a routes list rather than decorators for better testability and IDE support
- Use path parameters with type hints: `Route('/users/{user_id:int}', user_detail)` for automatic conversion and validation
- Order routes from most specific to least specific to prevent path conflicts
- Use `Mount` for serving static files: `Mount('/static', StaticFiles(directory='static'), name='static')`
- Implement custom path parameter convertors when needed with `register_url_convertor()`
- Use `methods` parameter to restrict allowed HTTP methods: `Route('/users', create_user, methods=['POST'])`

```python
from starlette.routing import Route, Mount
from starlette.staticfiles import StaticFiles

routes = [
    Route('/users/me', current_user),  # More specific first
    Route('/users/{user_id:int}', user_detail),  # Less specific after
    Route('/api/v1/health', health_check, methods=['GET']),
    Mount('/static', StaticFiles(directory='static'), name='static'),
]
```

## Middleware Implementation and Ordering

- **CRITICAL**: Avoid `BaseHTTPMiddleware` when possible due to cancellation and context propagation issues
- Use pure ASGI middleware (function-based) for better control and reliability
- Apply middleware in correct order: security middleware first, then authentication, then application-specific middleware
- Never read `request.body()` in middleware unless you buffer and restore it for downstream handlers
- Use `Middleware` wrapper for cleaner configuration in application setup
- Wrap entire application with global middleware like CORS for consistent error handling

```python
from starlette.middleware import Middleware
from starlette.middleware.cors import CORSMiddleware
from starlette.middleware.sessions import SessionMiddleware

# Pure ASGI middleware example
class TimingMiddleware:
    def __init__(self, app):
        self.app = app
    
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        start_time = time.time()
        try:
            await self.app(scope, receive, send)
        finally:
            process_time = time.time() - start_time
            # Log timing without affecting response

middleware = [
    Middleware(TimingMiddleware),
    Middleware(SessionMiddleware, secret_key=settings.SECRET_KEY),
]

app = Starlette(routes=routes, middleware=middleware)
# Global CORS wrapper
app = CORSMiddleware(app, allow_origins=["*"])
```

## Async Handling and Request Processing

- Always use `async def` for endpoint functions that perform I/O operations
- Use `await request.json()` for JSON bodies, `await request.form()` for form data
- Handle request body streaming with `async for chunk in request.stream():` for memory efficiency
- Use `BackgroundTask` or `BackgroundTasks` for post-response processing instead of fire-and-forget tasks
- Implement proper cancellation handling in long-running operations
- Access request properties synchronously: `request.path_params`, `request.query_params`, `request.headers`

```python
from starlette.background import BackgroundTask
from starlette.responses import JSONResponse

async def process_upload(request):
    # Stream large files instead of loading into memory
    body = b''
    async for chunk in request.stream():
        body += chunk
    
    # Background task for post-processing
    task = BackgroundTask(send_notification, user_id=request.path_params['user_id'])
    return JSONResponse({'status': 'processing'}, background=task)

async def send_notification(user_id: int):
    # This runs after response is sent
    pass
```

## Response Types and Content Handling

- Use appropriate response classes: `JSONResponse`, `HTMLResponse`, `PlainTextResponse`, `RedirectResponse`
- Set correct media types and status codes explicitly
- Use `StreamingResponse` for large content or real-time data
- Implement custom response classes by overriding `render()` method for specialized serialization
- Return response objects from endpoints, never plain dictionaries or strings
- Use `send_push_promise()` for HTTP/2 server push when serving assets

```python
from starlette.responses import JSONResponse, StreamingResponse
import orjson

class ORJSONResponse(JSONResponse):
    def render(self, content: Any) -> bytes:
        return orjson.dumps(content)

async def streaming_data(request):
    async def generate():
        for i in range(1000):
            yield f"data: {i}\n"
            await asyncio.sleep(0.1)
    
    return StreamingResponse(generate(), media_type="text/plain")
```

## WebSocket Management

- Use `WebSocketRoute` for WebSocket endpoints in routing configuration
- Always `await websocket.accept()` before sending/receiving messages
- Handle WebSocket disconnections gracefully with proper cleanup
- Use `websocket.path_params` for URL parameters in WebSocket routes
- Implement authorization checks before accepting WebSocket connections
- Handle different message types: text, bytes, JSON appropriately

```python
from starlette.routing import WebSocketRoute
from starlette.exceptions import HTTPException

async def websocket_endpoint(websocket):
    # Authorization before accepting
    if not is_authorized(websocket.headers):
        raise HTTPException(status_code=401, detail="Unauthorized")
    
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        # Handle disconnect cleanup
        pass
    finally:
        # Ensure cleanup happens
        await cleanup_websocket_resources()

routes = [
    WebSocketRoute('/ws/{room_id:int}', websocket_endpoint),
]
```

## Testing Patterns and Best Practices

- Use `TestClient` for HTTP testing with automatic lifespan management
- Use `httpx.AsyncClient` with `ASGITransport` for async testing scenarios
- Test WebSockets using `client.websocket_connect()` as context manager
- Set custom headers globally with `client.headers` or per-request with `headers` parameter
- Use different async backends (asyncio, trio) by specifying `backend` parameter
- Mock external dependencies in tests rather than testing with real services

```python
from starlette.testclient import TestClient
from httpx import AsyncClient, ASGITransport
import pytest

def test_homepage():
    client = TestClient(app)
    response = client.get('/')
    assert response.status_code == 200
    assert response.json() == {'hello': 'world'}

def test_websocket():
    client = TestClient(app)
    with client.websocket_connect('/ws') as websocket:
        websocket.send_text("hello")
        data = websocket.receive_text()
        assert data == "Echo: hello"

@pytest.mark.asyncio
async def test_async_endpoint():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        response = await client.get('/')
        assert response.status_code == 200
```

## Security Implementation

- Use `TrustedHostMiddleware` to validate Host headers and prevent Host header attacks
- Enable `HTTPSRedirectMiddleware` in production to enforce HTTPS
- Configure `SessionMiddleware` with secure settings: `https_only=True`, appropriate `same_site` values
- Implement proper authentication backends with `AuthenticationBackend` and `AuthenticationMiddleware`
- Use `@requires()` decorator for route-level permission checks
- Validate and sanitize all user inputs, especially path parameters and query strings

```python
from starlette.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware
from starlette.authentication import requires

middleware = [
    Middleware(TrustedHostMiddleware, allowed_hosts=['example.com', '*.example.com']),
    Middleware(HTTPSRedirectMiddleware),
    Middleware(SessionMiddleware, 
               secret_key=settings.SECRET_KEY,
               https_only=True,
               same_site='strict'),
]

@requires('authenticated', redirect='login')
async def protected_endpoint(request):
    return JSONResponse({'user': request.user.display_name})
```

## Performance Optimization

- Use `GZipMiddleware` for response compression with appropriate `minimum_size` and `compresslevel`
- Implement response caching strategies for static or semi-static content
- Use connection pooling for database connections through lifespan handlers
- Stream large responses instead of loading into memory
- Optimize static file serving with `StaticFiles` and proper caching headers
- Use `uvloop` for improved async performance in production

```python
from starlette.middleware.gzip import GZipMiddleware
from contextlib import asynccontextmanager
import httpx

@asynccontextmanager
async def lifespan(app):
    # Startup: create shared resources
    async with httpx.AsyncClient() as client:
        app.state.http_client = client
        yield
    # Shutdown: cleanup happens automatically

middleware = [
    Middleware(GZipMiddleware, minimum_size=1000, compresslevel=6),
]

app = Starlette(routes=routes, middleware=middleware, lifespan=lifespan)
```

## Exception Handling and Error Management

- Define custom exception handlers for different error types and status codes
- Use `HTTPException` for expected HTTP errors with proper status codes
- Implement global exception handlers for unhandled errors with logging
- Return consistent error response formats across your application
- Handle WebSocket exceptions separately with `WebSocketException`
- Log errors appropriately without exposing sensitive information

```python
from starlette.exceptions import HTTPException
from starlette.requests import Request
from starlette.responses import JSONResponse

async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail, "status_code": exc.status_code}
    )

async def general_exception_handler(request: Request, exc: Exception):
    # Log error for debugging
    logger.error(f"Unhandled error: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"error": "Internal server error"}
    )

exception_handlers = {
    HTTPException: http_exception_handler,
    Exception: general_exception_handler,
}

app = Starlette(routes=routes, exception_handlers=exception_handlers)
```

## Template and Static File Handling

- Use `Jinja2Templates` for server-side rendering with proper template directory configuration
- Configure template environment with custom filters and functions when needed
- Use `StaticFiles` with `Mount` for serving static assets efficiently
- Set proper `name` parameters for URL reversing with `url_for()`
- Serve static files from packages using `packages` parameter for reusable components
- Enable HTML mode for directory browsing if needed: `StaticFiles(html=True)`

```python
from starlette.templating import Jinja2Templates
from starlette.staticfiles import StaticFiles

templates = Jinja2Templates(directory='templates')

async def homepage(request):
    return templates.TemplateResponse(
        request=request,
        name='index.html',
        context={'title': 'Home Page'}
    )

routes = [
    Route('/', homepage),
    Mount('/static', StaticFiles(directory='static'), name='static'),
    Mount('/assets', StaticFiles(packages=['bootstrap4']), name='assets'),
]
```

## Known Issues and Mitigations

- **BaseHTTPMiddleware Cancellation**: Avoid `BaseHTTPMiddleware` in production; use pure ASGI middleware instead
- **Request Body Consumption**: Never read `request.body()` in middleware without buffering for downstream use
- **WebSocket State Management**: Implement proper cleanup in WebSocket disconnect handlers to prevent memory leaks
- **Middleware Ordering**: Test middleware combinations thoroughly as ordering affects behavior significantly
- **Task Cancellation**: Use Starlette's built-in background task mechanisms rather than creating raw asyncio tasks
- **Static File Conflicts**: Ensure static file routes don't conflict with API routes by proper ordering

## Production Deployment

- Run with production ASGI servers: Uvicorn, Hypercorn, or Daphne
- Set environment-specific configuration using `Config` class with `.env` files
- Enable proper logging configuration for production monitoring
- Use reverse proxy (nginx) for static file serving and SSL termination
- Configure health check endpoints for load balancer monitoring
- Set appropriate worker counts and resource limits for the ASGI server

```python
# Production server command
# uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4 --access-log

# Health check endpoint
async def health_check(request):
    return JSONResponse({"status": "healthy", "version": "1.0.0"})

routes = [
    Route('/health', health_check, methods=['GET']),
    # ... other routes
]
```