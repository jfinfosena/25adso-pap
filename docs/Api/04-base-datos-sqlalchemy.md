# Base de Datos y SQLAlchemy

## Introducción

SQLAlchemy es el ORM (Object-Relational Mapping) más popular de Python. Nos permite trabajar con bases de datos usando objetos Python en lugar de escribir SQL directamente. En este tema aprenderemos a configurar SQLAlchemy con FastAPI.

## ¿Qué es un ORM?

Un ORM es una técnica que permite mapear objetos de programación a tablas de base de datos:

```python
# Sin ORM (SQL directo)
result = cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))
user_data = result.fetchone()

# Con ORM (SQLAlchemy)
user = db.query(User).filter(User.id == user_id).first()
```

### Ventajas del ORM
- **Abstracción**: No necesitas escribir SQL
- **Portabilidad**: Funciona con diferentes bases de datos
- **Seguridad**: Previene inyección SQL automáticamente
- **Productividad**: Menos código, más funcionalidad
- **Mantenimiento**: Cambios de esquema más fáciles

## Instalación de dependencias

```bash
pip install sqlalchemy
pip install databases[sqlite]  # Para SQLite
pip install psycopg2-binary    # Para PostgreSQL
pip install pymysql            # Para MySQL
```

## Configuración de la base de datos

### 1. Archivo de configuración (`app/config.py`)

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Settings:
    # Configuración de base de datos
    DATABASE_URL: str = os.getenv(
        "DATABASE_URL", 
        "sqlite:///./inventory.db"
    )
    
    # Para PostgreSQL en producción
    # DATABASE_URL: str = os.getenv(
    #     "DATABASE_URL", 
    #     "postgresql://user:password@localhost/inventory"
    # )
    
    # Configuración SQLAlchemy
    SQLALCHEMY_ECHO: bool = os.getenv("DEBUG", "False").lower() == "true"
    
settings = Settings()
```

### 2. Configuración de conexión (`app/database/connection.py`)

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.config import settings

# Crear engine de SQLAlchemy
engine = create_engine(
    settings.DATABASE_URL,
    # Configuraciones específicas para SQLite
    connect_args={"check_same_thread": False} if "sqlite" in settings.DATABASE_URL else {},
    # Mostrar SQL en consola (útil para desarrollo)
    echo=settings.SQLALCHEMY_ECHO
)

# Crear SessionLocal para manejar sesiones de base de datos
SessionLocal = sessionmaker(
    autocommit=False,  # No hacer commit automático
    autoflush=False,   # No hacer flush automático
    bind=engine        # Vincular al engine
)
```

### 3. Funciones de base de datos (`app/database/database.py`)

```python
from sqlalchemy.orm import Session
from app.database.connection import SessionLocal

def get_db() -> Session:
    """
    Dependencia para obtener una sesión de base de datos.
    
    Esta función se usa como dependencia en FastAPI para inyectar
    una sesión de base de datos en los endpoints.
    
    Yields:
        Session: Sesión de SQLAlchemy
    """
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def create_tables():
    """
    Crear todas las tablas en la base de datos.
    
    Esta función debe llamarse al iniciar la aplicación
    para asegurar que todas las tablas existan.
    """
    from app.database.base import Base
    Base.metadata.create_all(bind=engine)

def drop_tables():
    """
    Eliminar todas las tablas de la base de datos.
    
    ⚠️ CUIDADO: Esta función elimina todos los datos.
    Solo usar en desarrollo o testing.
    """
    from app.database.base import Base
    Base.metadata.drop_all(bind=engine)
```

## Modelo base

### Creando la clase base (`app/database/base.py`)

