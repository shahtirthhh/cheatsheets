# FastAPI — Complete Staff-Level Guide
*From zero to production architecture · Everything a 40LPA backend engineer needs*

---

# Part 1: Project Setup & Scaffolding

## Installation

```bash
# Core
pip install fastapi uvicorn[standard]

# Full production stack
pip install fastapi uvicorn[standard] sqlalchemy asyncpg alembic \
    motor beanie pydantic-settings python-jose[cryptography] \
    passlib[bcrypt] python-multipart httpx celery redis \
    apscheduler python-dotenv loguru pytest
```

## Project Structure (Staff-Level Architecture)

```
NestJS pattern:           FastAPI equivalent:
  Controller                Router (routes/endpoints)
  Service                   Service (business logic)
  Repository                Repository (database queries)
  Module                    Router + Depends() wiring
  DTO                       Pydantic Schema
  Guard                     Dependency that raises HTTPException
  Interceptor               Middleware or Dependency
  Pipe                      Pydantic validation (automatic)
  Filter                    Exception Handler
```

```
project/
├── app/
│   ├── __init__.py
│   ├── main.py                    # app factory, startup/shutdown, middleware
│   ├── config.py                  # settings (env vars, secrets)
│   ├── database.py                # DB engines, sessions
│   │
│   ├── auth/                      # auth module
│   │   ├── __init__.py
│   │   ├── router.py              # routes (like NestJS controller)
│   │   ├── service.py             # business logic
│   │   ├── schemas.py             # Pydantic DTOs
│   │   ├── models.py              # SQLAlchemy/Beanie models
│   │   └── dependencies.py        # guards, current_user
│   │
│   ├── users/                     # users module
│   │   ├── router.py
│   │   ├── service.py
│   │   ├── repository.py          # DB queries
│   │   ├── schemas.py
│   │   └── models.py
│   │
│   ├── common/                    # shared utilities
│   │   ├── dependencies.py        # shared Depends (pagination, db session)
│   │   ├── exceptions.py          # custom exceptions + handlers
│   │   ├── middleware.py          # timing, logging, CORS
│   │   ├── decorators.py          # custom decorators
│   │   └── responses.py          # standard response format
│   │
│   └── workers/                   # background jobs
│       ├── celery_app.py
│       ├── tasks.py
│       └── scheduler.py           # cron jobs
│
├── alembic/                       # database migrations
│   ├── versions/
│   └── env.py
├── tests/
│   ├── conftest.py                # fixtures
│   ├── test_auth.py
│   └── test_users.py
├── alembic.ini
├── .env                           # local secrets
├── .env.staging
├── .env.production
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── pyproject.toml
```

