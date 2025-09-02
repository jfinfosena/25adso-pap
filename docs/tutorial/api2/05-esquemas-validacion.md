# 5. Esquemas de Validación con Pydantic

## ¿Qué es Pydantic?

Pydantic es una librería de Python que utiliza **type hints** para:
- **Validar datos** automáticamente
- **Serializar/Deserializar** entre JSON y objetos Python
- **Documentar** la estructura de datos
- **Generar** documentación automática de APIs

### ¿Por qué usar Pydantic?

**Sin Pydantic:**
```python
# Datos sin validar
def create_user(data):
    name = data.get('name')  # ¿Existe? ¿Es string?
    email = data.get('email')  # ¿Es email válido?
    age = data.get('age')  # ¿Es número? ¿Es positivo?
    
    # Validación manual (tedioso y propenso a errores)
    if not name or not isinstance(name, str):
        raise ValueError("Nombre inválido")
    if not email or '@' not in email:
        raise ValueError("Email inválido")
    # ... más validaciones
```

**Con Pydantic:**
```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    age: int

def create_user(user: UserCreate):
    # ¡Datos ya validados automáticamente!
    print(f"Usuario: {user.name}, Email: {user.email}")
```

## Conceptos Fundamentales

### BaseModel

Todos los esquemas heredan de `BaseModel`:

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int = None  # Opcional con valor por defecto
    name: str       # Requerido
    email: str      # Requerido
    age: int        # Requerido
```

### Type Hints

Pydantic usa las anotaciones de tipo de Python:

```python
from typing import Optional, List
from datetime import datetime

class Product(BaseModel):
    id: Optional[int] = None    # Opcional
    name: str                   # String requerido
    price: float               # Float requerido
    tags: List[str] = []       # Lista de strings, por defecto vacía
    created_at: datetime       # Datetime requerido
```

## Creación de Esquemas

### Paso 1: Esquemas Básicos

Creamos `app/schemas/schemas.py`:

```python
from pydantic import BaseModel
from typing import Optional

class User(BaseModel):
    id: Optional[int] = None
    name: str
    email: str
    
    class Config:
        from_attributes = True

class Product(BaseModel):
    id: Optional[int] = None
    name: str
    price: float
    
    class Config:
        from_attributes = True

class Order(BaseModel):
    id: Optional[int] = None
    user_id: int
    product_id: int
    quantity: int
    
    class Config:
        from_attributes = True
```

### Explicación de `Config`

```python
class Config:
    from_attributes = True
```

- **from_attributes**: Permite crear el esquema desde objetos SQLAlchemy
- **Antes se llamaba**: `orm_mode = True` (versiones anteriores)
- **Función**: Convierte `user.name` en `user['name']` automáticamente

### Ejemplo de Conversión

```python
# Objeto SQLAlchemy
user_db = User(id=1, name="Juan", email="juan@email.com")

# Convertir a esquema Pydantic
user_schema = UserSchema.from_orm(user_db)  # Versión antigua
user_schema = UserSchema.model_validate(user_db)  # Versión nueva

# Convertir a JSON
user_json = user_schema.model_dump_json()
# Resultado: '{"id": 1, "name": "Juan", "email": "juan@email.com"}'
```

## Tipos de Esquemas

### Patrón Base/Create/Response

Una práctica común es crear diferentes esquemas para diferentes propósitos:

```python
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

# Esquema base con campos comunes
class UserBase(BaseModel):
    name: str
    email: str

# Esquema para crear (sin ID)
class UserCreate(UserBase):
    password: str  # Solo para creación

# Esquema para actualizar (campos opcionales)
class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None
    password: Optional[str] = None

# Esquema para respuesta (con ID y sin password)
class UserResponse(UserBase):
    id: int
    created_at: datetime
    
    class Config:
        from_attributes = True
```

### Uso en Endpoints

```python
from fastapi import APIRouter

router = APIRouter()

@router.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate):
    # user.password está disponible aquí
    # pero no se incluirá en la respuesta
    pass

@router.put("/users/{user_id}", response_model=UserResponse)
def update_user(user_id: int, user: UserUpdate):
    # Solo los campos proporcionados se actualizarán
    pass
```

## Validaciones Avanzadas

### Validadores de Campo

```python
from pydantic import BaseModel, validator, Field
import re

