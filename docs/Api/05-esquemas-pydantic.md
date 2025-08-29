# Esquemas Pydantic

## Introducción

Pydantic es una librería de validación de datos que utiliza type hints de Python. En FastAPI, Pydantic se usa para:

- **Validar datos de entrada** (request body, query parameters)
- **Serializar datos de salida** (response models)
- **Documentar automáticamente** la API (OpenAPI/Swagger)
- **Convertir tipos de datos** automáticamente

## ¿Por qué usar esquemas?

### Sin esquemas (problemático)
```python
@app.post("/users/")
def create_user(user_data: dict):
    # ¿Qué campos tiene user_data?
    # ¿Son válidos los datos?
    # ¿Qué tipo de datos esperamos?
    return {"message": "Usuario creado"}
```

### Con esquemas (recomendado)
```python
@app.post("/users/", response_model=UserResponse)
def create_user(user: UserCreate):
    # Datos validados automáticamente
    # Tipos garantizados
    # Documentación automática
    return created_user
```

## Instalación

```bash
pip install pydantic[email]  # Incluye validación de email
```

## Conceptos básicos de Pydantic

### 1. Modelo básico

```python
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class User(BaseModel):
    name: str
    email: str
    age: Optional[int] = None
    is_active: bool = True
    created_at: datetime = datetime.now()

# Uso
user_data = {
    "name": "Juan Pérez",
    "email": "juan@example.com",
    "age": 25
}

user = User(**user_data)
print(user.name)  # Juan Pérez
print(user.dict())  # Convierte a diccionario
print(user.json())  # Convierte a JSON
```

### 2. Validación automática

```python
from pydantic import BaseModel, validator, EmailStr

class User(BaseModel):
    name: str
    email: EmailStr  # Valida formato de email
    age: int
    
    @validator('age')
    def validate_age(cls, v):
        if v < 0 or v > 120:
            raise ValueError('La edad debe estar entre 0 y 120')
        return v
    
    @validator('name')
    def validate_name(cls, v):
        if len(v.strip()) < 2:
            raise ValueError('El nombre debe tener al menos 2 caracteres')
        return v.strip().title()

# Esto lanzará una excepción
try:
    user = User(name="A", email="invalid-email", age=-5)
except ValueError as e:
    print(e)
```

## Esquemas para el proyecto de inventario

### 1. Esquemas base

```python
# app/schemas/base.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class BaseSchema(BaseModel):
    """
    Esquema base con configuración común.
    """
    class Config:
        # Permite crear instancias desde objetos SQLAlchemy
        from_attributes = True
        # Valida asignaciones
        validate_assignment = True
        # Usa enums por valor
        use_enum_values = True

class TimestampSchema(BaseSchema):
    """
    Esquema base con timestamps.
    """
    id: int
    created_at: datetime
    updated_at: datetime
```

### 2. Esquemas de Usuario (`app/schemas/user.py`)

```python
from pydantic import BaseModel, EmailStr, validator
from typing import Optional, List
from datetime import datetime
from app.schemas.base import BaseSchema, TimestampSchema

class UserBase(BaseSchema):
    """
    Campos base compartidos entre esquemas de usuario.
    """
    username: str
    email: EmailStr
    full_name: Optional[str] = None
    is_active: bool = True
    
    @validator('username')
    def validate_username(cls, v):
        if len(v) < 3:
            raise ValueError('El nombre de usuario debe tener al menos 3 caracteres')
        if len(v) > 50:
            raise ValueError('El nombre de usuario no puede exceder 50 caracteres')
        if not v.isalnum():
            raise ValueError('El nombre de usuario solo puede contener letras y números')
        return v.lower()
    
    @validator('full_name')
    def validate_full_name(cls, v):
        if v is not None:
            v = v.strip()
            if len(v) < 2:
                raise ValueError('El nombre completo debe tener al menos 2 caracteres')
            if len(v) > 100:
                raise ValueError('El nombre completo no puede exceder 100 caracteres')
            return v.title()
        return v

class UserCreate(UserBase):
    """
    Esquema para crear un usuario.
    Solo incluye los campos necesarios para la creación.
    """
    pass

class UserUpdate(BaseSchema):
    """
    Esquema para actualizar un usuario.
    Todos los campos son opcionales.
    """
    username: Optional[str] = None
    email: Optional[EmailStr] = None
    full_name: Optional[str] = None
    is_active: Optional[bool] = None
    
    @validator('username')
    def validate_username(cls, v):
        if v is not None:
            if len(v) < 3 or len(v) > 50 or not v.isalnum():
                raise ValueError('Nombre de usuario inválido')
            return v.lower()
        return v

class UserResponse(UserBase, TimestampSchema):
    """
    Esquema para respuestas que incluyen un usuario.
    Incluye todos los campos del usuario más timestamps.
    """
    pass

class UserWithLoans(UserResponse):
    """
    Esquema de usuario que incluye sus préstamos.
    """
    from app.schemas.loan import LoanResponse
    loans: List[LoanResponse] = []

# Esquemas para listas y paginación
class UserList(BaseSchema):
    """
    Esquema para listas de usuarios con paginación.
    """
    users: List[UserResponse]
    total: int
    page: int
    size: int
    pages: int
```