## App Factory (main.py)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.config import settings
from app.database import init_db, close_db
from app.auth.router import router as auth_router
from app.users.router import router as users_router
from app.common.exceptions import register_exception_handlers
from app.common.middleware import TimingMiddleware, RequestIdMiddleware
from app.workers.scheduler import start_scheduler, stop_scheduler

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events."""
    # ── Startup ──
    await init_db()
    start_scheduler()
    yield
    # ── Shutdown ──
    stop_scheduler()
    await close_db()

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.APP_NAME,
        version=settings.APP_VERSION,
        docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
        redoc_url=None,
        lifespan=lifespan,
    )

    # Middleware (order matters — last added runs first)
    app.add_middleware(RequestIdMiddleware)
    app.add_middleware(TimingMiddleware)
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # Exception handlers
    register_exception_handlers(app)

    # Routers (like NestJS app.module imports)
    app.include_router(auth_router, prefix="/api/auth", tags=["Auth"])
    app.include_router(users_router, prefix="/api/users", tags=["Users"])

    return app

app = create_app()
```

---

# Part 2: Configuration & Environments

```python
# app/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # App
    APP_NAME: str = "MyApp"
    APP_VERSION: str = "1.0.0"
    ENVIRONMENT: str = "local"           # local | staging | production
    DEBUG: bool = True
    
    # Server
    HOST: str = "0.0.0.0"
    PORT: int = 8000
    WORKERS: int = 1                      # uvicorn workers (1 for dev, 4+ for prod)
    
    # Database
    DATABASE_URL: str = "postgresql+asyncpg://user:pass@localhost:5432/mydb"
    MONGO_URL: str = "mongodb://localhost:27017"
    MONGO_DB_NAME: str = "mydb"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"
    
    # Auth
    JWT_SECRET: str = "change-me-in-production"
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    
    # CORS
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]
    
    model_config = {
        "env_file": ".env",               # loads from .env file
        "env_file_encoding": "utf-8",
        "case_sensitive": True,
    }

@lru_cache()                              # singleton — same object every call
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

```bash
# .env (local)
ENVIRONMENT=local
DEBUG=true
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/mydb_dev

# .env.staging
ENVIRONMENT=staging
DEBUG=false
DATABASE_URL=postgresql+asyncpg://user:pass@staging-db:5432/mydb_staging

# .env.production
ENVIRONMENT=production
DEBUG=false
DATABASE_URL=postgresql+asyncpg://user:pass@prod-db:5432/mydb_prod
WORKERS=4
```

---

# Part 3: Pydantic Schemas (DTOs)

```python
# app/users/schemas.py
from pydantic import BaseModel, EmailStr, Field, field_validator, model_validator
from datetime import datetime
from enum import Enum

class Role(str, Enum):
    ADMIN = "admin"
    MANAGER = "manager"
    USER = "user"

class Permission(str, Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    MANAGE_USERS = "manage_users"

# ── Base (shared fields) ──
class UserBase(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)
    
# ── Create DTO (what client sends to create) ──
class UserCreate(UserBase):
    password: str = Field(min_length=8, max_length=128)
    confirm_password: str
    role: Role = Role.USER

    @field_validator("password")
    @classmethod
    def password_strength(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain an uppercase letter")
        if not any(c.isdigit() for c in v):
            raise ValueError("Password must contain a digit")
        return v

    @model_validator(mode="after")
    def passwords_match(self):
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")
        return self

# ── Update DTO (all fields optional) ──
class UserUpdate(BaseModel):
    name: str | None = Field(None, min_length=1, max_length=100)
    email: EmailStr | None = None
    role: Role | None = None

# ── Response DTO (what client receives — NO password!) ──
class UserResponse(UserBase):
    id: int
    role: Role
    is_active: bool
    created_at: datetime
    
    model_config = {"from_attributes": True}    # read from ORM objects

# ── List Response (paginated) ──
class PaginatedResponse(BaseModel):
    data: list[UserResponse]
    total: int
    page: int
    size: int
    pages: int
```

---

# Part 4: Authentication (JWT + Role-Based + Permission-Based)

```python
# app/auth/service.py
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from passlib.context import CryptContext
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(data: dict) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode = {**data, "exp": expire, "type": "access"}
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.JWT_ALGORITHM)

def create_refresh_token(data: dict) -> str:
    expire = datetime.now(timezone.utc) + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode = {**data, "exp": expire, "type": "refresh"}
    return jwt.encode(to_encode, settings.JWT_SECRET, algorithm=settings.JWT_ALGORITHM)

def decode_token(token: str) -> dict:
    return jwt.decode(token, settings.JWT_SECRET, algorithms=[settings.JWT_ALGORITHM])
```

```python
# app/auth/dependencies.py — THE GUARDS SYSTEM
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from app.auth.service import decode_token
from app.users.repository import UserRepository
from app.common.dependencies import get_db

security = HTTPBearer()

# ── Guard 1: Get Current User (like NestJS @UseGuards(JwtAuthGuard)) ──
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db=Depends(get_db),
):
    try:
        payload = decode_token(credentials.credentials)
        if payload.get("type") != "access":
            raise HTTPException(status_code=401, detail="Invalid token type")
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid or expired token")
    
    user_repo = UserRepository(db)
    user = await user_repo.get_by_id(payload["sub"])
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    if not user.is_active:
        raise HTTPException(status_code=403, detail="Account deactivated")
    return user

# ── Guard 2: Role-Based (like NestJS @Roles('admin') + RolesGuard) ──
def require_roles(*roles: str):
    """Factory that returns a dependency — checks if user has one of the required roles."""
    async def dependency(user=Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(
                status_code=403,
                detail=f"Requires one of: {', '.join(roles)}. You have: {user.role}",
            )
        return user
    return dependency

# ── Guard 3: Permission-Based (granular) ──
ROLE_PERMISSIONS = {
    "admin": {"read", "write", "delete", "manage_users"},
    "manager": {"read", "write", "delete"},
    "user": {"read", "write"},
}

def require_permissions(*perms: str):
    async def dependency(user=Depends(get_current_user)):
        user_perms = ROLE_PERMISSIONS.get(user.role, set())
        missing = set(perms) - user_perms
        if missing:
            raise HTTPException(
                status_code=403,
                detail=f"Missing permissions: {', '.join(missing)}",
            )
        return user
    return dependency

# ── Usage ──
# @router.delete("/{user_id}")
# async def delete_user(user_id: int, admin=Depends(require_roles("admin"))):
#
# @router.put("/{user_id}")
# async def update_user(user_id: int, user=Depends(require_permissions("write", "manage_users"))):
```

```python
# app/auth/router.py
from fastapi import APIRouter, Depends, HTTPException
from app.auth.service import hash_password, verify_password, create_access_token, create_refresh_token
from app.auth.schemas import LoginRequest, TokenResponse, RegisterRequest
from app.auth.dependencies import get_current_user
from app.users.repository import UserRepository
from app.common.dependencies import get_db

router = APIRouter()

@router.post("/register", response_model=TokenResponse, status_code=201)
async def register(data: RegisterRequest, db=Depends(get_db)):
    repo = UserRepository(db)
    existing = await repo.get_by_email(data.email)
    if existing:
        raise HTTPException(status_code=409, detail="Email already registered")
    
    user = await repo.create(
        email=data.email,
        name=data.name,
        hashed_password=hash_password(data.password),
    )
    return TokenResponse(
        access_token=create_access_token({"sub": str(user.id)}),
        refresh_token=create_refresh_token({"sub": str(user.id)}),
        token_type="bearer",
    )

@router.post("/login", response_model=TokenResponse)
async def login(data: LoginRequest, db=Depends(get_db)):
    repo = UserRepository(db)
    user = await repo.get_by_email(data.email)
    if not user or not verify_password(data.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    return TokenResponse(
        access_token=create_access_token({"sub": str(user.id)}),
        refresh_token=create_refresh_token({"sub": str(user.id)}),
        token_type="bearer",
    )

@router.get("/me")
async def get_me(user=Depends(get_current_user)):
    return user
```

---

# Part 5: Database — SQLAlchemy (PostgreSQL)

```python
# app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=settings.DEBUG, pool_size=20, max_overflow=10)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)  # dev only; use Alembic in prod

async def close_db():
    await engine.dispose()
```

```python
# app/users/models.py — SQLAlchemy Model
from sqlalchemy import Column, Integer, String, Boolean, DateTime, Enum, ForeignKey, Table
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.database import Base

# Many-to-many: user_roles
user_roles = Table(
    "user_roles", Base.metadata,
    Column("user_id", Integer, ForeignKey("users.id", ondelete="CASCADE")),
    Column("role_id", Integer, ForeignKey("roles.id", ondelete="CASCADE")),
)

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, index=True, nullable=False)
    name = Column(String(100), nullable=False)
    hashed_password = Column(String(128), nullable=False)
    role = Column(String(20), default="user", nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    
    # Relationships
    orders = relationship("Order", back_populates="user", cascade="all, delete-orphan")

class Order(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    total = Column(Integer, nullable=False)
    status = Column(String(20), default="pending")
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    user = relationship("User", back_populates="orders")
```

```python
# app/users/repository.py — Database Queries (like NestJS Repository)
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession

class UserRepository:
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_by_id(self, user_id: int):
        result = await self.db.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()
    
    async def get_by_email(self, email: str):
        result = await self.db.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()
    
    async def list(self, skip: int = 0, limit: int = 20, role: str | None = None):
        query = select(User)
        if role:
            query = query.where(User.role == role)
        query = query.offset(skip).limit(limit).order_by(User.created_at.desc())
        result = await self.db.execute(query)
        return result.scalars().all()
    
    async def count(self, role: str | None = None):
        query = select(func.count(User.id))
        if role:
            query = query.where(User.role == role)
        result = await self.db.execute(query)
        return result.scalar()
    
    async def create(self, **kwargs):
        user = User(**kwargs)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
    
    async def update(self, user_id: int, **kwargs):
        user = await self.get_by_id(user_id)
        if not user:
            return None
        for key, value in kwargs.items():
            if value is not None:
                setattr(user, key, value)
        await self.db.commit()
        await self.db.refresh(user)
        return user
    
    async def delete(self, user_id: int) -> bool:
        user = await self.get_by_id(user_id)
        if not user:
            return False
        await self.db.delete(user)
        await self.db.commit()
        return True
```

```python
# app/common/dependencies.py — DB Session Dependency
from app.database import AsyncSessionLocal

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

```bash
# Alembic migrations (like NestJS TypeORM migrations)
alembic init alembic
# Edit alembic/env.py to use your async engine and Base.metadata
alembic revision --autogenerate -m "create users table"
alembic upgrade head
alembic downgrade -1
```

---

# Part 6: Database — Beanie (MongoDB)

```python
# app/database.py (MongoDB section)
from motor.motor_asyncio import AsyncIOMotorClient
from beanie import init_beanie
from app.config import settings

mongo_client = AsyncIOMotorClient(settings.MONGO_URL)

async def init_mongo():
    await init_beanie(
        database=mongo_client[settings.MONGO_DB_NAME],
        document_models=[User, Order],    # register all Beanie document models
    )

async def close_mongo():
    mongo_client.close()
```

```python
# app/users/models.py (Beanie — MongoDB ODM)
from beanie import Document, Indexed
from pydantic import EmailStr, Field
from datetime import datetime

class User(Document):
    email: Indexed(EmailStr, unique=True)
    name: str
    hashed_password: str
    role: str = "user"
    is_active: bool = True
    permissions: list[str] = []
    metadata: dict = {}
    created_at: datetime = Field(default_factory=datetime.utcnow)
    
    class Settings:
        name = "users"              # collection name
        use_state_management = True  # track changes

# Usage:
# user = User(email="alice@test.com", name="Alice", hashed_password="...")
# await user.insert()
# users = await User.find(User.role == "admin").to_list()
# user = await User.find_one(User.email == "alice@test.com")
# user.name = "Alice Smith"
# await user.save()
# await user.delete()
# count = await User.find(User.is_active == True).count()
```

---

# Part 7: Router → Service → Repository Pattern

```python
# app/users/router.py (Controller layer — handles HTTP, validates, delegates)
from fastapi import APIRouter, Depends, HTTPException, Query
from app.users.service import UserService
from app.users.schemas import UserCreate, UserUpdate, UserResponse, PaginatedResponse
from app.auth.dependencies import get_current_user, require_roles, require_permissions
from app.common.dependencies import get_db

router = APIRouter()

@router.get("/", response_model=PaginatedResponse)
async def list_users(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
    role: str | None = None,
    user=Depends(get_current_user),         # must be authenticated
    db=Depends(get_db),
):
    service = UserService(db)
    return await service.list_users(page=page, size=size, role=role)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, user=Depends(get_current_user), db=Depends(get_db)):
    service = UserService(db)
    result = await service.get_user(user_id)
    if not result:
        raise HTTPException(status_code=404, detail="User not found")
    return result

@router.put("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int, data: UserUpdate,
    user=Depends(require_permissions("write", "manage_users")),
    db=Depends(get_db),
):
    service = UserService(db)
    result = await service.update_user(user_id, data)
    if not result:
        raise HTTPException(status_code=404, detail="User not found")
    return result

@router.delete("/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    admin=Depends(require_roles("admin")),
    db=Depends(get_db),
):
    service = UserService(db)
    deleted = await service.delete_user(user_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="User not found")
```

```python
# app/users/service.py (Business logic — no HTTP awareness)
import math
from app.users.repository import UserRepository
from app.users.schemas import UserCreate, UserUpdate

class UserService:
    def __init__(self, db):
        self.repo = UserRepository(db)
    
    async def list_users(self, page: int, size: int, role: str | None = None):
        skip = (page - 1) * size
        users = await self.repo.list(skip=skip, limit=size, role=role)
        total = await self.repo.count(role=role)
        return {
            "data": users,
            "total": total,
            "page": page,
            "size": size,
            "pages": math.ceil(total / size),
        }
    
    async def get_user(self, user_id: int):
        return await self.repo.get_by_id(user_id)
    
    async def update_user(self, user_id: int, data: UserUpdate):
        update_data = data.model_dump(exclude_unset=True)
        return await self.repo.update(user_id, **update_data)
    
    async def delete_user(self, user_id: int) -> bool:
        return await self.repo.delete(user_id)
```

---

# Part 8: Custom Middleware (Interceptors)

```python
# app/common/middleware.py
import time
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from loguru import logger

class TimingMiddleware(BaseHTTPMiddleware):
    """Like NestJS LoggingInterceptor — logs request duration."""
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = (time.perf_counter() - start) * 1000
        response.headers["X-Process-Time-Ms"] = f"{duration:.2f}"
        logger.info(f"{request.method} {request.url.path} → {response.status_code} ({duration:.1f}ms)")
        return response

class RequestIdMiddleware(BaseHTTPMiddleware):
    """Adds unique request ID for tracing (like NestJS CorrelationIdInterceptor)."""
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

---

# Part 9: Custom Exception Handling

```python
# app/common/exceptions.py
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from loguru import logger

class AppException(Exception):
    def __init__(self, message: str, code: str, status_code: int = 400, details: dict = None):
        self.message = message
        self.code = code
        self.status_code = status_code
        self.details = details

class NotFoundException(AppException):
    def __init__(self, resource: str, resource_id):
        super().__init__(f"{resource} with id '{resource_id}' not found", "NOT_FOUND", 404)

class ConflictException(AppException):
    def __init__(self, message: str):
        super().__init__(message, "CONFLICT", 409)

class ForbiddenException(AppException):
    def __init__(self, message: str = "You don't have permission"):
        super().__init__(message, "FORBIDDEN", 403)

def register_exception_handlers(app: FastAPI):
    @app.exception_handler(AppException)
    async def app_exception_handler(request: Request, exc: AppException):
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": exc.message, "code": exc.code, "details": exc.details},
        )

    @app.exception_handler(Exception)
    async def unhandled_exception_handler(request: Request, exc: Exception):
        logger.exception(f"Unhandled error: {exc}")
        return JSONResponse(
            status_code=500,
            content={"error": "Internal server error", "code": "INTERNAL_ERROR"},
        )
```

---

# Part 10: Background Workers (BullMQ equivalent)

```python
# app/workers/celery_app.py — Celery (Python's BullMQ)
from celery import Celery
from app.config import settings

celery_app = Celery(
    "workers",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],
    timezone="UTC",
    task_track_started=True,
    task_acks_late=True,              # re-queue if worker crashes
    worker_prefetch_multiplier=1,     # don't prefetch (fair distribution)
)
```

```python
# app/workers/tasks.py
from app.workers.celery_app import celery_app

@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def send_email_task(self, to: str, subject: str, body: str):
    try:
        email_service.send(to=to, subject=subject, body=body)
    except Exception as exc:
        self.retry(exc=exc)           # retry up to 3 times, 60s apart

@celery_app.task
def process_document_task(document_id: str):
    # Heavy processing: PDF parsing, embedding, vector store indexing
    doc = load_document(document_id)
    chunks = split_into_chunks(doc)
    embeddings = embed_chunks(chunks)
    store_in_vectordb(embeddings)
```

```python
# app/users/router.py — dispatch background task
from app.workers.tasks import send_email_task

@router.post("/", status_code=201)
async def create_user(data: UserCreate, db=Depends(get_db)):
    service = UserService(db)
    user = await service.create_user(data)
    
    # Dispatch background job (like BullMQ .add())
    send_email_task.delay(to=user.email, subject="Welcome!", body="Hello!")
    
    return user
```

```bash
# Run Celery worker (separate terminal)
celery -A app.workers.celery_app worker --loglevel=info --concurrency=4

# Run Celery beat (for scheduled tasks)
celery -A app.workers.celery_app beat --loglevel=info
```

---

# Part 11: CRON Jobs (Scheduled Tasks)

```python
# app/workers/scheduler.py
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from loguru import logger

scheduler = AsyncIOScheduler()

async def cleanup_expired_tokens():
    logger.info("Cleaning up expired tokens...")
    # delete from tokens where expires_at < now()

async def generate_daily_report():
    logger.info("Generating daily report...")

async def sync_external_data():
    logger.info("Syncing external data...")

def start_scheduler():
    scheduler.add_job(cleanup_expired_tokens, CronTrigger(hour=2, minute=0))       # 2 AM daily
    scheduler.add_job(generate_daily_report, CronTrigger(hour=6, minute=0))        # 6 AM daily
    scheduler.add_job(sync_external_data, CronTrigger(minute="*/15"))              # every 15 min
    scheduler.start()
    logger.info("Scheduler started")

def stop_scheduler():
    scheduler.shutdown()
```

---

# Part 12: SSE Streaming & WebSockets

```python
# SSE for LLM token streaming
from fastapi.responses import StreamingResponse
import json

@router.post("/generate")
async def generate(prompt: str, user=Depends(get_current_user)):
    async def event_stream():
        async for token in llm.astream(prompt):
            data = json.dumps({"type": "token", "content": token.content})
            yield f"data: {data}\n\n"
        yield f"data: {json.dumps({'type': 'done'})}\n\n"
    
    return StreamingResponse(event_stream(), media_type="text/event-stream",
                             headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})

# WebSocket
from fastapi import WebSocket, WebSocketDisconnect

class ConnectionManager:
    def __init__(self):
        self.active: dict[str, WebSocket] = {}
    
    async def connect(self, user_id: str, ws: WebSocket):
        await ws.accept()
        self.active[user_id] = ws
    
    def disconnect(self, user_id: str):
        self.active.pop(user_id, None)
    
    async def send_to_user(self, user_id: str, message: dict):
        ws = self.active.get(user_id)
        if ws:
            await ws.send_json(message)

manager = ConnectionManager()

@router.websocket("/ws/{user_id}")
async def websocket_endpoint(ws: WebSocket, user_id: str):
    await manager.connect(user_id, ws)
    try:
        while True:
            data = await ws.receive_json()
            await manager.send_to_user(data["to"], {"from": user_id, "message": data["message"]})
    except WebSocketDisconnect:
        manager.disconnect(user_id)
```

---

# Part 13: FastAPI Internals — What Makes It Fast

```
ASGI Stack:
  Client → Nginx (reverse proxy) → Uvicorn (ASGI server) → Starlette → FastAPI → Your code

FastAPI = Starlette (ASGI web framework) + Pydantic (validation)

Why it's fast:
  1. ASGI (not WSGI): async I/O, handles thousands of connections on one thread
     (like Node.js event loop vs Python Flask's one-thread-per-request)
  2. Pydantic v2: validation compiled to Rust (pydantic-core). 5-50x faster than v1.
  3. Uvicorn: async server written with uvloop (libuv bindings, same as Node.js)
  4. Zero overhead routing: Starlette's router is compiled at startup
  5. No ORM overhead in the framework itself (you choose your own)

async def vs def:
  async def: runs on the event loop. Use with async libraries (asyncpg, httpx, motor).
  def: auto-wrapped in a thread pool. Use with blocking libraries (requests, psycopg2).
  
  ⚠️ NEVER call blocking code in async def — it blocks the ENTIRE event loop.
  ⚠️ If you use `def` (not `async def`), FastAPI runs it in a thread pool automatically.

Uvicorn workers:
  Single worker = single process = one event loop.
  Multiple workers = multiple processes (like Node.js cluster).
  Production: 2× CPU cores + 1 workers (e.g., 9 workers on 4-core machine).
```

---

# Part 14: Production Build & Deployment

```bash
# Development
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Production (multi-worker with Gunicorn)
gunicorn app.main:app \
    -w 4 \                          # 4 worker processes
    -k uvicorn.workers.UvicornWorker \  # async worker class
    -b 0.0.0.0:8000 \
    --access-logfile - \
    --error-logfile - \
    --timeout 120 \
    --graceful-timeout 30 \
    --keep-alive 5
```

```dockerfile
# Dockerfile (multi-stage production build)
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM base AS production
COPY . .
ENV ENVIRONMENT=production
EXPOSE 8000
CMD ["gunicorn", "app.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000", "--timeout", "120"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["8000:8000"]
    env_file: .env.production
    depends_on: [postgres, redis, mongo]
    restart: unless-stopped
    
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes: [postgres_data:/var/lib/postgresql/data]
    
  redis:
    image: redis:7-alpine
    
  mongo:
    image: mongo:7
    volumes: [mongo_data:/data/db]
    
  celery_worker:
    build: .
    command: celery -A app.workers.celery_app worker --loglevel=info -c 4
    env_file: .env.production
    depends_on: [redis, postgres]
    
  celery_beat:
    build: .
    command: celery -A app.workers.celery_app beat --loglevel=info
    env_file: .env.production
    depends_on: [redis]

volumes:
  postgres_data:
  mongo_data:
```

---

# Part 15: Testing

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.database import AsyncSessionLocal

@pytest.fixture
async def client():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac

@pytest.fixture
async def auth_client(client):
    """Client with valid auth token."""
    response = await client.post("/api/auth/register", json={
        "email": "test@test.com", "name": "Test", "password": "Test1234", "confirm_password": "Test1234"
    })
    token = response.json()["access_token"]
    client.headers["Authorization"] = f"Bearer {token}"
    yield client
```

```python
# tests/test_users.py
import pytest

@pytest.mark.asyncio
async def test_list_users_requires_auth(client):
    response = await client.get("/api/users/")
    assert response.status_code == 403

@pytest.mark.asyncio
async def test_list_users_authenticated(auth_client):
    response = await auth_client.get("/api/users/")
    assert response.status_code == 200
    data = response.json()
    assert "data" in data
    assert "total" in data

@pytest.mark.asyncio
async def test_create_user_validation(auth_client):
    response = await auth_client.post("/api/auth/register", json={
        "email": "invalid", "name": "", "password": "weak",
    })
    assert response.status_code == 422
```

---

# Part 16: 🧩 Interview Q&A

**Q: How does FastAPI's dependency injection compare to NestJS?**
A: NestJS uses constructor injection with decorators (`@Injectable()`, `@Inject()`), a module-scoped IoC container, and TypeScript's `emitDecoratorMetadata`. FastAPI uses function-based injection with `Depends()`. Dependencies are functions (sync or async) whose return values are injected into route handlers. They compose: a dependency can depend on other dependencies, forming a tree. FastAPI resolves the tree per-request, calling each dependency once. It's simpler than NestJS but equally powerful — guards become dependencies that raise HTTPException, interceptors become middleware, and pipes are replaced by Pydantic validation.

**Q: How do you handle different environments in FastAPI?**
A: Use `pydantic-settings` with `.env` files. Create `.env` (local), `.env.staging`, `.env.production`. The `Settings` class reads env vars with type validation and defaults. Use `ENVIRONMENT` variable to conditionally configure: disable `/docs` in production, set different DB URLs, adjust worker counts. In Docker, pass `--env-file .env.production`. Settings are cached with `@lru_cache` for singleton behavior.

**Q: What makes FastAPI "fast"?**
A: Three things. (1) ASGI: async I/O model like Node.js — handles thousands of concurrent connections on one thread, no thread-per-request overhead. (2) Pydantic v2: validation engine compiled to Rust, 5-50x faster than pure Python validation. (3) Uvicorn with uvloop: same libuv event loop that powers Node.js, running Python async code at near-C speeds. Combined, FastAPI benchmarks close to Go and Node.js frameworks, far ahead of Django and Flask.

**Q: How do you handle background jobs in FastAPI?**
A: Three options. (1) `BackgroundTasks` — built-in, runs after response is sent, no persistence, no retries. Use for fire-and-forget (logging, simple emails). (2) Celery + Redis — production-grade task queue with retries, scheduling, monitoring (Flower). Python's equivalent of BullMQ. Use for heavy processing, reliable job execution. (3) ARQ — lighter async alternative to Celery. Use when you want async-native job processing without Celery's complexity.

**Q: Controller → Service → Repository — how does this work in FastAPI?**
A: Router (controller) handles HTTP concerns: route definitions, request parsing, response formatting, auth guards via Depends. Service contains business logic: validation rules, calculations, orchestration — no HTTP or DB awareness. Repository handles database queries: raw SQLAlchemy/Beanie operations. The router creates a Service instance (passing DB session), the service creates a Repository instance. This separation makes business logic testable without HTTP or database, and repositories swappable between SQL and MongoDB.
