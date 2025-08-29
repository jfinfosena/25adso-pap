# Middleware, Autenticación y Autorización

## Introducción

En este tema aprenderemos sobre tres conceptos fundamentales para crear APIs seguras y robustas:

- **Middleware**: Funciones que se ejecutan antes/después de cada request
- **Autenticación**: Verificar la identidad del usuario
- **Autorización**: Verificar los permisos del usuario

## ¿Qué es Middleware?

El middleware es código que se ejecuta antes y/o después de que tu endpoint procese una request. Es útil para:

- **Logging** de requests
- **Autenticación** y autorización
- **Manejo de CORS**
- **Rate limiting**
- **Compresión** de respuestas
- **Manejo de errores** globales

### Tipos de Middleware en FastAPI

1. **Middleware de aplicación**: Se aplica a toda la aplicación
2. **Middleware de router**: Se aplica a un router específico
3. **Dependencies**: Se aplican a endpoints específicos

## Implementación de Middleware

### 1. Middleware básico

```python
# app/middleware/logging.py
import time
import logging
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

# Configurar logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LoggingMiddleware(BaseHTTPMiddleware):
    """
    Middleware para logging de requests y responses.
    """
    
    async def dispatch(self, request: Request, call_next):
        # Información del request
        start_time = time.time()
        client_ip = request.client.host
        method = request.method
        url = str(request.url)
        
        # Log del request entrante
        logger.info(f"Request: {method} {url} from {client_ip}")
        
        # Procesar el request
        response = await call_next(request)
        
        # Calcular tiempo de procesamiento
        process_time = time.time() - start_time
        
        # Log del response
        logger.info(
            f"Response: {response.status_code} for {method} {url} "
            f"in {process_time:.4f}s"
        )
        
        # Agregar header con tiempo de procesamiento
        response.headers["X-Process-Time"] = str(process_time)
        
        return response
```

### 2. Middleware de CORS personalizado

```python
# app/middleware/cors.py
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response as StarletteResponse

class CustomCORSMiddleware(BaseHTTPMiddleware):
    """
    Middleware personalizado para manejo de CORS.
    """
    
    def __init__(self, app, allowed_origins: list = None):
        super().__init__(app)
        self.allowed_origins = allowed_origins or ["*"]
    
    async def dispatch(self, request: Request, call_next):
        # Obtener el origin del request
        origin = request.headers.get("origin")
        
        # Manejar preflight requests (OPTIONS)
        if request.method == "OPTIONS":
            response = StarletteResponse()
        else:
            response = await call_next(request)
        
        # Agregar headers CORS
        if origin and ("*" in self.allowed_origins or origin in self.allowed_origins):
            response.headers["Access-Control-Allow-Origin"] = origin
        
        response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
        response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
        response.headers["Access-Control-Allow-Credentials"] = "true"
        
        return response
```

### 3. Middleware de rate limiting

```python
# app/middleware/rate_limit.py
import time
from collections import defaultdict
from fastapi import Request, HTTPException, status
from starlette.middleware.base import BaseHTTPMiddleware

class RateLimitMiddleware(BaseHTTPMiddleware):
    """
    Middleware para limitar la cantidad de requests por IP.
    """
    
    def __init__(self, app, calls: int = 100, period: int = 60):
        super().__init__(app)
        self.calls = calls  # Número de llamadas permitidas
        self.period = period  # Período en segundos
        self.clients = defaultdict(list)  # Almacenar requests por IP
    
    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host
        now = time.time()
        
        # Limpiar requests antiguos
        self.clients[client_ip] = [
            timestamp for timestamp in self.clients[client_ip]
            if now - timestamp < self.period
        ]
        
        # Verificar límite
        if len(self.clients[client_ip]) >= self.calls:
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail="Rate limit exceeded"
            )
        
        # Registrar el request actual
        self.clients[client_ip].append(now)
        
        # Procesar request
        response = await call_next(request)
        
        # Agregar headers informativos
        response.headers["X-RateLimit-Limit"] = str(self.calls)
        response.headers["X-RateLimit-Remaining"] = str(
            self.calls - len(self.clients[client_ip])
        )
        response.headers["X-RateLimit-Reset"] = str(
            int(now + self.period)
        )
        
        return response
```

## Autenticación

### 1. Autenticación con JWT

Primero, instalemos las dependencias necesarias:

```bash
pip install python-jose[cryptography] passlib[bcrypt] python-multipart
```

