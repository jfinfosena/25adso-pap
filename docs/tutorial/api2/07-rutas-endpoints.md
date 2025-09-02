# 7. Rutas y Endpoints

## ¿Qué son las Rutas y Endpoints?

### Definiciones

- **Ruta**: El camino URL que identifica un recurso (`/users`, `/products/{id}`)
- **Endpoint**: La combinación de una ruta + método HTTP (`GET /users`, `POST /products`)
- **Router**: Agrupador de endpoints relacionados

### Anatomía de un Endpoint

```
MÉTODO + RUTA + FUNCIÓN = ENDPOINT

POST + /users + create_user() = Endpoint para crear usuarios
GET + /users/{id} + get_user() = Endpoint para obtener un usuario
```

### Métodos HTTP y su Propósito

| Método | Propósito | Ejemplo | Idempotente* |
|--------|-----------|---------|-------------|
| `GET` | Obtener datos | `GET /users` | ✅ |
| `POST` | Crear recursos | `POST /users` | ❌ |
| `PUT` | Actualizar completo | `PUT /users/1` | ✅ |
| `PATCH` | Actualizar parcial | `PATCH /users/1` | ❌ |
| `DELETE` | Eliminar | `DELETE /users/1` | ✅ |

*Idempotente: Múltiples llamadas producen el mismo resultado

## Estructura de Routers en FastAPI

### APIRouter

FastAPI usa `APIRouter` para organizar endpoints:

```python
from fastapi import APIRouter

# Crear router con prefijo
router = APIRouter(
    prefix="/users",           # Todas las rutas empezarán con /users
    tags=["users"],           # Agrupación en documentación
    responses={404: {"description": "Not found"}}
)

@router.get("/")              # Ruta completa: GET /users/
def get_users():
    pass

@router.get("/{user_id}")     # Ruta completa: GET /users/{user_id}
def get_user(user_id: int):
    pass
```

### Organización por Recursos

Cada recurso (User, Product, Order) tiene su propio router:

```
app/routers/
├── users.py      # Endpoints de usuarios
├── products.py   # Endpoints de productos
└── orders.py     # Endpoints de órdenes
```

## Implementación de Routers

### Router de Usuarios

Veamos `app/routers/users.py` en detalle:

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from app.crud import crud
from app.schemas import schemas
from app.models import models
from app.database.database import get_db

# Crear router
router = APIRouter(
    prefix="/users",
    tags=["users"]
)

@router.post("/", response_model=schemas.User)
def create_user(user: schemas.User, db: Session = Depends(get_db)):
    """Crear un nuevo usuario."""
    return crud.create_item(db, models.User, user)

@router.get("/", response_model=List[schemas.User])
def get_users(db: Session = Depends(get_db)):
    """Obtener todos los usuarios."""
    return crud.get_all(db, models.User)