### 3. Esquemas de Categoría (`app/schemas/category.py`)

```python
from pydantic import validator
from typing import Optional, List
from app.schemas.base import BaseSchema, TimestampSchema

class CategoryBase(BaseSchema):
    """
    Campos base de categoría.
    """
    name: str
    description: Optional[str] = None
    
    @validator('name')
    def validate_name(cls, v):
        v = v.strip()
        if len(v) < 2:
            raise ValueError('El nombre debe tener al menos 2 caracteres')
        if len(v) > 100:
            raise ValueError('El nombre no puede exceder 100 caracteres')
        return v.title()
    
    @validator('description')
    def validate_description(cls, v):
        if v is not None:
            v = v.strip()
            if len(v) > 500:
                raise ValueError('La descripción no puede exceder 500 caracteres')
            return v if v else None
        return v

class CategoryCreate(CategoryBase):
    """
    Esquema para crear categoría.
    """
    pass

class CategoryUpdate(BaseSchema):
    """
    Esquema para actualizar categoría.
    """
    name: Optional[str] = None
    description: Optional[str] = None
    
    @validator('name')
    def validate_name(cls, v):
        if v is not None:
            v = v.strip()
            if len(v) < 2 or len(v) > 100:
                raise ValueError('Nombre inválido')
            return v.title()
        return v

class CategoryResponse(CategoryBase, TimestampSchema):
    """
    Esquema de respuesta de categoría.
    """
    pass

class CategoryWithItems(CategoryResponse):
    """
    Categoría con sus artículos.
    """
    from app.schemas.item import ItemResponse
    items: List[ItemResponse] = []
    items_count: int = 0
```

### 4. Esquemas de Artículo (`app/schemas/item.py`)

```python
from pydantic import validator
from typing import Optional, List
from enum import Enum
from app.schemas.base import BaseSchema, TimestampSchema

class ItemStatus(str, Enum):
    """
    Estados posibles de un artículo.
    """
    AVAILABLE = "available"
    LOANED = "loaned"
    MAINTENANCE = "maintenance"
    RETIRED = "retired"

class ItemBase(BaseSchema):
    """
    Campos base de artículo.
    """
    name: str
    description: Optional[str] = None
    serial_number: str
    status: ItemStatus = ItemStatus.AVAILABLE
    category_id: int
    
    @validator('name')
    def validate_name(cls, v):
        v = v.strip()
        if len(v) < 2:
            raise ValueError('El nombre debe tener al menos 2 caracteres')
        if len(v) > 200:
            raise ValueError('El nombre no puede exceder 200 caracteres')
        return v.title()
    
    @validator('serial_number')
    def validate_serial_number(cls, v):
        v = v.strip().upper()
        if len(v) < 3:
            raise ValueError('El número de serie debe tener al menos 3 caracteres')
        if len(v) > 100:
            raise ValueError('El número de serie no puede exceder 100 caracteres')
        # Validar formato alfanumérico
        if not v.replace('-', '').replace('_', '').isalnum():
            raise ValueError('El número de serie solo puede contener letras, números, guiones y guiones bajos')
        return v
    
    @validator('description')
    def validate_description(cls, v):
        if v is not None:
            v = v.strip()
            if len(v) > 1000:
                raise ValueError('La descripción no puede exceder 1000 caracteres')
            return v if v else None
        return v

class ItemCreate(ItemBase):
    """
    Esquema para crear artículo.
    """
    pass

class ItemUpdate(BaseSchema):
    """
    Esquema para actualizar artículo.
    """
    name: Optional[str] = None
    description: Optional[str] = None
    serial_number: Optional[str] = None
    status: Optional[ItemStatus] = None
    category_id: Optional[int] = None
    
    @validator('name')
    def validate_name(cls, v):
        if v is not None:
            v = v.strip()
            if len(v) < 2 or len(v) > 200:
                raise ValueError('Nombre inválido')
            return v.title()
        return v
    
    @validator('serial_number')
    def validate_serial_number(cls, v):
        if v is not None:
            v = v.strip().upper()
            if len(v) < 3 or len(v) > 100:
                raise ValueError('Número de serie inválido')
            return v
        return v

class ItemResponse(ItemBase, TimestampSchema):
    """
    Esquema de respuesta de artículo.
    """
    from app.schemas.category import CategoryResponse
    category: CategoryResponse

class ItemWithLoans(ItemResponse):
    """
    Artículo con sus préstamos.
    """
    from app.schemas.loan import LoanResponse
    loans: List[LoanResponse] = []
    current_loan: Optional[LoanResponse] = None

class ItemSearch(BaseSchema):
    """
    Esquema para búsqueda de artículos.
    """
    name: Optional[str] = None
    category_id: Optional[int] = None
    status: Optional[ItemStatus] = None
    serial_number: Optional[str] = None
```

