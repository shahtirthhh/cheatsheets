# FastAPI — 0 to Master Cheatsheet
*From "what is an API" to production-grade architecture*

---

# Part 1: The Basics (What Even Is This?)

## What Is an API?

An API (Application Programming Interface) is a way for two programs to talk to each other. When your React app fetches user data, it sends an HTTP request to a URL. The server at that URL processes the request and sends back data (usually JSON). That URL + the logic behind it = an API.

```
React App                     Server (FastAPI)
─────────                     ────────────────
"Give me user 5"      →      GET /api/users/5
                      ←      { "name": "Alice", "age": 30 }

"Create a new user"   →      POST /api/users  (with JSON body)
                      ←      { "id": 6, "name": "Bob" }
```

## What Is FastAPI?

FastAPI is a Python web framework for building APIs. It's the Python equivalent of Express.js (Node.js) or NestJS.

```
Express.js (Node)              FastAPI (Python)
──────────────                 ────────────────
app.get('/users', handler)     @app.get("/users")
app.post('/users', handler)    @app.post("/users")
req.params.id                  path parameter: user_id: int
req.query.page                 query parameter: page: int = 1
req.body                       Pydantic model (auto-validated!)
middleware()                   Middleware / Depends()
```

**Why FastAPI over Flask/Django?**
- **Auto-validation** — request data is validated by Pydantic before your code runs
- **Auto-documentation** — Swagger UI at `/docs` is generated automatically
- **Async support** — native `async/await` like Node.js
- **Type hints** — Python types become your documentation AND validation
- **Fast** — one of the fastest Python frameworks (on par with Node.js)

## Installation and First App

```bash
pip install fastapi uvicorn
```

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")                          # ← this is a "route decorator"
def read_root():                       # ← this is a "route handler" / "endpoint"
    return {"message": "Hello, World!"}

@app.get("/users/{user_id}")           # ← {user_id} is a path parameter
def read_user(user_id: int):           # ← type hint = auto-validation + auto-docs
    return {"user_id": user_id}

# Run: uvicorn main:app --reload
# Visit: http://localhost:8000/docs  ← auto-generated Swagger UI
```

```
Equivalent in Express:

const app = express();

app.get("/", (req, res) => {
  res.json({ message: "Hello, World!" });
});

app.get("/users/:user_id", (req, res) => {
  // YOU must validate req.params.user_id is a number
  res.json({ user_id: parseInt(req.params.user_id) });
});
```

**The magic of FastAPI:** If someone hits `/users/abc`, FastAPI automatically returns `422 Unprocessable Entity` with a clear error message. In Express, you'd get `NaN` unless you add validation yourself.

---

# Part 2: Route Parameters and Request Data

## Path Parameters

```python
@app.get("/users/{user_id}")
def get_user(user_id: int):         # ← int means FastAPI rejects non-integers
    return {"user_id": user_id}

# GET /users/5     → {"user_id": 5}
# GET /users/abc   → 422 error (not an integer)

# Multiple path params
@app.get("/users/{user_id}/posts/{post_id}")
def get_post(user_id: int, post_id: int):
    return {"user_id": user_id, "post_id": post_id}

# Enum path params (restrict to specific values)
from enum import Enum

class ModelName(str, Enum):
    gpt4 = "gpt-4"
    claude = "claude"
    llama = "llama"

@app.get("/models/{model_name}")
def get_model(model_name: ModelName):
    return {"model": model_name.value}

# GET /models/gpt-4   → {"model": "gpt-4"}
# GET /models/gemini   → 422 error (not in enum)
```

## Query Parameters

Any function parameter that's NOT in the URL path becomes a query parameter.

```python
@app.get("/users")
def list_users(
    skip: int = 0,            # optional, default 0
    limit: int = 10,          # optional, default 10
    active: bool = True,      # optional, default True
    search: str | None = None # optional, can be None
):
    return {"skip": skip, "limit": limit, "active": active, "search": search}

# GET /users                      → skip=0, limit=10, active=True, search=None
# GET /users?skip=20&limit=5      → skip=20, limit=5
# GET /users?active=false&search=alice  → active=False, search="alice"
# GET /users?limit=abc            → 422 error (not an integer)
```

```
NestJS equivalent:
  @Get('users')
  listUsers(
    @Query('skip') skip: number = 0,
    @Query('limit') limit: number = 10,
  ) { ... }

