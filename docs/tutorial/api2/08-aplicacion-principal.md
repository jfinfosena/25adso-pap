# 8. Aplicación Principal

## ¿Qué es main.py?

El archivo `main.py` es el **punto de entrada** de nuestra aplicación FastAPI. Es donde:

- Se crea la instancia de FastAPI
- Se configuran los routers
- Se inicializa la base de datos
- Se configuran middleware y CORS
- Se definen endpoints globales

### Arquitectura de la Aplicación

```
main.py (Aplicación Principal)
    ↓
┌─────────────────────────────────────┐
│  FastAPI App                        │
├─────────────────────────────────────┤
│  • Configuración                    │
│  • Middleware                       │
│  • Routers                          │
│  • Manejo de errores                │
│  • Documentación                    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Routers                            │
├─────────────────────────────────────┤
│  • /users    (users.py)             │
│  • /products (products.py)          │
│  • /orders   (orders.py)            │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│  Base de Datos                      │
├─────────────────────────────────────┤
│  • Modelos SQLAlchemy               │
│  • Sesiones                         │
│  • Tablas                           │
└─────────────────────────────────────┘
```

## Implementación Básica

### Versión Simple (Nuestro Proyecto)

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

### Análisis Paso a Paso

#### 1. **Importaciones**

```python
from fastapi import FastAPI
from app.database.database import engine
from app.models import models
from app.routers import users, products, orders
```

- `FastAPI`: Clase principal de la aplicación
- `engine`: Motor de base de datos SQLAlchemy
- `models`: Modelos de datos (User, Product, Order)
- `routers`: Módulos con endpoints organizados

#### 2. **Creación de Tablas**

```python
models.Base.metadata.create_all(bind=engine)
```

**¿Qué hace esto?**
- Examina todos los modelos que heredan de `Base`
- Genera SQL CREATE TABLE para cada modelo
- Ejecuta las sentencias en la base de datos
- **Idempotente**: No falla si las tablas ya existen

**SQL Generado:**
```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER NOT NULL PRIMARY KEY,
    name VARCHAR,
    email VARCHAR
);

CREATE TABLE IF NOT EXISTS products (
    id INTEGER NOT NULL PRIMARY KEY,
    name VARCHAR,
    price FLOAT
);

CREATE TABLE IF NOT EXISTS orders (
    id INTEGER NOT NULL PRIMARY KEY,
    user_id INTEGER,
    product_id INTEGER,
    quantity INTEGER
);
```

#### 3. **Creación de la Aplicación**

```python
app = FastAPI()
```

Crea una instancia de FastAPI con configuración por defecto.

#### 4. **Registro de Routers**

```python
app.include_router(users.router)
app.include_router(products.router)
app.include_router(orders.router)
```

**¿Qué hace `include_router()`?**
- Registra todos los endpoints del router
- Combina prefijos de ruta
- Agrupa endpoints por tags
- Incluye en la documentación automática

**Resultado:**
```
Endpoints registrados:
GET    /users/
POST   /users/
GET    /users/{user_id}
GET    /products/
POST   /products/
GET    /products/{product_id}
GET    /orders/
POST   /orders/
GET    /orders/{order_id}
```

#### 5. **Endpoint Raíz**

```python
@app.get("/")
def root():
    return {"message": "API Simple"}
```

Endpoint básico para verificar que la API está funcionando.

## Configuración Avanzada

### Aplicación con Metadatos

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import logging

# Configurar logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Crear aplicación con metadatos
app = FastAPI(
    title="API Simple",
    description="""Una API REST simple para aprender FastAPI.
    
    Esta API permite gestionar:
    * **Usuarios**: Crear y consultar usuarios
    * **Productos**: Gestionar catálogo de productos
    * **Órdenes**: Procesar pedidos de usuarios
    """,
    version="1.0.0",
    terms_of_service="http://example.com/terms/",
    contact={
        "name": "Equipo de Desarrollo",
        "url": "http://example.com/contact/",
        "email": "dev@example.com",
    },
    license_info={
        "name": "MIT",
        "url": "https://opensource.org/licenses/MIT",
    },
    docs_url="/docs",      # URL de Swagger UI
    redoc_url="/redoc",    # URL de ReDoc
    openapi_url="/openapi.json"  # URL del esquema OpenAPI
)
```

### Configuración de CORS

```python
# Configurar CORS para aplicaciones web
origins = [
    "http://localhost",
    "http://localhost:3000",  # React dev server
    "http://localhost:8080",  # Vue dev server
    "https://myapp.com",      # Producción
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)
```

### Middleware Personalizado

```python
from fastapi import Request, Response
import time

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    """Agregar tiempo de procesamiento a las respuestas."""
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response

