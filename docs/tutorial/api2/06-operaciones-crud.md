# 6. Operaciones CRUD

## ¿Qué es CRUD?

CRUD es un acrónimo que representa las cuatro operaciones básicas de persistencia de datos:

- **C**reate (Crear) - Insertar nuevos registros
- **R**ead (Leer) - Consultar registros existentes
- **U**pdate (Actualizar) - Modificar registros existentes
- **D**elete (Eliminar) - Borrar registros

### ¿Por qué una Capa CRUD?

La capa CRUD actúa como un **intermediario** entre los endpoints de la API y la base de datos:

```
🌐 API Endpoints
       ↕️
🔧 Capa CRUD        ← Lógica de negocio
       ↕️
🗄️ Base de Datos
```

**Ventajas:**
- **Reutilización**: Una función CRUD puede usarse en múltiples endpoints
- **Mantenimiento**: Cambios en la lógica de datos en un solo lugar
- **Testing**: Fácil probar la lógica de negocio independientemente
- **Abstracción**: Los endpoints no necesitan conocer detalles de la BD

## Estructura de Nuestro CRUD

### Enfoque Genérico

Nuestro proyecto usa un enfoque **genérico** que funciona con cualquier modelo:

```python
# app/crud/crud.py
from sqlalchemy.orm import Session
from app.models import models
from app.schemas import schemas

def get_all(db: Session, model):
    """Obtener todos los registros de un modelo."""
    return db.query(model).all()

def get_by_id(db: Session, model, item_id: int):
    """Obtener un registro por ID."""
    return db.query(model).filter(model.id == item_id).first()

def create_item(db: Session, model, item_data):
    """Crear un nuevo registro."""
    db_item = model(**item_data.dict(exclude={'id'}))
    db.add(db_item)
    db.commit()
    db.refresh(db_item)
    return db_item
```

### Ventajas del Enfoque Genérico

1. **Menos Código**: Una función para todos los modelos
2. **Consistencia**: Mismo comportamiento para todas las entidades
3. **Mantenimiento**: Cambios en un solo lugar
4. **Escalabilidad**: Fácil agregar nuevos modelos

## Análisis Detallado de Cada Operación

### 1. CREATE - Crear Registros

```python
def create_item(db: Session, model, item_data):
    """Crear un nuevo registro en la base de datos.
    
    Args:
        db: Sesión de base de datos
        model: Modelo SQLAlchemy (User, Product, Order)
        item_data: Esquema Pydantic con los datos
    
    Returns:
        El objeto creado con ID asignado
    """
    # 1. Convertir esquema Pydantic a diccionario
    data_dict = item_data.dict(exclude={'id'})
    
    # 2. Crear instancia del modelo
    db_item = model(**data_dict)
    
    # 3. Agregar a la sesión
    db.add(db_item)
    
    # 4. Confirmar cambios
    db.commit()
    
    # 5. Refrescar para obtener ID generado
    db.refresh(db_item)
    
    return db_item
```

#### Paso a Paso del CREATE

**1. Conversión de Datos**
```python
# Esquema Pydantic
user_data = UserCreate(name="Juan", email="juan@email.com")

# Convertir a diccionario (excluyendo ID)
data_dict = user_data.dict(exclude={'id'})
# Resultado: {'name': 'Juan', 'email': 'juan@email.com'}
```

**2. Creación del Objeto**
```python
# Crear instancia del modelo SQLAlchemy
db_user = User(**data_dict)
# Equivale a: db_user = User(name="Juan", email="juan@email.com")
```

**3. Persistencia**
```python
# Agregar a la sesión (aún no se guarda)
db.add(db_user)

# Confirmar cambios (ejecuta INSERT)
db.commit()

# Refrescar para obtener datos actualizados (como ID)
db.refresh(db_user)
print(db_user.id)  # Ahora tiene ID: 1, 2, 3...
```

### 2. READ - Leer Registros

#### Obtener Todos los Registros

```python
def get_all(db: Session, model):
    """Obtener todos los registros de un modelo.
    
    Args:
        db: Sesión de base de datos
        model: Modelo SQLAlchemy
    
    Returns:
        Lista de todos los registros
    """
    return db.query(model).all()
```

