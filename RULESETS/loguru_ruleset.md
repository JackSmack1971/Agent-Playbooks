---
trigger: glob
description: Comprehensive ruleset for loguru Python logging library, covering installation, configuration, usage patterns, multiprocessing, async operations, performance optimization, and security best practices
globs: ["**/*.py"]
---

# Loguru Rules

## Installation and Setup

- **Install loguru using pip**: `pip install loguru` - no additional dependencies required
- **Import with single line**: `from loguru import logger` - ready to use immediately with zero configuration
- **Verify installation**: Check with `import loguru; print(loguru.__version__)` to confirm proper installation
- **Environment setup**: Load environment variables and configuration before importing loguru to ensure proper initialization
- **Virtual environment**: Always install in virtual environment for dependency isolation

```python
# Correct installation verification
import loguru
print(loguru.__version__)

# Basic usage - works immediately
from loguru import logger
logger.info("Loguru is ready!")
```

## Basic Usage and Configuration

- **Remove default handler first**: Always call `logger.remove()` before adding custom handlers to avoid duplicate logs
- **Use logger.add() for configuration**: Add sinks with `logger.add(sink, **options)` - supports files, stdout/stderr, functions, and custom handlers
- **Set appropriate log levels**: Use DEBUG for development, INFO/WARNING for production to control verbosity
- **Configure before application logic**: Set up all logging configuration before starting main application code

```python
# Proper basic configuration
from loguru import logger
import sys

# Remove default handler to avoid duplicates
logger.remove()

# Add console handler with custom format
logger.add(
    sys.stderr,
    format="{time:YYYY-MM-DD HH:mm:ss.SSS} | {level: <8} | {name}:{function}:{line} - {message}",
    level="INFO",
    colorize=True
)

# Add file handler with rotation
logger.add(
    "app.log",
    format="{time:YYYY-MM-DD HH:mm:ss.SSS} | {level: <8} | {name}:{function}:{line} - {message}",
    level="DEBUG",
    rotation="10 MB",
    retention="10 days",
    compression="zip"
)
```

## Message Formatting and Context

- **Use braces for interpolation**: Use `logger.info("User {name} logged in", name=username)` instead of f-strings for lazy evaluation
- **Avoid f-strings in log messages**: F-strings are evaluated even when log level is disabled, wasting CPU cycles
- **Use .bind() for context**: Add context with `logger.bind(user_id=123)` to include extra fields in structured logs
- **Structure log messages consistently**: Include relevant context like user IDs, request IDs, or operation identifiers

```python
# Correct - lazy evaluation with braces
logger.info("Processing file {filename} for user {user_id}", filename=file.name, user_id=user.id)

# Incorrect - f-strings always evaluate
logger.info(f"Processing file {file.name} for user {user.id}")

# Adding context
context_logger = logger.bind(request_id="req-123", user_id=456)
context_logger.info("Operation started")
context_logger.info("Operation completed successfully")
```

## File Logging and Rotation

- **Enable file rotation**: Use `rotation` parameter to prevent log files from growing indefinitely
- **Set retention policy**: Use `retention` parameter to automatically clean up old log files
- **Enable compression**: Use `compression="zip"` or `compression="gz"` to save disk space
- **Use appropriate file permissions**: Set `opener` parameter with custom function to control file access permissions

```python
import os

def secure_opener(file, flags):
    return os.open(file, flags, 0o600)  # Owner read/write only

logger.add(
    "secure.log",
    rotation="500 MB",      # Rotate when file reaches 500MB
    retention="30 days",    # Keep logs for 30 days
    compression="zip",      # Compress rotated files
    opener=secure_opener,   # Secure file permissions
    enqueue=True           # Thread-safe writing
)
```

## Exception Handling and Debugging

- **Use logger.exception() for caught exceptions**: Automatically includes traceback information
- **Enable backtrace and diagnose for debugging**: Set `backtrace=True` and `diagnose=True` for detailed error context
- **Use @logger.catch decorator**: Automatically log unhandled exceptions in functions with full traceback
- **Disable diagnose in production**: Set `diagnose=False` in production to avoid exposing sensitive variable values

```python
# Exception logging with context
try:
    result = risky_operation()
except Exception as e:
    logger.exception("Failed to complete operation for user {user_id}", user_id=current_user.id)

# Catch decorator for automatic exception handling
@logger.catch
def critical_function():
    # Any unhandled exception will be automatically logged
    return perform_operation()

# Development vs production configuration
logger.add(
    "errors.log",
    level="ERROR",
    backtrace=True,
    diagnose=False,  # Set to False in production for security
    format="{time} | {level} | {message}"
)
```