```python
from sqlalchemy import Column, Integer, DateTime
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

# Crear la clase base para todos los modelos
Base = declarative_base()

class BaseModel(Base):
    """
    Clase base para todos los modelos de la aplicación.
    
    Proporciona campos comunes que todos los modelos necesitan:
    - id: Clave primaria
    - created_at: Fecha de creación
    - updated_at: Fecha de última actualización
    """
    __abstract__ = True  # No crear tabla para esta clase
    
    # Clave primaria auto-incremental
    id = Column(
        Integer, 
        primary_key=True, 
        index=True,
        comment="Identificador único del registro"
    )
    
    # Timestamp de creación
    created_at = Column(
        DateTime, 
        default=datetime.utcnow,
        nullable=False,
        comment="Fecha y hora de creación del registro"
    )
    
    # Timestamp de última actualización
    updated_at = Column(
        DateTime, 
        default=datetime.utcnow, 
        onupdate=datetime.utcnow,
        nullable=False,
        comment="Fecha y hora de última actualización"
    )
    
    def __repr__(self):
        """
        Representación string del objeto para debugging.
        """
        return f"<{self.__class__.__name__}(id={self.id})>"
```

## Creando modelos

### 1. Modelo de Usuario (`app/models/user.py`)

```python
from sqlalchemy import Column, String, Boolean, Text
from sqlalchemy.orm import relationship
from app.database.base import BaseModel

class User(BaseModel):
    """
    Modelo para representar usuarios del sistema.
    
    Attributes:
        username: Nombre de usuario único
        email: Correo electrónico único
        full_name: Nombre completo del usuario
        is_active: Si el usuario está activo
        loans: Relación con préstamos del usuario
    """
    __tablename__ = "users"
    
    # Campos del usuario
    username = Column(
        String(50), 
        unique=True, 
        index=True, 
        nullable=False,
        comment="Nombre de usuario único"
    )
    
    email = Column(
        String(255), 
        unique=True, 
        index=True, 
        nullable=False,
        comment="Correo electrónico del usuario"
    )
    
    full_name = Column(
        String(100), 
        nullable=True,
        comment="Nombre completo del usuario"
    )
    
    is_active = Column(
        Boolean, 
        default=True, 
        nullable=False,
        comment="Indica si el usuario está activo"
    )
    
    # Relaciones
    loans = relationship(
        "Loan", 
        back_populates="user",
        cascade="all, delete-orphan",
        lazy="dynamic"  # Carga bajo demanda
    )
    
    def __str__(self):
        return f"{self.username} ({self.email})"
```

### 2. Modelo de Categoría (`app/models/category.py`)

```python
from sqlalchemy import Column, String, Text
from sqlalchemy.orm import relationship
from app.database.base import BaseModel

class Category(BaseModel):
    """
    Modelo para representar categorías de artículos.
    
    Attributes:
        name: Nombre de la categoría
        description: Descripción de la categoría
        items: Relación con artículos de esta categoría
    """
    __tablename__ = "categories"
    
    # Campos de la categoría
    name = Column(
        String(100), 
        unique=True, 
        index=True, 
        nullable=False,
        comment="Nombre único de la categoría"
    )
    
    description = Column(
        Text, 
        nullable=True,
        comment="Descripción detallada de la categoría"
    )
    
    # Relaciones
    items = relationship(
        "Item", 
        back_populates="category",
        cascade="all, delete-orphan",
        lazy="dynamic"
    )
    
    def __str__(self):
        return self.name
```

### 3. Modelo de Artículo (`app/models/item.py`)

