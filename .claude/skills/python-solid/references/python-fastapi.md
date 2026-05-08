# FastAPI — Patterns und Best Practices

## Router-Struktur

```python
# presentation/api/user_routes.py
from fastapi import APIRouter, Depends, HTTPException, status

router = APIRouter(prefix="/api/users", tags=["users"])

@router.get("/", response_model=list[UserResponseDto])
async def get_users(service: IUserService = Depends(get_user_service)):
    return [UserResponseDto.from_domain(u) for u in await service.get_all()]

@router.get("/{user_id}", response_model=UserResponseDto)
async def get_user(user_id: str, service: IUserService = Depends(get_user_service)):
    user = await service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Nicht gefunden")
    return UserResponseDto.from_domain(user)

@router.post("/", response_model=UserResponseDto, status_code=status.HTTP_201_CREATED)
async def create_user(data: CreateUserDto, service: IUserService = Depends(get_user_service)):
    user = await service.create(data)
    return UserResponseDto.from_domain(user)

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: str, service: IUserService = Depends(get_user_service)):
    await service.delete(user_id)
```

## App-Setup

```python
# main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from presentation.api import user_routes, auth_routes

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield  # Startup/Shutdown hier

app = FastAPI(title="API", lifespan=lifespan)
app.include_router(user_routes.router)
app.include_router(auth_routes.router)
```

## Exception Handler

```python
# main.py — globale Fehlerbehandlung
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"detail": str(exc)}
    )

@app.exception_handler(PermissionError)
async def permission_error_handler(request: Request, exc: PermissionError):
    return JSONResponse(
        status_code=status.HTTP_403_FORBIDDEN,
        content={"detail": "Keine Berechtigung"}
    )
```

## Response Models

```python
# application/dtos/user_dto.py
from pydantic import BaseModel
from datetime import datetime

class UserResponseDto(BaseModel):
    id: str
    email: str
    role: str
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)

    @classmethod
    def from_domain(cls, user: User) -> "UserResponseDto":
        return cls(
            id=str(user.id),
            email=str(user.email),
            role=user.role,
            created_at=user.created_at
        )
```

## SSE (Server-Sent Events)

```python
# Für Echtzeit-Streams (z.B. ipNINX Dashboard)
from fastapi.responses import StreamingResponse
import asyncio

@router.get("/events")
async def event_stream():
    async def generate():
        while True:
            data = await get_latest_data()
            yield f"data: {data.model_dump_json()}\n\n"
            await asyncio.sleep(1)

    return StreamingResponse(generate(), media_type="text/event-stream")
```

## Background Tasks

```python
from fastapi import BackgroundTasks

@router.post("/users")
async def create_user(
    data: CreateUserDto,
    background_tasks: BackgroundTasks,
    service: IUserService = Depends(get_user_service)
):
    user = await service.create(data)
    background_tasks.add_task(send_welcome_email, user.email)
    return UserResponseDto.from_domain(user)
```

## CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://192.168.2.242"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Pagination Pattern

```python
from pydantic import BaseModel

class PaginationParams(BaseModel):
    page: int = 1
    size: int = 20

@router.get("/", response_model=list[UserResponseDto])
async def get_users(
    pagination: PaginationParams = Depends(),
    service: IUserService = Depends(get_user_service)
):
    return await service.get_paginated(pagination.page, pagination.size)
```
