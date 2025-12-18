# FastAPI Code Review Skill

## Overview

This skill provides code review guidelines for FastAPI applications, focusing on async patterns, Pydantic v2 models, dependency injection, and API best practices.

## Key Review Areas

### 1. Pydantic Models

**Check for proper model usage:**
- Use Pydantic v2 syntax and features
- Define proper validation rules
- Use appropriate field types
- Separate request/response models when needed

```python
# Good: Pydantic v2 model with validation
from pydantic import BaseModel, Field, field_validator, EmailStr
from typing import Annotated

class UserCreate(BaseModel):
    email: EmailStr
    username: Annotated[str, Field(min_length=3, max_length=50)]
    password: Annotated[str, Field(min_length=8)]

    @field_validator('username')
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v.lower()

class UserResponse(BaseModel):
    id: int
    email: EmailStr
    username: str

    model_config = {'from_attributes': True}

# Avoid: Using dict or Any types when structure is known
def bad_endpoint(data: dict):  # Loses validation benefits
    pass
```

### 2. Async Best Practices

**Verify async patterns:**
- Use async def for I/O-bound operations
- Avoid blocking calls in async functions
- Use proper async database drivers
- Consider background tasks for long operations

```python
# Good: Proper async usage
from sqlalchemy.ext.asyncio import AsyncSession

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

# Good: Background tasks for slow operations
from fastapi import BackgroundTasks

@app.post("/send-notification")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    background_tasks.add_task(send_email, email)
    return {"message": "Notification queued"}

# Avoid: Blocking calls in async context
@app.get("/bad")
async def bad_endpoint():
    time.sleep(5)  # Blocks the event loop!
    requests.get("...")  # Use httpx instead
```

### 3. Dependency Injection

**Review dependency patterns:**
- Use Depends for reusable dependencies
- Implement proper resource cleanup with yield
- Cache dependencies when appropriate
- Use sub-dependencies for composition

```python
# Good: Database session dependency with cleanup
from contextlib import asynccontextmanager

async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# Good: Authentication dependency
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: AsyncSession = Depends(get_db)
) -> User:
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await db.get(User, user_id)
    if user is None:
        raise credentials_exception
    return user

# Good: Composing dependencies
async def get_current_active_user(
    current_user: Annotated[User, Depends(get_current_user)]
) -> User:
    if not current_user.is_active:
        raise HTTPException(status_code=400, detail="Inactive user")
    return current_user
```

### 4. API Design

**Check API patterns:**
- Use proper HTTP methods and status codes
- Implement consistent error responses
- Version APIs when needed
- Use proper response models

```python
# Good: Well-designed endpoints
from fastapi import APIRouter, status

router = APIRouter(prefix="/api/v1/users", tags=["users"])

@router.get("/", response_model=list[UserResponse])
async def list_users(
    skip: int = 0,
    limit: Annotated[int, Field(le=100)] = 10,
    db: AsyncSession = Depends(get_db)
) -> list[User]:
    result = await db.execute(
        select(User).offset(skip).limit(limit)
    )
    return result.scalars().all()

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user: UserCreate,
    db: AsyncSession = Depends(get_db)
) -> User:
    db_user = User(**user.model_dump())
    db.add(db_user)
    await db.commit()
    await db.refresh(db_user)
    return db_user

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    await db.delete(user)
    await db.commit()
```

### 5. Error Handling

**Verify error handling:**
- Use HTTPException for API errors
- Implement custom exception handlers
- Return consistent error formats
- Don't expose internal errors to clients

```python
# Good: Custom exception handler
from fastapi import Request
from fastapi.responses import JSONResponse

class AppException(Exception):
    def __init__(self, status_code: int, detail: str, error_code: str):
        self.status_code = status_code
        self.detail = detail
        self.error_code = error_code

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.error_code,
            "detail": exc.detail,
            "path": str(request.url)
        }
    )

# Good: Validation error handler
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content={
            "error": "VALIDATION_ERROR",
            "detail": exc.errors()
        }
    )

# Avoid: Exposing internal errors
@app.get("/bad")
async def bad_endpoint():
    try:
        # ...
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))  # Exposes internals!
```

### 6. Security

**Review security patterns:**
- Implement proper authentication (OAuth2, JWT)
- Use CORS middleware correctly
- Validate and sanitize inputs
- Rate limit sensitive endpoints

```python
# Good: Security configuration
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.com"],  # Specific origins, not "*"
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

# Good: Rate limiting sensitive endpoints
@app.post("/auth/login")
@limiter.limit("5/minute")
async def login(request: Request, credentials: LoginCredentials):
    # ...

# Good: Input validation
@app.get("/search")
async def search(
    q: Annotated[str, Query(min_length=1, max_length=100)],
    page: Annotated[int, Query(ge=1, le=1000)] = 1
):
    # q is validated and safe to use
    pass
```

### 7. Testing

**Verify testing practices:**
- Use TestClient for endpoint testing
- Mock dependencies properly
- Test validation and error cases
- Use pytest fixtures for setup

```python
# Good: Comprehensive API tests
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession

@pytest.fixture
async def async_client(app):
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
async def auth_headers(async_client, test_user):
    response = await async_client.post(
        "/auth/login",
        json={"email": test_user.email, "password": "testpass"}
    )
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}

@pytest.mark.asyncio
async def test_create_user(async_client, auth_headers):
    response = await async_client.post(
        "/api/v1/users/",
        json={"email": "new@example.com", "username": "newuser", "password": "securepass"},
        headers=auth_headers
    )
    assert response.status_code == 201
    assert response.json()["email"] == "new@example.com"

@pytest.mark.asyncio
async def test_create_user_invalid_email(async_client, auth_headers):
    response = await async_client.post(
        "/api/v1/users/",
        json={"email": "invalid", "username": "test", "password": "testpass"},
        headers=auth_headers
    )
    assert response.status_code == 422
```

## Common Issues to Flag

1. **Blocking calls** in async functions (time.sleep, requests, file I/O)
2. **Missing response models** losing automatic documentation
3. **Improper dependency cleanup** (missing yield or try/finally)
4. **Overly permissive CORS** settings
5. **Missing input validation** on query/path parameters
6. **Exposing internal errors** to API consumers
7. **N+1 queries** from lazy loading in ORM
8. **Missing rate limiting** on authentication endpoints

## Review Checklist

- [ ] Pydantic models properly validate input
- [ ] Async functions don't contain blocking calls
- [ ] Dependencies clean up resources properly
- [ ] API responses use proper status codes
- [ ] Error responses are consistent and safe
- [ ] Authentication/authorization implemented correctly
- [ ] CORS configured appropriately
- [ ] Tests cover critical paths
