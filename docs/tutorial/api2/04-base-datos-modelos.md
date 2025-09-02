# 4. Base de Datos y Modelos

## ¿Qué es SQLAlchemy?

SQLAlchemy es un **ORM (Object-Relational Mapping)** que nos permite:
- Trabajar con bases de datos usando objetos Python
- Escribir menos SQL manual
- Cambiar de base de datos fácilmente
- Tener mayor seguridad contra inyecciones SQL

### ORM vs SQL Tradicional

**SQL Tradicional:**
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO users (name, email) VALUES ('Juan', 'juan@email.com');
SELECT * FROM users WHERE id = 1;
```

**Con SQLAlchemy ORM:**
```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)

# Crear usuario
user = User(name="Juan", email="juan@email.com")
db.add(user)
db.commit()

# Obtener usuario
user = db.query(User).filter(User.id == 1).first()
```

## Configuración de la Base de Datos

### Paso 1: Crear el archivo de configuración

Creamos `app/database/database.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

# URL de conexión a la base de datos
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"

# Crear el motor de base de datos
engine = create_engine(SQLALCHEMY_DATABASE_URL)

# Crear la clase de sesión
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Crear la clase base para los modelos
Base = declarative_base()

# Función para obtener una sesión de base de datos
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Explicación Detallada

#### 🔧 `create_engine()`
```python
engine = create_engine(SQLALCHEMY_DATABASE_URL)
```
- **Propósito**: Crea la conexión principal a la base de datos
- **SQLite**: Base de datos en archivo local (ideal para desarrollo)
- **Archivo**: `test.db` se crea automáticamente

#### 📝 `sessionmaker()`
```python
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```
- **autocommit=False**: Las transacciones deben confirmarse manualmente
- **autoflush=False**: Los cambios no se envían automáticamente
- **bind=engine**: Vincula las sesiones al motor de base de datos

#### 🏗️ `declarative_base()`
```python
Base = declarative_base()
```
- **Propósito**: Clase base para todos nuestros modelos
- **Funcionalidad**: Proporciona metadatos y funciones ORM

#### 🔄 `get_db()`
```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```
- **Patrón**: Dependency Injection de FastAPI
- **yield**: Genera una sesión y la mantiene abierta
- **finally**: Garantiza que la sesión se cierre

## Creación de Modelos

### Paso 2: Definir los modelos

Creamos `app/models/models.py`:

```python
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

### Anatomía de un Modelo

Veamos el modelo `User` en detalle:

```python
class User(Base):                    # 1. Hereda de Base
    __tablename__ = "users"          # 2. Nombre de la tabla
    
    id = Column(Integer, primary_key=True)  # 3. Clave primaria
    name = Column(String)                    # 4. Columna de texto
    email = Column(String)                   # 5. Otra columna
```

#### 1. **Herencia de Base**
- Todos los modelos heredan de `Base`
- Proporciona funcionalidad ORM automática
- Permite crear tablas automáticamente

#### 2. **`__tablename__`**
- Define el nombre de la tabla en la base de datos
- Convención: plural, snake_case
- Ejemplos: `users`, `products`, `order_items`

#### 3. **Clave Primaria**
```python
id = Column(Integer, primary_key=True)
```
- **Integer**: Tipo de dato entero
- **primary_key=True**: Identifica únicamente cada registro
- **Auto-incremento**: SQLite lo hace automáticamente

#### 4. **Columnas de Datos**
```python
name = Column(String)
email = Column(String)
price = Column(Float)
quantity = Column(Integer)
```

### Tipos de Datos Comunes

| Tipo SQLAlchemy | Tipo Python | Descripción | Ejemplo |
|-----------------|-------------|-------------|----------|
| `Integer` | `int` | Números enteros | `age = Column(Integer)` |
| `String` | `str` | Texto variable | `name = Column(String(100))` |
| `Float` | `float` | Números decimales | `price = Column(Float)` |
| `Boolean` | `bool` | Verdadero/Falso | `active = Column(Boolean)` |
| `DateTime` | `datetime` | Fecha y hora | `created_at = Column(DateTime)` |
| `Text` | `str` | Texto largo | `description = Column(Text)` |

### Ejemplo con Más Tipos

```python
from sqlalchemy import Column, Integer, String, Float, Boolean, DateTime, Text
from datetime import datetime

