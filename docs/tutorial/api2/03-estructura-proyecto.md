# 3. Estructura del Proyecto

## Arquitectura General

Nuestro proyecto sigue el patrón de **arquitectura en capas**, donde cada capa tiene una responsabilidad específica y bien definida.

```
api_simple/
├── 📁 app/                    # Aplicación principal
│   ├── 📁 database/          # Configuración de base de datos
│   ├── 📁 models/            # Modelos de datos (SQLAlchemy)
│   ├── 📁 schemas/           # Esquemas de validación (Pydantic)
│   ├── 📁 crud/              # Operaciones CRUD
│   ├── 📁 routers/           # Rutas y endpoints
│   └── 📄 __init__.py        # Inicialización del paquete
├── 📁 docs/                  # Documentación del proyecto
├── 📄 main.py               # Punto de entrada de la aplicación
├── 📄 requirements.txt      # Dependencias del proyecto
└── 📄 .gitignore            # Archivos a ignorar en Git
```

## Explicación de Cada Capa

### 🗄️ Capa de Base de Datos (`database/`)

**Propósito**: Configurar la conexión y sesión de base de datos.

```python
# app/database/database.py
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# Configuración de la base de datos
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Función para obtener sesión de BD
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**Responsabilidades**:
- Configurar la conexión a la base de datos
- Crear el motor de SQLAlchemy
- Gestionar sesiones de base de datos
- Proporcionar la clase base para modelos

### 🏗️ Capa de Modelos (`models/`)

**Propósito**: Definir la estructura de las tablas de base de datos.

```python
# app/models/models.py
from sqlalchemy import Column, Integer, String, Float
from app.database.database import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)

class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    price = Column(Float)

class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer)
    product_id = Column(Integer)
    quantity = Column(Integer)
```

**Responsabilidades**:
- Definir estructura de tablas
- Especificar tipos de datos
- Establecer relaciones entre tablas
- Mapear objetos Python a tablas SQL

### ✅ Capa de Esquemas (`schemas/`)

**Propósito**: Validar y serializar datos de entrada y salida.

```python
# app/schemas/schemas.py
from pydantic import BaseModel

class User(BaseModel):
    id: int = None
    name: str
    email: str
    
    class Config:
        from_attributes = True

class Product(BaseModel):
    id: int = None
    name: str
    price: float
    
    class Config:
        from_attributes = True

class Order(BaseModel):
    id: int = None
    user_id: int
    product_id: int
    quantity: int
    
    class Config:
        from_attributes = True
```

**Responsabilidades**:
- Validar datos de entrada
- Serializar datos de salida
- Documentar la estructura de la API
- Convertir entre formatos (JSON ↔ Python)

### 🔧 Capa CRUD (`crud/`)

**Propósito**: Implementar operaciones de base de datos.

```python
# app/crud/crud.py
from sqlalchemy.orm import Session
from app.models import models
from app.schemas import schemas

def get_all(db: Session, model):
    return db.query(model).all()

def get_by_id(db: Session, model, item_id: int):
    return db.query(model).filter(model.id == item_id).first()

def create_item(db: Session, model, item_data):
    db_item = model(**item_data.dict(exclude={'id'}))
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    return db_item
```

**Responsabilidades**:
- Ejecutar consultas SQL
- Manejar transacciones
- Implementar lógica de negocio
- Abstraer operaciones de base de datos

### 🌐 Capa de Rutas (`routers/`)

**Propósito**: Definir endpoints y manejar peticiones HTTP.

```python
# app/routers/users.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session
from app.crud import crud
from app.schemas import schemas
from app.models import models
from app.database.database import get_db

router = APIRouter(prefix="/users")

@router.post("/")
def create_user(user: schemas.User, db: Session = Depends(get_db)):
    return crud.create_item(db, models.User, user)

@router.get("/")
def get_users(db: Session = Depends(get_db)):
    return crud.get_all(db, models.User)