FastAPI: no decorators needed. Type hints DO everything.
```

## Request Body (Pydantic Models)

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

# Define the shape of incoming data
class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150)       # greater/equal 0, less/equal 150
    role: str = "user"                    # optional with default
    tags: list[str] = []                  # optional, defaults to empty list

@app.post("/users", status_code=201)     # ← custom status code
def create_user(user: UserCreate):        # ← body is auto-parsed and validated
    return {"id": 1, **user.model_dump()}

# POST /users with body: {"name": "Alice", "email": "alice@test.com", "age": 30}
# → {"id": 1, "name": "Alice", "email": "alice@test.com", "age": 30, "role": "user", "tags": []}

# POST with invalid data: {"name": "", "email": "not-an-email", "age": -5}
# → 422 with detailed errors for EACH field
```

```
NestJS equivalent:
  class CreateUserDto {
    @IsString() @MinLength(1) name: string;
    @IsEmail() email: string;
    @IsInt() @Min(0) @Max(150) age: number;
  }

  @Post('users')
  createUser(@Body() dto: CreateUserDto) { ... }

FastAPI: the Pydantic model IS the DTO + validation. No decorators needed.
```

## All Request Data Sources

```python
from fastapi import Path, Query, Body, Header, Cookie, File, Form, UploadFile

@app.put("/users/{user_id}")
def update_user(
    user_id: int = Path(ge=1),                           # from URL path
    q: str | None = Query(None, max_length=50),          # from ?q=...
    user: UserUpdate = Body(),                            # from request body
    x_token: str = Header(),                              # from X-Token header
    session_id: str | None = Cookie(None),               # from cookies
):
    return {"user_id": user_id, "q": q, "token": x_token}
```

---

# Part 3: Pydantic — The Type System That Makes FastAPI Work

## Pydantic Is to FastAPI What Zod Is to NestJS

Every piece of data flowing through FastAPI is validated by Pydantic. Request bodies, query params, headers, responses — all Pydantic.

### All Field Types

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Literal, Annotated
from datetime import datetime, date
from enum import Enum

class FullExample(BaseModel):
    # Basic types
    name: str                                    # required string
    age: int                                     # required integer
    score: float                                 # required float
    active: bool                                 # required boolean

    # Optional (can be None)
    bio: str | None = None                       # optional, defaults to None
    nickname: str = "anonymous"                  # optional with default

    # Constrained types
    username: str = Field(min_length=3, max_length=20, pattern=r"^[a-z0-9_]+$")
    rating: float = Field(ge=0, le=5)           # 0.0 to 5.0
    quantity: int = Field(gt=0, lt=1000)         # 1 to 999

    # Collections
    tags: list[str] = []                         # list of strings
    scores: dict[str, int] = {}                  # dict with string keys, int values
    unique_ids: set[int] = set()                 # set of integers
    coordinates: tuple[float, float] = (0, 0)    # exactly 2 floats

    # Enums
    role: Literal["admin", "user", "guest"] = "user"   # one of these exact strings

    # Nested models
    address: "Address | None" = None             # another Pydantic model

    # Dates
    created_at: datetime = Field(default_factory=datetime.now)
    birth_date: date | None = None

class Address(BaseModel):
    street: str
    city: str
    zip_code: str = Field(pattern=r"^\d{5}$")
```

### Validators (Custom Rules)

```python
class User(BaseModel):
    name: str
    email: str
    password: str
    confirm_password: str

    # Validate a single field
    @field_validator("name")
    @classmethod
    def name_must_not_be_blank(cls, v):
        if not v.strip():
            raise ValueError("Name cannot be blank")
        return v.strip()

    @field_validator("email")
    @classmethod
    def email_must_be_lowercase(cls, v):
        return v.lower()

    # Validate across multiple fields
    @model_validator(mode="after")
    def passwords_must_match(self):
        if self.password != self.confirm_password:
            raise ValueError("Passwords do not match")
        return self
```

### Model Inheritance (DRY Pattern)

```python
# Base with shared fields
class UserBase(BaseModel):
    name: str
    email: EmailStr

# Input model (what the client sends)
class UserCreate(UserBase):
    password: str = Field(min_length=8)

# Update model (all fields optional)
class UserUpdate(BaseModel):
    name: str | None = None
    email: EmailStr | None = None

# Output model (what the client receives — no password!)
class UserResponse(UserBase):
    id: int
    created_at: datetime
    model_config = {"from_attributes": True}    # read from ORM objects

# Internal model (what's in the database)
class UserInDB(UserBase):
    id: int
    hashed_password: str
    created_at: datetime
```

### Response Model (Control What You Send Back)

```python
@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate):
    # Even if your internal object has password, hashed_password, etc.
    # FastAPI strips everything not in UserResponse before sending
    db_user = save_to_db(user)
    return db_user    # only id, name, email, created_at are sent