## Multiprocessing and Concurrency

- **Use enqueue=True for multiprocessing**: Enables thread-safe and process-safe logging through internal queue
- **Specify multiprocessing context**: Pass `context` parameter when using different multiprocessing start methods
- **Initialize logger before forking**: Set up logging configuration in main process before creating worker processes
- **Avoid fork-unsafe operations**: Don't add handlers with threads after forking processes

```python
import multiprocessing
from loguru import logger

def worker_function(task_id):
    logger.info("Worker {} processing task", task_id)
    return task_id * 2

if __name__ == "__main__":
    # Configure logger before creating processes
    logger.remove()
    logger.add(
        "multiprocess.log",
        enqueue=True,  # Required for multiprocessing safety
        format="{time} | {level} | {process} | {message}"
    )
    
    # Use spawn context for better compatibility
    context = multiprocessing.get_context("spawn")
    with context.Pool(4) as pool:
        results = pool.map(worker_function, range(10))
```

## Async and Asynchronous Operations

- **Use enqueue=True for async applications**: Prevents blocking of async event loops during log writes
- **Avoid blocking sinks in async code**: Use async-compatible sinks or queue-based writing
- **Configure before event loop**: Set up logging before starting async event loops or frameworks

```python
import asyncio
from loguru import logger

# Configure for async usage
logger.remove()
logger.add(
    "async_app.log",
    enqueue=True,  # Non-blocking async-safe logging
    format="{time} | {level} | {message}",
    level="INFO"
)

async def async_operation(item_id):
    logger.info("Starting async operation for item {}", item_id)
    await asyncio.sleep(1)  # Simulate async work
    logger.info("Completed async operation for item {}", item_id)

async def main():
    tasks = [async_operation(i) for i in range(5)]
    await asyncio.gather(*tasks)

if __name__ == "__main__":
    asyncio.run(main())
```

## Structured Logging and JSON Output

- **Enable JSON serialization**: Use `serialize=True` for machine-readable log format
- **Include correlation IDs**: Use .bind() to add request IDs, trace IDs, or correlation identifiers
- **Structure extra fields consistently**: Maintain consistent field names across your application
- **Use for log aggregation systems**: JSON format integrates well with ELK stack, Splunk, or cloud logging

```python
# JSON logging configuration
logger.add(
    "structured.log",
    serialize=True,  # Output in JSON format
    format="{time} | {level} | {message}",
    level="INFO"
)

# Adding structured context
request_logger = logger.bind(
    request_id="req-12345",
    user_id=789,
    endpoint="/api/users",
    method="POST"
)

request_logger.info("Request started")
request_logger.info("Validation completed")
request_logger.info("Request completed", response_time_ms=150)
```

## Performance Optimization

- **Set appropriate log levels**: Use INFO or WARNING in production, avoid DEBUG level for performance
- **Use lazy evaluation**: Prefer brace formatting over f-strings for conditional log evaluation
- **Enable enqueue for I/O intensive logging**: Prevents blocking main thread during file writes
- **Monitor log volume**: Implement log sampling or rate limiting for high-volume applications

```python
# Performance-optimized configuration
logger.remove()

# Production console logging - minimal overhead
logger.add(
    sys.stderr,
    level="WARNING",  # Only warnings and errors to console
    format="{time:HH:mm:ss} | {level} | {message}",
    colorize=False    # Disable colors for better performance
)

# Async file logging for better performance
logger.add(
    "app.log",
    level="INFO",
    enqueue=True,     # Non-blocking writes
    rotation="100 MB",
    compression="gz",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level} | {message}"
)

# Conditional expensive logging
if logger.isEnabledFor("DEBUG"):
    expensive_data = compute_debug_info()
    logger.debug("Debug info: {}", expensive_data)
```

## Security and Production Considerations

- **Disable diagnose in production**: Set `diagnose=False` to prevent exposing variable values in tracebacks
- **Sanitize sensitive data**: Never log passwords, API keys, or personal information
- **Set secure file permissions**: Use custom opener to restrict log file access
- **Implement log rotation and retention**: Prevent log files from consuming excessive disk space
- **Monitor for sensitive data exposure**: Regularly audit logs for accidentally logged sensitive information

