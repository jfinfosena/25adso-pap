# Testing y Pruebas

## Introducción

El testing es una parte fundamental del desarrollo de software que nos permite:

- **Verificar** que nuestro código funciona correctamente
- **Prevenir** errores en producción
- **Facilitar** refactoring y mantenimiento
- **Documentar** el comportamiento esperado
- **Aumentar** la confianza en nuestro código

## Tipos de pruebas

### 1. Pruebas unitarias
- Prueban **funciones individuales** o métodos
- Son **rápidas** y **aisladas**
- No dependen de recursos externos

### 2. Pruebas de integración
- Prueban **interacciones** entre componentes
- Incluyen base de datos, APIs externas, etc.
- Son más **lentas** pero más **realistas**

### 3. Pruebas end-to-end (E2E)
- Prueban **flujos completos** de usuario
- Simulan **comportamiento real**
- Son las más **lentas** pero más **completas**

## Configuración del entorno de testing

### 1. Instalación de dependencias

```bash
pip install pytest pytest-asyncio httpx pytest-cov
```

### 2. Estructura de directorios

```
proyecto/
├── app/
│   ├── main.py
│   ├── models/
│   ├── routers/
│   └── ...
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Configuración compartida
│   ├── test_main.py         # Pruebas de la aplicación principal
│   ├── unit/                # Pruebas unitarias
│   │   ├── __init__.py
│   │   ├── test_crud.py
│   │   ├── test_auth.py
│   │   └── test_utils.py
│   └── integration/         # Pruebas de integración
│       ├── __init__.py
│       ├── test_users.py
│       ├── test_items.py
│       └── test_loans.py
├── pytest.ini              # Configuración de pytest
└── requirements-test.txt    # Dependencias de testing
```

### 3. Configuración de pytest

```ini
# pytest.ini
[tool:pytest]
minversion = 6.0
addopts = 
    -ra
    -q
    --strict-markers
    --strict-config
    --cov=app
    --cov-report=term-missing
    --cov-report=html
    --cov-fail-under=80
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    auth: marks tests related to authentication
```

## Configuración de fixtures

### 1. Configuración base (conftest.py)

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

from app.main import app
from app.database.base import BaseModel
from app.database.database import get_db
from app.auth.jwt import create_access_token
from app.models.user import User
from app.models.category import Category
from app.models.item import Item
from app.models.loan import Loan

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
    autocommit=False, 
    autoflush=False, 
    bind=engine
)

@pytest.fixture(scope="function")
def db_session():
    """
    Crear una sesión de base de datos para testing.
    """
    # Crear todas las tablas
    BaseModel.metadata.create_all(bind=engine)
    
    # Crear sesión
    session = TestingSessionLocal()
    
    try:
        yield session
    finally:
        session.close()
        # Limpiar todas las tablas
        BaseModel.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def client(db_session):
    """
    Crear un cliente de testing con base de datos de prueba.
    """
    def override_get_db():
        try:
            yield db_session
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as test_client:
        yield test_client
    
    # Limpiar overrides
    app.dependency_overrides.clear()

@pytest.fixture
def sample_user_data():
    """
    Datos de ejemplo para crear usuarios.
    """
    return {
        "username": "testuser",
        "email": "test@example.com",
        "full_name": "Test User",
        "phone": "1234567890",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW"  # secret
    }

@pytest.fixture
def sample_category_data():
    """
    Datos de ejemplo para crear categorías.
    """
    return {
        "name": "Electrónicos",
        "description": "Dispositivos electrónicos"
    }

@pytest.fixture
def sample_item_data():
    """
    Datos de ejemplo para crear artículos.
    """
    return {
        "name": "Laptop Dell",
        "description": "Laptop para desarrollo",
        "serial_number": "DL123456",
        "category_id": 1
    }

@pytest.fixture
def create_user(db_session, sample_user_data):
    """
    Crear un usuario de prueba en la base de datos.
    """
    user = User(**sample_user_data)
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)
    return user

