---
name: fastapi-developer
description: FastAPI development: Pydantic v2 models, dependency injection, async patterns, background tasks, WebSocket endpoints, OpenAPI customization, and deployment.
---

# FastAPI Developer

Production-grade FastAPI development patterns for Python 3.11+. Covers the full lifecycle from Pydantic v2 data modeling through async database access, authentication, real-time WebSockets, and containerized deployment. All examples use Pydantic v2 semantics, SQLAlchemy 2.0 async style, and the modern `lifespan` context manager.

## Table of Contents

- [Pydantic V2 Models](#pydantic-v2-models)
- [Dependency Injection](#dependency-injection)
- [Routing and Middleware](#routing-and-middleware)
- [Async Patterns](#async-patterns)
- [Database Integration](#database-integration)
- [Authentication](#authentication)
- [WebSockets](#websockets)
- [Testing and Deployment](#testing-and-deployment)

---

## Pydantic V2 Models

Pydantic v2 is backed by `pydantic-core` in Rust. Use `model_validator`, `field_validator`, `ConfigDict`, and `computed_field` instead of the legacy v1 API.

### Model Definitions and ConfigDict

```python
from pydantic import BaseModel, ConfigDict, Field
from datetime import datetime
from uuid import UUID, uuid4

class UserBase(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        from_attributes=True,        # replaces orm_mode
        populate_by_name=True,       # replaces allow_population_by_field_name
    )
    email: str = Field(..., pattern=r"^[\w.+-]+@[\w-]+\.[\w.]+$")
    display_name: str = Field(..., max_length=100, alias="name")

class UserCreate(UserBase):
    password: str = Field(..., min_length=12, max_length=128)

class UserRead(UserBase):
    id: UUID = Field(default_factory=uuid4)
    created_at: datetime
    is_active: bool = True
```

### Validators and Computed Fields

```python
from pydantic import field_validator, model_validator, computed_field, BaseModel

class SignupRequest(BaseModel):
    username: str
    password: str
    password_confirm: str

    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("Username must be alphanumeric")
        return v.lower()

    @model_validator(mode="after")
    def passwords_match(self) -> "SignupRequest":
        if self.password != self.password_confirm:
            raise ValueError("Passwords do not match")
        return self

class Product(BaseModel):
    price_cents: int
    tax_rate: float = 0.08

    @computed_field
    @property
    def price_display(self) -> str:
        total = self.price_cents * (1 + self.tax_rate)
        return f"${total / 100:.2f}"
```

### Discriminated Unions

```python
from pydantic import BaseModel, Field
from typing import Literal, Annotated, Union

class CreditCard(BaseModel):
    method: Literal["credit_card"] = "credit_card"
    card_number: str
    exp_month: int

class BankTransfer(BaseModel):
    method: Literal["bank_transfer"] = "bank_transfer"
    routing_number: str
    account_number: str

PaymentMethod = Annotated[
    Union[CreditCard, BankTransfer],
    Field(discriminator="method"),
]

class Order(BaseModel):
    amount_cents: int
    payment: PaymentMethod    # Fast O(1) dispatch on "method" field
```

### JSON Schema Customization

```python
from pydantic import BaseModel, Field
from typing import Annotated

Latitude = Annotated[float, Field(ge=-90, le=90, json_schema_extra={"example": 37.7749})]
Longitude = Annotated[float, Field(ge=-180, le=180, json_schema_extra={"example": -122.4194})]

class Location(BaseModel):
    lat: Latitude
    lng: Longitude
```

### Anti-Patterns

```python
# WRONG: v1-style Config inner class — silently ignored in v2
class Bad(BaseModel):
    class Config:
        orm_mode = True          # Has no effect in Pydantic v2

# WRONG: v1 @validator decorator — raises deprecation warning
from pydantic import validator   # Don't import this in v2

# CORRECT: use model_config = ConfigDict(...) and @field_validator
```

---

## Dependency Injection

FastAPI's `Depends()` provides composable, testable, type-safe injection.

### Basic Dependencies

```python
from fastapi import Depends, Query, FastAPI
from typing import Annotated

app = FastAPI()

async def pagination(
    skip: int = Query(0, ge=0),
    limit: int = Query(20, ge=1, le=100),
) -> dict[str, int]:
    return {"skip": skip, "limit": limit}

PaginationDep = Annotated[dict[str, int], Depends(pagination)]

@app.get("/items")
async def list_items(pagination: PaginationDep):
    return {"skip": pagination["skip"], "limit": pagination["limit"]}
```

### Sub-Dependencies and Chaining

```python
from fastapi import Depends, Header, HTTPException
from typing import Annotated

async def get_token(authorization: str = Header(...)) -> str:
    scheme, _, token = authorization.partition(" ")
    if scheme.lower() != "bearer" or not token:
        raise HTTPException(401, "Invalid auth header")
    return token

async def get_current_user(token: str = Depends(get_token)) -> dict:
    user = await decode_and_lookup(token)
    if not user:
        raise HTTPException(401, "Invalid token")
    return user

async def require_admin(user: dict = Depends(get_current_user)) -> dict:
    if "admin" not in user.get("roles", []):
        raise HTTPException(403, "Admin required")
    return user

CurrentUser = Annotated[dict, Depends(get_current_user)]
AdminUser = Annotated[dict, Depends(require_admin)]
```

### Yield Dependencies (Resource Cleanup)

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

async def get_db(
    session_factory: async_sessionmaker[AsyncSession] = Depends(get_session_factory),
) -> AsyncGenerator[AsyncSession, None]:
    async with session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

DbSession = Annotated[AsyncSession, Depends(get_db)]
```

### Dependency Overrides for Testing

```python
from fastapi.testclient import TestClient

def fake_current_user() -> dict:
    return {"id": "test-user", "roles": ["admin"]}

app.dependency_overrides[get_current_user] = fake_current_user
client = TestClient(app)
response = client.get("/admin/dashboard")
assert response.status_code == 200
app.dependency_overrides.clear()
```

### Anti-Patterns

```python
# WRONG: instantiating a DB session inside the route body
@app.get("/items")
async def list_items():
    session = SessionLocal()          # No cleanup guarantee
    items = await session.execute(...)
    session.close()                   # Skipped if exception occurs
    return items

# CORRECT: use a yield dependency so cleanup is guaranteed
```

---

## Routing and Middleware

### APIRouter and Lifespan

```python
# app/routers/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/api/v1/users", tags=["users"])

@router.get("/")
async def list_users(): ...

@router.get("/{user_id}")
async def get_user(user_id: int): ...
```

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.pool = await create_db_pool()   # Startup
    yield
    await app.state.pool.close()              # Shutdown

app = FastAPI(title="My Service", lifespan=lifespan)
app.include_router(users.router)
app.include_router(admin.router, prefix="/admin", dependencies=[Depends(require_admin)])
```

### CORS and Custom Middleware

```python
from fastapi.middleware.cors import CORSMiddleware
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],   # Never ["*"] in production
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        start = time.perf_counter()
        response = await call_next(request)
        response.headers["X-Process-Time-Ms"] = f"{(time.perf_counter() - start) * 1000:.1f}"
        return response

app.add_middleware(TimingMiddleware)
```

### Exception Handlers

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class DomainError(Exception):
    def __init__(self, code: str, detail: str, status: int = 400):
        self.code, self.detail, self.status = code, detail, status

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError) -> JSONResponse:
    return JSONResponse(status_code=exc.status, content={"error": {"code": exc.code, "detail": exc.detail}})
```

### Anti-Patterns

```python
# WRONG: deprecated @app.on_event decorator
@app.on_event("startup")        # Use lifespan instead
async def startup(): ...

# WRONG: wildcard CORS in production
app.add_middleware(CORSMiddleware, allow_origins=["*"])  # Security risk
```

---

## Async Patterns

FastAPI runs on ASGI (Uvicorn). Use async for I/O-bound work; use plain `def` for CPU-bound or blocking-library calls (FastAPI auto-threads them).

### Async vs Sync Endpoints

```python
import httpx

# async def: for I/O with async-capable libraries
@app.get("/external")
async def call_external():
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://api.example.com/data")
    return resp.json()

# plain def: for CPU-bound or blocking libraries — auto-threaded
@app.get("/cpu-heavy")
def compute_report():
    return heavy_pandas_computation()
```

### Shared HTTP Client via Lifespan

```python
from contextlib import asynccontextmanager
import httpx

@asynccontextmanager
async def lifespan(app):
    app.state.http = httpx.AsyncClient(
        timeout=httpx.Timeout(10.0, connect=5.0),
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
    )
    yield
    await app.state.http.aclose()

@app.get("/proxy")
async def proxy(request: Request):
    resp = await request.app.state.http.get("https://api.example.com/resource")
    return resp.json()
```

### Task Groups and Background Tasks

```python
import asyncio
from fastapi import BackgroundTasks

@app.get("/aggregate")
async def aggregate(request: Request):
    client = request.app.state.http
    async def fetch(url: str) -> dict:
        return (await client.get(url)).json()

    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch("https://api.example.com/users"))
        t2 = tg.create_task(fetch("https://api.example.com/orders"))
    return {"users": t1.result(), "orders": t2.result()}

@app.post("/users", status_code=201)
async def create_user(body: UserCreate, bg: BackgroundTasks):
    user = await user_repo.create(body)
    bg.add_task(send_welcome_email, user.email, user.display_name)
    bg.add_task(write_audit_log, str(user.id), "user_created")
    return user
```

### Anti-Patterns

```python
# WRONG: blocking I/O inside async def
@app.get("/bad")
async def bad_endpoint():
    import requests                     # Blocks the event loop!
    return requests.get("https://api.example.com").json()

# WRONG: new httpx client per request — TCP handshake every time
@app.get("/wasteful")
async def wasteful():
    async with httpx.AsyncClient() as client:
        return (await client.get("https://api.example.com")).json()

# CORRECT: shared client from lifespan; plain def for blocking libs
```

---

## Database Integration

### SQLAlchemy 2.0 Async Engine

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost:5432/mydb",
    pool_size=20, max_overflow=10, pool_pre_ping=True, pool_recycle=3600,
)
SessionFactory = async_sessionmaker(engine, expire_on_commit=False)
```

### Declarative Models

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, ForeignKey, func
from datetime import datetime
from uuid import UUID, uuid4

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="selectin")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    title: Mapped[str] = mapped_column(String(200))
    body: Mapped[str]
    author_id: Mapped[UUID] = mapped_column(ForeignKey("users.id"))
    author: Mapped["User"] = relationship(back_populates="posts")
```

### Repository Pattern

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

class UserRepository:
    def __init__(self, session: AsyncSession):
        self._s = session

    async def get_by_id(self, user_id: UUID) -> User | None:
        return await self._s.get(User, user_id)

    async def get_by_email(self, email: str) -> User | None:
        result = await self._s.execute(select(User).where(User.email == email))
        return result.scalar_one_or_none()

    async def create(self, email: str, hashed_pw: str) -> User:
        user = User(email=email, hashed_password=hashed_pw)
        self._s.add(user)
        await self._s.flush()
        return user
```

### Alembic Async Migrations

```python
# alembic/env.py
import asyncio
from alembic import context
from sqlalchemy.ext.asyncio import create_async_engine
from app.db.models import Base

def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=Base.metadata)
    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations():
    engine = create_async_engine(config.get_main_option("sqlalchemy.url"))
    async with engine.connect() as conn:
        await conn.run_sync(do_run_migrations)
    await engine.dispose()

def run_migrations_online():
    asyncio.run(run_async_migrations())
```

### Anti-Patterns

```python
# WRONG: synchronous driver with async engine
engine = create_async_engine("postgresql://...")   # Needs +asyncpg

# WRONG: lazy loading in async context
user = await session.get(User, uid)
print(user.posts)  # MissingGreenlet error — lazy load is sync

# CORRECT: eager load with selectinload
from sqlalchemy.orm import selectinload
stmt = select(User).options(selectinload(User.posts)).where(User.id == uid)
```

---

## Authentication

### OAuth2 Password Bearer with JWT

```python
from datetime import datetime, timedelta, timezone
import jwt
from pydantic import BaseModel

SECRET_KEY = "change-me-use-env-variable"
ALGORITHM = "HS256"

class TokenPayload(BaseModel):
    sub: str
    exp: datetime
    roles: list[str] = []

def create_access_token(user_id: str, roles: list[str], minutes: int = 30) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=minutes)
    return jwt.encode(
        TokenPayload(sub=user_id, exp=expire, roles=roles).model_dump(),
        SECRET_KEY, algorithm=ALGORITHM,
    )

def decode_access_token(token: str) -> TokenPayload:
    return TokenPayload(**jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM]))
```

### Security Dependencies

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from typing import Annotated

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> TokenPayload:
    try:
        return decode_access_token(token)
    except jwt.ExpiredSignatureError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token")

def require_roles(*required: str):
    async def checker(user: TokenPayload = Depends(get_current_user)) -> TokenPayload:
        if not set(required).intersection(user.roles):
            raise HTTPException(status.HTTP_403_FORBIDDEN, "Insufficient permissions")
        return user
    return checker

CurrentUser = Annotated[TokenPayload, Depends(get_current_user)]
AdminUser = Annotated[TokenPayload, Depends(require_roles("admin"))]
```

### Login Endpoint

```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from passlib.context import CryptContext

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])
pwd_ctx = CryptContext(schemes=["bcrypt"], deprecated="auto")

@router.post("/token")
async def login(form: OAuth2PasswordRequestForm = Depends(), db: DbSession = Depends()):
    user = await UserRepository(db).get_by_email(form.username)
    if not user or not pwd_ctx.verify(form.password, user.hashed_password):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Bad credentials")
    return {"access_token": create_access_token(str(user.id), user.roles), "token_type": "bearer"}
```

### API Key Authentication

```python
from fastapi.security import APIKeyHeader
from fastapi import Security, HTTPException, status

api_key_header = APIKeyHeader(name="X-API-Key")
VALID_KEYS = {"sk-live-abc123": "service-a", "sk-live-def456": "service-b"}

async def verify_api_key(key: str = Security(api_key_header)) -> str:
    client = VALID_KEYS.get(key)
    if not client:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Invalid API key")
    return client

@app.get("/internal/metrics", dependencies=[Security(verify_api_key)])
async def internal_metrics():
    return {"requests_today": 42_000}
```

### Anti-Patterns

```python
# WRONG: storing raw passwords
user.password = form.password               # Never store plaintext

# WRONG: no expiration on tokens
jwt.encode({"sub": user_id}, key)           # No "exp" — lives forever

# WRONG: checking roles with string equality
if user.role == "admin":                     # Fails with multiple roles

# CORRECT: hash passwords, set exp claim, check with set intersection
```

---

## WebSockets

### Basic Endpoint

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
        pass
```

### Connection Manager with Broadcasting

```python
import asyncio
from fastapi import WebSocket

class ConnectionManager:
    def __init__(self):
        self._connections: dict[str, WebSocket] = {}
        self._lock = asyncio.Lock()

    async def connect(self, client_id: str, ws: WebSocket) -> None:
        await ws.accept()
        async with self._lock:
            self._connections[client_id] = ws

    async def disconnect(self, client_id: str) -> None:
        async with self._lock:
            self._connections.pop(client_id, None)

    async def broadcast(self, message: str, exclude: str | None = None) -> None:
        async with self._lock:
            targets = [ws for cid, ws in self._connections.items() if cid != exclude]
        await asyncio.gather(*(ws.send_text(message) for ws in targets), return_exceptions=True)

manager = ConnectionManager()
```

### Chat Room with Heartbeats

```python
import asyncio, json
from fastapi import WebSocket, WebSocketDisconnect, Query

HEARTBEAT_INTERVAL = 30

@app.websocket("/ws/chat/{room}")
async def chat(ws: WebSocket, room: str, user: str = Query(...)):
    cid = f"{room}:{user}"
    await manager.connect(cid, ws)
    await manager.broadcast(json.dumps({"event": "join", "user": user}), exclude=cid)

    async def heartbeat():
        try:
            while True:
                await asyncio.sleep(HEARTBEAT_INTERVAL)
                await ws.send_json({"event": "ping"})
        except Exception:
            pass

    hb_task = asyncio.create_task(heartbeat())
    try:
        while True:
            payload = json.loads(await ws.receive_text())
            payload["user"] = user
            await manager.broadcast(json.dumps(payload))
    except WebSocketDisconnect:
        hb_task.cancel()
        await manager.disconnect(cid)
        await manager.broadcast(json.dumps({"event": "leave", "user": user}))
```

### Client-Side Reconnection

```javascript
function createSocket(url, maxRetries = 10) {
  let retries = 0;
  function connect() {
    const ws = new WebSocket(url);
    ws.onopen = () => { retries = 0; };
    ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      if (data.event === "ping") { ws.send(JSON.stringify({event:"pong"})); return; }
      handleMessage(data);
    };
    ws.onclose = () => {
      if (retries < maxRetries) setTimeout(connect, Math.min(1000 * 2 ** retries++, 30000));
    };
    return ws;
  }
  return connect();
}
```

### Anti-Patterns

```python
# WRONG: no heartbeat — idle connections dropped by proxies
# WRONG: no try/except WebSocketDisconnect — ghost connections accumulate
# WRONG: sequential broadcast blocks on slow clients
for ws in connections:
    await ws.send_text(msg)            # Use asyncio.gather instead
```

---

## Testing and Deployment

### TestClient (Synchronous)

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client():
    with TestClient(app) as c:
        yield c

def test_health(client: TestClient):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.json() == {"status": "ok"}

def test_create_user(client: TestClient):
    resp = client.post("/api/v1/users", json={"email": "t@example.com", "name": "T", "password": "securepass123"})
    assert resp.status_code == 201
```

### Async Testing with httpx

```python
import pytest, httpx
from app.main import app

@pytest.fixture
async def async_client():
    async with httpx.AsyncClient(transport=httpx.ASGITransport(app=app), base_url="http://test") as ac:
        yield ac

@pytest.mark.anyio
async def test_list_items(async_client: httpx.AsyncClient):
    resp = await async_client.get("/api/v1/items")
    assert resp.status_code == 200
```

### Database Fixtures with Rollback

```python
import pytest, httpx
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from app.db.models import Base
from app.deps import get_db

@pytest.fixture(scope="session")
async def engine():
    eng = create_async_engine("postgresql+asyncpg://test:test@localhost/test_db")
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield eng
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await eng.dispose()

@pytest.fixture
async def db_session(engine):
    factory = async_sessionmaker(engine, expire_on_commit=False)
    async with factory() as session:
        async with session.begin():
            yield session
            await session.rollback()

@pytest.fixture
async def client(db_session):
    async def override():
        yield db_session
    app.dependency_overrides[get_db] = override
    async with httpx.AsyncClient(transport=httpx.ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

### Dockerfile

```dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .

FROM base AS production
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### Uvicorn Production Config

```python
# gunicorn_conf.py — run with: gunicorn app.main:app -c gunicorn_conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"
timeout = 120
keepalive = 5
accesslog = "-"
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["8000:8000"]
    command: uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
    environment:
      DATABASE_URL: postgresql+asyncpg://dev:dev@db:5432/devdb
    depends_on:
      db: { condition: service_healthy }
  db:
    image: postgres:16-alpine
    environment: { POSTGRES_USER: dev, POSTGRES_PASSWORD: dev, POSTGRES_DB: devdb }
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev"]
      interval: 5s
      timeout: 5s
      retries: 5
volumes:
  pgdata:
```

### Anti-Patterns

```python
# WRONG: --reload in production — file watcher overhead + instability
# WRONG: single worker — wastes multi-core capacity
# WRONG: no /health endpoint — orchestrators can't probe readiness

# CORRECT: gunicorn + uvicorn workers, no reload, health endpoint
@app.get("/health")
async def health():
    return {"status": "ok"}
```
