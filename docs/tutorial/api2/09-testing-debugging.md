# 9. Testing y Debugging

## ¿Por qué Testing?

El **testing** (pruebas) es fundamental para:

- **Confiabilidad**: Asegurar que el código funciona como se espera
- **Mantenimiento**: Detectar errores al modificar código
- **Documentación**: Los tests sirven como documentación viva
- **Refactoring**: Cambiar código con confianza
- **Calidad**: Mejorar la calidad general del software

### Tipos de Testing

```
Pirámide de Testing

    ┌─────────────────┐
    │   E2E Tests     │  ← Pocos, lentos, costosos
    │   (End-to-End) │
    ├─────────────────┤
    │ Integration     │  ← Algunos, moderados
    │ Tests           │
    ├─────────────────┤
    │ Unit Tests      │  ← Muchos, rápidos, baratos
    │                 │
    └─────────────────┘
```

## Configuración de Testing

### Instalación de Dependencias

```bash
# Instalar pytest y dependencias de testing
pip install pytest pytest-asyncio httpx

# Para coverage (cobertura de código)
pip install pytest-cov

# Para testing de base de datos
pip install pytest-mock
```

### Estructura de Archivos de Test

```
api_simple/
├── app/
│   ├── models/
│   ├── schemas/
│   ├── crud/
│   └── routers/
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Configuración de pytest
│   ├── test_main.py         # Tests de la aplicación principal
│   ├── test_models.py       # Tests de modelos
│   ├── test_schemas.py      # Tests de schemas
│   ├── test_crud.py         # Tests de CRUD
│   └── test_routers/
│       ├── __init__.py
│       ├── test_users.py    # Tests de endpoints de usuarios
│       ├── test_products.py # Tests de endpoints de productos
│       └── test_orders.py   # Tests de endpoints de órdenes
├── pytest.ini              # Configuración de pytest
└── requirements-test.txt    # Dependencias de testing
```

## Configuración de Pytest

### pytest.ini

```ini
# pytest.ini
[tool:pytest]
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
testpaths = tests
addopts = 
    -v
    --tb=short
    --strict-markers
    --disable-warnings
    --cov=app
    --cov-report=term-missing
    --cov-report=html
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

### conftest.py

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

from app.main import app
from app.database.database import get_db, Base
from app.models.models import User, Product, Order

# Base de datos en memoria para testing
SQLALCHEMY_DATABASE_URL = "sqlite:///:memory:"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={
        "check_same_thread": False,
    },
    poolclass=StaticPool,
)

TestingSessionLocal = sessionmaker(
    autocommit=False, autoflush=False, bind=engine
)

@pytest.fixture(scope="function")
def db_session():
    """Crear una sesión de base de datos para testing."""
    Base.metadata.create_all(bind=engine)
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def client(db_session):
    """Crear un cliente de testing con base de datos de prueba."""
    def override_get_db():
        try:
            yield db_session
        finally:
            db_session.close()
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as test_client:
        yield test_client
    
    app.dependency_overrides.clear()

@pytest.fixture
def sample_user_data():
    """Datos de ejemplo para usuario."""
    return {
        "name": "Juan Pérez",
        "email": "juan@example.com"
    }

@pytest.fixture
def sample_product_data():
    """Datos de ejemplo para producto."""
    return {
        "name": "Laptop Gaming",
        "price": 1299.99
    }

@pytest.fixture
def sample_order_data():
    """Datos de ejemplo para orden."""
    return {
        "user_id": 1,
        "product_id": 1,
        "quantity": 2
    }

@pytest.fixture
def create_sample_user(db_session):
    """Crear un usuario de ejemplo en la base de datos."""
    user = User(name="Test User", email="test@example.com")
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)
    return user

@pytest.fixture
def create_sample_product(db_session):
    """Crear un producto de ejemplo en la base de datos."""
    product = Product(name="Test Product", price=99.99)
    db_session.add(product)
    db_session.commit()
    db_session.refresh(product)
    return product
```

