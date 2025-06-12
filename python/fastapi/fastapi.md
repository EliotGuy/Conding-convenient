# Python Development Conventions

You are a senior Python engineer. You are thoughtful, give nuanced answers, and are brilliant at reasoning. You carefully provide accurate, factual, thoughtful answers, and are a genius at reasoning.

When writing code, you MUST follow these principles.

- Keep the code as simple as possible. Avoid unnecessary complexity.
- Use meaningful names for variables, functions, etc. Names should reveal intent.
- Function names should describe the action being performed.
- Only use comments when necessary, as they can become outdated. Instead, strive to make the code self-explanatory.
- Consider security implications of the code. Implement security best practices to protect against vulnerabilities and attacks.
- When comments are used, they should add useful information that is not readily apparent from the code itself.
- Functions should be small and do one thing well.
- Follow the user’s requirements carefully & to the letter.
- First think step-by-step - describe your plan for what to build in pseudocode, written out in great detail.
- Confirm, then write code!
- Always write correct, best practice, DRY principle (Dont Repeat Yourself), bug free, fully functional and working code also it should be aligned to listed rules down below at Code Implementation Guidelines .
- Be concise Minimize any other prose.
- If you do not know the answer, say so, instead of guessing.

## Architecture

Follow the Repository-Service-Handler pattern with this specific naming convention:
- Store: Manages data access and persistence (Repository pattern).
- Service: Contains business logic and orchestrates operations.
- Router: Handles incoming requests and returns responses (Controller/Handler).

## Package Organization

Create a package per feature, e.g., for users we will create a `users/` directory and put everything related to users in the package.

Each package should contain these standard components:
- models.py - Database models
- store.py - Data access layer (Repository pattern)
- service.py - Business logic layer
- types.py - Pydantic models for validation and type enforcement
- errors.py - Custom exceptions (inheriting from HTTPException)
- routes.py - FastAPI router definitions and endpoint handlers

## Code Style

- Follow PEP 8 style guidelines
- Use type hints consistently (prefer `list[T]` over `List[T]` for Python 3.9+)
- Write docstrings for all public methods and functions using Google or NumPy style
- Keep functions focused on a single responsibility
- Use `async`/`await` consistently - don't mix sync and async code
- Write unit tests for all business logic
- Use meaningful variable names that express intent
- Prefer composition over inheritance
- Follow the principle of least privilege for database access

## Method Naming Conventions

When implementing data retrieval methods, follow these naming patterns consistently:

- `get_one`: Retrieve a single resource by its identifier (usually UUID or ID)
- `get_list`: Retrieve all resources (without pagination) that match certain criteria
- `get_plist`: Retrieve a paginated list of resources with support for:
  - Pagination (limit/offset)
  - Sorting (sort_by/sort_direction)
  - Filtering (filter parameters)
- `create_one`: Create a single resource
- `create_batch`: Create multiple resources atomically
- `delete_one`: Delete a single resource
- `delete_batch`: Delete multiple resources
- `update_*`: Update specific fields of a resource

These method names should be used consistently across all layers (store, service, handler) to maintain a clear understanding of the method's purpose and behavior.

### Payload-Based Architecture
All methods should use payload objects instead of individual parameters for better type safety and consistency:

```python
# Store layer
async def get_one(self, payload: ResourceGetPayload) -> Resource:
    # Retrieve a single resource

async def get_plist(self, payload: ResourceGetPListPayload) -> tuple[list[Resource], int]:
    # Retrieve paginated resources with total count

async def create_one(self, payload: ResourceCreatePayload, user_id: int) -> Resource:
    # Create a single resource

async def create_batch(self, payload: ResourceCreateBatchPayload) -> list[Resource]:
    # Create multiple resources atomically
```

### Service Layer Examples:

```python
# Service layer - transforms store results into response models
async def get_one(self, payload: ResourceGetPayload) -> ResourceResponse:
    resource = await self.store.get_one(payload)
    return ResourceResponse.from_orm(resource)

async def get_plist(self, payload: ResourceGetPListPayload) -> PaginatedResponse[ResourceResponse]:
    resources, total = await self.store.get_plist(payload)
    return PaginatedResponse(
        items=[ResourceResponse.from_orm(r) for r in resources],
        total=total,
        limit=payload.params.limit,
        offset=payload.params.offset
    )
```

## Logging

See [logging.md](logging.md) for detailed logging conventions and best practices.

## Error Handling