**SQL Generado:**
```sql
SELECT users.id, users.name, users.email 
FROM users;
```

#### Obtener por ID

```python
def get_by_id(db: Session, model, item_id: int):
    """Obtener un registro específico por ID.
    
    Args:
        db: Sesión de base de datos
        model: Modelo SQLAlchemy
        item_id: ID del registro a buscar
    
    Returns:
        El registro encontrado o None
    """
    return db.query(model).filter(model.id == item_id).first()
```

**SQL Generado:**
```sql
SELECT users.id, users.name, users.email 
FROM users 
WHERE users.id = 1 
LIMIT 1;
```

### 3. UPDATE - Actualizar Registros

Aunque nuestro proyecto simple no incluye UPDATE, aquí está la implementación:

```python
def update_item(db: Session, model, item_id: int, item_data):
    """Actualizar un registro existente.
    
    Args:
        db: Sesión de base de datos
        model: Modelo SQLAlchemy
        item_id: ID del registro a actualizar
        item_data: Esquema con los nuevos datos
    
    Returns:
        El registro actualizado o None si no existe
    """
    # 1. Buscar el registro existente
    db_item = db.query(model).filter(model.id == item_id).first()
    
    if not db_item:
        return None
    
    # 2. Obtener datos a actualizar (excluyendo None)
    update_data = item_data.dict(exclude_unset=True, exclude={'id'})
    
    # 3. Actualizar campos
    for field, value in update_data.items():
        setattr(db_item, field, value)
    
    # 4. Confirmar cambios
    db.commit()
    db.refresh(db_item)
    
    return db_item
```

#### Actualización Parcial

```python
# Esquema para actualización
class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

# Uso
user_update = UserUpdate(name="Juan Carlos")  # Solo actualizar nombre
update_data = user_update.dict(exclude_unset=True)
# Resultado: {'name': 'Juan Carlos'} - email no se incluye
```

### 4. DELETE - Eliminar Registros

```python
def delete_item(db: Session, model, item_id: int):
    """Eliminar un registro por ID.
    
    Args:
        db: Sesión de base de datos
        model: Modelo SQLAlchemy
        item_id: ID del registro a eliminar
    
    Returns:
        True si se eliminó, False si no existía
    """
    # 1. Buscar el registro
    db_item = db.query(model).filter(model.id == item_id).first()
    
    if not db_item:
        return False
    
    # 2. Eliminar
    db.delete(db_item)
    db.commit()
    
    return True
```

## Operaciones Avanzadas

### Búsqueda con Filtros

```python
def get_users_by_name(db: Session, name: str):
    """Buscar usuarios por nombre (búsqueda parcial)."""
    return db.query(User).filter(User.name.like(f"%{name}%")).all()

def get_products_by_price_range(db: Session, min_price: float, max_price: float):
    """Buscar productos en un rango de precio."""
    return db.query(Product).filter(
        Product.price >= min_price,
        Product.price <= max_price
    ).all()

def get_orders_by_user(db: Session, user_id: int):
    """Obtener todas las órdenes de un usuario."""
    return db.query(Order).filter(Order.user_id == user_id).all()
```

### Paginación

```python
def get_paginated(db: Session, model, skip: int = 0, limit: int = 10):
    """Obtener registros con paginación.
    
    Args:
        db: Sesión de base de datos
        model: Modelo SQLAlchemy
        skip: Número de registros a saltar
        limit: Número máximo de registros a retornar
    
    Returns:
        Lista de registros paginados
    """
    return db.query(model).offset(skip).limit(limit).all()

# Uso
# Página 1: skip=0, limit=10 (registros 1-10)
# Página 2: skip=10, limit=10 (registros 11-20)
# Página 3: skip=20, limit=10 (registros 21-30)
```

### Ordenamiento

```python
from sqlalchemy import desc, asc

def get_products_sorted(db: Session, sort_by: str = "name", order: str = "asc"):
    """Obtener productos ordenados."""
    query = db.query(Product)
    
    if sort_by == "name":
        column = Product.name
    elif sort_by == "price":
        column = Product.price
    else:
        column = Product.id
    
    if order == "desc":
        query = query.order_by(desc(column))
    else:
        query = query.order_by(asc(column))
    
    return query.all()
```