#### Configuración de JWT

```python
# app/auth/jwt.py
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from app.config import settings

# Configuración de encriptación
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# Configuración JWT
SECRET_KEY = settings.SECRET_KEY
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    Verificar si una contraseña coincide con su hash.
    
    Args:
        plain_password: Contraseña en texto plano
        hashed_password: Contraseña hasheada
        
    Returns:
        True si coinciden, False si no
    """
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """
    Generar hash de una contraseña.
    
    Args:
        password: Contraseña en texto plano
        
    Returns:
        Hash de la contraseña
    """
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """
    Crear un token JWT de acceso.
    
    Args:
        data: Datos a incluir en el token
        expires_delta: Tiempo de expiración personalizado
        
    Returns:
        Token JWT como string
    """
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt

def verify_token(token: str) -> Optional[dict]:
    """
    Verificar y decodificar un token JWT.
    
    Args:
        token: Token JWT a verificar
        
    Returns:
        Payload del token si es válido, None si no
    """
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        return None

def get_user_from_token(token: str) -> Optional[str]:
    """
    Obtener el username del usuario desde un token.
    
    Args:
        token: Token JWT
        
    Returns:
        Username si el token es válido, None si no
    """
    payload = verify_token(token)
    if payload:
        return payload.get("sub")
    return None
```

#### Esquemas de autenticación

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr
from typing import Optional

class UserLogin(BaseModel):
    """
    Esquema para login de usuario.
    """
    username: str
    password: str

class Token(BaseModel):
    """
    Esquema para respuesta de token.
    """
    access_token: str
    token_type: str = "bearer"
    expires_in: int  # Segundos hasta expiración

class TokenData(BaseModel):
    """
    Esquema para datos del token.
    """
    username: Optional[str] = None

class UserRegister(BaseModel):
    """
    Esquema para registro de usuario.
    """
    username: str
    email: EmailStr
    full_name: str
    password: str
    phone: Optional[str] = None

class PasswordChange(BaseModel):
    """
    Esquema para cambio de contraseña.
    """
    current_password: str
    new_password: str

class PasswordReset(BaseModel):
    """
    Esquema para reset de contraseña.
    """
    email: EmailStr

class PasswordResetConfirm(BaseModel):
    """
    Esquema para confirmar reset de contraseña.
    """
    token: str
    new_password: str
```

#### Dependencias de autenticación

```python
# app/auth/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.orm import Session
from typing import Optional

from app.auth.jwt import get_user_from_token
from app.crud.users import user as crud_user
from app.database.database import get_db
from app.models.user import User

# Configurar esquema de seguridad
security = HTTPBearer()

def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
) -> User:
    """
    Obtener el usuario actual desde el token JWT.
    
    Args:
        credentials: Credenciales del header Authorization
        db: Sesión de base de datos
        
    Returns:
        Usuario autenticado
        
    Raises:
        HTTPException: Si el token es inválido o el usuario no existe
    """
    # Verificar que hay credenciales
    if not credentials:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token de acceso requerido",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # Obtener username del token
    username = get_user_from_token(credentials.credentials)
    if not username:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token inválido",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # Obtener usuario de la base de datos
    user = crud_user.get_by_username(db, username=username)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Usuario no encontrado",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # Verificar que el usuario está activo
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Usuario inactivo",
        )
    
    return user

def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """
    Obtener usuario activo (alias para claridad).
    
    Args:
        current_user: Usuario actual
        
    Returns:
        Usuario activo
    """
    return current_user

def get_optional_current_user(
    credentials: Optional[HTTPAuthorizationCredentials] = Depends(security),
    db: Session = Depends(get_db)
) -> Optional[User]:
    """
    Obtener usuario actual de forma opcional (para endpoints públicos).
    
    Args:
        credentials: Credenciales opcionales
        db: Sesión de base de datos
        
    Returns:
        Usuario si está autenticado, None si no
    """
    if not credentials:
        return None
    
    try:
        return get_current_user(credentials, db)
    except HTTPException:
        return None
```

#### Router de autenticación

```python
# app/routers/auth.py
from datetime import timedelta
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from app.auth.jwt import (
    verify_password, 
    create_access_token, 
    get_password_hash,
    ACCESS_TOKEN_EXPIRE_MINUTES
)
from app.auth.dependencies import get_current_user
from app.crud.users import user as crud_user
from app.database.database import get_db
from app.schemas.auth import (
    Token, 
    UserRegister, 
    PasswordChange,
    PasswordReset,
    PasswordResetConfirm
)
from app.schemas.user import UserResponse, UserCreate
from app.models.user import User