## Unit Tests

### Testing de Modelos

```python
# tests/test_models.py
import pytest
from app.models.models import User, Product, Order

class TestUserModel:
    """Tests para el modelo User."""
    
    def test_create_user(self, db_session):
        """Test crear usuario."""
        user = User(name="Test User", email="test@example.com")
        db_session.add(user)
        db_session.commit()
        
        assert user.id is not None
        assert user.name == "Test User"
        assert user.email == "test@example.com"
    
    def test_user_representation(self, db_session):
        """Test representación string del usuario."""
        user = User(name="Test User", email="test@example.com")
        db_session.add(user)
        db_session.commit()
        
        assert str(user) == "<User(name='Test User', email='test@example.com')>"
    
    def test_user_email_unique(self, db_session):
        """Test que el email sea único."""
        user1 = User(name="User 1", email="same@example.com")
        user2 = User(name="User 2", email="same@example.com")
        
        db_session.add(user1)
        db_session.commit()
        
        db_session.add(user2)
        
        with pytest.raises(Exception):  # IntegrityError
            db_session.commit()

class TestProductModel:
    """Tests para el modelo Product."""
    
    def test_create_product(self, db_session):
        """Test crear producto."""
        product = Product(name="Test Product", price=99.99)
        db_session.add(product)
        db_session.commit()
        
        assert product.id is not None
        assert product.name == "Test Product"
        assert product.price == 99.99
    
    def test_product_price_positive(self, db_session):
        """Test que el precio sea positivo."""
        # Esto dependería de validaciones en el modelo
        product = Product(name="Test Product", price=-10.0)
        db_session.add(product)
        
        # En un modelo más robusto, esto debería fallar
        # with pytest.raises(ValueError):
        #     db_session.commit()

class TestOrderModel:
    """Tests para el modelo Order."""
    
    def test_create_order(self, db_session, create_sample_user, create_sample_product):
        """Test crear orden."""
        user = create_sample_user
        product = create_sample_product
        
        order = Order(
            user_id=user.id,
            product_id=product.id,
            quantity=2
        )
        db_session.add(order)
        db_session.commit()
        
        assert order.id is not None
        assert order.user_id == user.id
        assert order.product_id == product.id
        assert order.quantity == 2
    
    def test_order_quantity_positive(self, db_session, create_sample_user, create_sample_product):
        """Test que la cantidad sea positiva."""
        user = create_sample_user
        product = create_sample_product
        
        order = Order(
            user_id=user.id,
            product_id=product.id,
            quantity=0  # Cantidad inválida
        )
        db_session.add(order)
        
        # En un modelo más robusto, esto debería fallar
        # with pytest.raises(ValueError):
        #     db_session.commit()
```

### Testing de Schemas