@app.middleware("http")
async def log_requests(request: Request, call_next):
    """Registrar todas las peticiones."""
    logger.info(f"Petición: {request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Respuesta: {response.status_code}")
    return response
```

## Manejo de Errores Global

### Exception Handlers

```python
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from sqlalchemy.exc import IntegrityError
from pydantic import ValidationError

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    """Manejar errores HTTP personalizados."""
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": "HTTP Error",
            "detail": exc.detail,
            "status_code": exc.status_code
        }
    )

@app.exception_handler(ValidationError)
async def validation_exception_handler(request: Request, exc: ValidationError):
    """Manejar errores de validación de Pydantic."""
    return JSONResponse(
        status_code=422,
        content={
            "error": "Validation Error",
            "detail": exc.errors(),
            "body": exc.body
        }
    )

@app.exception_handler(IntegrityError)
async def integrity_error_handler(request: Request, exc: IntegrityError):
    """Manejar errores de integridad de base de datos."""
    return JSONResponse(
        status_code=400,
        content={
            "error": "Database Integrity Error",
            "detail": "Los datos violan restricciones de la base de datos"
        }
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    """Manejar errores generales no capturados."""
    logger.error(f"Error no manejado: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal Server Error",
            "detail": "Ha ocurrido un error interno del servidor"
        }
    )
```

## Eventos de Ciclo de Vida

### Startup y Shutdown

```python
@app.on_event("startup")
async def startup_event():
    """Ejecutar al iniciar la aplicación."""
    logger.info("Iniciando aplicación...")
    
    # Crear tablas
    models.Base.metadata.create_all(bind=engine)
    logger.info("Tablas de base de datos creadas")
    
    # Inicializar datos de prueba (opcional)
    # await create_initial_data()
    
    logger.info("Aplicación iniciada correctamente")

@app.on_event("shutdown")
async def shutdown_event():
    """Ejecutar al cerrar la aplicación."""
    logger.info("Cerrando aplicación...")
    
    # Cerrar conexiones de base de datos
    engine.dispose()
    
    logger.info("Aplicación cerrada correctamente")
```

### Datos Iniciales

```python
from app.database.database import SessionLocal
from app.models.models import User, Product

async def create_initial_data():
    """Crear datos iniciales si no existen."""
    db = SessionLocal()
    try:
        # Verificar si ya hay datos
        user_count = db.query(User).count()
        if user_count == 0:
            # Crear usuarios de ejemplo
            users = [
                User(name="Admin", email="admin@example.com"),
                User(name="Usuario Demo", email="demo@example.com")
            ]
            db.add_all(users)
            
            # Crear productos de ejemplo
            products = [
                Product(name="Laptop", price=999.99),
                Product(name="Mouse", price=29.99),
                Product(name="Teclado", price=79.99)
            ]
            db.add_all(products)
            
            db.commit()
            logger.info("Datos iniciales creados")
        else:
            logger.info("Datos iniciales ya existen")
    except Exception as e:
        logger.error(f"Error creando datos iniciales: {e}")
        db.rollback()
    finally:
        db.close()
```

## Configuración por Entorno

### Variables de Entorno

```python
import os
from functools import lru_cache
from pydantic import BaseSettings

class Settings(BaseSettings):
    """Configuración de la aplicación."""
    app_name: str = "API Simple"
    debug: bool = False
    database_url: str = "sqlite:///./test.db"
    secret_key: str = "your-secret-key-here"
    
    class Config:
        env_file = ".env"

@lru_cache()
def get_settings():
    return Settings()

# Usar configuración
settings = get_settings()

app = FastAPI(
    title=settings.app_name,
    debug=settings.debug
)
```

### Archivo .env

```env
# .env
APP_NAME="API Simple - Desarrollo"
DEBUG=true
DATABASE_URL="sqlite:///./dev.db"
SECRET_KEY="dev-secret-key-123"
```

## Organización Modular

### Estructura Avanzada

```python
# main.py
from fastapi import FastAPI
from app.core.config import settings
from app.core.database import init_db
from app.api.api_v1.api import api_router
from app.core.middleware import setup_middleware
from app.core.exceptions import setup_exception_handlers

def create_application() -> FastAPI:
    """Factory para crear la aplicación FastAPI."""
    app = FastAPI(
        title=settings.PROJECT_NAME,
        version=settings.VERSION,
        description=settings.DESCRIPTION,
        openapi_url=f"{settings.API_V1_STR}/openapi.json"
    )
    
    # Configurar middleware
    setup_middleware(app)
    
    # Configurar manejo de errores
    setup_exception_handlers(app)
    
    # Incluir routers
    app.include_router(api_router, prefix=settings.API_V1_STR)
    
    return app

# Crear aplicación
app = create_application()