router = APIRouter(
    prefix="/auth",
    tags=["authentication"]
)

@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def register(
    user_data: UserRegister,
    db: Session = Depends(get_db)
):
    """
    Registrar un nuevo usuario.
    
    - **username**: Nombre de usuario único
    - **email**: Email único
    - **full_name**: Nombre completo
    - **password**: Contraseña (será hasheada)
    - **phone**: Teléfono (opcional)
    """
    # Verificar si el usuario ya existe
    if crud_user.get_by_username(db, username=user_data.username):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="El nombre de usuario ya está registrado"
        )
    
    if crud_user.get_by_email(db, email=user_data.email):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="El email ya está registrado"
        )
    
    # Hashear la contraseña
    hashed_password = get_password_hash(user_data.password)
    
    # Crear usuario
    user_create = UserCreate(
        username=user_data.username,
        email=user_data.email,
        full_name=user_data.full_name,
        phone=user_data.phone,
        hashed_password=hashed_password
    )
    
    return crud_user.create(db=db, obj_in=user_create)

@router.post("/login", response_model=Token)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """
    Iniciar sesión y obtener token de acceso.
    
    - **username**: Nombre de usuario o email
    - **password**: Contraseña del usuario
    """
    # Buscar usuario por username o email
    user = crud_user.get_by_username(db, username=form_data.username)
    if not user:
        user = crud_user.get_by_email(db, email=form_data.username)
    
    # Verificar usuario y contraseña
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Credenciales incorrectas",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    # Verificar que el usuario está activo
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Usuario inactivo"
        )
    
    # Crear token de acceso
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=access_token_expires
    )
    
    return {
        "access_token": access_token,
        "token_type": "bearer",
        "expires_in": ACCESS_TOKEN_EXPIRE_MINUTES * 60
    }

@router.get("/me", response_model=UserResponse)
def read_users_me(
    current_user: User = Depends(get_current_user)
):
    """
    Obtener información del usuario actual.
    """
    return current_user

@router.post("/change-password")
def change_password(
    password_data: PasswordChange,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    """
    Cambiar la contraseña del usuario actual.
    
    - **current_password**: Contraseña actual
    - **new_password**: Nueva contraseña
    """
    # Verificar contraseña actual
    if not verify_password(password_data.current_password, current_user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Contraseña actual incorrecta"
        )
    
    # Actualizar contraseña
    hashed_password = get_password_hash(password_data.new_password)
    current_user.hashed_password = hashed_password
    db.commit()
    
    return {"message": "Contraseña actualizada exitosamente"}

@router.post("/logout")
def logout(
    current_user: User = Depends(get_current_user)
):
    """
    Cerrar sesión (en una implementación real, invalidarías el token).
    """
    return {"message": "Sesión cerrada exitosamente"}
```

## Autorización

### 1. Sistema de roles y permisos

```python
# app/models/role.py
from sqlalchemy import Column, Integer, String, Boolean, Table, ForeignKey
from sqlalchemy.orm import relationship
from app.database.base import BaseModel

# Tabla de asociación para usuarios y roles
user_roles = Table(
    'user_roles',
    BaseModel.metadata,
    Column('user_id', Integer, ForeignKey('users.id'), primary_key=True),
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True)
)

# Tabla de asociación para roles y permisos
role_permissions = Table(
    'role_permissions',
    BaseModel.metadata,
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True),
    Column('permission_id', Integer, ForeignKey('permissions.id'), primary_key=True)
)

class Role(BaseModel):
    """
    Modelo para roles de usuario.
    """
    __tablename__ = "roles"
    
    name = Column(String(50), unique=True, nullable=False)
    description = Column(String(200))
    is_active = Column(Boolean, default=True)
    
    # Relaciones
    users = relationship("User", secondary=user_roles, back_populates="roles")
    permissions = relationship("Permission", secondary=role_permissions, back_populates="roles")

class Permission(BaseModel):
    """
    Modelo para permisos.
    """
    __tablename__ = "permissions"
    
    name = Column(String(50), unique=True, nullable=False)
    description = Column(String(200))
    resource = Column(String(50))  # ej: "users", "items", "loans"
    action = Column(String(50))    # ej: "create", "read", "update", "delete"
    
    # Relaciones
    roles = relationship("Role", secondary=role_permissions, back_populates="permissions")