```python
from sqlalchemy import Column, String, Text, Integer, ForeignKey, Enum as SQLEnum
from sqlalchemy.orm import relationship
from enum import Enum
from app.database.base import BaseModel

class ItemStatus(str, Enum):
    """
    Estados posibles de un artículo.
    """
    AVAILABLE = "available"      # Disponible
    LOANED = "loaned"           # Prestado
    MAINTENANCE = "maintenance" # En mantenimiento
    RETIRED = "retired"         # Retirado

class Item(BaseModel):
    """
    Modelo para representar artículos del inventario.
    
    Attributes:
        name: Nombre del artículo
        description: Descripción del artículo
        serial_number: Número de serie único
        status: Estado actual del artículo
        category_id: ID de la categoría
        category: Relación con la categoría
        loans: Relación con préstamos del artículo
    """
    __tablename__ = "items"
    
    # Campos del artículo
    name = Column(
        String(200), 
        nullable=False, 
        index=True,
        comment="Nombre del artículo"
    )
    
    description = Column(
        Text, 
        nullable=True,
        comment="Descripción detallada del artículo"
    )
    
    serial_number = Column(
        String(100), 
        unique=True, 
        index=True, 
        nullable=False,
        comment="Número de serie único del artículo"
    )
    
    status = Column(
        SQLEnum(ItemStatus), 
        default=ItemStatus.AVAILABLE, 
        nullable=False,
        comment="Estado actual del artículo"
    )
    
    # Clave foránea a categoría
    category_id = Column(
        Integer, 
        ForeignKey("categories.id"), 
        nullable=False,
        comment="ID de la categoría del artículo"
    )
    
    # Relaciones
    category = relationship(
        "Category", 
        back_populates="items"
    )
    
    loans = relationship(
        "Loan", 
        back_populates="item",
        cascade="all, delete-orphan",
        lazy="dynamic"
    )
    
    def __str__(self):
        return f"{self.name} ({self.serial_number})"
    
    @property
    def is_available(self) -> bool:
        """Verifica si el artículo está disponible para préstamo."""
        return self.status == ItemStatus.AVAILABLE
```

### 4. Modelo de Préstamo (`app/models/loan.py`)

```python
from sqlalchemy import Column, Integer, ForeignKey, DateTime, Text, Enum as SQLEnum
from sqlalchemy.orm import relationship
from datetime import datetime, timedelta
from enum import Enum
from app.database.base import BaseModel

class LoanStatus(str, Enum):
    """
    Estados posibles de un préstamo.
    """
    ACTIVE = "active"       # Préstamo activo
    RETURNED = "returned"   # Devuelto
    OVERDUE = "overdue"     # Vencido
    CANCELLED = "cancelled" # Cancelado

class Loan(BaseModel):
    """
    Modelo para representar préstamos de artículos.
    
    Attributes:
        loan_date: Fecha del préstamo
        due_date: Fecha de vencimiento
        return_date: Fecha de devolución (si aplica)
        status: Estado del préstamo
        notes: Notas adicionales
        user_id: ID del usuario que solicita
        item_id: ID del artículo prestado
        user: Relación con el usuario
        item: Relación con el artículo
    """
    __tablename__ = "loans"
    
    # Fechas del préstamo
    loan_date = Column(
        DateTime, 
        default=datetime.utcnow, 
        nullable=False,
        comment="Fecha y hora del préstamo"
    )
    
    due_date = Column(
        DateTime, 
        nullable=False,
        comment="Fecha y hora de vencimiento"
    )
    
    return_date = Column(
        DateTime, 
        nullable=True,
        comment="Fecha y hora de devolución"
    )
    
    # Estado y notas
    status = Column(
        SQLEnum(LoanStatus), 
        default=LoanStatus.ACTIVE, 
        nullable=False,
        comment="Estado actual del préstamo"
    )
    
    notes = Column(
        Text, 
        nullable=True,
        comment="Notas adicionales sobre el préstamo"
    )
    
    # Claves foráneas
    user_id = Column(
        Integer, 
        ForeignKey("users.id"), 
        nullable=False,
        comment="ID del usuario que solicita el préstamo"
    )
    
    item_id = Column(
        Integer, 
        ForeignKey("items.id"), 
        nullable=False,
        comment="ID del artículo prestado"
    )
    
    # Relaciones
    user = relationship(
        "User", 
        back_populates="loans"
    )
    
    item = relationship(
        "Item", 
        back_populates="loans"
    )
    
    def __str__(self):
        return f"Préstamo {self.id}: {self.item.name} a {self.user.username}"
    
    @property
    def is_overdue(self) -> bool:
        """Verifica si el préstamo está vencido."""
        if self.status != LoanStatus.ACTIVE:
            return False
        return datetime.utcnow() > self.due_date
    
    @property
    def days_until_due(self) -> int:
        """Calcula días hasta el vencimiento."""
        if self.status != LoanStatus.ACTIVE:
            return 0
        delta = self.due_date - datetime.utcnow()
        return delta.days
```

