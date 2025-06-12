# Python/FastAPI Logging Conventions

## Library

Use `structlog` for structured logging:

```bash
pip install structlog
```

## Logger Setup

Create a logger configuration in a dedicated module:

```python
# app/log.py
import os
import structlog
from structlog.stdlib import LoggerFactory

def configure_logging():
    """Configure structlog for the application"""
    
    # Development configuration
    if os.getenv("DEBUG") == "true":
        structlog.configure(
            processors=[
                structlog.stdlib.filter_by_level,
                structlog.stdlib.add_logger_name,
                structlog.stdlib.add_log_level,
                structlog.stdlib.PositionalArgumentsFormatter(),
                structlog.processors.TimeStamper(fmt="iso"),
                structlog.processors.StackInfoRenderer(),
                structlog.processors.format_exc_info,
                structlog.dev.ConsoleRenderer()
            ],
            context_class=dict,
            logger_factory=LoggerFactory(),
            wrapper_class=structlog.stdlib.BoundLogger,
            cache_logger_on_first_use=True,
        )
    else:
        # Production configuration
        structlog.configure(
            processors=[
                structlog.stdlib.filter_by_level,
                structlog.stdlib.add_logger_name,
                structlog.stdlib.add_log_level,
                structlog.stdlib.PositionalArgumentsFormatter(),
                structlog.processors.TimeStamper(fmt="iso"),
                structlog.processors.StackInfoRenderer(),
                structlog.processors.format_exc_info,
                structlog.processors.JSONRenderer()
            ],
            context_class=dict,
            logger_factory=LoggerFactory(),
            wrapper_class=structlog.stdlib.BoundLogger,
            cache_logger_on_first_use=True,
        )
```

## Logger Usage

Get a logger instance in each module:

```python
import structlog

logger = structlog.get_logger(__name__)
```

## Logging Levels

Follow these guidelines for logging levels:

### DEBUG
For detailed information about function inputs and parameters:

```python
logger.debug("Processing user request", user_id=user_id, params=params)
```

### INFO
For tracking successful operations:

```python
logger.info("User created successfully", user_id=user.id, email=user.email)
```

### WARNING
For non-critical issues that might need attention:

```python
logger.warning("Rate limit approaching", user_id=user_id, current_requests=count)
```

### ERROR
For operation failures with error details:

```python
logger.error("Failed to create user", error=str(err), email=email)
```

## Layer-Specific Guidelines

### Router/Handler Layer
Primary location for logging. Log all significant operations:

```python
@router.post("/users")
async def create_user(user_data: UserCreateRequest):
    logger.debug("Creating user", email=user_data.email)
    
    try:
        user = await user_service.create_user(user_data)
        logger.info("User created successfully", user_id=user.id, email=user.email)
        return user
    except UserAlreadyExistsError as err:
        logger.warning("User creation failed - already exists", email=user_data.email)
        raise HTTPException(status_code=409, detail=str(err))
```

### Service Layer
Log only when detailed debugging is needed or for business logic insights:

```python
async def create_user(self, user_data: UserCreateRequest) -> User:
    logger.debug("Validating user data", email=user_data.email)
    
    # Business logic here
    
    try:
        user = await self.store.create_user(user_data)
        return user
    except IntegrityError:
        logger.debug("User already exists in database", email=user_data.email)
        raise UserAlreadyExistsError(f"User with email {user_data.email} already exists")
```

### Store Layer
Minimal logging, only for exceptional debugging:

```python
async def create_user(self, user_data: UserCreateRequest) -> User:
    # Let database exceptions propagate
    # Only log if absolutely necessary for debugging
    return await self.db.create(User(**user_data.dict()))
```

## Security Considerations

Never log sensitive information:

```python
# ❌ DON'T DO THIS
logger.info("User login", email=email, password=password)

# ✅ DO THIS
logger.info("User login attempt", email=email)
```

Common sensitive fields to avoid:
- Passwords
- API keys/tokens
- Credit card numbers
- Social security numbers
- Personal identification numbers

## Context and Correlation

Use correlation IDs for request tracing:

```python
# Middleware to add correlation ID
import uuid
from fastapi import Request

@app.middleware("http")
async def add_correlation_id(request: Request, call_next):
    correlation_id = str(uuid.uuid4())
    
    # Bind correlation ID to logger context
    structlog.contextvars.bind_contextvars(correlation_id=correlation_id)
    
    response = await call_next(request)
    return response
```

Then in your handlers:

```python
logger.info("Processing request", endpoint="/users", method="POST")
```

## Error Logging

When logging errors, include relevant context:

```python
try:
    result = await some_operation()
except SomeSpecificError as err:
    logger.error(
        "Operation failed",
        operation="user_creation",
        error_type=type(err).__name__,
        error_message=str(err),
        user_id=user_id
    )
    raise
```

## Performance Logging

For performance monitoring:

```python
import time

start_time = time.time()
result = await expensive_operation()
duration = time.time() - start_time

logger.info(
    "Operation completed",
    operation="expensive_operation",
    duration_seconds=duration,
    result_count=len(result)
)
```

## Testing

In tests, you can capture and assert on log messages:

```python
import pytest
from structlog.testing import LogCapture

def test_user_creation_logs(caplog):
    with LogCapture() as cap:
        # Your test code here
        pass
    
    # Assert on log messages
    assert cap.entries[0]["event"] == "User created successfully"
    assert cap.entries[0]["user_id"] == expected_user_id
```