```

### 2. Actualizar modelo de usuario

```python
# Agregar a app/models/user.py
from sqlalchemy.orm import relationship
from app.models.role import user_roles

class User(BaseModel):
    # ... campos existentes ...
    
    # Agregar relación con roles
    roles = relationship("Role", secondary=user_roles, back_populates="users")
    
    def has_permission(self, permission_name: str) -> bool:
        """
        Verificar si el usuario tiene un permiso específico.
        
        Args:
            permission_name: Nombre del permiso a verificar
            
        Returns:
            True si tiene el permiso, False si no
        """
        for role in self.roles:
            if role.is_active:
                for permission in role.permissions:
                    if permission.name == permission_name:
                        return True
        return False
    
    def has_role(self, role_name: str) -> bool:
        """
        Verificar si el usuario tiene un rol específico.
        
        Args:
            role_name: Nombre del rol a verificar
            
        Returns:
            True si tiene el rol, False si no
        """
        for role in self.roles:
            if role.is_active and role.name == role_name:
                return True
        return False
```

### 3. Dependencias de autorización

```python
# app/auth/permissions.py
from functools import wraps
from fastapi import Depends, HTTPException, status
from typing import List, Callable

from app.auth.dependencies import get_current_user
from app.models.user import User

def require_permissions(permissions: List[str]):
    """
    Decorator para requerir permisos específicos.
    
    Args:
        permissions: Lista de permisos requeridos
        
    Returns:
        Función de dependencia
    """
    def permission_checker(current_user: User = Depends(get_current_user)):
        for permission in permissions:
            if not current_user.has_permission(permission):
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail=f"Permiso requerido: {permission}"
                )
        return current_user
    
    return permission_checker

def require_roles(roles: List[str]):
    """
    Decorator para requerir roles específicos.
    
    Args:
        roles: Lista de roles requeridos (cualquiera de ellos)
        
    Returns:
        Función de dependencia
    """
    def role_checker(current_user: User = Depends(get_current_user)):
        has_required_role = any(
            current_user.has_role(role) for role in roles
        )
        
        if not has_required_role:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Rol requerido: {' o '.join(roles)}"
            )
        return current_user
    
    return role_checker

def require_admin():
    """
    Dependencia para requerir rol de administrador.
    
    Returns:
        Función de dependencia
    """
    return require_roles(["admin"])

def require_staff():
    """
    Dependencia para requerir rol de staff o admin.
    
    Returns:
        Función de dependencia
    """
    return require_roles(["admin", "staff"])

def can_modify_user(target_user_id: int):
    """
    Verificar si el usuario puede modificar a otro usuario.
    
    Args:
        target_user_id: ID del usuario a modificar
        
    Returns:
        Función de dependencia
    """
    def user_modifier_checker(current_user: User = Depends(get_current_user)):
        # Los admins pueden modificar cualquier usuario
        if current_user.has_role("admin"):
            return current_user
        
        # Los usuarios solo pueden modificarse a sí mismos
        if current_user.id == target_user_id:
            return current_user
        
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="No tienes permisos para modificar este usuario"
        )
    
    return user_modifier_checker
```

### 4. Uso en endpoints

```python
# Ejemplo de uso en routers
from app.auth.permissions import require_permissions, require_admin, require_staff

# Endpoint que requiere permiso específico
@router.post("/items/", response_model=ItemResponse)
def create_item(
    item: ItemCreate,
    db: Session = Depends(get_db),
    current_user: User = Depends(require_permissions(["items:create"]))
):
    """
    Crear artículo (requiere permiso items:create).
    """
    return crud_item.create_item(db=db, item=item)

# Endpoint que requiere rol de admin
@router.delete("/users/{user_id}")
def delete_user(
    user_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(require_admin())
):
    """
    Eliminar usuario (solo admins).
    """
    return crud_user.remove(db=db, id=user_id)

# Endpoint que requiere ser staff o admin
@router.get("/reports/loans")
def get_loans_report(
    db: Session = Depends(get_db),
    current_user: User = Depends(require_staff())
):
    """
    Obtener reporte de préstamos (staff o admin).
    """
    return {"message": "Reporte de préstamos"}