## Inicialización en main.py

```python
# app/main.py
from fastapi import FastAPI
from app.database.connection import engine
from app.database.base import Base

# Importar todos los modelos para que SQLAlchemy los registre
from app.models import user, category, item, loan

# Crear todas las tablas
Base.metadata.create_all(bind=engine)

app = FastAPI(title="Inventory API")

# ... resto de la configuración
```

## Conceptos importantes de SQLAlchemy

### 1. Relaciones

#### One-to-Many (Uno a Muchos)
```python
# Una categoría tiene muchos artículos
class Category(BaseModel):
    items = relationship("Item", back_populates="category")

class Item(BaseModel):
    category_id = Column(Integer, ForeignKey("categories.id"))
    category = relationship("Category", back_populates="items")
```

#### Many-to-One (Muchos a Uno)
```python
# Muchos préstamos pertenecen a un usuario
class Loan(BaseModel):
    user_id = Column(Integer, ForeignKey("users.id"))
    user = relationship("User", back_populates="loans")

class User(BaseModel):
    loans = relationship("Loan", back_populates="user")
```

### 2. Lazy Loading

```python
# Diferentes estrategias de carga
loans = relationship("Loan", lazy="select")     # Carga cuando se accede
loans = relationship("Loan", lazy="joined")     # Carga con JOIN
loans = relationship("Loan", lazy="dynamic")    # Retorna query object
loans = relationship("Loan", lazy="subquery")   # Carga con subquery
```

### 3. Cascade

```python
# Operaciones en cascada
loans = relationship(
    "Loan", 
    cascade="all, delete-orphan"  # Elimina préstamos si se elimina usuario
)
```

### 4. Índices

```python
# Crear índices para mejorar rendimiento
email = Column(String(255), index=True)  # Índice simple
username = Column(String(50), unique=True, index=True)  # Índice único
```

## Comandos útiles para desarrollo

### Crear tablas manualmente
```python
# En Python shell
from app.database.connection import engine
from app.database.base import Base
from app.models import user, category, item, loan

# Crear todas las tablas
Base.metadata.create_all(bind=engine)

# Eliminar todas las tablas (¡CUIDADO!)
Base.metadata.drop_all(bind=engine)
```

### Inspeccionar base de datos
```python
# Ver tablas creadas
from sqlalchemy import inspect
from app.database.connection import engine

inspector = inspect(engine)
print(inspector.get_table_names())
```

## Mejores prácticas

### 1. Nomenclatura
- **Tablas**: snake_case plural (users, loan_items)
- **Columnas**: snake_case (user_id, created_at)
- **Modelos**: PascalCase singular (User, LoanItem)

### 2. Validaciones
```python
from sqlalchemy import CheckConstraint

class Item(BaseModel):
    price = Column(Numeric(10, 2), CheckConstraint('price > 0'))
```

### 3. Comentarios
```python
username = Column(
    String(50), 
    comment="Nombre de usuario único del sistema"
)
```

### 4. Valores por defecto
```python
is_active = Column(Boolean, default=True)
created_at = Column(DateTime, default=datetime.utcnow)
```

## Próximos pasos

En el siguiente tema aprenderemos sobre esquemas Pydantic, que nos permitirán validar y serializar los datos que entran y salen de nuestra API.

---

**💡 Tips importantes**:

1. **Siempre usa migraciones** en producción (Alembic)
2. **Índices en columnas frecuentemente consultadas** mejoran el rendimiento
3. **Relaciones lazy='dynamic'** para colecciones grandes
4. **Usa enums** para campos con valores limitados
5. **Comentarios en columnas** facilitan el mantenimiento

**🔗 Enlaces útiles**:
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [SQLAlchemy ORM Tutorial](https://docs.sqlalchemy.org/en/14/orm/tutorial.html)
- [FastAPI with SQLAlchemy](https://fastapi.tiangolo.com/tutorial/sql-databases/)