class Product(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100))              # Límite de caracteres
    description = Column(Text)              # Texto largo
    price = Column(Float)
    in_stock = Column(Boolean, default=True) # Valor por defecto
    created_at = Column(DateTime, default=datetime.utcnow)
```

## Relaciones Entre Modelos

Aunque nuestro proyecto usa un enfoque simple, es importante entender las relaciones:

### Tipos de Relaciones

#### 1. **Uno a Muchos (One-to-Many)**
```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    
    # Relación: Un usuario puede tener muchas órdenes
    orders = relationship("Order", back_populates="user")

class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"))  # Clave foránea
    
    # Relación: Una orden pertenece a un usuario
    user = relationship("User", back_populates="orders")
```

#### 2. **Muchos a Muchos (Many-to-Many)**
```python
from sqlalchemy import Table

# Tabla de asociación
user_product_association = Table(
    'user_products',
    Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id')),
    Column('product_id', Integer, ForeignKey('products.id'))
)

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    
    # Un usuario puede tener muchos productos favoritos
    favorite_products = relationship(
        "Product",
        secondary=user_product_association,
        back_populates="favorited_by"
    )

class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    
    # Un producto puede ser favorito de muchos usuarios
    favorited_by = relationship(
        "User",
        secondary=user_product_association,
        back_populates="favorite_products"
    )
```

## Creación de Tablas

### Método Automático

En `main.py`, las tablas se crean automáticamente:

```python
from app.database.database import engine
from app.models import models

# Crear todas las tablas definidas en los modelos
models.Base.metadata.create_all(bind=engine)
```

### ¿Qué hace `create_all()`?

1. **Examina** todos los modelos que heredan de `Base`
2. **Genera** el SQL necesario para crear las tablas
3. **Ejecuta** las sentencias CREATE TABLE
4. **Ignora** tablas que ya existen

### SQL Generado

Para nuestros modelos, SQLAlchemy genera:

```sql
CREATE TABLE users (
    id INTEGER NOT NULL PRIMARY KEY,
    name VARCHAR,
    email VARCHAR
);

CREATE TABLE products (
    id INTEGER NOT NULL PRIMARY KEY,
    name VARCHAR,
    price FLOAT
);

CREATE TABLE orders (
    id INTEGER NOT NULL PRIMARY KEY,
    user_id INTEGER,
    product_id INTEGER,
    quantity INTEGER
);
```

## Operaciones Básicas con Modelos

### Crear un Registro

```python
from sqlalchemy.orm import Session
from app.models.models import User
from app.database.database import SessionLocal

# Crear sesión
db = SessionLocal()

# Crear usuario
user = User(name="Juan Pérez", email="juan@email.com")

# Agregar a la sesión
db.add(user)

# Confirmar cambios
db.commit()

# Refrescar para obtener el ID generado
db.refresh(user)

print(f"Usuario creado con ID: {user.id}")

# Cerrar sesión
db.close()
```

### Consultar Registros

```python
# Obtener todos los usuarios
users = db.query(User).all()

# Obtener usuario por ID
user = db.query(User).filter(User.id == 1).first()

# Obtener usuario por email
user = db.query(User).filter(User.email == "juan@email.com").first()

# Obtener múltiples usuarios con filtro
users = db.query(User).filter(User.name.like("%Juan%")).all()
```

### Actualizar Registros

```python
# Obtener usuario
user = db.query(User).filter(User.id == 1).first()

# Modificar datos
user.name = "Juan Carlos Pérez"
user.email = "juancarlos@email.com"

# Confirmar cambios
db.commit()
```

### Eliminar Registros

```python
# Obtener usuario
user = db.query(User).filter(User.id == 1).first()