```python
import os
import re

def secure_opener(file, flags):
    """Custom opener with secure permissions"""
    return os.open(file, flags, 0o600)

def sanitize_message(record):
    """Custom function to sanitize sensitive data"""
    # Remove credit card numbers, emails, etc.
    message = record["message"]
    message = re.sub(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b', '[CARD-REDACTED]', message)
    message = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL-REDACTED]', message)
    record["message"] = message
    return record

# Production-ready secure configuration
logger.add(
    "secure_app.log",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level} | {name}:{line} | {message}",
    level="INFO",
    rotation="50 MB",
    retention="90 days",
    compression="gz",
    opener=secure_opener,
    enqueue=True,
    backtrace=True,
    diagnose=False,  # Never expose variable values in production
    filter=sanitize_message  # Sanitize sensitive data
)
```

## Library Integration and Testing

- **Disable library loggers in production**: Use `logger.disable("library_name")` to suppress third-party library logs
- **Enable for testing**: Use `logger.enable("library_name")` to re-enable logs during debugging
- **Configure for testing frameworks**: Integrate with pytest and other testing tools appropriately
- **Use consistent logger names**: Use `__name__` for logger identification in modules

```python
# Library integration
from loguru import logger

# Disable noisy third-party libraries
logger.disable("urllib3")
logger.disable("requests")
logger.disable("boto3")

# Re-enable for debugging when needed
# logger.enable("requests")

# Module-specific logging
module_logger = logger.bind(module=__name__)

def library_function():
    module_logger.info("Function called in {}", __name__)
```

## Environment-Specific Configuration

- **Load config before imports**: Ensure environment variables are set before importing loguru
- **Use environment variables**: Leverage `LOGURU_LEVEL`, `LOGURU_FORMAT` for runtime configuration
- **Implement environment-specific settings**: Different log levels and formats for dev/staging/production
- **Centralize configuration**: Use configuration files or environment-based settings

```python
import os
from loguru import logger

# Environment-specific configuration
def configure_logging():
    logger.remove()
    
    environment = os.getenv("ENVIRONMENT", "development")
    
    if environment == "production":
        # Production: JSON logs, higher level
        logger.add(
            "app.log",
            serialize=True,
            level="INFO",
            rotation="100 MB",
            retention="30 days",
            compression="gz",
            enqueue=True,
            diagnose=False
        )
    elif environment == "development":
        # Development: Colorized console, debug level
        logger.add(
            sys.stderr,
            colorize=True,
            level="DEBUG",
            format="<green>{time:HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}:{function}:{line}</cyan> - <level>{message}</level>"
        )
    
    return logger

# Call before application starts
configure_logging()
```

## Known Issues and Mitigations

- **Environment variable timing**: Load environment configuration before importing loguru to ensure proper settings
- **Stream redirection issues**: Tools like pytest may redirect stdout/stderr, causing handler exceptions - use dynamic stream references
- **Multiprocessing deadlocks**: Initialize loggers before starting threads or processes, use `enqueue=True` for safety
- **Log file locking**: On Windows, log files may be locked by antivirus - use `enqueue=True` to mitigate
- **High memory usage**: With `enqueue=True`, monitor queue size in high-volume scenarios

```python
# Mitigation for stream redirection issues
import sys
from loguru import logger

# Dynamic stream reference to handle redirection
logger.add(lambda msg: sys.stderr.write(msg), colorize=True)

# Mitigation for environment variable timing
import os
os.environ["LOGURU_LEVEL"] = "DEBUG"  # Set before import
from loguru import logger

# Mitigation for multiprocessing issues
if __name__ == "__main__":
    # Configure logging BEFORE starting processes
    logger.remove()
    logger.add("safe.log", enqueue=True)
    
    # Now safe to start multiprocessing
    import multiprocessing
    # ... rest of code
```

## Best Practices Summary

- **Always call logger.remove() first** when configuring custom handlers
- **Use enqueue=True for concurrent applications** (multiprocessing, threading, async)
- **Set diagnose=False in production** to avoid exposing sensitive variable data
- **Implement proper log rotation and retention** to manage disk space
- **Use structured logging with context** for better observability
- **Load configuration before importing loguru** to ensure proper initialization
- **Monitor log volume and performance impact** in production environments
- **Sanitize sensitive data** before logging to prevent data exposure
- **Use appropriate log levels** for different environments (DEBUG in dev, INFO+ in prod)
- **Test logging configuration** in all deployment environments before production release