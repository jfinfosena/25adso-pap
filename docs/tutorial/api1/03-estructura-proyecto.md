# Estructura del Proyecto

## Introducción

Una estructura de proyecto bien organizada es fundamental para el mantenimiento, escalabilidad y colaboración en el desarrollo. En este tema aprenderemos cómo organizar un proyecto FastAPI siguiendo las mejores prácticas.

## Estructura recomendada

```
fastapi-inventory/
├── .venv/                    # Entorno virtual (no versionar)
├── app/                      # Código principal de la aplicación
│   ├── __init__.py          # Hace que app sea un paquete Python
│   ├── main.py              # Punto de entrada de la aplicación
│   ├── config.py            # Configuraciones globales
│   ├── database/            # Configuración de base de datos
│   │   ├── __init__.py
│   │   ├── base.py          # Clase base para modelos
│   │   ├── connection.py    # Configuración de conexión
│   │   └── database.py      # Funciones de base de datos
│   ├── models/              # Modelos SQLAlchemy (ORM)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── category.py
│   │   ├── item.py
│   │   └── loan.py
│   ├── schemas/             # Esquemas Pydantic (validación)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── category.py
│   │   ├── item.py
│   │   └── loan.py
│   ├── crud/                # Operaciones CRUD
│   │   ├── __init__.py
│   │   ├── users.py
│   │   ├── categories.py
│   │   ├── items.py
│   │   └── loans.py
│   ├── routers/             # Rutas de la API
│   │   ├── __init__.py
│   │   ├── users.py
│   │   ├── categories.py
│   │   ├── items.py
│   │   └── loans.py
│   └── utils/               # Utilidades y helpers
│       ├── __init__.py
│       ├── dependencies.py  # Dependencias comunes
│       └── exceptions.py    # Excepciones personalizadas
├── tests/                   # Pruebas unitarias e integración
│   ├── __init__.py
│   ├── test_users.py
│   ├── test_items.py
│   └── conftest.py
├── docs/                    # Documentación adicional
├── .env                     # Variables de entorno (no versionar)
├── .env.example             # Ejemplo de variables de entorno
├── .gitignore              # Archivos a ignorar en Git
├── requirements.txt        # Dependencias de producción
├── requirements-dev.txt    # Dependencias de desarrollo
└── README.md              # Documentación del proyecto
```

## Explicación de cada directorio

### 1. Directorio `app/`

Contiene todo el código de la aplicación:

#### `main.py` - Punto de entrada
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.database.connection import engine
from app.database.base import Base
from app.routers import users, items, categories, loans
from app.config import settings

# Crear tablas en la base de datos
Base.metadata.create_all(bind=engine)

# Crear instancia de FastAPI
app = FastAPI(
    title="Inventory API",
    description="API REST para gestión de inventario",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc"
)

# CORS Middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Incluir routers
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])
app.include_router(items.router, prefix="/api/v1/items", tags=["items"])
app.include_router(categories.router, prefix="/api/v1/categories", tags=["categories"])
app.include_router(loans.router, prefix="/api/v1/loans", tags=["loans"])

@app.get("/")
def read_root():
    return {
        "message": "Inventory API",
        "version": "1.0.0",
        "docs": "/docs"
    }
```

#### `config.py` - Configuraciones
```python
import os
from dotenv import load_dotenv

load_dotenv()

class Settings:
    # Base de datos
    DATABASE_URL: str = os.getenv("DATABASE_URL", "sqlite:///./inventory.db")
    
    # Configuración de la aplicación
    DEBUG: bool = os.getenv("DEBUG", "False").lower() == "true"
    SECRET_KEY: str = os.getenv("SECRET_KEY", "your-secret-key")
    
    # Configuración de CORS
    ALLOWED_ORIGINS: list = [
        "http://localhost:3000",
        "http://localhost:8080",
    ]
    
    # Configuración de paginación
    DEFAULT_PAGE_SIZE: int = 20
    MAX_PAGE_SIZE: int = 100

