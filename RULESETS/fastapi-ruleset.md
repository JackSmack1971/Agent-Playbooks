---
trigger: glob
description: Comprehensive ruleset for FastAPI development covering setup, security, performance, testing, and deployment best practices based on v0.115.12
globs: ["**/*.py", "**/requirements.txt", "**/Dockerfile", "**/docker-compose.yml", "**/pyproject.toml"]
---

# FastAPI Rules

## Project Structure and Setup

- **Use proper package structure**: Organize larger applications with separate modules for routes, models, dependencies, and configuration
- **Install with standard dependencies**: Use `pip install "fastapi[standard]"` for most use cases, or `pip install "fastapi[standard-no-fastapi-cloud-cli]"` if cloud CLI is not needed
- **Pin dependencies properly**: Use exact versions for production (`fastapi==0.115.12`) or compatible ranges (`fastapi>=0.115.0,<0.116.0`) for development
- **Create virtual environments**: Always use isolated Python environments (`python -m venv env`) for dependency management

```python
# Recommended project structure
app/
├── __init__.py
├── main.py           # FastAPI app instance
├── dependencies.py   # Shared dependencies
├── config.py        # Application settings
├── routers/         # API route modules
│   ├── __init__.py
│   ├── items.py
│   └── users.py
└── models/          # Pydantic models
    ├── __init__.py
    └── item.py
```

## Application Configuration

- **Use Pydantic Settings for configuration**: Leverage `BaseSettings` for environment-based configuration management
- **Store secrets in environment variables**: Never hardcode sensitive data in source code
- **Configure OpenAPI conditionally**: Use environment variables to control documentation availability in production
- **Set up dependency injection for settings**: Use `@lru_cache()` decorator for singleton settings objects

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")
    
    app_name: str = "FastAPI App"
    admin_email: str
    secret_key: str
    database_url: str
    openapi_url: str = "/openapi.json"  # Set to "" in production to disable

@lru_cache()
def get_settings():
    return Settings()

app = FastAPI(openapi_url=get_settings().openapi_url)
```

## Security Best Practices

- **Always use HTTPS in production**: Configure SSL/TLS certificates and disable HTTP
- **Implement proper CORS policies**: Configure `CORSMiddleware` with specific allowed origins, not wildcard "*"
- **Validate all user inputs**: Use Pydantic models for request validation and sanitize user-generated content
- **Implement rate limiting**: Protect against DoS attacks with request throttling
- **Use dependency injection for authentication**: Centralize auth logic in reusable dependencies
- **Store passwords securely**: Always hash passwords, never store plaintext
- **Disable debug mode in production**: Set debug=False and remove auto-reload
- **Implement proper authorization**: Check user permissions at both endpoint and business logic levels

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    # Validate JWT token and return user
    if not validate_token(credentials.credentials):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authentication credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return get_user_from_token(credentials.credentials)

@app.get("/protected")
async def protected_endpoint(user = Depends(get_current_user)):
    return {"message": f"Hello {user.username}"}
```

## Router and Endpoint Configuration

- **Use APIRouter for modular organization**: Group related endpoints with shared configuration
- **Configure router-level dependencies**: Apply common dependencies like authentication at router level
- **Set proper HTTP status codes**: Use appropriate status codes for different response scenarios
- **Add comprehensive documentation**: Include summary, description, and response examples
- **Handle errors gracefully**: Use HTTPException with proper status codes and error details

```python
from fastapi import APIRouter, Depends, HTTPException, status

router = APIRouter(
    prefix="/items",
    tags=["items"],
    dependencies=[Depends(get_current_user)],
    responses={404: {"description": "Not found"}},
)

@router.get(
    "/{item_id}",
    summary="Get item by ID",
    description="Retrieve a specific item using its unique identifier",
    response_model=ItemResponse,
)
async def get_item(item_id: int):
    if item_id < 1:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Item ID must be positive"
        )
    # Implementation here
```

## Database and ORM Integration

- **Use SQLAlchemy ORM**: Avoid raw SQL queries to prevent injection attacks
- **Implement proper session management**: Use dependency injection for database sessions
- **Handle database connections properly**: Ensure connections are closed after use
- **Use database migrations**: Implement Alembic for schema versioning
- **Validate database operations**: Check for foreign key constraints and business rules

```python
from sqlalchemy.orm import Session
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

## Async Programming Best Practices

- **Use async/await consistently**: Define async functions for I/O operations, sync for CPU-bound tasks
- **Avoid blocking operations in async functions**: Use asyncio-compatible libraries for database, HTTP clients
- **Handle async exceptions properly**: Use try/catch blocks around async operations
- **Implement background tasks correctly**: Use BackgroundTasks for fire-and-forget operations
- **Configure async database sessions**: Use async SQLAlchemy for better performance

```python
from fastapi import BackgroundTasks

async def send_notification(email: str, message: str):
    # Async operation - use aiosmtplib or similar
    await async_send_email(email, message)