class User(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    email: str
    age: int = Field(..., ge=0, le=120)  # ge = greater equal, le = less equal
    
    @validator('email')
    def validate_email(cls, v):
        if '@' not in v:
            raise ValueError('Email debe contener @')
        return v.lower()  # Convertir a minúsculas
    
    @validator('name')
    def validate_name(cls, v):
        if not v.strip():
            raise ValueError('Nombre no puede estar vacío')
        return v.strip().title()  # Capitalizar
```

### Field() para Restricciones

```python
from pydantic import BaseModel, Field
from typing import List

class Product(BaseModel):
    name: str = Field(
        ...,  # Requerido
        min_length=3,
        max_length=100,
        description="Nombre del producto"
    )
    price: float = Field(
        ...,
        gt=0,  # Greater than 0
        description="Precio en USD"
    )
    tags: List[str] = Field(
        default=[],
        max_items=10,
        description="Etiquetas del producto"
    )
    description: str = Field(
        None,  # Opcional
        max_length=1000,
        description="Descripción detallada"
    )
```

### Validadores Root

```python
from pydantic import BaseModel, root_validator

class Order(BaseModel):
    user_id: int
    product_id: int
    quantity: int
    discount: float = 0.0
    
    @root_validator
    def validate_order(cls, values):
        quantity = values.get('quantity')
        discount = values.get('discount')
        
        if quantity and quantity <= 0:
            raise ValueError('Cantidad debe ser positiva')
        
        if discount and (discount < 0 or discount > 1):
            raise ValueError('Descuento debe estar entre 0 y 1')
        
        return values
```

## Tipos de Datos Especiales

### EmailStr y HttpUrl

```python
from pydantic import BaseModel, EmailStr, HttpUrl
from typing import Optional

class User(BaseModel):
    name: str
    email: EmailStr  # Validación automática de email
    website: Optional[HttpUrl] = None  # URL válida
    
# Nota: Requiere instalar email-validator
# pip install email-validator
```

### UUID y Datetime

```python
from pydantic import BaseModel
from uuid import UUID
from datetime import datetime
from typing import Optional

class User(BaseModel):
    id: Optional[UUID] = None
    name: str
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None
```

### Enums

```python
from enum import Enum
from pydantic import BaseModel

class StatusEnum(str, Enum):
    ACTIVE = "active"
    INACTIVE = "inactive"
    PENDING = "pending"

class User(BaseModel):
    name: str
    status: StatusEnum = StatusEnum.ACTIVE
```

## Serialización y Deserialización

### De JSON a Objeto

```python
# JSON string
json_data = '{"name": "Juan", "email": "juan@email.com", "age": 25}'

# Crear objeto desde JSON
user = User.model_validate_json(json_data)
print(user.name)  # "Juan"

# Desde diccionario
dict_data = {"name": "Ana", "email": "ana@email.com", "age": 30}
user = User.model_validate(dict_data)
```

### De Objeto a JSON

```python
user = User(name="Carlos", email="carlos@email.com", age=28)

# A diccionario
user_dict = user.model_dump()
# {'name': 'Carlos', 'email': 'carlos@email.com', 'age': 28}

# A JSON string
user_json = user.model_dump_json()
# '{"name": "Carlos", "email": "carlos@email.com", "age": 28}'

# Excluir campos
user_dict = user.model_dump(exclude={'age'})
# {'name': 'Carlos', 'email': 'carlos@email.com'}

# Solo incluir campos específicos
user_dict = user.model_dump(include={'name', 'email'})
```

## Esquemas Anidados

### Relaciones Simples

```python
class Address(BaseModel):
    street: str
    city: str
    country: str

class User(BaseModel):
    name: str
    email: str
    address: Address  # Esquema anidado

# Uso
user_data = {
    "name": "Juan",
    "email": "juan@email.com",
    "address": {
        "street": "Calle 123",
        "city": "Madrid",
        "country": "España"
    }
}

user = User.model_validate(user_data)
print(user.address.city)  # "Madrid"
```

### Listas de Objetos

```python
from typing import List

class OrderItem(BaseModel):
    product_id: int
    quantity: int
    price: float

class Order(BaseModel):
    id: int
    user_id: int
    items: List[OrderItem]  # Lista de esquemas
    total: float

# Uso
order_data = {
    "id": 1,
    "user_id": 123,
    "items": [
        {"product_id": 1, "quantity": 2, "price": 10.0},
        {"product_id": 2, "quantity": 1, "price": 25.0}
    ],
    "total": 45.0
}

order = Order.model_validate(order_data)
print(len(order.items))  # 2
```

## Configuración Avanzada

### Alias de Campos

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    name: str = Field(alias="full_name")
    email: str = Field(alias="email_address")
    
    class Config:
        allow_population_by_field_name = True  # Permite usar ambos nombres

# Uso con alias
user_data = {"full_name": "Juan", "email_address": "juan@email.com"}
user = User.model_validate(user_data)

# Uso con nombre original (si allow_population_by_field_name = True)
user_data = {"name": "Juan", "email": "juan@email.com"}
user = User.model_validate(user_data)
```

### Configuraciones Útiles

```python
class User(BaseModel):
    name: str
    email: str
    
    class Config:
        # Permitir campos extra (por defecto se ignoran)
        extra = "allow"  # "allow", "ignore", "forbid"
        
        # Validar asignaciones después de la creación
        validate_assignment = True
        
        # Usar enum values en lugar de nombres
        use_enum_values = True
        
        # Permitir mutación de campos
        allow_mutation = True
        
        # Esquema JSON personalizado
        schema_extra = {
            "example": {
                "name": "Juan Pérez",
                "email": "juan@example.com"
            }
        }
```

## Manejo de Errores

### Capturar Errores de Validación

```python
from pydantic import ValidationError

try:
    user = User(name="", email="email-invalido", age=-5)
except ValidationError as e:
    print(e.json(indent=2))
    # Muestra errores detallados en JSON
```

### Errores Personalizados

```python
from pydantic import BaseModel, validator, ValidationError

class User(BaseModel):
    username: str
    password: str
    
    @validator('username')
    def username_must_be_unique(cls, v):
        # Simular verificación en base de datos
        existing_users = ["admin", "root", "test"]
        if v.lower() in existing_users:
            raise ValueError(f'Username "{v}" ya está en uso')
        return v
    
    @validator('password')
    def password_strength(cls, v):
        if len(v) < 8:
            raise ValueError('Password debe tener al menos 8 caracteres')
        if not any(c.isupper() for c in v):
            raise ValueError('Password debe tener al menos una mayúscula')
        return v
```

## Integración con FastAPI

### Documentación Automática

FastAPI usa los esquemas Pydantic para generar documentación automática:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

app = FastAPI()

class UserCreate(BaseModel):
    """Esquema para crear un usuario nuevo."""
    name: str = Field(..., description="Nombre completo del usuario")
    email: str = Field(..., description="Dirección de correo electrónico")
    age: int = Field(..., ge=0, le=120, description="Edad en años")

class UserResponse(BaseModel):
    """Esquema de respuesta con datos del usuario."""
    id: int
    name: str
    email: str
    age: int

@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate):
    """Crear un nuevo usuario en el sistema."""
    # La documentación se genera automáticamente
    pass