@pytest.fixture
def create_category(db_session, sample_category_data):
    """
    Crear una categoría de prueba en la base de datos.
    """
    category = Category(**sample_category_data)
    db_session.add(category)
    db_session.commit()
    db_session.refresh(category)
    return category

@pytest.fixture
def create_item(db_session, sample_item_data, create_category):
    """
    Crear un artículo de prueba en la base de datos.
    """
    item_data = sample_item_data.copy()
    item_data["category_id"] = create_category.id
    
    item = Item(**item_data)
    db_session.add(item)
    db_session.commit()
    db_session.refresh(item)
    return item

@pytest.fixture
def auth_headers(create_user):
    """
    Headers de autenticación para requests.
    """
    access_token = create_access_token(data={"sub": create_user.username})
    return {"Authorization": f"Bearer {access_token}"}

@pytest.fixture
def admin_user(db_session):
    """
    Crear un usuario administrador.
    """
    admin_data = {
        "username": "admin",
        "email": "admin@example.com",
        "full_name": "Admin User",
        "hashed_password": "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW",
        "is_superuser": True
    }
    
    user = User(**admin_data)
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)
    return user

@pytest.fixture
def admin_headers(admin_user):
    """
    Headers de autenticación para admin.
    """
    access_token = create_access_token(data={"sub": admin_user.username})
    return {"Authorization": f"Bearer {access_token}"}
```

## Pruebas unitarias

### 1. Testing de operaciones CRUD

```python
# tests/unit/test_crud.py
import pytest
from sqlalchemy.orm import Session

from app.crud.users import user as crud_user
from app.crud.categories import category as crud_category
from app.crud.items import item as crud_item
from app.schemas.user import UserCreate, UserUpdate
from app.schemas.category import CategoryCreate, CategoryUpdate
from app.schemas.item import ItemCreate, ItemUpdate

class TestUserCRUD:
    """
    Pruebas para operaciones CRUD de usuarios.
    """
    
    def test_create_user(self, db_session: Session):
        """
        Probar creación de usuario.
        """
        user_data = UserCreate(
            username="newuser",
            email="newuser@example.com",
            full_name="New User",
            hashed_password="hashedpassword"
        )
        
        user = crud_user.create(db_session, obj_in=user_data)
        
        assert user.username == "newuser"
        assert user.email == "newuser@example.com"
        assert user.full_name == "New User"
        assert user.id is not None
        assert user.created_at is not None
    
    def test_get_user_by_id(self, db_session: Session, create_user):
        """
        Probar obtención de usuario por ID.
        """
        user = crud_user.get(db_session, id=create_user.id)
        
        assert user is not None
        assert user.id == create_user.id
        assert user.username == create_user.username
    
    def test_get_user_by_username(self, db_session: Session, create_user):
        """
        Probar obtención de usuario por username.
        """
        user = crud_user.get_by_username(db_session, username=create_user.username)
        
        assert user is not None
        assert user.username == create_user.username
    
    def test_get_user_by_email(self, db_session: Session, create_user):
        """
        Probar obtención de usuario por email.
        """
        user = crud_user.get_by_email(db_session, email=create_user.email)
        
        assert user is not None
        assert user.email == create_user.email
    
    def test_update_user(self, db_session: Session, create_user):
        """
        Probar actualización de usuario.
        """
        update_data = UserUpdate(
            full_name="Updated Name",
            phone="9876543210"
        )
        
        updated_user = crud_user.update(
            db_session, 
            db_obj=create_user, 
            obj_in=update_data
        )
        
        assert updated_user.full_name == "Updated Name"
        assert updated_user.phone == "9876543210"
        assert updated_user.username == create_user.username  # No cambió
    
    def test_delete_user(self, db_session: Session, create_user):
        """
        Probar eliminación de usuario.
        """
        user_id = create_user.id
        
        deleted_user = crud_user.remove(db_session, id=user_id)
        
        assert deleted_user.id == user_id
        
        # Verificar que ya no existe
        user = crud_user.get(db_session, id=user_id)
        assert user is None
    
    def test_get_multi_users(self, db_session: Session):
        """
        Probar obtención de múltiples usuarios.
        """
        # Crear varios usuarios
        for i in range(3):
            user_data = UserCreate(
                username=f"user{i}",
                email=f"user{i}@example.com",
                full_name=f"User {i}",
                hashed_password="hashedpassword"
            )
            crud_user.create(db_session, obj_in=user_data)
        
        users = crud_user.get_multi(db_session, skip=0, limit=10)
        
        assert len(users) == 3
        assert all(user.username.startswith("user") for user in users)