```python
# tests/test_schemas.py
import pytest
from pydantic import ValidationError
from app.schemas.schemas import User, Product, Order

class TestUserSchema:
    """Tests para el schema User."""
    
    def test_valid_user_schema(self):
        """Test schema válido de usuario."""
        user_data = {
            "name": "Juan Pérez",
            "email": "juan@example.com"
        }
        user = User(**user_data)
        
        assert user.name == "Juan Pérez"
        assert user.email == "juan@example.com"
    
    def test_user_schema_with_id(self):
        """Test schema de usuario con ID."""
        user_data = {
            "id": 1,
            "name": "Juan Pérez",
            "email": "juan@example.com"
        }
        user = User(**user_data)
        
        assert user.id == 1
        assert user.name == "Juan Pérez"
        assert user.email == "juan@example.com"
    
    def test_user_schema_missing_name(self):
        """Test schema de usuario sin nombre."""
        user_data = {
            "email": "juan@example.com"
        }
        
        with pytest.raises(ValidationError) as exc_info:
            User(**user_data)
        
        errors = exc_info.value.errors()
        assert len(errors) == 1
        assert errors[0]["loc"] == ("name",)
        assert errors[0]["type"] == "missing"
    
    def test_user_schema_invalid_email(self):
        """Test schema de usuario con email inválido."""
        user_data = {
            "name": "Juan Pérez",
            "email": "email-invalido"  # Email sin formato válido
        }
        
        # Nota: Esto pasaría con nuestro schema actual
        # Para validación de email real, necesitaríamos EmailStr
        user = User(**user_data)
        assert user.email == "email-invalido"

class TestProductSchema:
    """Tests para el schema Product."""
    
    def test_valid_product_schema(self):
        """Test schema válido de producto."""
        product_data = {
            "name": "Laptop Gaming",
            "price": 1299.99
        }
        product = Product(**product_data)
        
        assert product.name == "Laptop Gaming"
        assert product.price == 1299.99
    
    def test_product_schema_negative_price(self):
        """Test schema de producto con precio negativo."""
        product_data = {
            "name": "Laptop Gaming",
            "price": -100.0
        }
        
        # Con nuestro schema actual, esto pasaría
        # Para validación, necesitaríamos Field(gt=0)
        product = Product(**product_data)
        assert product.price == -100.0

class TestOrderSchema:
    """Tests para el schema Order."""
    
    def test_valid_order_schema(self):
        """Test schema válido de orden."""
        order_data = {
            "user_id": 1,
            "product_id": 1,
            "quantity": 2
        }
        order = Order(**order_data)
        
        assert order.user_id == 1
        assert order.product_id == 1
        assert order.quantity == 2
    
    def test_order_schema_missing_fields(self):
        """Test schema de orden con campos faltantes."""
        order_data = {
            "user_id": 1
            # Faltan product_id y quantity
        }
        
        with pytest.raises(ValidationError) as exc_info:
            Order(**order_data)
        
        errors = exc_info.value.errors()
        assert len(errors) == 2  # product_id y quantity
        
        error_fields = [error["loc"][0] for error in errors]
        assert "product_id" in error_fields
        assert "quantity" in error_fields
```

### Testing de CRUD

```python
# tests/test_crud.py
import pytest
from app.crud.crud import get_all, get_by_id, create_item
from app.models.models import User, Product, Order
from app.schemas.schemas import User as UserSchema, Product as ProductSchema

class TestCRUDOperations:
    """Tests para operaciones CRUD."""
    
    def test_create_user(self, db_session):
        """Test crear usuario via CRUD."""
        user_data = UserSchema(name="Test User", email="test@example.com")
        user = create_item(db_session, User, user_data)
        
        assert user.id is not None
        assert user.name == "Test User"
        assert user.email == "test@example.com"
    
    def test_get_all_users_empty(self, db_session):
        """Test obtener todos los usuarios cuando no hay ninguno."""
        users = get_all(db_session, User)
        assert users == []
    
    def test_get_all_users_with_data(self, db_session, create_sample_user):
        """Test obtener todos los usuarios con datos."""
        user = create_sample_user
        users = get_all(db_session, User)
        
        assert len(users) == 1
        assert users[0].id == user.id
        assert users[0].name == user.name
    
    def test_get_user_by_id_exists(self, db_session, create_sample_user):
        """Test obtener usuario por ID que existe."""
        user = create_sample_user
        found_user = get_by_id(db_session, User, user.id)
        
        assert found_user is not None
        assert found_user.id == user.id
        assert found_user.name == user.name
    
    def test_get_user_by_id_not_exists(self, db_session):
        """Test obtener usuario por ID que no existe."""
        user = get_by_id(db_session, User, 999)
        assert user is None
    
    def test_create_product(self, db_session):
        """Test crear producto via CRUD."""
        product_data = ProductSchema(name="Test Product", price=99.99)
        product = create_item(db_session, Product, product_data)
        
        assert product.id is not None
        assert product.name == "Test Product"
        assert product.price == 99.99
    
    def test_create_multiple_items(self, db_session):
        """Test crear múltiples elementos."""
        # Crear usuarios
        user1_data = UserSchema(name="User 1", email="user1@example.com")
        user2_data = UserSchema(name="User 2", email="user2@example.com")
        
        user1 = create_item(db_session, User, user1_data)
        user2 = create_item(db_session, User, user2_data)
        
        # Verificar que se crearon correctamente
        users = get_all(db_session, User)
        assert len(users) == 2
        
        user_names = [user.name for user in users]
        assert "User 1" in user_names
        assert "User 2" in user_names
```