@router.get("/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    return crud.get_by_id(db, models.User, user_id)
```

**Responsabilidades**:
- Definir rutas HTTP
- Manejar parámetros de petición
- Validar entrada
- Coordinar entre capas
- Retornar respuestas HTTP

### 🚀 Aplicación Principal (`main.py`)

**Propósito**: Inicializar y configurar la aplicación FastAPI.

```python
# main.py
from fastapi import FastAPI
from app.database.database import engine
from app.models import models
from app.routers import users, products, orders

# Crear tablas en la base de datos
models.Base.metadata.create_all(bind=engine)

# Crear aplicación FastAPI
app = FastAPI()

# Incluir routers
app.include_router(users.router)
app.include_router(products.router)
app.include_router(orders.router)

@app.get("/")
def root():
    return {"message": "API Simple"}
```

**Responsabilidades**:
- Inicializar FastAPI
- Configurar la aplicación
- Registrar routers
- Crear tablas de base de datos

## Flujo de Datos

Veamos cómo fluyen los datos a través de las capas:

```
1. 🌐 Cliente hace petición HTTP
   ↓
2. 🛣️ Router recibe y valida la petición
   ↓
3. ✅ Schema valida los datos de entrada
   ↓
4. 🔧 CRUD ejecuta operación en base de datos
   ↓
5. 🗄️ Model interactúa con la base de datos
   ↓
6. 🔧 CRUD retorna datos a router
   ↓
7. ✅ Schema serializa datos de salida
   ↓
8. 🛣️ Router envía respuesta HTTP
   ↓
9. 🌐 Cliente recibe respuesta
```

### Ejemplo Práctico: Crear un Usuario

1. **Cliente envía**: `POST /users` con JSON
2. **Router** (`users.py`) recibe la petición
3. **Schema** (`schemas.User`) valida los datos
4. **CRUD** (`crud.create_item`) crea el usuario
5. **Model** (`models.User`) define la estructura
6. **Database** guarda en SQLite
7. **CRUD** retorna el usuario creado
8. **Schema** serializa la respuesta
9. **Router** envía JSON al cliente

## Ventajas de Esta Arquitectura

### 🔄 Separación de Responsabilidades
- Cada capa tiene una función específica
- Fácil de entender y mantener
- Cambios en una capa no afectan otras

### 🧪 Testabilidad
- Cada capa se puede probar independientemente
- Fácil crear mocks y stubs
- Tests más rápidos y confiables

### 🔧 Mantenibilidad
- Código organizado y predecible
- Fácil encontrar y modificar funcionalidades
- Menos bugs por acoplamiento

### 📈 Escalabilidad
- Fácil agregar nuevas funcionalidades
- Reutilización de componentes
- Paralelización del desarrollo

## Patrones de Diseño Utilizados

### 1. **Repository Pattern** (CRUD)
- Abstrae el acceso a datos
- Facilita cambios de base de datos
- Mejora la testabilidad

### 2. **Dependency Injection** (FastAPI)
- Inyección de dependencias automática
- Fácil gestión de recursos
- Mejor control del ciclo de vida

### 3. **Data Transfer Object** (Schemas)
- Transferencia segura de datos
- Validación automática
- Documentación implícita

### 4. **Layered Architecture**
- Organización en capas
- Flujo unidireccional
- Bajo acoplamiento

## Convenciones de Nomenclatura

### Archivos y Carpetas
- **Carpetas**: snake_case (`app/`, `models/`)
- **Archivos Python**: snake_case (`models.py`, `crud.py`)
- **Clases**: PascalCase (`User`, `Product`)
- **Funciones**: snake_case (`get_user`, `create_item`)
- **Variables**: snake_case (`user_id`, `db_session`)

### Base de Datos
- **Tablas**: plural, snake_case (`users`, `products`)
- **Columnas**: snake_case (`user_id`, `created_at`)
- **Índices**: descriptivos (`idx_user_email`)

### API Endpoints
- **Rutas**: kebab-case si es necesario (`/users`, `/user-profiles`)
- **Parámetros**: snake_case (`user_id`, `page_size`)

## Configuración Adicional

### Variables de Entorno
Para configuraciones sensibles, usa un archivo `.env`:

```env
DATABASE_URL=sqlite:///./test.db
SECRET_KEY=your-secret-key
DEBUG=True
```

### Logging
Configura logging para debugging:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
```

### Configuración de CORS
Para aplicaciones web frontend:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Próximos Pasos

Ahora que entendemos la estructura, en los siguientes capítulos implementaremos cada capa paso a paso:

1. **Base de datos y modelos** - Configurar SQLAlchemy
2. **Esquemas** - Crear validaciones con Pydantic
3. **CRUD** - Implementar operaciones de base de datos
4. **Routers** - Crear endpoints HTTP
5. **Integración** - Conectar todo en main.py

---

**Siguiente:** [Base de Datos y Modelos](04-base-datos-modelos.md)

**Anterior:** [Configuración del Entorno](02-configuracion-entorno.md)