@router.get("/{user_id}", response_model=schemas.User)
def get_user(user_id: int, db: Session = Depends(get_db)):
    """Obtener un usuario por ID."""
    user = crud.get_by_id(db, models.User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Usuario no encontrado")
    return user
```

### Análisis Detallado

#### 1. **Decoradores de Ruta**

```python
@router.post("/", response_model=schemas.User)
```

- `@router.post`: Define método HTTP POST
- `"/"`: Ruta relativa (se combina con prefix)
- `response_model`: Esquema para la respuesta

#### 2. **Parámetros de Función**

```python
def create_user(user: schemas.User, db: Session = Depends(get_db)):
```

- `user: schemas.User`: Cuerpo de la petición (JSON → Pydantic)
- `db: Session = Depends(get_db)`: Inyección de dependencia

#### 3. **Dependency Injection**

```python
db: Session = Depends(get_db)
```

FastAPI automáticamente:
1. Llama a `get_db()`
2. Obtiene una sesión de base de datos
3. La pasa como parámetro `db`
4. Cierra la sesión al terminar

#### 4. **Response Model**

```python
response_model=schemas.User
```

- Valida que la respuesta cumpla el esquema
- Genera documentación automática
- Serializa el objeto a JSON
- Excluye campos no definidos en el esquema

## Tipos de Parámetros

### 1. Path Parameters (Parámetros de Ruta)

```python
@router.get("/users/{user_id}")
def get_user(user_id: int):
    """user_id se extrae de la URL."""
    pass

# Ejemplos de URLs:
# GET /users/1     → user_id = 1
# GET /users/123   → user_id = 123
```

#### Validación de Path Parameters

```python
from fastapi import Path

@router.get("/users/{user_id}")
def get_user(
    user_id: int = Path(..., gt=0, description="ID del usuario")
):
    """user_id debe ser mayor a 0."""
    pass
```

### 2. Query Parameters (Parámetros de Consulta)

```python
@router.get("/users/")
def get_users(
    skip: int = 0,
    limit: int = 10,
    name: str = None,
    db: Session = Depends(get_db)
):
    """Parámetros opcionales en la URL."""
    # Implementar filtros y paginación
    pass

# Ejemplos de URLs:
# GET /users/                           → skip=0, limit=10, name=None
# GET /users/?skip=20&limit=5          → skip=20, limit=5, name=None
# GET /users/?name=Juan&limit=100      → skip=0, limit=100, name="Juan"
```

#### Validación de Query Parameters

```python
from fastapi import Query

@router.get("/users/")
def get_users(
    skip: int = Query(0, ge=0, description="Registros a saltar"),
    limit: int = Query(10, ge=1, le=100, description="Máximo registros"),
    name: str = Query(None, min_length=2, description="Filtrar por nombre")
):
    pass
```

### 3. Request Body (Cuerpo de Petición)

```python
@router.post("/users/")
def create_user(user: schemas.UserCreate):
    """user se extrae del JSON del cuerpo."""
    pass

# Ejemplo de petición:
# POST /users/
# Content-Type: application/json
# {
#   "name": "Juan Pérez",
#   "email": "juan@example.com"
# }
```

### 4. Combinando Parámetros

```python
@router.put("/users/{user_id}")
def update_user(
    user_id: int,                    # Path parameter
    user: schemas.UserUpdate,        # Request body
    force: bool = False,             # Query parameter
    db: Session = Depends(get_db)    # Dependency
):
    """Actualizar usuario con múltiples tipos de parámetros."""
    pass

# Ejemplo de petición:
# PUT /users/123?force=true
# {
#   "name": "Juan Carlos"
# }
```

## Manejo de Respuestas HTTP

### Códigos de Estado

```python
from fastapi import status

@router.post("/users/", status_code=status.HTTP_201_CREATED)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    """Retorna 201 Created en lugar de 200 OK."""
    return crud.create_item(db, models.User, user)

@router.delete("/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    """Retorna 204 No Content (sin cuerpo de respuesta)."""
    # Lógica de eliminación
    pass
```

### Respuestas de Error

```python
from fastapi import HTTPException

@router.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = crud.get_by_id(db, models.User, user_id)
    
    if not user:
        raise HTTPException(
            status_code=404,
            detail="Usuario no encontrado"
        )
    
    return user
```

### Múltiples Respuestas Posibles

```python
@router.get(
    "/users/{user_id}",
    response_model=schemas.User,
    responses={
        404: {"description": "Usuario no encontrado"},
        400: {"description": "ID de usuario inválido"}
    }
)
def get_user(user_id: int, db: Session = Depends(get_db)):
    if user_id <= 0:
        raise HTTPException(400, "ID debe ser positivo")
    
    user = crud.get_by_id(db, models.User, user_id)
    if not user:
        raise HTTPException(404, "Usuario no encontrado")
    
    return user
```

## Endpoints Completos por Recurso

### Usuarios (CRUD Completo)

```python
# app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List, Optional

from app.crud import crud
from app.schemas import schemas
from app.models import models
from app.database.database import get_db

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=schemas.User, status_code=status.HTTP_201_CREATED)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    """Crear un nuevo usuario."""
    # Verificar si el email ya existe
    existing_user = db.query(models.User).filter(
        models.User.email == user.email
    ).first()
    
    if existing_user:
        raise HTTPException(
            status_code=400,
            detail="Email ya está registrado"
        )
    
    return crud.create_item(db, models.User, user)

@router.get("/", response_model=List[schemas.User])
def get_users(
    skip: int = 0,
    limit: int = 100,
    name: Optional[str] = None,
    db: Session = Depends(get_db)
):
    """Obtener lista de usuarios con filtros opcionales."""
    query = db.query(models.User)
    
    if name:
        query = query.filter(models.User.name.like(f"%{name}%"))
    
    users = query.offset(skip).limit(limit).all()
    return users

@router.get("/{user_id}", response_model=schemas.User)
def get_user(user_id: int, db: Session = Depends(get_db)):
    """Obtener un usuario por ID."""
    user = crud.get_by_id(db, models.User, user_id)
    
    if not user:
        raise HTTPException(
            status_code=404,
            detail=f"Usuario con ID {user_id} no encontrado"
        )
    
    return user

@router.put("/{user_id}", response_model=schemas.User)
def update_user(
    user_id: int,
    user_update: schemas.UserUpdate,
    db: Session = Depends(get_db)
):
    """Actualizar un usuario existente."""
    user = crud.get_by_id(db, models.User, user_id)
    
    if not user:
        raise HTTPException(404, "Usuario no encontrado")
    
    # Actualizar campos
    update_data = user_update.dict(exclude_unset=True)
    for field, value in update_data.items():
        setattr(user, field, value)
    
    db.commit()
    db.refresh(user)
    return user

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    """Eliminar un usuario."""
    user = crud.get_by_id(db, models.User, user_id)
    
    if not user:
        raise HTTPException(404, "Usuario no encontrado")
    
    db.delete(user)
    db.commit()
```

### Productos con Búsqueda Avanzada

```python
# app/routers/products.py
from fastapi import APIRouter, Depends, Query
from typing import List, Optional

router = APIRouter(prefix="/products", tags=["products"])

@router.get("/", response_model=List[schemas.Product])
def get_products(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    name: Optional[str] = Query(None, min_length=2),
    min_price: Optional[float] = Query(None, ge=0),
    max_price: Optional[float] = Query(None, ge=0),
    sort_by: str = Query("name", regex="^(name|price|id)$"),
    order: str = Query("asc", regex="^(asc|desc)$"),
    db: Session = Depends(get_db)
):
    """Buscar productos con filtros avanzados."""
    query = db.query(models.Product)
    
    # Aplicar filtros
    if name:
        query = query.filter(models.Product.name.like(f"%{name}%"))
    if min_price is not None:
        query = query.filter(models.Product.price >= min_price)
    if max_price is not None:
        query = query.filter(models.Product.price <= max_price)
    
    # Aplicar ordenamiento
    if sort_by == "name":
        column = models.Product.name
    elif sort_by == "price":
        column = models.Product.price
    else:
        column = models.Product.id
    
    if order == "desc":
        query = query.order_by(column.desc())
    else:
        query = query.order_by(column.asc())
    
    # Aplicar paginación
    products = query.offset(skip).limit(limit).all()
    return products

@router.get("/search", response_model=List[schemas.Product])
def search_products(
    q: str = Query(..., min_length=2, description="Término de búsqueda"),
    db: Session = Depends(get_db)
):
    """Búsqueda de texto completo en productos."""
    products = db.query(models.Product).filter(
        models.Product.name.like(f"%{q}%")
    ).all()
    
    return products
```

### Órdenes con Validaciones de Negocio

```python
# app/routers/orders.py
from fastapi import APIRouter, Depends, HTTPException

router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("/", response_model=schemas.Order, status_code=201)
def create_order(order: schemas.OrderCreate, db: Session = Depends(get_db)):
    """Crear una nueva orden con validaciones."""
    # Validar que el usuario existe
    user = crud.get_by_id(db, models.User, order.user_id)
    if not user:
        raise HTTPException(404, "Usuario no encontrado")
    
    # Validar que el producto existe
    product = crud.get_by_id(db, models.Product, order.product_id)
    if not product:
        raise HTTPException(404, "Producto no encontrado")
    
    # Validar cantidad
    if order.quantity <= 0:
        raise HTTPException(400, "La cantidad debe ser positiva")
    
    return crud.create_item(db, models.Order, order)

@router.get("/user/{user_id}", response_model=List[schemas.Order])
def get_user_orders(user_id: int, db: Session = Depends(get_db)):
    """Obtener todas las órdenes de un usuario."""
    # Verificar que el usuario existe
    user = crud.get_by_id(db, models.User, user_id)
    if not user:
        raise HTTPException(404, "Usuario no encontrado")
    
    orders = db.query(models.Order).filter(
        models.Order.user_id == user_id
    ).all()
    
    return orders
```

## Middleware y Hooks

### Middleware de Logging

```python
from fastapi import Request
import time
import logging

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    
    # Procesar petición
    response = await call_next(request)
    
    # Calcular tiempo de procesamiento
    process_time = time.time() - start_time
    
    logger.info(
        f"{request.method} {request.url} - "
        f"Status: {response.status_code} - "
        f"Time: {process_time:.4f}s"
    )
    
    return response
```

### Manejo Global de Errores

```python
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
from sqlalchemy.exc import IntegrityError

@app.exception_handler(IntegrityError)
async def integrity_error_handler(request: Request, exc: IntegrityError):
    return JSONResponse(
        status_code=400,
        content={"detail": "Error de integridad en la base de datos"}
    )

@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=400,
        content={"detail": str(exc)}
    )
```

## Documentación Automática

### Configuración de OpenAPI

```python
from fastapi import FastAPI

app = FastAPI(
    title="API Simple",
    description="Una API REST simple para aprender FastAPI",
    version="1.0.0",
    docs_url="/docs",      # Swagger UI
    redoc_url="/redoc",    # ReDoc
    openapi_url="/openapi.json"
)
```

### Metadatos de Endpoints

```python
@router.post(
    "/users/",
    response_model=schemas.User,
    status_code=201,
    summary="Crear usuario",
    description="Crear un nuevo usuario en el sistema",
    response_description="Usuario creado exitosamente",
    tags=["users"],
    responses={
        400: {"description": "Email ya registrado"},
        422: {"description": "Datos de entrada inválidos"}
    }
)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    """Crear un nuevo usuario.
    
    - **name**: Nombre completo del usuario
    - **email**: Dirección de correo electrónico única
    """
    pass
```

## Testing de Endpoints

### Test con TestClient

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_user():
    response = client.post(
        "/users/",
        json={"name": "Test User", "email": "test@example.com"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test User"
    assert data["email"] == "test@example.com"
    assert "id" in data

def test_get_user():
    # Crear usuario primero
    create_response = client.post(
        "/users/",
        json={"name": "Test User", "email": "test@example.com"}
    )
    user_id = create_response.json()["id"]
    
    # Obtener usuario
    response = client.get(f"/users/{user_id}")
    assert response.status_code == 200
    data = response.json()
    assert data["id"] == user_id

def test_get_nonexistent_user():
    response = client.get("/users/999")
    assert response.status_code == 404
    assert "no encontrado" in response.json()["detail"]
```

## Mejores Prácticas

### 1. **Consistencia en URLs**

```python
# ✅ Bueno: URLs consistentes
/users/          # GET (lista), POST (crear)
/users/{id}      # GET (obtener), PUT (actualizar), DELETE (eliminar)
/users/{id}/orders  # GET órdenes del usuario

# ❌ Malo: URLs inconsistentes
/getUsers/
/user/create
/deleteUser/{id}
```

### 2. **Códigos de Estado Apropiados**

```python
# ✅ Bueno
@router.post("/users/", status_code=201)  # Created
@router.get("/users/")                    # 200 OK (por defecto)
@router.put("/users/{id}")                # 200 OK
@router.delete("/users/{id}", status_code=204)  # No Content

# ❌ Malo: Siempre 200
@router.post("/users/")  # Debería ser 201
@router.delete("/users/{id}")  # Debería ser 204
```

### 3. **Validación Completa**

```python
@router.get("/users/{user_id}")
def get_user(
    user_id: int = Path(..., gt=0, description="ID del usuario"),
    db: Session = Depends(get_db)
):
    """Validar que user_id sea positivo."""
    pass
```

### 4. **Manejo de Errores Descriptivo**

```python
# ✅ Bueno: Errores descriptivos
if not user:
    raise HTTPException(
        status_code=404,
        detail=f"Usuario con ID {user_id} no encontrado"
    )

# ❌ Malo: Errores genéricos
if not user:
    raise HTTPException(404, "Not found")
```

## Ejercicios Prácticos

### Ejercicio 1: Endpoint de Estadísticas
Crea un endpoint `GET /stats` que retorne:
- Total de usuarios
- Total de productos
- Total de órdenes
- Producto más vendido

### Ejercicio 2: Búsqueda Avanzada
Implementa `GET /search` que permita buscar en:
- Usuarios por nombre o email
- Productos por nombre
- Órdenes por ID de usuario

### Ejercicio 3: Validaciones de Negocio
Agrega validaciones para:
- No permitir crear órdenes de productos inexistentes
- Verificar que el email sea único al crear usuarios
- Validar rangos de precio en productos

### Ejercicio 4: Paginación Completa
Implementa paginación que incluya:
- Número total de registros
- Número total de páginas
- Enlaces a página anterior/siguiente

## Próximos Pasos

En el siguiente capítulo aprenderemos sobre la **Aplicación Principal**, donde:
- Configuraremos la aplicación FastAPI
- Registraremos todos los routers
- Configuraremos middleware
- Manejaremos la inicialización de la base de datos

---

**Siguiente:** [Aplicación Principal](08-aplicacion-principal.md)

**Anterior:** [Operaciones CRUD](06-operaciones-crud.md)