## Integration Tests

### Testing de Endpoints

```python
# tests/test_routers/test_users.py
import pytest
from fastapi import status

class TestUserEndpoints:
    """Tests para endpoints de usuarios."""
    
    def test_create_user_success(self, client, sample_user_data):
        """Test crear usuario exitosamente."""
        response = client.post("/users/", json=sample_user_data)
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data["name"] == sample_user_data["name"]
        assert data["email"] == sample_user_data["email"]
        assert "id" in data
    
    def test_create_user_invalid_data(self, client):
        """Test crear usuario con datos inválidos."""
        invalid_data = {
            "name": ""  # Nombre vacío
            # Falta email
        }
        
        response = client.post("/users/", json=invalid_data)
        assert response.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
    
    def test_get_users_empty(self, client):
        """Test obtener usuarios cuando no hay ninguno."""
        response = client.get("/users/")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data == []
    
    def test_get_users_with_data(self, client, sample_user_data):
        """Test obtener usuarios con datos."""
        # Crear usuario primero
        create_response = client.post("/users/", json=sample_user_data)
        assert create_response.status_code == status.HTTP_200_OK
        
        # Obtener usuarios
        response = client.get("/users/")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert len(data) == 1
        assert data[0]["name"] == sample_user_data["name"]
    
    def test_get_user_by_id_exists(self, client, sample_user_data):
        """Test obtener usuario por ID que existe."""
        # Crear usuario primero
        create_response = client.post("/users/", json=sample_user_data)
        user_id = create_response.json()["id"]
        
        # Obtener usuario por ID
        response = client.get(f"/users/{user_id}")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data["id"] == user_id
        assert data["name"] == sample_user_data["name"]
    
    def test_get_user_by_id_not_exists(self, client):
        """Test obtener usuario por ID que no existe."""
        response = client.get("/users/999")
        
        # Dependiendo de la implementación, podría ser 404 o None
        # Con nuestra implementación actual, retorna None
        assert response.status_code == status.HTTP_200_OK
        assert response.json() is None
    
    def test_create_multiple_users(self, client):
        """Test crear múltiples usuarios."""
        users_data = [
            {"name": "User 1", "email": "user1@example.com"},
            {"name": "User 2", "email": "user2@example.com"},
            {"name": "User 3", "email": "user3@example.com"}
        ]
        
        created_users = []
        for user_data in users_data:
            response = client.post("/users/", json=user_data)
            assert response.status_code == status.HTTP_200_OK
            created_users.append(response.json())
        
        # Verificar que todos se crearon
        response = client.get("/users/")
        assert response.status_code == status.HTTP_200_OK
        
        all_users = response.json()
        assert len(all_users) == 3
        
        user_names = [user["name"] for user in all_users]
        assert "User 1" in user_names
        assert "User 2" in user_names
        assert "User 3" in user_names
```

### Testing de Productos