### Conteo de Registros

```python
def count_items(db: Session, model):
    """Contar total de registros."""
    return db.query(model).count()

def count_users_by_domain(db: Session, domain: str):
    """Contar usuarios por dominio de email."""
    return db.query(User).filter(
        User.email.like(f"%@{domain}")
    ).count()
```

## Manejo de Errores

### Errores Comunes y Soluciones

```python
from sqlalchemy.exc import IntegrityError
from fastapi import HTTPException

def create_user_safe(db: Session, user_data: UserCreate):
    """Crear usuario con manejo de errores."""
    try:
        # Verificar si el email ya existe
        existing_user = db.query(User).filter(
            User.email == user_data.email
        ).first()
        
        if existing_user:
            raise HTTPException(
                status_code=400,
                detail="Email ya está registrado"
            )
        
        # Crear usuario
        db_user = User(**user_data.dict(exclude={'id'}))
        db.add(db_user)
        db.commit()
        db.refresh(db_user)
        
        return db_user
        
    except IntegrityError:
        db.rollback()
        raise HTTPException(
            status_code=400,
            detail="Error de integridad en la base de datos"
        )
    except Exception as e:
        db.rollback()
        raise HTTPException(
            status_code=500,
            detail=f"Error interno: {str(e)}"
        )
```

### Transacciones

```python
def create_order_with_items(db: Session, order_data, items_data):
    """Crear orden con múltiples items en una transacción."""
    try:
        # Iniciar transacción implícita
        
        # 1. Crear la orden
        db_order = Order(**order_data.dict(exclude={'id'}))
        db.add(db_order)
        db.flush()  # Obtener ID sin hacer commit
        
        # 2. Crear los items
        for item_data in items_data:
            db_item = OrderItem(
                order_id=db_order.id,
                **item_data.dict(exclude={'id'})
            )
            db.add(db_item)
        
        # 3. Confirmar toda la transacción
        db.commit()
        db.refresh(db_order)
        
        return db_order
        
    except Exception as e:
        # Revertir todos los cambios
        db.rollback()
        raise e
```

## Optimización de Consultas

### Eager Loading (Carga Anticipada)

```python
from sqlalchemy.orm import joinedload

def get_orders_with_user(db: Session):
    """Obtener órdenes con datos del usuario (una sola consulta)."""
    return db.query(Order).options(
        joinedload(Order.user)
    ).all()

# Sin joinedload: N+1 consultas (1 para órdenes + N para usuarios)
# Con joinedload: 1 sola consulta con JOIN
```

### Consultas Específicas

```python
def get_user_summary(db: Session, user_id: int):
    """Obtener resumen de usuario con estadísticas."""
    from sqlalchemy import func
    
    result = db.query(
        User.id,
        User.name,
        User.email,
        func.count(Order.id).label('total_orders'),
        func.sum(Order.total).label('total_spent')
    ).outerjoin(Order).filter(
        User.id == user_id
    ).group_by(User.id).first()
    
    return result
```

### Índices para Rendimiento

```python
# En los modelos
from sqlalchemy import Index

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String, index=True)  # Índice simple
    
    # Índice compuesto
    __table_args__ = (
        Index('idx_name_email', 'name', 'email'),
    )
```

## Testing de CRUD

### Test Unitario

```python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.database.database import Base
from app.models.models import User
from app.crud.crud import create_item, get_by_id
from app.schemas.schemas import User as UserSchema

# Configurar base de datos de prueba
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

@pytest.fixture
def db_session():
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

def test_create_user(db_session):
    # Datos de prueba
    user_data = UserSchema(name="Test User", email="test@example.com")
    
    # Crear usuario
    created_user = create_item(db_session, User, user_data)
    
    # Verificaciones
    assert created_user.id is not None
    assert created_user.name == "Test User"
    assert created_user.email == "test@example.com"

def test_get_user_by_id(db_session):
    # Crear usuario primero
    user_data = UserSchema(name="Test User", email="test@example.com")
    created_user = create_item(db_session, User, user_data)
    
    # Buscar por ID
    found_user = get_by_id(db_session, User, created_user.id)
    
    # Verificaciones
    assert found_user is not None
    assert found_user.id == created_user.id
    assert found_user.name == "Test User"
```