@app.post("/notify")
async def create_notification(
    notification: NotificationCreate,
    background_tasks: BackgroundTasks
):
    # Add background task without waiting
    background_tasks.add_task(send_notification, notification.email, notification.message)
    return {"message": "Notification scheduled"}
```

## Testing Configuration

- **Use TestClient for API testing**: Import from `fastapi.testclient` for synchronous tests
- **Install required test dependencies**: Install `httpx` for TestClient and `pytest` for test framework
- **Override dependencies for testing**: Use `app.dependency_overrides` to inject test configurations
- **Test both success and error scenarios**: Include tests for validation errors and edge cases
- **Use fixtures for test data**: Create reusable test data with pytest fixtures
- **Test async endpoints properly**: Use `pytest-asyncio` for async test functions

```python
from fastapi.testclient import TestClient
from .main import app, get_settings

def test_settings_override():
    def get_test_settings():
        return Settings(admin_email="test@example.com")
    
    app.dependency_overrides[get_settings] = get_test_settings
    
    client = TestClient(app)
    response = client.get("/info")
    assert response.status_code == 200
    assert response.json()["admin_email"] == "test@example.com"
    
    # Clean up
    app.dependency_overrides.clear()
```

## Docker Deployment

- **Use multi-stage builds for Poetry**: Separate dependency resolution from runtime image
- **Configure proper CMD instruction**: Use uvicorn with correct module path and options
- **Enable proxy headers for reverse proxies**: Add `--proxy-headers` flag when behind load balancers
- **Set appropriate working directory**: Use `/code` as standard working directory
- **Optimize layer caching**: Copy requirements first, then source code
- **Use slim Python images**: Choose `python:3.11-slim` for smaller image size

```dockerfile
FROM python:3.11-slim

WORKDIR /code

# Install dependencies first for better caching
COPY ./requirements.txt /code/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt

# Copy application code
COPY ./app /code/app

# Configure for production deployment
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80", "--proxy-headers"]
```

## Performance Optimization

- **Configure appropriate worker processes**: Use multiple Uvicorn workers for CPU-bound applications
- **Implement response caching**: Cache expensive computations and database queries
- **Use streaming responses for large data**: Implement StreamingResponse for large file downloads
- **Optimize database queries**: Use eager loading and avoid N+1 query problems
- **Configure connection pooling**: Set appropriate database connection pool sizes
- **Monitor performance metrics**: Implement logging and monitoring for response times

```python
from fastapi.responses import StreamingResponse
import asyncio

@app.get("/large-file")
async def download_large_file():
    def generate_data():
        for chunk in get_file_chunks():
            yield chunk
    
    return StreamingResponse(
        generate_data(),
        media_type="application/octet-stream",
        headers={"Content-Disposition": "attachment; filename=data.txt"}
    )
```

## Known Issues and Mitigations

- **Background task execution failures**: Ensure database sessions are properly scoped for background tasks
- **Async execution bottlenecks**: Avoid blocking calls in async functions; use `asyncio.create_task()` for concurrent operations
- **Memory leaks in long-running applications**: Properly close database connections and clear large objects
- **CORS preflight issues**: Configure OPTIONS method handling for complex CORS scenarios
- **File upload size limits**: Configure appropriate request size limits in your ASGI server
- **Dependency override cleanup**: Always clear `app.dependency_overrides` after testing

```python
# Fix for background task database session issues
from contextlib import contextmanager

@contextmanager
def get_db_for_background_task():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def background_db_operation(item_id: int):
    with get_db_for_background_task() as db:
        # Perform database operations
        item = db.query(Item).filter(Item.id == item_id).first()
        # Process item
```

## Production Deployment Checklist

- **Environment variables configured**: All secrets and config in environment, not code
- **HTTPS enabled**: SSL/TLS certificates configured and HTTP redirects in place
- **Debug mode disabled**: `debug=False` and no `--reload` flag
- **Proper logging configured**: Structured logging with appropriate levels
- **Health checks implemented**: `/health` endpoint for load balancer monitoring
- **Rate limiting active**: API rate limiting and request size limits configured
- **Monitoring in place**: Error tracking (Sentry) and performance monitoring
- **Database migrations applied**: Schema is current and migrations tested
- **Security headers configured**: CORS, CSP, and other security headers set
- **Backup and recovery tested**: Database backups and restore procedures verified

```python
@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow()}

# Production logging configuration
import logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)
```

## Version Compatibility Notes

- **Current version tested**: FastAPI 0.115.12
- **Python compatibility**: Supports Python 3.8+, recommended Python 3.11+
- **Pydantic v2**: Use `pydantic_settings` for BaseSettings import
- **Starlette compatibility**: Ensure compatible Starlette version for middleware
- **Breaking changes**: Review release notes when upgrading minor versions
- **Dependency updates**: Regular security updates for all dependencies