```

---

# Part 4: Dependency Injection

## What Is Depends()?

`Depends()` is FastAPI's dependency injection system. It lets you extract reusable logic (database connections, auth, pagination) into functions that are automatically called and injected.

```
NestJS:  @Injectable() + providers array + @Inject()
FastAPI: just a function + Depends()
```

### Basic: Database Session

```python
from fastapi import Depends

# Dependency function
def get_db():
    db = SessionLocal()          # create database session
    try:
        yield db                 # ← this is what gets injected
    finally:
        db.close()               # cleanup after request finishes

# Use it in any endpoint
@app.get("/users")
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()

@app.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    return db.query(User).get(user_id)

# `get_db` is called automatically before each endpoint
# `db.close()` runs automatically after each endpoint
# Just like NestJS OnModuleDestroy or Express middleware cleanup
```

### Authentication Dependency

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

# Dependency: extract and verify JWT
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db),                # dependencies can depend on OTHER dependencies
):
    token = credentials.credentials
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user = db.query(User).get(payload["sub"])
        if not user:
            raise HTTPException(status_code=401, detail="User not found")
        return user
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Use it — user is automatically authenticated
@app.get("/me")
def read_me(user: User = Depends(get_current_user)):
    return user

# Role-based auth (factory function returns a dependency)
def require_role(*roles: str):
    def dependency(user: User = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(status_code=403, detail="Forbidden")
        return user
    return dependency

@app.delete("/users/{user_id}")
def delete_user(user_id: int, admin: User = Depends(require_role("admin"))):
    # Only admins reach here
    ...
```

```
Dependency tree (FastAPI resolves automatically):

  get_current_user
    ├── security (extracts Bearer token from header)
    └── get_db (database session)

  require_role("admin")
    └── get_current_user
          ├── security
          └── get_db

Each dependency is called ONCE per request, even if multiple params use it.
```

### Pagination Dependency

```python
class PaginationParams:
    def __init__(self, page: int = 1, size: int = 20):
        self.page = max(1, page)
        self.size = min(100, size)
        self.skip = (self.page - 1) * self.size

@app.get("/users")
def list_users(
    pagination: PaginationParams = Depends(),      # note: Depends() with no argument
    db: Session = Depends(get_db),
):
    users = db.query(User).offset(pagination.skip).limit(pagination.size).all()
    return {"data": users, "page": pagination.page, "size": pagination.size}
```

---

# Part 5: Middleware, Error Handling, Background Tasks

## Middleware

```python
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
import time

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.time()
        response = await call_next(request)            # call the actual endpoint
        duration = time.time() - start
        response.headers["X-Process-Time"] = f"{duration:.4f}"
        return response

app.add_middleware(TimingMiddleware)

# CORS (like cors() in Express)
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Error Handling

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse

# Throw errors anywhere
@app.get("/users/{user_id}")
def get_user(user_id: int):
    user = db.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# Custom exception class
class AppError(Exception):
    def __init__(self, message: str, code: str, status_code: int = 400):
        self.message = message
        self.code = code
        self.status_code = status_code

# Global exception handler
@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.message, "code": exc.code},
    )

# Now anywhere in your code:
raise AppError("Insufficient credits", code="CREDITS_EXHAUSTED", status_code=402)
```

## Background Tasks

```python
from fastapi import BackgroundTasks

def send_email(to: str, subject: str, body: str):
    # This runs AFTER the response is sent
    email_service.send(to=to, subject=subject, body=body)

@app.post("/register")
def register(user: UserCreate, background_tasks: BackgroundTasks):
    new_user = create_user(user)

    # Schedule background work (runs after response is sent)
    background_tasks.add_task(send_email, new_user.email, "Welcome!", "Hello!")
    background_tasks.add_task(log_signup, new_user.id)

    return new_user    # client gets response immediately
```

---

# Part 6: Async, Streaming, WebSockets

## Async vs Sync Handlers

```python
# ASYNC handler — use when calling async libraries (httpx, asyncpg, motor)
@app.get("/users")
async def get_users():
    users = await async_db.fetch_all("SELECT * FROM users")
    return users

# SYNC handler — FastAPI auto-wraps in a thread pool
@app.get("/legacy")
def get_legacy():
    data = requests.get("http://legacy-api/data")    # blocking but safe
    return data.json()

# ⚠️ DANGER: blocking code in async handler
@app.get("/broken")
async def broken():
    data = requests.get("http://slow-api")    # BLOCKS the event loop!
    return data.json()
# Fix: use `def` (not `async def`) or use httpx (async HTTP client)
```