## Mejores Prácticas

### 1. **Separación de Responsabilidades**

```python
# ✅ Bueno: CRUD solo maneja datos
def create_user(db: Session, user_data: UserCreate):
    db_user = User(**user_data.dict(exclude={'id'}))
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

# ❌ Malo: CRUD con lógica de negocio
def create_user(db: Session, user_data: UserCreate):
    # Enviar email de bienvenida (debería estar en otra capa)
    send_welcome_email(user_data.email)
    
    db_user = User(**user_data.dict(exclude={'id'}))
    db.add(db_user)
    db.commit()
    return db_user
```

### 2. **Manejo Consistente de Errores**

```python
def get_user_or_404(db: Session, user_id: int):
    """Obtener usuario o lanzar error 404."""
    user = get_by_id(db, User, user_id)
    if not user:
        raise HTTPException(
            status_code=404,
            detail=f"Usuario con ID {user_id} no encontrado"
        )
    return user
```

### 3. **Validaciones de Negocio**

```python
def create_order(db: Session, order_data: OrderCreate):
    """Crear orden con validaciones de negocio."""
    # Verificar que el usuario existe
    user = get_user_or_404(db, order_data.user_id)
    
    # Verificar que el producto existe
    product = get_by_id(db, Product, order_data.product_id)
    if not product:
        raise HTTPException(404, "Producto no encontrado")
    
    # Verificar stock (si tuviéramos ese campo)
    # if product.stock < order_data.quantity:
    #     raise HTTPException(400, "Stock insuficiente")
    
    # Crear la orden
    return create_item(db, Order, order_data)
```

### 4. **Documentación Clara**

```python
def get_filtered_products(
    db: Session,
    name: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    skip: int = 0,
    limit: int = 100
):
    """Obtener productos con filtros opcionales.
    
    Args:
        db: Sesión de base de datos
        name: Filtrar por nombre (búsqueda parcial)
        min_price: Precio mínimo (inclusive)
        max_price: Precio máximo (inclusive)
        skip: Número de registros a saltar para paginación
        limit: Número máximo de registros a retornar
    
    Returns:
        Lista de productos que cumplen los criterios
    
    Example:
        # Buscar productos caros con "phone" en el nombre
        products = get_filtered_products(
            db, 
            name="phone", 
            min_price=500.0,
            skip=0, 
            limit=10
        )
    """
    query = db.query(Product)
    
    if name:
        query = query.filter(Product.name.like(f"%{name}%"))
    if min_price is not None:
        query = query.filter(Product.price >= min_price)
    if max_price is not None:
        query = query.filter(Product.price <= max_price)
    
    return query.offset(skip).limit(limit).all()
```

## Ejercicios Prácticos

### Ejercicio 1: CRUD Completo
Implementa las operaciones faltantes:
- `update_item()`
- `delete_item()`
- Pruébalas con diferentes modelos

### Ejercicio 2: Búsquedas Avanzadas
Crea funciones para:
- Buscar usuarios por dominio de email
- Obtener productos más vendidos
- Calcular total de ventas por usuario

### Ejercicio 3: Validaciones
Agrega validaciones para:
- No permitir órdenes con cantidad 0
- Verificar que el email sea único al crear usuarios
- Validar que el precio sea positivo

### Ejercicio 4: Optimización
Implementa:
- Paginación con información de total de páginas
- Búsqueda con múltiples criterios
- Cache de consultas frecuentes

## Próximos Pasos

En el siguiente capítulo aprenderemos sobre **Rutas y Endpoints**, donde:
- Conectaremos las operaciones CRUD con HTTP
- Definiremos los endpoints de nuestra API
- Manejaremos parámetros de consulta y cuerpo de peticiones
- Implementaremos respuestas HTTP apropiadas

---

**Siguiente:** [Rutas y Endpoints](07-rutas-endpoints.md)

**Anterior:** [Esquemas de Validación](05-esquemas-validacion.md)