class TestCategoryCRUD:
    """
    Pruebas para operaciones CRUD de categorías.
    """
    
    def test_create_category(self, db_session: Session):
        """
        Probar creación de categoría.
        """
        category_data = CategoryCreate(
            name="Test Category",
            description="Test Description"
        )
        
        category = crud_category.create(db_session, obj_in=category_data)
        
        assert category.name == "Test Category"
        assert category.description == "Test Description"
        assert category.id is not None
    
    def test_get_category_by_name(self, db_session: Session, create_category):
        """
        Probar obtención de categoría por nombre.
        """
        category = crud_category.get_by_name(db_session, name=create_category.name)
        
        assert category is not None
        assert category.name == create_category.name

class TestItemCRUD:
    """
    Pruebas para operaciones CRUD de artículos.
    """
    
    def test_create_item(self, db_session: Session, create_category):
        """
        Probar creación de artículo.
        """
        item_data = ItemCreate(
            name="Test Item",
            description="Test Description",
            serial_number="TEST123",
            category_id=create_category.id
        )
        
        item = crud_item.create(db_session, obj_in=item_data)
        
        assert item.name == "Test Item"
        assert item.serial_number == "TEST123"
        assert item.category_id == create_category.id
    
    def test_get_item_by_serial(self, db_session: Session, create_item):
        """
        Probar obtención de artículo por número de serie.
        """
        item = crud_item.get_by_serial_number(
            db_session, 
            serial_number=create_item.serial_number
        )
        
        assert item is not None
        assert item.serial_number == create_item.serial_number
    
    def test_get_items_by_category(self, db_session: Session, create_category):
        """
        Probar obtención de artículos por categoría.
        """
        # Crear varios artículos en la misma categoría
        for i in range(3):
            item_data = ItemCreate(
                name=f"Item {i}",
                description=f"Description {i}",
                serial_number=f"SERIAL{i}",
                category_id=create_category.id
            )
            crud_item.create(db_session, obj_in=item_data)
        
        items = crud_item.get_by_category(db_session, category_id=create_category.id)
        
        assert len(items) == 3
        assert all(item.category_id == create_category.id for item in items)
```

### 2. Testing de autenticación

```python
# tests/unit/test_auth.py
import pytest
from datetime import datetime, timedelta

from app.auth.jwt import (
    create_access_token,
    verify_token,
    get_user_from_token,
    verify_password,
    get_password_hash
)

class TestJWTFunctions:
    """
    Pruebas para funciones JWT.
    """
    
    def test_create_access_token(self):
        """
        Probar creación de token de acceso.
        """
        data = {"sub": "testuser"}
        token = create_access_token(data)
        
        assert isinstance(token, str)
        assert len(token) > 0
    
    def test_create_access_token_with_expiration(self):
        """
        Probar creación de token con expiración personalizada.
        """
        data = {"sub": "testuser"}
        expires_delta = timedelta(minutes=15)
        token = create_access_token(data, expires_delta)
        
        payload = verify_token(token)
        assert payload is not None
        assert payload["sub"] == "testuser"
    
    def test_verify_valid_token(self):
        """
        Probar verificación de token válido.
        """
        data = {"sub": "testuser"}
        token = create_access_token(data)
        
        payload = verify_token(token)
        
        assert payload is not None
        assert payload["sub"] == "testuser"
        assert "exp" in payload
    
    def test_verify_invalid_token(self):
        """
        Probar verificación de token inválido.
        """
        invalid_token = "invalid.token.here"
        
        payload = verify_token(invalid_token)
        
        assert payload is None
    
    def test_get_user_from_token(self):
        """
        Probar obtención de usuario desde token.
        """
        username = "testuser"
        data = {"sub": username}
        token = create_access_token(data)
        
        extracted_username = get_user_from_token(token)
        
        assert extracted_username == username
    
    def test_get_user_from_invalid_token(self):
        """
        Probar obtención de usuario desde token inválido.
        """
        invalid_token = "invalid.token.here"
        
        username = get_user_from_token(invalid_token)
        
        assert username is None