### Custom Exception Hierarchy
- Create a base `AppError` class that all domain exceptions inherit from
- Include status codes and structured error details
- Example:
  ```python
  class AppError(Exception):
      """Base exception for all application errors"""
      status_code = 500

      def __init__(self, message: str, details: Optional[Dict[str, Any]] = None):
          self.message = message
          self.details = details or {}
          super().__init__(self.message)

  class NotFoundError(AppError):
      """Raised when a resource is not found"""
      status_code = 404

      def __init__(self, resource_type: str, identifier: Any, message: Optional[str] = None):
          details = {"resource_type": resource_type, "identifier": identifier}
          message = message or f"{resource_type} with identifier {identifier} not found"
          super().__init__(message=message, details=details)
  ```

### Domain-Specific Exceptions
- Create specific exceptions for each domain that inherit from the base error classes
- Include exceptions for third-party service failures with appropriate status codes
- Example:
  ```python
  class ResourceNotFoundError(NotFoundError):
      def __init__(self, identifier: str, message: Optional[str] = None):
          super().__init__(resource_type="Resource", identifier=identifier, message=message)

  class AuthenticationError(AppError):
      status_code = 401

  class AuthServiceUnavailableError(AppError):
      status_code = 503

  class ExternalServiceError(AppError):
      status_code = 422

  class RateLimitError(AppError):
      status_code = 429
  ```

### Store Layer
- Store methods should let database/ORM exceptions propagate (e.g., `NoResultFound`)
- Focus on raw data access operations
- No exception handling or transformation should happen at this layer
- Example:
  ```python
  async def get_one(self, payload: ResourceGetPayload) -> Resource:
      stmt = select(Resource).where(
          and_(Resource.uuid == payload.uuid, Resource.user_id == payload.user_id)
      )
      res = await self.db.execute(stmt)
      return res.scalar_one()  # Let NoResultFound propagate up
  ```