# Endpoint con autorización personalizada
@router.put("/users/{user_id}")
def update_user(
    user_id: int,
    user_update: UserUpdate,
    db: Session = Depends(get_db),
    current_user: User = Depends(can_modify_user(user_id))
):
    """
    Actualizar usuario (solo el mismo usuario o admin).
    """
    user = crud_user.get(db, id=user_id)
    return crud_user.update(db=db, db_obj=user, obj_in=user_update)
```

## Configuración en main.py

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.middleware.logging import LoggingMiddleware
from app.middleware.rate_limit import RateLimitMiddleware
from app.routers import users, categories, items, loans, auth
from app.config import settings

app = FastAPI(
    title="Sistema de Inventario API",
    description="API REST para gestión de inventario y préstamos",
    version="1.0.0"
)

# Agregar middleware (el orden importa)
app.add_middleware(LoggingMiddleware)
app.add_middleware(
    RateLimitMiddleware,
    calls=100,  # 100 requests
    period=60   # por minuto
)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_HOSTS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Incluir routers
app.include_router(auth.router, prefix="/api/v1")
app.include_router(users.router, prefix="/api/v1")
app.include_router(categories.router, prefix="/api/v1")
app.include_router(items.router, prefix="/api/v1")
app.include_router(loans.router, prefix="/api/v1")

@app.get("/")
def read_root():
    return {"message": "Sistema de Inventario API con Autenticación"}
```

## Mejores prácticas de seguridad

### 1. Configuración segura

```python
# app/config.py
import secrets
from pydantic import BaseSettings

class Settings(BaseSettings):
    # Generar clave secreta segura
    SECRET_KEY: str = secrets.token_urlsafe(32)
    
    # Configuración de base de datos
    DATABASE_URL: str = "sqlite:///./inventory.db"
    
    # Configuración CORS
    ALLOWED_HOSTS: list = ["http://localhost:3000"]
    
    # Configuración JWT
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    
    # Rate limiting
    RATE_LIMIT_CALLS: int = 100
    RATE_LIMIT_PERIOD: int = 60
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### 2. Validación de entrada

```python
# Siempre validar entrada del usuario
from pydantic import validator

class UserRegister(BaseModel):
    username: str
    password: str
    
    @validator('username')
    def username_must_be_alphanumeric(cls, v):
        if not v.isalnum():
            raise ValueError('Username debe ser alfanumérico')
        if len(v) < 3:
            raise ValueError('Username debe tener al menos 3 caracteres')
        return v
    
    @validator('password')
    def password_must_be_strong(cls, v):
        if len(v) < 8:
            raise ValueError('Password debe tener al menos 8 caracteres')
        if not any(c.isupper() for c in v):
            raise ValueError('Password debe tener al menos una mayúscula')
        if not any(c.islower() for c in v):
            raise ValueError('Password debe tener al menos una minúscula')
        if not any(c.isdigit() for c in v):
            raise ValueError('Password debe tener al menos un número')
        return v
```

### 3. Logging de seguridad

```python
# app/middleware/security_logging.py
import logging
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

security_logger = logging.getLogger("security")

class SecurityLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Log intentos de autenticación
        if request.url.path.startswith("/api/v1/auth/"):
            security_logger.info(
                f"Auth attempt: {request.method} {request.url.path} "
                f"from {request.client.host}"
            )
        
        response = await call_next(request)
        
        # Log fallos de autenticación
        if response.status_code == 401:
            security_logger.warning(
                f"Auth failed: {request.method} {request.url.path} "
                f"from {request.client.host}"
            )
        
        # Log accesos denegados
        if response.status_code == 403:
            security_logger.warning(
                f"Access denied: {request.method} {request.url.path} "
                f"from {request.client.host}"
            )
        
        return response
```

## Próximos pasos

En el siguiente tema aprenderemos sobre testing y cómo escribir pruebas para nuestra API.

---

**💡 Tips importantes**:

1. **Nunca hardcodear secretos** - usar variables de entorno
2. **Hashear contraseñas** - nunca almacenar en texto plano
3. **Validar entrada** - siempre validar datos del usuario
4. **Principio de menor privilegio** - dar solo permisos necesarios
5. **Logging de seguridad** - registrar eventos importantes

**🔗 Enlaces útiles**:
- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [JWT.io](https://jwt.io/)
- [OWASP API Security](https://owasp.org/www-project-api-security/)
- [Python Cryptography](https://cryptography.io/)