```

### Validación Automática

```python
@app.post("/users/")
def create_user(user: UserCreate):
    # FastAPI valida automáticamente:
    # - Tipos de datos
    # - Campos requeridos
    # - Restricciones de Field()
    # - Validadores personalizados
    
    # Si hay errores, retorna 422 automáticamente
    return {"message": "Usuario creado", "user": user}
```

## Mejores Prácticas

### 1. **Organización de Esquemas**

```python
# app/schemas/user.py
from pydantic import BaseModel
from typing import Optional

class UserBase(BaseModel):
    name: str
    email: str

class UserCreate(UserBase):
    password: str

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None

class UserInDB(UserBase):
    id: int
    hashed_password: str
    
    class Config:
        from_attributes = True

class UserResponse(UserBase):
    id: int
    
    class Config:
        from_attributes = True
```

### 2. **Reutilización de Validadores**

```python
from pydantic import validator
import re

def validate_email(email: str) -> str:
    """Validador reutilizable para emails."""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, email):
        raise ValueError('Formato de email inválido')
    return email.lower()

class User(BaseModel):
    email: str
    
    @validator('email')
    def email_validator(cls, v):
        return validate_email(v)

class Contact(BaseModel):
    email: str
    
    @validator('email')
    def email_validator(cls, v):
        return validate_email(v)
```

### 3. **Documentación Clara**

```python
from pydantic import BaseModel, Field
from typing import Optional

class Product(BaseModel):
    """Modelo de producto para el catálogo.
    
    Este esquema representa un producto en nuestro sistema de e-commerce.
    Incluye validaciones para precio y nombre.
    """
    
    name: str = Field(
        ...,
        min_length=3,
        max_length=100,
        description="Nombre del producto",
        example="iPhone 13 Pro"
    )
    
    price: float = Field(
        ...,
        gt=0,
        description="Precio en USD",
        example=999.99
    )
    
    description: Optional[str] = Field(
        None,
        max_length=1000,
        description="Descripción detallada del producto",
        example="Smartphone con cámara profesional"
    )
    
    class Config:
        schema_extra = {
            "example": {
                "name": "MacBook Pro",
                "price": 1299.99,
                "description": "Laptop profesional para desarrolladores"
            }
        }
```

## Ejercicios Prácticos

### Ejercicio 1: Esquema de Producto Completo
Crea un esquema `ProductCreate` con:
- Nombre (3-100 caracteres)
- Precio (mayor a 0)
- Categoría (enum: electronics, clothing, books)
- Tags (lista de máximo 5 strings)
- Descripción (opcional, máximo 500 caracteres)

### Ejercicio 2: Validación de Pedido
Crea un esquema `OrderCreate` que valide:
- La cantidad sea positiva
- El descuento esté entre 0% y 50%
- La fecha de entrega sea futura

### Ejercicio 3: Esquema Anidado
Crea esquemas para:
- `Address` (calle, ciudad, código postal)
- `User` que incluya una dirección
- Validar que el código postal tenga el formato correcto

## Próximos Pasos

En el siguiente capítulo aprenderemos sobre **Operaciones CRUD**, donde utilizaremos estos esquemas para:
- Validar datos antes de guardar en la base de datos
- Convertir entre modelos SQLAlchemy y esquemas Pydantic
- Manejar errores de validación en las operaciones
- Implementar lógica de negocio robusta

---

**Siguiente:** [Operaciones CRUD](06-operaciones-crud.md)

**Anterior:** [Base de Datos y Modelos](04-base-datos-modelos.md)