class TestPasswordFunctions:
    """
    Pruebas para funciones de contraseñas.
    """
    
    def test_password_hashing(self):
        """
        Probar hash de contraseña.
        """
        password = "testpassword123"
        
        hashed = get_password_hash(password)
        
        assert hashed != password
        assert len(hashed) > 0
        assert hashed.startswith("$2b$")
    
    def test_password_verification_success(self):
        """
        Probar verificación exitosa de contraseña.
        """
        password = "testpassword123"
        hashed = get_password_hash(password)
        
        is_valid = verify_password(password, hashed)
        
        assert is_valid is True
    
    def test_password_verification_failure(self):
        """
        Probar verificación fallida de contraseña.
        """
        password = "testpassword123"
        wrong_password = "wrongpassword"
        hashed = get_password_hash(password)
        
        is_valid = verify_password(wrong_password, hashed)
        
        assert is_valid is False
    
    def test_different_passwords_different_hashes(self):
        """
        Probar que contraseñas diferentes generan hashes diferentes.
        """
        password1 = "password123"
        password2 = "password456"
        
        hash1 = get_password_hash(password1)
        hash2 = get_password_hash(password2)
        
        assert hash1 != hash2
    
    def test_same_password_different_hashes(self):
        """
        Probar que la misma contraseña genera hashes diferentes (salt).
        """
        password = "testpassword123"
        
        hash1 = get_password_hash(password)
        hash2 = get_password_hash(password)
        
        # Los hashes deben ser diferentes debido al salt
        assert hash1 != hash2
        
        # Pero ambos deben verificar correctamente
        assert verify_password(password, hash1)
        assert verify_password(password, hash2)
```

## Pruebas de integración

### 1. Testing de endpoints de usuarios

```python
# tests/integration/test_users.py
import pytest
from fastapi.testclient import TestClient