@app.on_event("startup")
async def startup():
    await init_db()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        reload=True
    )
```

## Documentación Automática

### Personalización de OpenAPI

```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="API Simple",
        version="1.0.0",
        description="API REST para gestión de usuarios, productos y órdenes",
        routes=app.routes,
    )
    
    # Personalizar esquema
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png"
    }
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

### Tags para Organización

```python
tags_metadata = [
    {
        "name": "users",
        "description": "Operaciones con usuarios. Permite crear y consultar usuarios.",
    },
    {
        "name": "products",
        "description": "Gestión de productos. Catálogo de productos disponibles.",
    },
    {
        "name": "orders",
        "description": "Procesamiento de órdenes. Gestión de pedidos de usuarios.",
    },
]

app = FastAPI(
    title="API Simple",
    description="API REST para e-commerce básico",
    version="1.0.0",
    openapi_tags=tags_metadata
)
```

## Testing de la Aplicación

### Test de Integración

```python
# test_main.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_read_main():
    """Test del endpoint raíz."""
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "API Simple"}

def test_docs_accessible():
    """Test que la documentación sea accesible."""
    response = client.get("/docs")
    assert response.status_code == 200

def test_openapi_schema():
    """Test del esquema OpenAPI."""
    response = client.get("/openapi.json")
    assert response.status_code == 200
    schema = response.json()
    assert "openapi" in schema
    assert "info" in schema
    assert "paths" in schema

def test_health_check():
    """Test de salud de la aplicación."""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "healthy"
```

### Endpoint de Salud

```python
@app.get("/health")
def health_check():
    """Endpoint para verificar el estado de la aplicación."""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "version": "1.0.0"
    }
```

## Deployment y Producción

### Configuración para Producción

```python
# main.py para producción
import os
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI(
    title="API Simple",
    version="1.0.0",
    docs_url=None if os.getenv("ENVIRONMENT") == "production" else "/docs",
    redoc_url=None if os.getenv("ENVIRONMENT") == "production" else "/redoc"
)

# Middleware de seguridad para producción
if os.getenv("ENVIRONMENT") == "production":
    app.add_middleware(
        TrustedHostMiddleware,
        allowed_hosts=["api.mycompany.com", "*.mycompany.com"]
    )

# CORS más restrictivo para producción
allowed_origins = [
    "https://myapp.com",
    "https://www.myapp.com"
] if os.getenv("ENVIRONMENT") == "production" else ["*"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)
```

### Comando de Inicio

```python
# Para desarrollo
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "main:app",
        host="127.0.0.1",
        port=8000,
        reload=True,
        log_level="info"
    )

# Para producción (usar gunicorn)
# gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
```

## Mejores Prácticas

### 1. **Separación de Configuración**

```python
# ✅ Bueno: Configuración separada
from app.core.config import settings

app = FastAPI(
    title=settings.APP_NAME,
    debug=settings.DEBUG
)

# ❌ Malo: Configuración hardcodeada
app = FastAPI(
    title="API Simple",
    debug=True
)
```

### 2. **Manejo de Errores Robusto**

```python
# ✅ Bueno: Manejo específico de errores
@app.exception_handler(ValidationError)
async def validation_error_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={"detail": exc.errors()}
    )

# ❌ Malo: Sin manejo de errores
# Los errores se propagan sin control
```

### 3. **Logging Apropiado**

```python
# ✅ Bueno: Logging estructurado
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)

@app.on_event("startup")
async def startup():
    logger.info("Aplicación iniciando...")

# ❌ Malo: Sin logging
# Difícil debuggear problemas
```

## Ejercicios Prácticos

### Ejercicio 1: Configuración Avanzada
Modifica `main.py` para incluir:
- Configuración por variables de entorno
- Middleware de logging
- Manejo de errores personalizado

### Ejercicio 2: Endpoint de Salud
Crea un endpoint `/health` que retorne:
- Estado de la aplicación
- Versión
- Timestamp
- Estado de la base de datos

### Ejercicio 3: Datos Iniciales
Implementa una función que cree datos de ejemplo:
- 5 usuarios
- 10 productos
- 15 órdenes

### Ejercicio 4: Testing Completo
Escribe tests para:
- Todos los endpoints principales
- Manejo de errores
- Configuración de la aplicación

## Próximos Pasos

En el siguiente capítulo aprenderemos sobre **Testing y Debugging**, donde:
- Escribiremos tests unitarios y de integración
- Configuraremos herramientas de debugging
- Implementaremos logging efectivo
- Aprenderemos a usar herramientas de profiling

---

**Siguiente:** [Testing y Debugging](09-testing-debugging.md)

**Anterior:** [Rutas y Endpoints](07-rutas-endpoints.md)