settings = Settings()
```

### 2. Directorio `database/`

Maneja toda la configuración de la base de datos:

#### `base.py` - Modelo base
```python
from sqlalchemy import Column, Integer, DateTime
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()

class BaseModel(Base):
    """Clase base para todos los modelos"""
    __abstract__ = True
    
    id = Column(Integer, primary_key=True, index=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

#### `connection.py` - Configuración de conexión
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.config import settings

# Crear engine de SQLAlchemy
engine = create_engine(
    settings.DATABASE_URL,
    connect_args={"check_same_thread": False}  # Solo para SQLite
)

# Crear SessionLocal
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

#### `database.py` - Funciones de base de datos
```python
from app.database.connection import SessionLocal

def get_db():
    """Dependencia para obtener sesión de base de datos"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### 3. Directorio `models/`

Contiene los modelos SQLAlchemy (representación de tablas):

```python
# models/user.py
from sqlalchemy import Column, String, Boolean
from sqlalchemy.orm import relationship
from app.database.base import BaseModel

class User(BaseModel):
    __tablename__ = "users"
    
    username = Column(String(50), unique=True, index=True, nullable=False)
    email = Column(String(255), unique=True, index=True, nullable=False)
    full_name = Column(String(100), nullable=True)
    is_active = Column(Boolean, default=True, nullable=False)
    
    # Relaciones
    loans = relationship("Loan", back_populates="user")
```

### 4. Directorio `schemas/`

Contiene los esquemas Pydantic (validación y serialización):

```python
# schemas/user.py
from pydantic import BaseModel, EmailStr
from typing import Optional
from datetime import datetime

class UserBase(BaseModel):
    username: str
    email: EmailStr
    full_name: Optional[str] = None
    is_active: bool = True

class UserCreate(UserBase):
    pass

class UserUpdate(BaseModel):
    username: Optional[str] = None
    email: Optional[EmailStr] = None
    full_name: Optional[str] = None
    is_active: Optional[bool] = None

class UserResponse(UserBase):
    id: int
    created_at: datetime
    updated_at: datetime
    
    class Config:
        from_attributes = True
```

### 5. Directorio `crud/`

Contiene las operaciones CRUD (Create, Read, Update, Delete):

```python
# crud/users.py
from sqlalchemy.orm import Session
from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate

def get_user(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

def get_users(db: Session, skip: int = 0, limit: int = 100):
    return db.query(User).offset(skip).limit(limit).all()

def create_user(db: Session, user: UserCreate):
    db_user = User(**user.dict())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

### 6. Directorio `routers/`

Contiene las rutas de la API:

```python
# routers/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from app.crud import users as crud_users
from app.schemas import user as user_schemas
from app.database.database import get_db

router = APIRouter()

@router.post("/", response_model=user_schemas.UserResponse, status_code=201)
def create_user(user: user_schemas.UserCreate, db: Session = Depends(get_db)):
    return crud_users.create_user(db=db, user=user)

@router.get("/", response_model=List[user_schemas.UserResponse])
def read_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    return crud_users.get_users(db, skip=skip, limit=limit)
```

### 7. Directorio `utils/`

Contiene utilidades y helpers:

```python
# utils/dependencies.py
from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session
from app.database.database import get_db

def get_database_session() -> Session:
    """Dependencia para obtener sesión de base de datos"""
    return Depends(get_db)
```

```python
# utils/exceptions.py
from fastapi import HTTPException, status

class InventoryException(HTTPException):
    """Excepción base para el sistema de inventario"""
    def __init__(self, status_code: int, detail: str):
        super().__init__(status_code, detail)

class UserNotFoundException(InventoryException):
    def __init__(self):
        super().__init__(status.HTTP_404_NOT_FOUND, "Usuario no encontrado")

class ItemNotFoundException(InventoryException):
    def __init__(self):
        super().__init__(status.HTTP_404_NOT_FOUND, "Artículo no encontrado")
```

## Archivos de configuración importantes

### `.env` - Variables de entorno
```env
# Base de datos
DATABASE_URL=sqlite:///./inventory.db

# Configuración de la aplicación
DEBUG=True
SECRET_KEY=your-super-secret-key-here

# Configuración de CORS
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8080
```

### `.gitignore` - Archivos a ignorar
```gitignore
# Entorno virtual
.venv/
venv/
env/

# Variables de entorno
.env

# Base de datos
*.db
*.sqlite
*.sqlite3

# Python
__pycache__/
*.py[cod]
*.pyo
*.pyd
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# Logs
*.log
logs/

# OS
.DS_Store
Thumbs.db
```

## Principios de organización

### 1. Separación de responsabilidades
- **Models**: Representan la estructura de datos
- **Schemas**: Validan entrada y salida
- **CRUD**: Operaciones de base de datos
- **Routers**: Lógica de endpoints
- **Utils**: Funciones auxiliares

### 2. Importaciones claras
```python
# ✅ Bueno: importaciones específicas
from app.models.user import User
from app.schemas.user import UserCreate, UserResponse
from app.crud.users import create_user, get_user

# ❌ Malo: importaciones genéricas
from app.models import *
from app.schemas import *
```

### 3. Nomenclatura consistente
- **Archivos**: snake_case (user.py, item_category.py)
- **Clases**: PascalCase (User, ItemCategory)
- **Funciones**: snake_case (get_user, create_item)
- **Variables**: snake_case (user_id, item_name)

### 4. Documentación en código
```python
def create_user(db: Session, user: UserCreate) -> User:
    """
    Crear un nuevo usuario en la base de datos.
    
    Args:
        db: Sesión de base de datos
        user: Datos del usuario a crear
        
    Returns:
        Usuario creado con ID asignado
        
    Raises:
        ValueError: Si el email ya existe
    """
    # Implementación...
```

## Ventajas de esta estructura

### 1. Mantenibilidad
- Código organizado y fácil de encontrar
- Cambios localizados en módulos específicos
- Fácil de entender para nuevos desarrolladores

### 2. Escalabilidad
- Fácil agregar nuevos modelos y endpoints
- Estructura modular permite crecimiento
- Separación clara de responsabilidades

### 3. Testabilidad
- Cada módulo se puede testear independientemente
- Mocking más fácil con dependencias claras
- Pruebas unitarias e integración separadas

### 4. Reutilización
- Funciones CRUD reutilizables
- Esquemas compartidos entre endpoints
- Utilidades comunes centralizadas

## Comandos para crear la estructura

```bash
# Crear directorios
mkdir -p app/{database,models,schemas,crud,routers,utils}
mkdir tests docs

# Crear archivos __init__.py
touch app/__init__.py
touch app/database/__init__.py
touch app/models/__init__.py
touch app/schemas/__init__.py
touch app/crud/__init__.py
touch app/routers/__init__.py
touch app/utils/__init__.py
touch tests/__init__.py

# Crear archivos principales
touch app/main.py
touch app/config.py
touch .env
touch .gitignore
touch requirements.txt
touch README.md
```

## Próximos pasos

En el siguiente tema aprenderemos sobre la configuración de la base de datos y SQLAlchemy, donde implementaremos los modelos que representarán nuestras tablas.

---

**💡 Tips importantes**:

1. **Mantén la estructura consistente** - facilita la navegación y mantenimiento
2. **Usa nombres descriptivos** - el código debe ser auto-documentado
3. **Separa responsabilidades** - cada archivo debe tener un propósito claro
4. **Documenta tu código** - especialmente funciones públicas y complejas

**🔗 Enlaces útiles**:
- [Guía de estructura de proyectos Python](https://docs.python-guide.org/writing/structure/)
- [FastAPI Project Structure](https://fastapi.tiangolo.com/tutorial/bigger-applications/)
- [PEP 8 - Style Guide for Python Code](https://www.python.org/dev/peps/pep-0008/)