```python
# tests/test_routers/test_products.py
import pytest
from fastapi import status

class TestProductEndpoints:
    """Tests para endpoints de productos."""
    
    def test_create_product_success(self, client, sample_product_data):
        """Test crear producto exitosamente."""
        response = client.post("/products/", json=sample_product_data)
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data["name"] == sample_product_data["name"]
        assert data["price"] == sample_product_data["price"]
        assert "id" in data
    
    def test_create_product_invalid_price(self, client):
        """Test crear producto con precio inválido."""
        invalid_data = {
            "name": "Test Product",
            "price": "not-a-number"  # Precio inválido
        }
        
        response = client.post("/products/", json=invalid_data)
        assert response.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY
    
    def test_get_products_empty(self, client):
        """Test obtener productos cuando no hay ninguno."""
        response = client.get("/products/")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data == []
    
    def test_get_product_by_id_exists(self, client, sample_product_data):
        """Test obtener producto por ID que existe."""
        # Crear producto primero
        create_response = client.post("/products/", json=sample_product_data)
        product_id = create_response.json()["id"]
        
        # Obtener producto por ID
        response = client.get(f"/products/{product_id}")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data["id"] == product_id
        assert data["name"] == sample_product_data["name"]
        assert data["price"] == sample_product_data["price"]
```

### Testing de Órdenes

```python
# tests/test_routers/test_orders.py
import pytest
from fastapi import status

class TestOrderEndpoints:
    """Tests para endpoints de órdenes."""
    
    def test_create_order_success(self, client, sample_user_data, sample_product_data):
        """Test crear orden exitosamente."""
        # Crear usuario y producto primero
        user_response = client.post("/users/", json=sample_user_data)
        user_id = user_response.json()["id"]
        
        product_response = client.post("/products/", json=sample_product_data)
        product_id = product_response.json()["id"]
        
        # Crear orden
        order_data = {
            "user_id": user_id,
            "product_id": product_id,
            "quantity": 2
        }
        
        response = client.post("/orders/", json=order_data)
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data["user_id"] == user_id
        assert data["product_id"] == product_id
        assert data["quantity"] == 2
        assert "id" in data
    
    def test_create_order_invalid_user(self, client, sample_product_data):
        """Test crear orden con usuario inexistente."""
        # Crear solo producto
        product_response = client.post("/products/", json=sample_product_data)
        product_id = product_response.json()["id"]
        
        # Intentar crear orden con usuario inexistente
        order_data = {
            "user_id": 999,  # Usuario que no existe
            "product_id": product_id,
            "quantity": 2
        }
        
        response = client.post("/orders/", json=order_data)
        
        # Con nuestra implementación actual, esto pasaría
        # En una implementación más robusta, debería fallar
        assert response.status_code == status.HTTP_200_OK
    
    def test_get_orders_empty(self, client):
        """Test obtener órdenes cuando no hay ninguna."""
        response = client.get("/orders/")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data == []
```

## Testing de la Aplicación Principal

```python
# tests/test_main.py
import pytest
from fastapi import status
from fastapi.testclient import TestClient
from app.main import app

class TestMainApplication:
    """Tests para la aplicación principal."""
    
    def test_read_main(self, client):
        """Test del endpoint raíz."""
        response = client.get("/")
        
        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data == {"message": "API Simple"}
    
    def test_docs_accessible(self):
        """Test que la documentación sea accesible."""
        with TestClient(app) as client:
            response = client.get("/docs")
            assert response.status_code == status.HTTP_200_OK
    
    def test_openapi_schema(self):
        """Test del esquema OpenAPI."""
        with TestClient(app) as client:
            response = client.get("/openapi.json")
            assert response.status_code == status.HTTP_200_OK
            
            schema = response.json()
            assert "openapi" in schema
            assert "info" in schema
            assert "paths" in schema
            
            # Verificar que nuestros endpoints estén en el schema
            paths = schema["paths"]
            assert "/users/" in paths
            assert "/products/" in paths
            assert "/orders/" in paths
    
    def test_cors_headers(self, client):
        """Test headers CORS si están configurados."""
        response = client.options("/")
        # Esto dependería de la configuración CORS
        # assert "Access-Control-Allow-Origin" in response.headers
    
    def test_404_endpoint(self, client):
        """Test endpoint que no existe."""
        response = client.get("/nonexistent")
        assert response.status_code == status.HTTP_404_NOT_FOUND
```

