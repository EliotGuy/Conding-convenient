# FastAPI Examples

Use these examples as reference for implementing the conventions described in fastapi.md.

## Key Patterns Demonstrated

- **Payload-based architecture**: All methods use payload objects for type safety
- **Generic types**: Reusable pagination and response structures
- **Error hierarchy**: Comprehensive error handling with base AppError class
- **Global exception handling**: Centralized error handling in handlers
- **Method naming**: Consistent `get_one`, `get_list`, `get_plist` patterns
- **Store/Service/Handler separation**: Clear separation of concerns

## Examples

```py
# auth/errors.py
from typing import Optional, Dict, Any
from proj.errors import AppError

class AuthenticationError(AppError):
    """Raised when authentication fails"""
    status_code = 401

class AuthServiceUnavailableError(AppError):
    """Raised when auth service is unavailable"""
    status_code = 503

class TokenExpiredError(AppError):
    """Raised when auth token has expired"""
    status_code = 401

# auth/service.py
import logging
from typing import Optional
from proj.auth.errors import AuthenticationError, AuthServiceUnavailableError, TokenExpiredError
from proj.users.models import User

logger = logging.getLogger(__name__)

class AuthService:
    def __init__(self, auth_provider):
        self.auth_provider = auth_provider

    async def authenticate_user(self, token: str) -> User:
        """
        Authenticate a user using a token
        
        :param token: Authentication token
        :return: Authenticated user
        :raises AuthenticationError: If token is invalid
        :raises AuthServiceUnavailableError: If auth service is down
        :raises TokenExpiredError: If token has expired
        """
        try:
            user_data = await self.auth_provider.validate_token(token)
            logger.info({"user_id": user_data.get("id")}, "User authenticated successfully")
            return User.from_dict(user_data)
        except AuthProviderInvalidTokenError as e:
            logger.warning({"token_prefix": token[:8]}, "Invalid authentication token")
            raise AuthenticationError(f"Invalid token: {str(e)}")
        except AuthProviderExpiredTokenError as e:
            logger.warning({"token_prefix": token[:8]}, "Expired authentication token")
            raise TokenExpiredError("Authentication token has expired")
        except ConnectionError as e:
            logger.error({"error": str(e)}, "Auth service connection failed")
            raise AuthServiceUnavailableError(f"Auth service unavailable: {str(e)}")
        except Exception as e:
            logger.exception("Unexpected error during authentication")
            raise AuthServiceUnavailableError(f"Authentication service error: {str(e)}")

# images/errors.py
from proj.errors import AppError

class AltTextGenerationError(AppError):
    """Raised when alt text generation fails"""
    status_code = 422

class AltTextRateLimitError(AppError):
    """Raised when AI service rate limit is exceeded"""
    status_code = 429

class ImageProcessingError(AppError):
    """Raised when image processing fails"""
    status_code = 422

class InvalidImageFormatError(AppError):
    """Raised when image format is not supported"""
    status_code = 400

# images/service.py (partial - showing third-party error handling)
import logging
from PIL import UnidentifiedImageError
from proj.images.errors import AltTextGenerationError, AltTextRateLimitError, ImageProcessingError, InvalidImageFormatError
from proj.images.store import ImageStore
from proj.images.types import ImageGetPayload, ImageResponse, ProcessedImageResponse

logger = logging.getLogger(__name__)

class ImageService:
    def __init__(self, store: ImageStore, ai_service):
        self.store = store
        self.ai_service = ai_service

    async def generate_alt_text(self, image_uuid: str) -> ImageResponse:
        """
        Generate alt text for an image using AI service
        
        :param image_uuid: UUID of the image
        :return: Updated image with alt text
        :raises AltTextGenerationError: If AI service fails
        :raises AltTextRateLimitError: If rate limit exceeded
        """
        try:
            image = await self.store.get_one(ImageGetPayload(uuid=image_uuid))
            
            # Call external AI service
            response = await self.ai_service.generate_alt_text(image.url)
            alt_text = response.alt_text
            
            # Update image with generated alt text
            updated_image = await self.store.update_alt_text(image_uuid, alt_text)
            
            logger.info({"image_uuid": image_uuid}, "Alt text generated successfully")
            return ImageResponse.from_orm(updated_image)
            
        except AIServiceError as e:
            logger.error({"image_uuid": image_uuid, "error": str(e)}, "AI service error")
            raise AltTextGenerationError(f"Failed to generate alt text: {str(e)}")
        except RateLimitError as e:
            logger.warning({"image_uuid": image_uuid}, "AI service rate limit exceeded")
            raise AltTextRateLimitError("Rate limit exceeded for AI service")
        except ConnectionError as e:
            logger.error({"image_uuid": image_uuid, "error": str(e)}, "AI service connection failed")
            raise AltTextGenerationError(f"AI service unavailable: {str(e)}")

    async def process_image(self, image_data: bytes) -> ProcessedImageResponse:
        """
        Process and resize an image
        
        :param image_data: Raw image bytes
        :return: Processed image response
        :raises InvalidImageFormatError: If image format not supported
        :raises ImageProcessingError: If processing fails
        """
        try:
            # Use utility function that may raise PIL exceptions
            processed_data = resize_image(image_data)
            
            logger.info("Image processed successfully")
            return ProcessedImageResponse(data=processed_data)
            
        except UnidentifiedImageError:
            logger.warning("Unsupported image format provided")
            raise InvalidImageFormatError("Unsupported image format")
        except MemoryError:
            logger.error("Image too large to process")
            raise ImageProcessingError("Image too large to process")
        except Exception as e:
            logger.exception("Unexpected error during image processing")
            raise ImageProcessingError(f"Image processing failed: {str(e)}")

# utils.py (global utils)
def apply_filters(query, model, filter_obj):
    """
    Apply filters to a SQLAlchemy query based on a Pydantic filter object.

    :param query: SQLAlchemy query object
    :param model: SQLAlchemy model class
    :param filter_obj: Pydantic model with filter fields
    :return: SQLAlchemy query with filters applied
    """
    if filter_obj is None:
        return query

    for field_name, value in filter_obj.dict(exclude_none=True).items():
        if field_name.endswith('_contains'):
            base_field = field_name.replace('_contains', '')
            query = query.where(getattr(model, base_field).ilike(f"%{value}%"))
        else:
            query = query.where(getattr(model, field_name) == value)

    return query

# types.py (global types)
from enum import Enum
from typing import TypeVar, Generic, List, Optional
from pydantic import BaseModel, Field

T = TypeVar("T")  # For response item type
F = TypeVar("F")  # For filter type
S = TypeVar("S")  # For sort field type

class SortDirection(str, Enum):
    """
    Enum for sort direction options

    Used to specify ascending or descending order in queries
    """
    ASC = "asc"
    DESC = "desc"

class SortField(str, Enum):
    """
    Base enum for sortable fields

    Contains common fields that can be used for sorting across different models
    """
    CREATED_AT = "created_at"
    UPDATED_AT = "updated_at"

class PaginationParams(BaseModel, Generic[F, S]):
    """
    Generic pagination parameters

    :param F: Filter type
    :param S: Sort field type
    """
    limit: int = Field(default=10, ge=1, le=100)
    offset: int = Field(default=0, ge=0)
    sort_by: S  # No default here
    sort_direction: SortDirection = Field(default=SortDirection.DESC)
    filter: Optional[F] = None

class PaginatedResponse(BaseModel, Generic[T]):
    """
    Generic paginated response

    :param T: Response item type
    """
    items: List[T]
    total: int
    limit: int
    offset: int


# proj/errors.py
from typing import Optional, Any, Dict, Type


class AppError(Exception):
    """
    Base exception for all application errors

    :param message: Error message
    :param details: Additional error details
    """
    status_code = 500

    def __init__(self, message: str, details: Optional[Dict[str, Any]] = None):
        self.message = message
        self.details = details or {}
        super().__init__(self.message)


class NotFoundError(AppError):
    """
    Raised when a resource is not found

    :param resource_type: Type of resource that was not found
    :param identifier: Identifier used to look up the resource
    :param message: Optional custom error message
    """
    status_code = 404

    def __init__(
        self,
        resource_type: str,
        identifier: Any,
        message: Optional[str] = None
    ):
        details = {"resource_type": resource_type, "identifier": identifier}
        message = message or f"{resource_type} with identifier {identifier} not found"
        super().__init__(message=message, details=details)


class MultipleFoundError(AppError):
    """
    Raised when multiple resources are found but only one was expected

    :param resource_type: Type of resource
    :param count: Number of resources found
    :param message: Optional custom error message
    """
    status_code = 409  # Conflict

    def __init__(
        self,
        resource_type: str,
        count: int,
        message: Optional[str] = None
    ):
        details = {"resource_type": resource_type, "count": count}
        message = message or f"Expected one {resource_type}, but found {count}"
        super().__init__(message=message, details=details)


# projects/errors.py
from typing import Optional, Dict, Any
from proj.errors import NotFoundError, MultipleFoundError, AppError


class ProjectError(Exception):
    """Base exception for all project-related errors"""
    pass


class ProjectNotFoundError(NotFoundError):
    """
    Raised when a project is not found

    :param identifier: Project identifier (usually UUID)
    :param message: Optional custom error message
    """
    def __init__(self, identifier: str, message: Optional[str] = None):
        super().__init__(
            resource_type="Project",
            identifier=identifier,
            message=message
        )


class ProjectMultipleFoundError(MultipleFoundError):
    """
    Raised when multiple projects are found but only one was expected

    :param count: Number of projects found
    :param message: Optional custom error message
    """
    def __init__(self, count: int, message: Optional[str] = None):
        super().__init__(
            resource_type="Project",
            count=count,
            message=message
        )


class ProjectCreationError(AppError):
    """
    Raised when a project cannot be created

    :param reason: Reason for the failure
    :param details: Additional error details
    """
    status_code = 400

    def __init__(self, reason: str, details: Optional[Dict[str, Any]] = None):
        message = f"Failed to create project: {reason}"
        super().__init__(message=message, details=details or {})

# projects/models.py
from datetime import datetime
from enum import Enum
from typing import Optional

from sqlalchemy import BigInteger, String, ForeignKey, func, text, UniqueConstraint, CheckConstraint
from sqlalchemy.orm import Mapped, mapped_column

from proj.database import Base

class ProjectType(str, Enum):
    WORDPRESS = "WORDPRESS"
    SHOPIFY = "SHOPIFY"

class Project(Base):
    __tablename__ = "projects"

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True, autoincrement=True)

    uuid: Mapped[str] = mapped_column(
        String, server_default=text("nanoid(8)"), nullable=False
    )

    created_at: Mapped[datetime] = mapped_column(
        nullable=False, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        nullable=False, server_default=func.now()
    )

    url: Mapped[str] = mapped_column(String, nullable=False)
    type: Mapped[ProjectType] = mapped_column(String, nullable=False)

    user_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("users.id"), nullable=False
    )

    __table_args__ = (
        UniqueConstraint("uuid", name="projects_uuid_unique"),
        UniqueConstraint("url", "user_id", name="projects_url_user_id_unique"),
        CheckConstraint("type IN ('WORDPRESS', 'SHOPIFY')", name="projects_type_check"),
        CheckConstraint("char_length(type) <= 255", name="projects_type_length_check"),
    )

# projects/types.py
from enum import Enum
from typing import List, Optional, Any, Dict
from pydantic import BaseModel, HttpUrl, Field, validator
from datetime import datetime

from proj.types import SortField, PaginationParams, PaginatedResponse
from proj.projects.models import ProjectType

class ProjectResponse(BaseModel):
    """
    Response model for Project data

    Used for serializing Project objects to API responses
    """
    uuid: str
    url: str
    type: ProjectType
    created_at: datetime
    updated_at: datetime

    class Config:
        orm_mode = True

# DELETE /projects/:uuid
class ProjectDeletePayload(BaseModel):
    """
    Payload for deleting a single project

    Used in delete_one operations
    """
    uuid: str
    user_id: int

# DELETE /projects/batch
class ProjectDeleteBatchPayload(BaseModel):
    """
    Payload for deleting multiple projects

    Used in delete_batch operations
    """
    uuids: List[str]
    user_id: int

# POST /projects/batch
class ProjectCreateBatchPayload(BaseModel):
    """
    Payload for creating multiple projects

    Used in create_batch operations
    """
    items: List[ProjectCreatePayload]
    user_id: int

# GET /projects/:uuid

class ProjectGetPayload(BaseModel):
    """
    Payload for retrieving a single project

    Used in get_one operations
    """
    uuid: str
    user_id: int

# GET /projects/

class ProjectGetListPayload(BaseModel):
    """
    Payload for retrieving all projects for a user

    Used in get_list operations
    """
    user_id: int

# GET /projects/?offset=1&limit=10

# Filter
class ProjectFilter(BaseModel):
    """
    Filter options for Project queries

    Used in paginated list operations
    """
    type: Optional[ProjectType] = None
    url_contains: Optional[str] = None

# Pagination
class ProjectSortField(SortField):
    """
    Sort field options for Project queries

    Extends the base SortField with Project-specific fields
    """
    URL = "url"
    TYPE = "type"

class ProjectPListParams(PaginationParams[ProjectFilter, ProjectSortField]):
    """
    Pagination parameters for Project list queries

    Used as query parameters in the API
    """
    sort_by: ProjectSortField = Field(default=ProjectSortField.CREATED_AT)

class ProjectGetPListPayload(BaseModel):
    """
    Payload for retrieving a paginated list of projects

    Used in get_plist operations
    """
    user_id: int
    params: ProjectPListParams

# POST /projects/
class ProjectCreatePayload(BaseModel):
    """
    Payload for creating a new project

    Used in create operations
    """
    url: HttpUrl = Field(..., description="Project URL")
    type: ProjectType = Field(..., description="Project type")
    
    @validator('url')
    def validate_url(cls, v):
        """Ensure URL is accessible and valid"""
        url_str = str(v)
        if not url_str.startswith(('http://', 'https://')):
            raise ValueError('URL must start with http:// or https://')
        return v

# projects/store.py
from sqlalchemy import select, and_, func, delete
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.exc import NoResultFound

from proj.projects.types import ProjectGetPayload, ProjectGetListPayload, ProjectGetPListPayload, ProjectCreatePayload, ProjectDeletePayload, ProjectDeleteBatchPayload, ProjectCreateBatchPayload
from proj.projects.models import Project
from proj.types import SortDirection
from proj.utils import apply_filters

class ProjectStore:
    """
    A store for Project objects.

    Used to interact with Project data in the database.

    :param db: AsyncSession instance for database operations.

    Usage::

        >>> from proj.db import get_session
        >>> async with get_session() as session:
        >>>     store = ProjectStore(session)
        >>>     project = await store.get_one(ProjectGetPayload(uuid="123", user_id=456))
    """
    def __init__(self, db: AsyncSession):
        self.db = db

    async def get_one(self, payload: ProjectGetPayload) -> Project:
        """
        Get a single Project by UUID

        :param payload: Contains UUID and user_id
        :return: Project object
        :raises NoResultFound: If no project matches the UUID and user_id
        """
        stmt = select(Project).where(
            and_(
                Project.uuid == payload.uuid,
                Project.user_id == payload.user_id,
            )
        )
        res = await self.db.execute(stmt)
        return res.scalar_one()  # Let NoResultFound propagate up

    async def get_list(self, payload: ProjectGetListPayload) -> list[Project]:
        """
        Get all Projects for a user

        :param payload: Contains user_id
        :return: List of Project objects
        """
        stmt = select(Project).where(
            Project.user_id == payload.user_id
        )
        res = await self.db.execute(stmt)
        return res.scalars().all()

    async def get_plist(self, payload: ProjectGetPListPayload) -> tuple[list[Project], int]:
        """
        Get a paginated list of Projects

        :param payload: Contains user_id and pagination parameters
        :return: Tuple containing list of projects and total count
        """
        base_query = select(Project).where(
            Project.user_id == payload.user_id
        )

        # apply filters if provided
        params = payload.params
        if params.filter:
            base_query = apply_filters(base_query, Project, params.filter)

        # count total before pagination
        count_query = select(func.count()).select_from(base_query.subquery())
        total = await self.db.scalar(count_query)

        # apply sorting
        sort_column = getattr(Project, params.sort_by.value)
        if params.sort_direction == SortDirection.DESC:
            sort_column = sort_column.desc()
        base_query = base_query.order_by(sort_column)

        # apply pagination
        base_query = base_query.offset(params.offset).limit(params.limit)

        # execute query
        result = await self.db.execute(base_query)
        projects = result.scalars().all()

        return projects, total

    async def create_one(self, payload: ProjectCreatePayload, user_id: int) -> Project:
        """
        Create a single Project

        :param payload: Project creation data
        :param user_id: User ID to associate with the project
        :return: Created Project object
        """
        project = Project(
            url=str(payload.url),
            type=payload.type,
            user_id=user_id
        )

        self.db.add(project)
        await self.db.flush()  # Flush to get the ID and other generated values

        return project

    async def create_batch(self, payload: ProjectCreateBatchPayload) -> list[Project]:
        """
        Create multiple Projects in a single transaction

        :param payload: Contains list of projects to create and user_id
        :return: List of created Project objects
        """
        projects = [
            Project(
                url=str(item.url),
                type=item.type,
                user_id=payload.user_id
            )
            for item in payload.items
        ]

        self.db.add_all(projects)
        await self.db.flush()

        return projects

    async def delete_one(self, payload: ProjectDeletePayload) -> bool:
        """
        Delete a single Project by UUID

        :param payload: Contains UUID and user_id
        :return: True if project was deleted, False if project was not found
        """
        stmt = delete(Project).where(
            and_(
                Project.uuid == payload.uuid,
                Project.user_id == payload.user_id
            )
        ).returning(Project.id)

        result = await self.db.execute(stmt)
        deleted = result.scalar_one_or_none()

        return deleted is not None

    async def delete_batch(self, payload: ProjectDeleteBatchPayload) -> int:
        """
        Delete multiple Projects by UUID

        :param payload: Contains list of UUIDs and user_id
        :return: Number of projects deleted
        """
        if not payload.uuids:
            return 0

        stmt = delete(Project).where(
            and_(
                Project.uuid.in_(payload.uuids),
                Project.user_id == payload.user_id
            )
        ).returning(Project.id)

        result = await self.db.execute(stmt)
        deleted_ids = result.scalars().all()

        return len(deleted_ids)

# projects/service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.exc import NoResultFound, MultipleResultsFound, IntegrityError
import logging

from proj.projects.types import (
    ProjectGetPayload, ProjectGetListPayload, ProjectGetPListPayload, 
    ProjectResponse, PaginatedResponse, ProjectCreatePayload, 
    ProjectDeletePayload, ProjectDeleteBatchPayload, ProjectCreateBatchPayload
)
from proj.projects.models import Project
from proj.projects.store import ProjectStore
from proj.projects.errors import ProjectNotFoundError, ProjectMultipleFoundError, ProjectCreationError

logger = logging.getLogger(__name__)

class ProjectService:
    def __init__(self, db: AsyncSession):
        self.store = ProjectStore(db)

    async def get_one(self, payload: ProjectGetPayload) -> ProjectResponse:
        """
        Retrieve a single project by UUID

        :param payload: Contains UUID and user_id
        :return: Project response object
        :raises ProjectNotFoundError: If project with given UUID doesn't exist
        :raises ProjectMultipleFoundError: If multiple projects match the criteria
        """
        try:
            project = await self.store.get_one(payload)
            return ProjectResponse.from_orm(project)
        except NoResultFound:
            raise ProjectNotFoundError(identifier=payload.uuid)
        except MultipleResultsFound:
            # This should be rare with proper constraints, but handle it anyway
            raise ProjectMultipleFoundError(count=2)  # We don't know exact count

    async def get_list(self, payload: ProjectGetListPayload) -> list[ProjectResponse]:
        """
        Retrieve all projects for a user

        :param payload: Contains user_id
        :return: List of project response objects
        """
        projects = await self.store.get_list(payload)
        return [ProjectResponse.from_orm(project) for project in projects]

    async def get_plist(self, payload: ProjectGetPListPayload) -> PaginatedResponse[ProjectResponse]:
        """
        Retrieve a paginated list of projects

        :param payload: Contains user_id and pagination parameters
        :return: Paginated response with projects
        """
        projects, total = await self.store.get_plist(payload)
        params = payload.params

        return PaginatedResponse(
            items=[ProjectResponse.from_orm(project) for project in projects],
            total=total,
            limit=params.limit,
            offset=params.offset
        )

    async def create_one(self, payload: ProjectCreatePayload, user_id: int) -> ProjectResponse:
        """
        Create a new project

        :param payload: Project creation data
        :param user_id: User ID to associate with the project
        :return: Created project response
        :raises ProjectCreationError: If project creation fails
        """
        try:
            project = await self.store.create_one(payload, user_id)
            return ProjectResponse.from_orm(project)
        except IntegrityError as e:
            # Handle database constraint violations
            if "projects_url_user_id_unique" in str(e):
                raise ProjectCreationError(
                    reason="A project with this URL already exists",
                    details={"url": str(payload.url)}
                )
            raise ProjectCreationError(reason=str(e))
        except Exception as e:
            logger.error("Failed to create project", error=str(e), url=str(payload.url))
            raise ProjectCreationError(reason=str(e))

    async def create_batch(self, payload: ProjectCreateBatchPayload) -> list[ProjectResponse]:
        """
        Create multiple projects in a single transaction

        :param payload: Contains list of projects to create and user_id
        :return: List of created project responses
        :raises ProjectCreationError: If project creation fails
        """
        try:
            projects = await self.store.create_batch(payload)
            return [ProjectResponse.from_orm(project) for project in projects]
        except IntegrityError as e:
            # Handle database constraint violations
            if "projects_url_user_id_unique" in str(e):
                raise ProjectCreationError(
                    reason="One or more projects with the same URL already exist",
                    details={"count": len(payload.items)}
                )
            raise ProjectCreationError(reason=str(e))
        except Exception as e:
            logger.error("Failed to create projects batch", error=str(e), count=len(payload.items))
            raise ProjectCreationError(reason=f"Failed to create projects batch: {str(e)}")

    async def delete_one(self, payload: ProjectDeletePayload) -> bool:
        """
        Delete a project by UUID

        :param payload: Contains UUID and user_id
        :return: True if project was deleted, False if project was not found
        :raises ProjectNotFoundError: If project with given UUID doesn't exist
        """
        deleted = await self.store.delete_one(payload)

        if not deleted:
            raise ProjectNotFoundError(identifier=payload.uuid)

        return True

    async def delete_batch(self, payload: ProjectDeleteBatchPayload) -> int:
        """
        Delete multiple projects by UUID

        :param payload: Contains list of UUIDs and user_id
        :return: Number of projects deleted
        """
        return await self.store.delete_batch(payload)

# proj/api/exception_handlers.py
import logging
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from proj.errors import AppError, NotFoundError, MultipleFoundError
from proj.projects.errors import ProjectCreationError

logger = logging.getLogger(__name__)

def register_exception_handlers(app: FastAPI) -> None:
    """
    Register global exception handlers for the FastAPI application

    :param app: FastAPI application instance
    """

    @app.exception_handler(AppError)
    async def handle_app_error(request: Request, exc: AppError) -> JSONResponse:
        """
        Handle all application errors

        :param request: FastAPI request
        :param exc: Exception instance
        :return: JSON response with error details
        """
        log_context = {
            "path": request.url.path,
            "method": request.method,
            "error_type": exc.__class__.__name__,
            **exc.details
        }

        if exc.status_code >= 500:
            logger.error(log_context, exc.message)
        elif exc.status_code >= 400:
            logger.warning(log_context, exc.message)

        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": exc.__class__.__name__,
                "message": exc.message,
                "details": exc.details
            }
        )

    @app.exception_handler(ValueError)
    async def handle_value_error(request: Request, exc: ValueError) -> JSONResponse:
        """
        Handle ValueError exceptions

        :param request: FastAPI request
        :param exc: Exception instance
        :return: JSON response with error details
        """
        logger.warning({
            "path": request.url.path,
            "method": request.method,
            "error": str(exc)
        }, "Validation error")

        return JSONResponse(
            status_code=400,
            content={
                "error": "ValidationError",
                "message": str(exc),
                "details": {}
            }
        )

# proj/main.py (partial update)
from fastapi import FastAPI
from proj.api.exception_handlers import register_exception_handlers
from proj.projects.handler import router as projects_router

app = FastAPI(title="Project API")

# Register global exception handlers
register_exception_handlers(app)

# Include routers
app.include_router(projects_router, prefix="/api/v1")

# projects/handler.py
from fastapi import Depends, APIRouter, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
import logging
from typing import List

from proj.projects.types import (
    ProjectGetPayload, ProjectGetListPayload, ProjectGetPListPayload, 
    ProjectResponse, PaginatedResponse, ProjectPListParams, ProjectCreatePayload, 
    ProjectDeletePayload, ProjectDeleteBatchPayload, ProjectCreateBatchPayload
)
from proj.projects.service import ProjectService
from proj.database import get_db
from proj.auth import get_current_user_id

router = APIRouter(prefix="/projects", tags=["projects"])
logger = logging.getLogger(__name__)

@router.get("/{uuid}", response_model=ProjectResponse)
async def get_project(
    uuid: str,
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db)
):
    """
    Get a project by UUID

    :param uuid: Project UUID
    :param user_id: Current user ID (from auth)
    :param db: Database session
    :return: Project details
    """
    logger.debug({"uuid": uuid, "user_id": user_id}, "Getting project by UUID")
    service = ProjectService(db)

    # No try/except needed - domain exceptions are handled by global handlers
    result = await service.get_one(ProjectGetPayload(uuid=uuid, user_id=user_id))
    logger.info({"uuid": uuid}, "Project retrieved successfully")
    return result

@router.get("/", response_model=PaginatedResponse[ProjectResponse])
async def get_projects_list(
    params: ProjectPListParams = Depends(),
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db)
):
    """
    Get a paginated list of projects

    :param params: Pagination, sorting and filtering parameters
    :param user_id: Current user ID (from auth)
    :param db: Database session
    :return: Paginated list of projects
    """
    logger.debug(
        {
            "user_id": user_id,
            "limit": params.limit,
            "offset": params.offset,
            "sort_by": params.sort_by.value,
            "sort_direction": params.sort_direction.value,
            "filter": params.filter.dict() if params.filter else None
        },
        "Getting paginated list of projects"
    )

    service = ProjectService(db)
    payload = ProjectGetPListPayload(user_id=user_id, params=params)
    result = await service.get_plist(payload)

    logger.info(
        {
            "user_id": user_id,
            "total": result.total,
            "count": len(result.items)
        },
        "Projects list retrieved successfully"
    )

    return result

@router.post("/", response_model=ProjectResponse, status_code=status.HTTP_201_CREATED)
async def create_project(
    payload: ProjectCreatePayload,
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db)
):
    """
    Create a new project

    :param payload: Project creation data
    :param user_id: Current user ID (from auth)
    :param db: Database session
    :return: Created project details
    """
    logger.debug(
        {
            "user_id": user_id,
            "url": str(payload.url),
            "type": payload.type.value
        },
        "Creating new project"
    )

    service = ProjectService(db)
    result = await service.create_one(payload, user_id)

    logger.info(
        {
            "uuid": result.uuid,
            "user_id": user_id
        },
        "Project created successfully"
    )

    return result

@router.post("/batch", response_model=List[ProjectResponse], status_code=status.HTTP_201_CREATED)
async def create_projects_batch(
    payload: List[ProjectCreatePayload],
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db)
):
    """
    Create multiple projects in a single transaction

    :param payload: List of project creation data
    :param user_id: Current user ID (from auth)
    :param db: Database session
    :return: List of created project details
    """
    logger.debug(
        {
            "user_id": user_id,
            "count": len(payload)
        },
        "Creating batch of projects"
    )

    service = ProjectService(db)
    batch_payload = ProjectCreateBatchPayload(items=payload, user_id=user_id)
    result = await service.create_batch(batch_payload)

    logger.info(
        {
            "user_id": user_id,
            "count": len(result)
        },
        "Projects batch created successfully"
    )

    return result

@router.delete("/{uuid}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_project(
    uuid: str,
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db)
):
    """
    Delete a project by UUID

    :param uuid: Project UUID
    :param user_id: Current user ID (from auth)
    :param db: Database session
    :return: No content
    :raises HTTPException: 404 if project not found
    """
    logger.debug(
        {
            "uuid": uuid,
            "user_id": user_id
        },
        "Deleting project"
    )

    service = ProjectService(db)
    await service.delete_one(ProjectDeletePayload(uuid=uuid, user_id=user_id))

    logger.info(
        {
            "uuid": uuid,
            "user_id": user_id
        },
        "Project deleted successfully"
    )

    return None

@router.delete("/batch", status_code=status.HTTP_200_OK)
async def delete_projects_batch(
    uuids: List[str],
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db)
):
    """
    Delete multiple projects by UUID

    :param uuids: List of project UUIDs
    :param user_id: Current user ID (from auth)
    :param db: Database session
    :return: Number of projects deleted
    """
    logger.debug(
        {
            "uuids": uuids,
            "user_id": user_id,
            "count": len(uuids)
        },
        "Deleting batch of projects"
    )

    service = ProjectService(db)
    count = await service.delete_batch(ProjectDeleteBatchPayload(uuids=uuids, user_id=user_id))

    logger.info(
        {
            "user_id": user_id,
            "count": count
        },
        "Projects batch deleted successfully"
    )

    return {"deleted": count}
```