### 5. Esquemas de Préstamo (`app/schemas/loan.py`)

```python
from pydantic import validator, root_validator
from typing import Optional, List
from datetime import datetime, timedelta
from enum import Enum
from app.schemas.base import BaseSchema, TimestampSchema

class LoanStatus(str, Enum):
    """
    Estados posibles de un préstamo.
    """
    ACTIVE = "active"
    RETURNED = "returned"
    OVERDUE = "overdue"
    CANCELLED = "cancelled"

class LoanBase(BaseSchema):
    """
    Campos base de préstamo.
    """
    due_date: datetime
    notes: Optional[str] = None
    user_id: int
    item_id: int
    
    @validator('due_date')
    def validate_due_date(cls, v):
        # La fecha de vencimiento debe ser futura
        if v <= datetime.utcnow():
            raise ValueError('La fecha de vencimiento debe ser futura')
        # No más de 1 año en el futuro
        max_date = datetime.utcnow() + timedelta(days=365)
        if v > max_date:
            raise ValueError('La fecha de vencimiento no puede ser más de 1 año en el futuro')
        return v
    
    @validator('notes')
    def validate_notes(cls, v):
        if v is not None:
            v = v.strip()
            if len(v) > 500:
                raise ValueError('Las notas no pueden exceder 500 caracteres')
            return v if v else None
        return v

class LoanCreate(LoanBase):
    """
    Esquema para crear préstamo.
    """
    pass

class LoanUpdate(BaseSchema):
    """
    Esquema para actualizar préstamo.
    """
    due_date: Optional[datetime] = None
    return_date: Optional[datetime] = None
    status: Optional[LoanStatus] = None
    notes: Optional[str] = None
    
    @root_validator
    def validate_return_and_status(cls, values):
        return_date = values.get('return_date')
        status = values.get('status')
        
        # Si se establece fecha de retorno, el estado debe ser RETURNED
        if return_date and status != LoanStatus.RETURNED:
            values['status'] = LoanStatus.RETURNED
        
        # Si el estado es RETURNED, debe haber fecha de retorno
        if status == LoanStatus.RETURNED and not return_date:
            values['return_date'] = datetime.utcnow()
        
        return values

class LoanReturn(BaseSchema):
    """
    Esquema específico para devolver un préstamo.
    """
    return_date: Optional[datetime] = None
    notes: Optional[str] = None
    
    @validator('return_date', pre=True, always=True)
    def set_return_date(cls, v):
        return v or datetime.utcnow()

class LoanResponse(LoanBase, TimestampSchema):
    """
    Esquema de respuesta de préstamo.
    """
    loan_date: datetime
    return_date: Optional[datetime] = None
    status: LoanStatus
    
    # Campos calculados
    is_overdue: bool = False
    days_until_due: int = 0
    
    @validator('is_overdue', pre=False, always=True)
    def calculate_is_overdue(cls, v, values):
        if values.get('status') != LoanStatus.ACTIVE:
            return False
        due_date = values.get('due_date')
        return due_date < datetime.utcnow() if due_date else False
    
    @validator('days_until_due', pre=False, always=True)
    def calculate_days_until_due(cls, v, values):
        if values.get('status') != LoanStatus.ACTIVE:
            return 0
        due_date = values.get('due_date')
        if due_date:
            delta = due_date - datetime.utcnow()
            return max(0, delta.days)
        return 0

class LoanWithDetails(LoanResponse):
    """
    Préstamo con detalles de usuario y artículo.
    """
    from app.schemas.user import UserResponse
    from app.schemas.item import ItemResponse
    
    user: UserResponse
    item: ItemResponse

class LoanSearch(BaseSchema):
    """
    Esquema para búsqueda de préstamos.
    """
    user_id: Optional[int] = None
    item_id: Optional[int] = None
    status: Optional[LoanStatus] = None
    overdue_only: bool = False
    date_from: Optional[datetime] = None
    date_to: Optional[datetime] = None
```