## SSE Streaming (For LLM Token Streaming)

```python
from fastapi.responses import StreamingResponse
import json

@app.post("/generate")
async def generate(prompt: str):
    async def event_stream():
        async for token in llm.astream(prompt):
            data = json.dumps({"type": "token", "content": token.content})
            yield f"data: {data}\n\n"
        yield f"data: {json.dumps({'type': 'done'})}\n\n"

    return StreamingResponse(
        event_stream(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

## WebSockets

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            data = await ws.receive_text()
            await ws.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

---

# Part 7: File Uploads, Forms, Static Files

```python
from fastapi import File, UploadFile, Form

# Single file upload
@app.post("/upload")
async def upload(file: UploadFile):
    contents = await file.read()
    return {
        "filename": file.filename,
        "size": len(contents),
        "content_type": file.content_type,
    }

# Multiple files
@app.post("/upload-many")
async def upload_many(files: list[UploadFile]):
    return [{"filename": f.filename} for f in files]

# File + form data together
@app.post("/submit")
async def submit(
    title: str = Form(),
    description: str = Form(""),
    file: UploadFile = File(),
):
    return {"title": title, "filename": file.filename}
```

---

# Part 8: Application Structure & Routers

## Splitting Into Multiple Files (Like Express Router or NestJS Modules)

```
project/
├── main.py                  # app creation, router registration
├── config.py                # settings
├── dependencies.py          # shared Depends (db, auth)
├── routers/
│   ├── users.py             # user endpoints
│   ├── auth.py              # auth endpoints
│   └── ai.py                # AI/LLM endpoints
├── schemas/                 # Pydantic models
│   ├── users.py
│   └── ai.py
├── services/                # business logic
│   ├── user_service.py
│   └── ai_service.py
├── models/                  # database models (SQLAlchemy/MongoDB)
│   └── user.py
└── tests/
```

```python
# routers/users.py
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/api/users",
    tags=["Users"],          # groups endpoints in Swagger docs
)

@router.get("/")
def list_users(): ...

@router.get("/{user_id}")
def get_user(user_id: int): ...

@router.post("/", status_code=201)
def create_user(user: UserCreate): ...

# main.py
from fastapi import FastAPI
from routers import users, auth, ai

app = FastAPI(title="My API", version="1.0.0")
app.include_router(users.router)
app.include_router(auth.router)
app.include_router(ai.router)
```

---

# Part 9: Testing

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello, World!"}

def test_create_user():
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@test.com",
        "age": 30,
    })
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Alice"
    assert "id" in data

def test_invalid_user():
    response = client.post("/api/users", json={
        "name": "",           # too short
        "email": "not-email", # invalid
        "age": -5,            # below 0
    })
    assert response.status_code == 422   # validation error

# Override dependencies for testing
from dependencies import get_db

def get_test_db():
    db = TestSession()
    try:
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = get_test_db
```

---

# Part 10: Deployment

```bash
# Development
uvicorn main:app --reload --host 0.0.0.0 --port 8000

# Production (multiple workers)
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8000

# Docker
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["gunicorn", "main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

---

# Part 11: 🧩 Interview Q&A

**Q: What makes FastAPI fast?**
A: It's built on Starlette (ASGI framework) and Pydantic (data validation). ASGI supports async natively. Pydantic v2 is written in Rust. Combined with Python's type hints, there's minimal overhead.

**Q: What's the difference between `def` and `async def` handlers?**
A: `async def` runs on the async event loop — use with async libraries. `def` runs in a thread pool — use with blocking/sync code. NEVER call blocking code inside `async def`.

**Q: How does FastAPI auto-generate documentation?**
A: It reads your type hints, Pydantic models, and route decorators at startup to build an OpenAPI schema. Swagger UI (`/docs`) and ReDoc (`/redoc`) render this schema.

**Q: Depends() vs middleware — when to use which?**
A: Middleware runs on EVERY request (logging, CORS, timing). Depends() runs only on routes that declare it (auth, pagination, DB sessions). Depends() is more granular and testable.

**Q: How does Pydantic validation work under the hood?**
A: Pydantic v2 compiles validators into Rust-powered validation functions at class creation time. When data arrives, it runs through the compiled schema — that's why it's fast.

**Q: What is ASGI vs WSGI?**
A: WSGI (Web Server Gateway Interface) is synchronous — one request per thread (Flask, Django). ASGI (Asynchronous Server Gateway Interface) is async — handles thousands of concurrent connections on a single thread (FastAPI, Starlette). ASGI is to Python what the event loop is to Node.js.