## Debugging

### Configuración de Logging

```python
# app/core/logging.py
import logging
import sys
from pathlib import Path

def setup_logging(log_level: str = "INFO"):
    """Configurar logging para la aplicación."""
    
    # Crear directorio de logs
    log_dir = Path("logs")
    log_dir.mkdir(exist_ok=True)
    
    # Configurar formato
    log_format = (
        "%(asctime)s - %(name)s - %(levelname)s - "
        "%(filename)s:%(lineno)d - %(message)s"
    )
    
    # Configurar logging
    logging.basicConfig(
        level=getattr(logging, log_level.upper()),
        format=log_format,
        handlers=[
            logging.StreamHandler(sys.stdout),
            logging.FileHandler(log_dir / "app.log"),
            logging.FileHandler(log_dir / "error.log", level=logging.ERROR)
        ]
    )
    
    # Configurar loggers específicos
    logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)
    logging.getLogger("uvicorn").setLevel(logging.INFO)
    
    return logging.getLogger(__name__)
```

### Debugging con pdb

```python
# Agregar breakpoints en el código
import pdb

def create_item(db, model, item_data):
    pdb.set_trace()  # Breakpoint aquí
    
    item_dict = item_data.dict()
    db_item = model(**item_dict)
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    return db_item
```

### Debugging con VS Code

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "FastAPI Debug",
            "type": "python",
            "request": "launch",
            "program": "${workspaceFolder}/main.py",
            "console": "integratedTerminal",
            "env": {
                "PYTHONPATH": "${workspaceFolder}"
            },
            "args": []
        },
        {
            "name": "Pytest Debug",
            "type": "python",
            "request": "launch",
            "module": "pytest",
            "args": [
                "tests/",
                "-v",
                "--tb=short"
            ],
            "console": "integratedTerminal",
            "env": {
                "PYTHONPATH": "${workspaceFolder}"
            }
        }
    ]
}
```

### Logging en Endpoints

```python
# app/routers/users.py
import logging
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/users", tags=["users"])

@router.post("/")
def create_user(user: User, db: Session = Depends(get_db)):
    logger.info(f"Creando usuario: {user.name} - {user.email}")
    
    try:
        db_user = create_item(db, UserModel, user)
        logger.info(f"Usuario creado exitosamente con ID: {db_user.id}")
        return db_user
    except Exception as e:
        logger.error(f"Error creando usuario: {str(e)}", exc_info=True)
        raise

@router.get("/{user_id}")
def read_user(user_id: int, db: Session = Depends(get_db)):
    logger.debug(f"Buscando usuario con ID: {user_id}")
    
    user = get_by_id(db, UserModel, user_id)
    
    if user:
        logger.debug(f"Usuario encontrado: {user.name}")
    else:
        logger.warning(f"Usuario con ID {user_id} no encontrado")
    
    return user
```

## Herramientas de Profiling

### Memory Profiling

```python
# Instalar: pip install memory-profiler

# Decorador para profiling de memoria
from memory_profiler import profile

@profile
def create_many_users(db, count=1000):
    """Función para probar uso de memoria."""
    users = []
    for i in range(count):
        user_data = User(name=f"User {i}", email=f"user{i}@example.com")
        user = create_item(db, UserModel, user_data)
        users.append(user)
    return users

# Ejecutar: python -m memory_profiler script.py
```

### Performance Profiling

```python
# Instalar: pip install line-profiler

# Decorador para profiling de tiempo
from line_profiler import LineProfiler

@profile
def slow_function():
    """Función para analizar performance."""
    import time
    
    # Operación lenta
    time.sleep(0.1)
    
    # Operación de base de datos
    users = get_all(db, UserModel)
    
    # Procesamiento
    result = [user.name.upper() for user in users]
    
    return result