## Esquemas de respuesta comunes

### 1. Respuestas de error

```python
# app/schemas/common.py
from pydantic import BaseModel
from typing import Optional, List, Any

class ErrorResponse(BaseModel):
    """
    Esquema estándar para respuestas de error.
    """
    error: str
    message: str
    details: Optional[Any] = None

class ValidationErrorResponse(BaseModel):
    """
    Esquema para errores de validación.
    """
    error: str = "Validation Error"
    message: str
    errors: List[dict]

class SuccessResponse(BaseModel):
    """
    Esquema para respuestas exitosas simples.
    """
    success: bool = True
    message: str
    data: Optional[Any] = None
```

### 2. Esquemas de paginación

```python
from typing import Generic, TypeVar, List
from pydantic import BaseModel
from pydantic.generics import GenericModel

T = TypeVar('T')

class PaginatedResponse(GenericModel, Generic[T]):
    """
    Esquema genérico para respuestas paginadas.
    """
    items: List[T]
    total: int
    page: int
    size: int
    pages: int
    has_next: bool
    has_prev: bool

class PaginationParams(BaseModel):
    """
    Parámetros de paginación.
    """
    page: int = 1
    size: int = 20
    
    @validator('page')
    def validate_page(cls, v):
        if v < 1:
            raise ValueError('La página debe ser mayor a 0')
        return v
    
    @validator('size')
    def validate_size(cls, v):
        if v < 1 or v > 100:
            raise ValueError('El tamaño debe estar entre 1 y 100')
        return v
```

## Uso en endpoints

```python
# app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from app.schemas.user import UserCreate, UserResponse, UserUpdate, PaginatedResponse
from app.schemas.common import SuccessResponse, ErrorResponse
from app.database.database import get_db
from app.crud import users as crud_users

router = APIRouter()

@router.post(
    "/", 
    response_model=UserResponse, 
    status_code=status.HTTP_201_CREATED,
    responses={
        400: {"model": ErrorResponse, "description": "Datos inválidos"},
        409: {"model": ErrorResponse, "description": "Usuario ya existe"}
    }
)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    """
    Crear un nuevo usuario.
    
    - **username**: Nombre de usuario único (3-50 caracteres alfanuméricos)
    - **email**: Correo electrónico válido
    - **full_name**: Nombre completo (opcional)
    - **is_active**: Si el usuario está activo (por defecto True)
    """
    # Los datos ya están validados por Pydantic
    return crud_users.create_user(db=db, user=user)

@router.get("/", response_model=PaginatedResponse[UserResponse])
def read_users(
    page: int = 1,
    size: int = 20,
    db: Session = Depends(get_db)
):
    """
    Obtener lista paginada de usuarios.
    """
    # Validación automática de parámetros
    users = crud_users.get_users(db, skip=(page-1)*size, limit=size)
    total = crud_users.get_users_count(db)
    
    return PaginatedResponse[
        items=users,
        total=total,
        page=page,
        size=size,
        pages=(total + size - 1) // size,
        has_next=page * size < total,
        has_prev=page > 1
    ]

@router.put("/{user_id}", response_model=UserResponse)
def update_user(
    user_id: int,
    user_update: UserUpdate,
    db: Session = Depends(get_db)
):
    """
    Actualizar un usuario existente.
    """
    user = crud_users.get_user(db, user_id=user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuario no encontrado"
        )
    
    return crud_users.update_user(db=db, user=user, user_update=user_update)
```