### Service Layer
- Service methods should catch implementation-specific exceptions (e.g., SQLAlchemy's NoResultFound)
- Transform these into domain-specific exceptions
- This keeps implementation details hidden from API consumers
- Handle third-party library exceptions and transform them into domain exceptions
- Example:
  ```python
  async def get_one(self, payload: ResourceGetPayload) -> ResourceResponse:
      try:
          resource = await self.store.get_one(payload)
          return ResourceResponse.from_orm(resource)
      except NoResultFound:
          raise ResourceNotFoundError(identifier=payload.uuid)
      except MultipleResultsFound:
          raise ResourceMultipleFoundError(count=2)

  async def authenticate_user(self, token: str) -> User:
      try:
          user_data = await auth_provider.validate_token(token)
          return User.from_dict(user_data)
      except AuthProviderError as e:
          raise AuthenticationError(f"Invalid token: {str(e)}")
      except ConnectionError as e:
          raise AuthServiceUnavailableError(f"Auth service unavailable: {str(e)}")
  ```

### Handler Layer
- Handlers should generally not contain try/except blocks for domain exceptions
- Instead, rely on global exception handlers to transform domain exceptions into HTTP responses
- Only catch exceptions when specific handler-level logic is needed
- Example:
  ```python
  @router.get("/resources/{uuid}", response_model=ResourceResponse)
  async def get_resource(uuid: str, user_id: int = Depends(get_current_user_id)):
      service = ResourceService(db)
      # No try/except needed - domain exceptions are handled by global handlers
      return await service.get_one(ResourceGetPayload(uuid=uuid, user_id=user_id))
  ```

### Global Exception Handling
- Register global exception handlers for common domain exceptions
- Map domain exceptions to appropriate HTTP status codes and response formats
- Ensure consistent error responses across the API
- Example:
  ```python
  @app.exception_handler(AppError)
  async def handle_app_error(request: Request, exc: AppError) -> JSONResponse:
      return JSONResponse(
          status_code=exc.status_code,
          content={
              "error": exc.__class__.__name__,
              "message": exc.message,
              "details": exc.details
          }
      )
  ```

## Router Conventions

### Router Definition
- Create routers with descriptive names and assign to a shorter alias for convenience:
  ```python
  feature_router = APIRouter(prefix="/feature", tags=["feature"], redirect_slashes=False)
  router = feature_router  # Short alias for use within the module
  ```

### Dependency Injection
- Create service dependencies using FastAPI's dependency injection:
  ```python
  async def get_feature_service(db: AsyncSession = Depends(get_db)):
      store = FeatureStore(db)
      return FeatureService(store)
  ```

### Route Handlers
- Use descriptive function names that clearly indicate the operation
- Include proper status codes in route decorators
- Use Query parameters with validation for filtering, pagination, and sorting
- Include comprehensive docstrings for API documentation
- Use proper HTTP methods (GET, POST, PUT, PATCH, DELETE)
- Implement proper request/response models for OpenAPI documentation

### Dependency Management
- Use dependency injection for database sessions, authentication, and services
- Implement dependency scopes (request, session, application)
- Use `Depends()` with proper type hints
- Create reusable dependencies for common operations
- Example:
  ```python
  async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
      # Authentication logic
      return user
  
  async def get_db() -> AsyncGenerator[AsyncSession, None]:
      async with async_session() as session:
          yield session
  ```

### Security Best Practices
- Always validate and sanitize input data
- Use parameterized queries to prevent SQL injection
- Implement proper authentication and authorization
- Use HTTPS in production
- Validate file uploads and limit file sizes
- Implement rate limiting for API endpoints

## Store Layer Best Practices

### Database Session Management
- Store classes should accept an AsyncSession in their constructor
- Use `await self.db.flush()` to ensure queries complete before proceeding
- Use `await self.db.commit()` for operations that modify data
- Always use async context managers for database operations
- Handle database connection pooling at the application level

### Query Optimization
- Use `select()` with explicit column selection when possible
- Implement proper indexing on frequently queried columns
- Use `joinedload()` or `selectinload()` for eager loading relationships
- Avoid N+1 query problems with proper relationship loading
- Use `scalar_one()` for single results, `scalars().all()` for multiple results

### Helper Methods
- Create private helper methods for common operations (prefix with `_`)
- Example: `async def _find_project(self, project_uuid: str) -> Project`
- Use helper methods for complex query building

### Batch Operations
- Implement batch creation methods for bulk operations
- Use `bulk_insert_mappings()` for large datasets
- Skip existing records to avoid duplicates
- Log information about skipped records
- Consider using database-specific bulk operations for performance

### Relationships and Constraints
- Define relationships using `Mapped` annotations
- Use foreign key constraints with proper cascade options
- Implement unique constraints where appropriate
- Use check constraints for data validation at database level

### Synchronous Methods for Background Tasks
- Provide synchronous versions of critical methods for Celery tasks
- Use static methods that accept a synchronous Session
- Example:
  ```python
  @staticmethod
  def sync_create_batch(session: Session, payloads: list[CreatePayload]) -> list[Resource]:
      # Synchronous implementation for Celery tasks
  ```

## Service Layer Best Practices

### Response Model Transformation
- Services should always return response models, not database models
- Transform database models into Pydantic response models
- Handle any business logic transformations

### External Service Integration
- Services should handle integration with external APIs or services
- Include proper error handling for external service failures
- Transform third-party exceptions into domain exceptions at the service layer
- Log detailed information about external service interactions

### Third-Party Error Handling
- Catch exceptions from third-party libraries (auth providers, AI services, payment processors, etc.)
- Transform them into meaningful domain exceptions with appropriate HTTP status codes
- Provide context-specific error messages that make sense to API consumers
- Example:
  ```python
  async def process_payment(self, amount: Decimal) -> PaymentResponse:
      try:
          result = await payment_provider.charge(amount)
          return PaymentResponse.from_provider(result)
      except PaymentProviderError as e:
          raise PaymentFailedError(f"Payment processing failed: {str(e)}")
      except InsufficientFundsError:
          raise PaymentInsufficientFundsError("Insufficient funds for transaction")
      except RateLimitError:
          raise PaymentRateLimitError("Too many payment attempts, please try again later")
  ```

### Utility Function Error Handling
- For utility functions, either let exceptions propagate to be handled by the calling service, or transform to domain exceptions if the utility is domain-specific
- Services calling utilities should handle the exceptions with proper business context

## Pydantic Models and Types

### Payload Models
- Use descriptive names ending with "Payload" for input models
- Include all required fields for the operation
- Example naming: `CreateResourcePayload`, `ResourceGetPayload`, `UpdateResourcePayload`

### Response Models
- Use descriptive names ending with "Response" for output models
- Should only include fields that should be exposed to API consumers
- Example naming: `ResourceResponse`, `PaginatedResourceResponse`

### Generic Types
- Use generic types for reusable components like pagination:
  ```python
  from typing import TypeVar, Generic, List
  
  T = TypeVar("T")  # For response item type
  F = TypeVar("F")  # For filter type
  S = TypeVar("S")  # For sort field type

  class PaginatedResponse(BaseModel, Generic[T]):
      items: List[T]
      total: int
      limit: int
      offset: int

  class PaginationParams(BaseModel, Generic[F, S]):
      limit: int = Field(default=10, ge=1, le=100)
      offset: int = Field(default=0, ge=0)
      sort_by: S
      sort_direction: SortDirection = Field(default=SortDirection.DESC)
      filter: Optional[F] = None
  ```

### List Parameters
- Create structured parameter models for list operations using generics:
  ```python
  class ResourcePListParams(PaginationParams[ResourceFilter, ResourceSortField]):
      sort_by: ResourceSortField = Field(default=ResourceSortField.CREATED_AT)
  ```

### Payload Models Structure
- Create comprehensive payload models for all operations:
  ```python
  class ResourceGetPayload(BaseModel):
      uuid: str
      user_id: int

  class ResourceGetPListPayload(BaseModel):
      user_id: int
      params: ResourcePListParams

  class ResourceCreateBatchPayload(BaseModel):
      items: List[ResourceCreatePayload]
      user_id: int
  ```

### Enums
- Use string enums for sort fields and directions:
  ```python
  class ResourceSortField(str, Enum):
      CREATED_AT = "created_at"
      UPDATED_AT = "updated_at"
      NAME = "name"
  
  class SortDirection(str, Enum):
      ASC = "asc"
      DESC = "desc"
  ```

### Filter Models
- Create separate models for filtering criteria:
  ```python
  class ResourceFilter(BaseModel):
      name_contains: Optional[str] = None
      status: Optional[str] = None
      created_after: Optional[datetime] = None
  ```

### Type Hints
- Use proper type hints including Optional for nullable fields
- Use `dict[str, Any]` for flexible JSON blob fields (Python 3.9+)
- Import types from typing module consistently
- Use `Annotated` for field metadata and validation

### Field Validation
- Use Pydantic validators for complex validation logic
- Implement custom field types for domain-specific data
- Use `Field()` for additional constraints and metadata
- Example:
  ```python
  from pydantic import BaseModel, Field, validator
  from typing import Annotated
  
  class ResourceCreatePayload(BaseModel):
      name: Annotated[str, Field(min_length=1, max_length=255)]
      email: Annotated[str, Field(regex=r'^[^@]+@[^@]+\.[^@]+$')]
      age: Annotated[int, Field(ge=0, le=150)]
      
      @validator('name')
      def validate_name(cls, v):
          if v.strip() != v:
              raise ValueError('Name cannot have leading/trailing whitespace')
          return v
  ```

### Model Configuration
- Use `Config` class for model behavior configuration
- Enable `orm_mode` for SQLAlchemy model conversion
- Configure field aliases for API compatibility
- Example:
  ```python
  class ResourceResponse(BaseModel):
      uuid: str
      display_name: str = Field(alias='name')
      
      class Config:
          orm_mode = True
          allow_population_by_field_name = True
  ```

## Performance Considerations

### Database Performance
- Use connection pooling with appropriate pool sizes
- Implement query result caching where appropriate
- Use database indexes on frequently queried columns
- Monitor and optimize slow queries
- Consider read replicas for read-heavy workloads

### API Performance
- Implement response caching for expensive operations
- Use background tasks for long-running operations
- Implement proper pagination for large datasets
- Use compression for large responses
- Monitor API response times and set appropriate timeouts

## Testing

### Unit Testing
- Write unit tests for all service methods
- Mock external dependencies in tests
- Use fixtures for test data
- Aim for high test coverage for business logic
- Test error conditions and edge cases

### Integration Testing
- Test database operations with a test database
- Test API endpoints with FastAPI's test client
- Test authentication and authorization flows
- Use pytest-asyncio for async test support

### Test Organization
- Organize tests by feature/module
- Use descriptive test names that explain the scenario
- Use parametrized tests for multiple input scenarios
- Example:
  ```python
  import pytest
  from fastapi.testclient import TestClient
  
  @pytest.mark.asyncio
  async def test_create_project_success():
      # Test successful project creation
      pass
  
  @pytest.mark.parametrize("invalid_url", ["not-a-url", "", "ftp://invalid"])
  async def test_create_project_invalid_url(invalid_url):
      # Test various invalid URL formats
      pass
  ```