# Eliminar
db.delete(user)
db.commit()
```

## Validaciones y Restricciones

### Restricciones a Nivel de Base de Datos

```python
from sqlalchemy import Column, Integer, String, Float, UniqueConstraint, CheckConstraint

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)      # No puede ser NULL
    email = Column(String(255), unique=True)        # Debe ser único
    age = Column(Integer, CheckConstraint('age >= 0'))  # Debe ser positivo
    
    # Restricción de tabla
    __table_args__ = (
        UniqueConstraint('name', 'email', name='unique_name_email'),
    )
```

### Valores por Defecto

```python
from datetime import datetime
from sqlalchemy import Column, Integer, String, DateTime, Boolean

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    active = Column(Boolean, default=True)                    # Valor por defecto
    created_at = Column(DateTime, default=datetime.utcnow)    # Función por defecto
```

## Índices para Rendimiento

```python
from sqlalchemy import Column, Integer, String, Index

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

## Mejores Prácticas

### 1. **Nomenclatura Consistente**
```python
# ✅ Bueno
class User(Base):
    __tablename__ = "users"  # Plural, snake_case
    
    id = Column(Integer, primary_key=True)
    first_name = Column(String)  # snake_case
    created_at = Column(DateTime)

# ❌ Malo
class user(Base):  # Debería ser PascalCase
    __tablename__ = "User"  # Debería ser plural y snake_case
    
    ID = Column(Integer, primary_key=True)  # Debería ser snake_case
    firstName = Column(String)  # Debería ser snake_case
```

### 2. **Documentación de Modelos**
```python
class User(Base):
    """Modelo para usuarios del sistema.
    
    Attributes:
        id: Identificador único del usuario
        name: Nombre completo del usuario
        email: Dirección de correo electrónico (único)
        created_at: Fecha y hora de creación
    """
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, comment="ID único del usuario")
    name = Column(String(100), nullable=False, comment="Nombre completo")
    email = Column(String(255), unique=True, comment="Email único")
```

### 3. **Validación de Datos**
```python
from sqlalchemy.orm import validates

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    name = Column(String)
    email = Column(String)
    
    @validates('email')
    def validate_email(self, key, address):
        if '@' not in address:
            raise ValueError("Email inválido")
        return address
    
    @validates('name')
    def validate_name(self, key, name):
        if len(name) < 2:
            raise ValueError("Nombre muy corto")
        return name
```

## Debugging y Logging

### Habilitar Logging SQL

```python
import logging

# Configurar logging para ver las consultas SQL
logging.basicConfig()
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Crear engine con echo=True para desarrollo
engine = create_engine(SQLALCHEMY_DATABASE_URL, echo=True)
```

### Inspeccionar la Base de Datos

```python
from sqlalchemy import inspect

# Crear inspector
inspector = inspect(engine)

# Obtener nombres de tablas
table_names = inspector.get_table_names()
print(f"Tablas: {table_names}")

# Obtener columnas de una tabla
columns = inspector.get_columns('users')
for column in columns:
    print(f"Columna: {column['name']}, Tipo: {column['type']}")
```

## Migración de Esquemas

Para proyectos en producción, considera usar **Alembic**:

```bash
# Instalar Alembic
pip install alembic

# Inicializar
alembic init alembic

# Crear migración
alembic revision --autogenerate -m "Crear tabla users"

# Aplicar migración
alembic upgrade head
```

## Ejercicios Prácticos

### Ejercicio 1: Modelo Extendido
Crea un modelo `Category` con:
- `id` (clave primaria)
- `name` (string, único)
- `description` (texto largo)
- `active` (boolean, por defecto True)

### Ejercicio 2: Validaciones
Agrega validaciones al modelo `Product`:
- El precio debe ser mayor a 0
- El nombre debe tener al menos 3 caracteres

### Ejercicio 3: Consultas
Escribe funciones para:
- Obtener productos por rango de precio
- Buscar usuarios por parte del nombre
- Contar órdenes por usuario

## Próximos Pasos

En el siguiente capítulo aprenderemos sobre **Esquemas de Validación** con Pydantic, que nos permitirán:
- Validar datos de entrada
- Serializar datos de salida
- Documentar automáticamente la API
- Convertir entre formatos JSON y Python

---

**Siguiente:** [Esquemas de Validación](05-esquemas-validacion.md)

**Anterior:** [Estructura del Proyecto](03-estructura-proyecto.md)