class TestUserEndpoints:
    """
    Pruebas de integración para endpoints de usuarios.
    """
    
    def test_create_user_success(self, client: TestClient):
        """
        Probar creación exitosa de usuario.
        """
        user_data = {
            "username": "newuser",
            "email": "newuser@example.com",
            "full_name": "New User",
            "password": "testpassword123"
        }
        
        response = client.post("/api/v1/auth/register", json=user_data)
        
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "newuser"
        assert data["email"] == "newuser@example.com"
        assert "id" in data
        assert "hashed_password" not in data  # No debe exponer la contraseña
    
    def test_create_user_duplicate_username(self, client: TestClient, create_user):
        """
        Probar error al crear usuario con username duplicado.
        """
        user_data = {
            "username": create_user.username,  # Username ya existe
            "email": "different@example.com",
            "full_name": "Different User",
            "password": "testpassword123"
        }
        
        response = client.post("/api/v1/auth/register", json=user_data)
        
        assert response.status_code == 400
        assert "ya está registrado" in response.json()["detail"]
    
    def test_create_user_duplicate_email(self, client: TestClient, create_user):
        """
        Probar error al crear usuario con email duplicado.
        """
        user_data = {
            "username": "differentuser",
            "email": create_user.email,  # Email ya existe
            "full_name": "Different User",
            "password": "testpassword123"
        }
        
        response = client.post("/api/v1/auth/register", json=user_data)
        
        assert response.status_code == 400
        assert "ya está registrado" in response.json()["detail"]
    
    def test_get_users_without_auth(self, client: TestClient):
        """
        Probar acceso a usuarios sin autenticación.
        """
        response = client.get("/api/v1/users/")
        
        assert response.status_code == 401
    
    def test_get_users_with_auth(self, client: TestClient, auth_headers):
        """
        Probar obtención de usuarios con autenticación.
        """
        response = client.get("/api/v1/users/", headers=auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        assert isinstance(data, list)
    
    def test_get_user_by_id(self, client: TestClient, auth_headers, create_user):
        """
        Probar obtención de usuario por ID.
        """
        response = client.get(
            f"/api/v1/users/{create_user.id}", 
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["id"] == create_user.id
        assert data["username"] == create_user.username
    
    def test_get_user_not_found(self, client: TestClient, auth_headers):
        """
        Probar obtención de usuario inexistente.
        """
        response = client.get("/api/v1/users/999", headers=auth_headers)
        
        assert response.status_code == 404
    
    def test_update_user_success(self, client: TestClient, auth_headers, create_user):
        """
        Probar actualización exitosa de usuario.
        """
        update_data = {
            "full_name": "Updated Name",
            "phone": "9876543210"
        }
        
        response = client.put(
            f"/api/v1/users/{create_user.id}",
            json=update_data,
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["full_name"] == "Updated Name"
        assert data["phone"] == "9876543210"
    
    def test_delete_user_success(self, client: TestClient, admin_headers, create_user):
        """
        Probar eliminación exitosa de usuario (solo admin).
        """
        response = client.delete(
            f"/api/v1/users/{create_user.id}",
            headers=admin_headers
        )
        
        assert response.status_code == 200
        
        # Verificar que el usuario ya no existe
        response = client.get(
            f"/api/v1/users/{create_user.id}",
            headers=admin_headers
        )
        assert response.status_code == 404

class TestUserAuthentication:
    """
    Pruebas de autenticación de usuarios.
    """
    
    def test_login_success(self, client: TestClient, create_user):
        """
        Probar login exitoso.
        """
        login_data = {
            "username": create_user.username,
            "password": "secret"  # Contraseña del fixture
        }
        
        response = client.post(
            "/api/v1/auth/login",
            data=login_data,  # OAuth2PasswordRequestForm usa form data
            headers={"Content-Type": "application/x-www-form-urlencoded"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"
        assert "expires_in" in data
    
    def test_login_invalid_username(self, client: TestClient):
        """
        Probar login con username inválido.
        """
        login_data = {
            "username": "nonexistent",
            "password": "secret"
        }
        
        response = client.post(
            "/api/v1/auth/login",
            data=login_data,
            headers={"Content-Type": "application/x-www-form-urlencoded"}
        )
        
        assert response.status_code == 401
        assert "Credenciales incorrectas" in response.json()["detail"]
    
    def test_login_invalid_password(self, client: TestClient, create_user):
        """
        Probar login con contraseña inválida.
        """
        login_data = {
            "username": create_user.username,
            "password": "wrongpassword"
        }
        
        response = client.post(
            "/api/v1/auth/login",
            data=login_data,
            headers={"Content-Type": "application/x-www-form-urlencoded"}
        )
        
        assert response.status_code == 401
        assert "Credenciales incorrectas" in response.json()["detail"]
    
    def test_get_current_user(self, client: TestClient, auth_headers):
        """
        Probar obtención de usuario actual.
        """
        response = client.get("/api/v1/auth/me", headers=auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        assert "username" in data
        assert "email" in data
        assert "id" in data
    
    def test_get_current_user_invalid_token(self, client: TestClient):
        """
        Probar obtención de usuario actual con token inválido.
        """
        invalid_headers = {"Authorization": "Bearer invalid_token"}
        
        response = client.get("/api/v1/auth/me", headers=invalid_headers)
        
        assert response.status_code == 401
```

### 2. Testing de endpoints de artículos

```python
# tests/integration/test_items.py
import pytest
from fastapi.testclient import TestClient

class TestItemEndpoints:
    """
    Pruebas de integración para endpoints de artículos.
    """
    
    def test_create_item_success(self, client: TestClient, auth_headers, create_category):
        """
        Probar creación exitosa de artículo.
        """
        item_data = {
            "name": "Test Item",
            "description": "Test Description",
            "serial_number": "TEST123",
            "category_id": create_category.id
        }
        
        response = client.post(
            "/api/v1/items/",
            json=item_data,
            headers=auth_headers
        )
        
        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "Test Item"
        assert data["serial_number"] == "TEST123"
        assert data["category_id"] == create_category.id
        assert "id" in data
    
    def test_create_item_duplicate_serial(self, client: TestClient, auth_headers, create_item):
        """
        Probar error al crear artículo con número de serie duplicado.
        """
        item_data = {
            "name": "Different Item",
            "description": "Different Description",
            "serial_number": create_item.serial_number,  # Serial duplicado
            "category_id": create_item.category_id
        }
        
        response = client.post(
            "/api/v1/items/",
            json=item_data,
            headers=auth_headers
        )
        
        assert response.status_code == 400
        assert "ya existe" in response.json()["detail"]
    
    def test_get_items(self, client: TestClient, auth_headers):
        """
        Probar obtención de lista de artículos.
        """
        response = client.get("/api/v1/items/", headers=auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        assert isinstance(data, list)
    
    def test_get_item_by_id(self, client: TestClient, auth_headers, create_item):
        """
        Probar obtención de artículo por ID.
        """
        response = client.get(
            f"/api/v1/items/{create_item.id}",
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["id"] == create_item.id
        assert data["name"] == create_item.name
    
    def test_search_items(self, client: TestClient, auth_headers, create_item):
        """
        Probar búsqueda de artículos.
        """
        response = client.get(
            f"/api/v1/items/search?q={create_item.name[:4]}",
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert isinstance(data, list)
        assert len(data) > 0
        assert any(item["id"] == create_item.id for item in data)
    
    def test_update_item(self, client: TestClient, auth_headers, create_item):
        """
        Probar actualización de artículo.
        """
        update_data = {
            "name": "Updated Item",
            "description": "Updated Description"
        }
        
        response = client.put(
            f"/api/v1/items/{create_item.id}",
            json=update_data,
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert data["name"] == "Updated Item"
        assert data["description"] == "Updated Description"
    
    def test_delete_item(self, client: TestClient, admin_headers, create_item):
        """
        Probar eliminación de artículo.
        """
        response = client.delete(
            f"/api/v1/items/{create_item.id}",
            headers=admin_headers
        )
        
        assert response.status_code == 200
        
        # Verificar que el artículo ya no existe
        response = client.get(
            f"/api/v1/items/{create_item.id}",
            headers=admin_headers
        )
        assert response.status_code == 404
```

## Pruebas de rendimiento

### 1. Testing de carga

```python
# tests/performance/test_load.py
import pytest
import time
import asyncio
from concurrent.futures import ThreadPoolExecutor
from fastapi.testclient import TestClient

@pytest.mark.slow
class TestPerformance:
    """
    Pruebas de rendimiento y carga.
    """
    
    def test_concurrent_user_creation(self, client: TestClient):
        """
        Probar creación concurrente de usuarios.
        """
        def create_user(index):
            user_data = {
                "username": f"user{index}",
                "email": f"user{index}@example.com",
                "full_name": f"User {index}",
                "password": "testpassword123"
            }
            
            start_time = time.time()
            response = client.post("/api/v1/auth/register", json=user_data)
            end_time = time.time()
            
            return {
                "status_code": response.status_code,
                "response_time": end_time - start_time,
                "index": index
            }
        
        # Crear 10 usuarios concurrentemente
        with ThreadPoolExecutor(max_workers=5) as executor:
            futures = [executor.submit(create_user, i) for i in range(10)]
            results = [future.result() for future in futures]
        
        # Verificar resultados
        successful_requests = [r for r in results if r["status_code"] == 201]
        assert len(successful_requests) == 10
        
        # Verificar tiempos de respuesta
        avg_response_time = sum(r["response_time"] for r in results) / len(results)
        assert avg_response_time < 1.0  # Menos de 1 segundo promedio
    
    def test_api_response_time(self, client: TestClient, auth_headers):
        """
        Probar tiempos de respuesta de la API.
        """
        endpoints = [
            "/api/v1/users/",
            "/api/v1/categories/",
            "/api/v1/items/",
            "/api/v1/loans/"
        ]
        
        response_times = []
        
        for endpoint in endpoints:
            start_time = time.time()
            response = client.get(endpoint, headers=auth_headers)
            end_time = time.time()
            
            response_time = end_time - start_time
            response_times.append(response_time)
            
            assert response.status_code == 200
            assert response_time < 0.5  # Menos de 500ms
        
        avg_response_time = sum(response_times) / len(response_times)
        assert avg_response_time < 0.3  # Menos de 300ms promedio
```

## Comandos útiles para testing

### 1. Ejecutar todas las pruebas

```bash
# Ejecutar todas las pruebas
pytest

# Ejecutar con verbose
pytest -v

# Ejecutar con coverage
pytest --cov=app

# Ejecutar solo pruebas unitarias
pytest tests/unit/

# Ejecutar solo pruebas de integración
pytest tests/integration/

# Ejecutar pruebas específicas
pytest tests/unit/test_crud.py::TestUserCRUD::test_create_user

# Ejecutar pruebas con marcadores
pytest -m "not slow"  # Excluir pruebas lentas
pytest -m "unit"      # Solo pruebas unitarias
pytest -m "auth"      # Solo pruebas de autenticación
```

### 2. Generar reportes

```bash
# Generar reporte HTML de coverage
pytest --cov=app --cov-report=html

# Generar reporte XML para CI/CD
pytest --cov=app --cov-report=xml --junitxml=test-results.xml

# Ejecutar con profiling
pytest --profile
```

### 3. Debugging

```bash
# Ejecutar con debugging
pytest --pdb

# Parar en el primer fallo
pytest -x

# Mostrar output de print
pytest -s

# Ejecutar en paralelo
pytest -n auto
```

## Integración con CI/CD

### 1. GitHub Actions

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
        pytest --cov=app --cov-report=xml --junitxml=test-results.xml
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        fail_ci_if_error: true
```

### 2. Pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: tests
        name: Run tests
        entry: pytest
        language: system
        pass_filenames: false
        always_run: true
      
      - id: coverage
        name: Check coverage
        entry: pytest --cov=app --cov-fail-under=80
        language: system
        pass_filenames: false
        always_run: true
```

## Mejores prácticas

### 1. Organización de pruebas

- **Separar** por tipo (unit, integration, e2e)
- **Usar fixtures** para datos de prueba
- **Nombrar** descriptivamente
- **Documentar** casos complejos

### 2. Datos de prueba

- **Usar factories** para generar datos
- **Limpiar** después de cada prueba
- **Aislar** pruebas entre sí
- **Usar** base de datos en memoria

### 3. Assertions

- **Ser específico** en las assertions
- **Verificar** tanto éxito como fallo
- **Probar** casos límite
- **Incluir** mensajes descriptivos

### 4. Coverage

- **Apuntar** a 80%+ de coverage
- **No obsesionarse** con 100%
- **Priorizar** código crítico
- **Revisar** reportes regularmente

## Próximos pasos

En el siguiente tema aprenderemos sobre deployment y cómo llevar nuestra API a producción.

---

**💡 Tips importantes**:

1. **Escribir pruebas primero** (TDD) cuando sea posible
2. **Mantener pruebas simples** y enfocadas
3. **Usar mocks** para dependencias externas
4. **Automatizar** ejecución de pruebas
5. **Revisar coverage** regularmente

**🔗 Enlaces útiles**:
- [Pytest Documentation](https://docs.pytest.org/)
- [FastAPI Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [Coverage.py](https://coverage.readthedocs.io/)
- [Testing Best Practices](https://docs.python-guide.org/writing/tests/)