## Validaciones avanzadas

### 1. Validadores personalizados

```python
from pydantic import validator, root_validator
import re

class UserAdvanced(BaseModel):
    username: str
    password: str
    confirm_password: str
    phone: Optional[str] = None
    
    @validator('password')
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError('La contraseña debe tener al menos 8 caracteres')
        if not re.search(r'[A-Z]', v):
            raise ValueError('La contraseña debe tener al menos una mayúscula')
        if not re.search(r'[a-z]', v):
            raise ValueError('La contraseña debe tener al menos una minúscula')
        if not re.search(r'\d', v):
            raise ValueError('La contraseña debe tener al menos un número')
        return v
    
    @validator('phone')
    def validate_phone(cls, v):
        if v is not None:
            # Remover espacios y caracteres especiales
            phone = re.sub(r'[^\d+]', '', v)
            if not re.match(r'^\+?[1-9]\d{1,14}$', phone):
                raise ValueError('Formato de teléfono inválido')
            return phone
        return v
    
    @root_validator
    def validate_passwords_match(cls, values):
        password = values.get('password')
        confirm_password = values.get('confirm_password')
        if password != confirm_password:
            raise ValueError('Las contraseñas no coinciden')
        return values
```

### 2. Validaciones condicionales

```python
class LoanAdvanced(BaseModel):
    item_id: int
    user_id: int
    due_date: datetime
    priority: str = "normal"
    
    @root_validator
    def validate_priority_and_due_date(cls, values):
        priority = values.get('priority')
        due_date = values.get('due_date')
        
        if priority == "urgent":
            # Préstamos urgentes no pueden ser por más de 7 días
            max_date = datetime.utcnow() + timedelta(days=7)
            if due_date > max_date:
                raise ValueError('Préstamos urgentes no pueden exceder 7 días')
        
        return values
```

## Configuración avanzada

### 1. Configuración de clase

```python
class UserConfig(BaseModel):
    username: str
    email: str
    
    class Config:
        # Configuraciones importantes
        from_attributes = True          # Para SQLAlchemy objects
        validate_assignment = True      # Validar al asignar valores
        use_enum_values = True         # Usar valores de enum
        allow_population_by_field_name = True  # Permitir alias
        json_encoders = {              # Encoders personalizados
            datetime: lambda v: v.isoformat()
        }
        schema_extra = {               # Ejemplo en documentación
            "example": {
                "username": "johndoe",
                "email": "john@example.com"
            }
        }
```

### 2. Alias de campos

```python
from pydantic import Field

class UserWithAlias(BaseModel):
    username: str = Field(..., alias="user_name")
    email: str = Field(..., description="Correo electrónico del usuario")
    full_name: str = Field(None, alias="fullName", max_length=100)
    
    class Config:
        allow_population_by_field_name = True
```

## Mejores prácticas

### 1. Organización de esquemas
- **Un archivo por modelo** (user.py, item.py, etc.)
- **Esquemas base reutilizables** (BaseSchema, TimestampSchema)
- **Separar por propósito** (Create, Update, Response)

### 2. Validaciones
- **Validar en el esquema**, no en el endpoint
- **Mensajes de error claros** y en español
- **Validaciones de negocio** en root_validator

### 3. Documentación
- **Usar docstrings** en esquemas y campos
- **Ejemplos en schema_extra** para documentación
- **Descripciones en Field()** para campos complejos

### 4. Performance
- **Esquemas específicos** para cada uso
- **Evitar esquemas muy anidados** en listas
- **Lazy loading** para relaciones opcionales

## Próximos pasos

En el siguiente tema aprenderemos sobre operaciones CRUD, donde utilizaremos estos esquemas para interactuar con la base de datos de manera segura y eficiente.

---

**💡 Tips importantes**:

1. **Siempre valida en el esquema** - no en el endpoint
2. **Usa type hints** - mejora la documentación y IDE
3. **Esquemas específicos** - Create, Update, Response separados
4. **Mensajes de error claros** - facilita el debugging
5. **Configuración from_attributes** - para objetos SQLAlchemy

**🔗 Enlaces útiles**:
- [Pydantic Documentation](https://pydantic-docs.helpmanual.io/)
- [FastAPI Request Body](https://fastapi.tiangolo.com/tutorial/body/)
- [FastAPI Response Model](https://fastapi.tiangolo.com/tutorial/response-model/)