# Ejecutar: kernprof -l -v script.py
```

## Coverage (Cobertura de Código)

### Configuración de Coverage

```ini
# .coveragerc
[run]
source = app
omit = 
    app/tests/*
    app/__init__.py
    */venv/*
    */virtualenv/*
    */.tox/*

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    raise AssertionError
    raise NotImplementedError
    if __name__ == .__main__.:

[html]
directory = htmlcov
```

### Ejecutar Coverage

```bash
# Ejecutar tests con coverage
pytest --cov=app --cov-report=html --cov-report=term-missing

# Ver reporte en HTML
# Abrir htmlcov/index.html en el navegador

# Coverage mínimo requerido
pytest --cov=app --cov-fail-under=80
```

## Continuous Integration

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.8, 3.9, 3.10]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v3
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-test.txt
    
    - name: Run tests
      run: |
        pytest --cov=app --cov-report=xml --cov-fail-under=80
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella
```

## Mejores Prácticas

### 1. **Organización de Tests**

```python
# ✅ Bueno: Tests organizados por funcionalidad
class TestUserCRUD:
    def test_create_user(self):
        pass
    
    def test_get_user(self):
        pass
    
    def test_update_user(self):
        pass

# ❌ Malo: Tests mezclados sin organización
def test_user_stuff():
    # Muchas cosas mezcladas
    pass
```

### 2. **Nombres Descriptivos**

```python
# ✅ Bueno: Nombres descriptivos
def test_create_user_with_valid_email_returns_user_with_id():
    pass

def test_get_user_by_nonexistent_id_returns_none():
    pass

# ❌ Malo: Nombres poco descriptivos
def test_user1():
    pass

def test_user2():
    pass
```

### 3. **Fixtures Reutilizables**

```python
# ✅ Bueno: Fixtures específicas y reutilizables
@pytest.fixture
def user_with_orders(db_session):
    user = User(name="Test User", email="test@example.com")
    db_session.add(user)
    db_session.commit()
    
    # Agregar órdenes
    for i in range(3):
        order = Order(user_id=user.id, product_id=1, quantity=i+1)
        db_session.add(order)
    
    db_session.commit()
    return user

# ❌ Malo: Configuración repetida en cada test
def test_something(db_session):
    # Repetir configuración en cada test
    user = User(...)
    # ...
```

### 4. **Assertions Claras**

```python
# ✅ Bueno: Assertions específicas
assert response.status_code == 200
assert "id" in response.json()
assert response.json()["name"] == "Expected Name"

# ❌ Malo: Assertions vagas
assert response  # ¿Qué estamos verificando?
assert len(data) > 0  # ¿Cuántos esperamos exactamente?
```

## Ejercicios Prácticos

### Ejercicio 1: Tests Básicos
Escribe tests para:
- Crear un usuario con email duplicado (debería fallar)
- Crear un producto con precio cero
- Crear una orden con cantidad negativa

### Ejercicio 2: Tests de Integración
Crea un test que:
1. Cree un usuario
2. Cree un producto
3. Cree una orden
4. Verifique que todo esté relacionado correctamente

### Ejercicio 3: Mocking
Implementa tests que usen mocks para:
- Simular errores de base de datos
- Simular respuestas lentas
- Simular servicios externos

### Ejercicio 4: Performance Testing
Crea tests que verifiquen:
- Tiempo de respuesta de endpoints
- Uso de memoria con muchos registros
- Comportamiento bajo carga

## Próximos Pasos

En el siguiente capítulo aprenderemos sobre **Documentación y Deployment**, donde:
- Generaremos documentación automática
- Configuraremos OpenAPI/Swagger
- Prepararemos la aplicación para producción
- Implementaremos estrategias de deployment

---

**Siguiente:** [Documentación y Deployment](10-documentacion-deployment.md)

**Anterior:** [Aplicación Principal](08-aplicacion-principal.md)