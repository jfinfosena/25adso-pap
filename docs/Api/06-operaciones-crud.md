# Operaciones CRUD

## Introducción

CRUD es un acrónimo que representa las cuatro operaciones básicas de persistencia de datos:

- **C**reate (Crear)
- **R**ead (Leer)
- **U**pdate (Actualizar)
- **D**elete (Eliminar)

En FastAPI con SQLAlchemy, estas operaciones se implementan como funciones que interactúan con la base de datos usando sesiones de SQLAlchemy.

## ¿Por qué separar las operaciones CRUD?

### Ventajas de la separación

1. **Reutilización**: Las mismas operaciones pueden usarse en diferentes endpoints
2. **Testabilidad**: Fácil de testear independientemente
3. **Mantenimiento**: Lógica de base de datos centralizada
4. **Consistencia**: Operaciones estandarizadas

### Estructura recomendada

```
app/crud/
├── __init__.py
├── base.py          # Operaciones CRUD base
├── users.py         # Operaciones específicas de usuarios
├── categories.py    # Operaciones específicas de categorías
├── items.py         # Operaciones específicas de artículos
└── loans.py         # Operaciones específicas de préstamos
```

## Clase CRUD base

### Implementación base (`app/crud/base.py`)

```python
from typing import Any, Dict, Generic, List, Optional, Type, TypeVar, Union
from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel
from sqlalchemy.orm import Session
from app.database.base import BaseModel as DBBaseModel

# Tipos genéricos
ModelType = TypeVar("ModelType", bound=DBBaseModel)
CreateSchemaType = TypeVar("CreateSchemaType", bound=BaseModel)
UpdateSchemaType = TypeVar("UpdateSchemaType", bound=BaseModel)

class CRUDBase(Generic[ModelType, CreateSchemaType, UpdateSchemaType]):
    """
    Clase base para operaciones CRUD.
    
    Proporciona operaciones básicas que pueden ser heredadas
    y extendidas por clases CRUD específicas.
    """
    
    def __init__(self, model: Type[ModelType]):
        """
        Inicializar con el modelo SQLAlchemy.
        
        Args:
            model: Clase del modelo SQLAlchemy
        """
        self.model = model
    
    def get(self, db: Session, id: Any) -> Optional[ModelType]:
        """
        Obtener un registro por ID.
        
        Args:
            db: Sesión de base de datos
            id: ID del registro
            
        Returns:
            Registro encontrado o None
        """
        return db.query(self.model).filter(self.model.id == id).first()
    
    def get_multi(
        self, 
        db: Session, 
        *, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[ModelType]:
        """
        Obtener múltiples registros con paginación.
        
        Args:
            db: Sesión de base de datos
            skip: Número de registros a saltar
            limit: Número máximo de registros a retornar
            
        Returns:
            Lista de registros
        """
        return db.query(self.model).offset(skip).limit(limit).all()
    
    def create(self, db: Session, *, obj_in: CreateSchemaType) -> ModelType:
        """
        Crear un nuevo registro.
        
        Args:
            db: Sesión de base de datos
            obj_in: Datos para crear el registro
            
        Returns:
            Registro creado
        """
        obj_in_data = jsonable_encoder(obj_in)
        db_obj = self.model(**obj_in_data)
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj
    
    def update(
        self,
        db: Session,
        *,
        db_obj: ModelType,
        obj_in: Union[UpdateSchemaType, Dict[str, Any]]
    ) -> ModelType:
        """
        Actualizar un registro existente.
        
        Args:
            db: Sesión de base de datos
            db_obj: Registro existente en la base de datos
            obj_in: Datos para actualizar
            
        Returns:
            Registro actualizado
        """
        obj_data = jsonable_encoder(db_obj)
        
        if isinstance(obj_in, dict):
            update_data = obj_in
        else:
            update_data = obj_in.dict(exclude_unset=True)
        
        for field in obj_data:
            if field in update_data:
                setattr(db_obj, field, update_data[field])
        
        db.add(db_obj)
        db.commit()
        db.refresh(db_obj)
        return db_obj
    
    def remove(self, db: Session, *, id: int) -> ModelType:
        """
        Eliminar un registro por ID.
        
        Args:
            db: Sesión de base de datos
            id: ID del registro a eliminar
            
        Returns:
            Registro eliminado
        """
        obj = db.query(self.model).get(id)
        db.delete(obj)
        db.commit()
        return obj
    
    def count(self, db: Session) -> int:
        """
        Contar el total de registros.
        
        Args:
            db: Sesión de base de datos
            
        Returns:
            Número total de registros
        """
        return db.query(self.model).count()
    
    def exists(self, db: Session, id: int) -> bool:
        """
        Verificar si existe un registro con el ID dado.
        
        Args:
            db: Sesión de base de datos
            id: ID a verificar
            
        Returns:
            True si existe, False si no
        """
        return db.query(self.model).filter(self.model.id == id).first() is not None
```

## Operaciones CRUD específicas

### 1. CRUD de Usuarios (`app/crud/users.py`)

```python
from typing import Optional, List
from sqlalchemy.orm import Session
from sqlalchemy import or_

from app.crud.base import CRUDBase
from app.models.user import User
from app.schemas.user import UserCreate, UserUpdate

class CRUDUser(CRUDBase[User, UserCreate, UserUpdate]):
    """
    Operaciones CRUD específicas para usuarios.
    """
    
    def get_by_email(self, db: Session, *, email: str) -> Optional[User]:
        """
        Obtener usuario por email.
        
        Args:
            db: Sesión de base de datos
            email: Email del usuario
            
        Returns:
            Usuario encontrado o None
        """
        return db.query(User).filter(User.email == email).first()
    
    def get_by_username(self, db: Session, *, username: str) -> Optional[User]:
        """
        Obtener usuario por nombre de usuario.
        
        Args:
            db: Sesión de base de datos
            username: Nombre de usuario
            
        Returns:
            Usuario encontrado o None
        """
        return db.query(User).filter(User.username == username).first()
    
    def search_users(
        self, 
        db: Session, 
        *, 
        query: str, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[User]:
        """
        Buscar usuarios por nombre de usuario, email o nombre completo.
        
        Args:
            db: Sesión de base de datos
            query: Término de búsqueda
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de usuarios que coinciden
        """
        search_term = f"%{query}%"
        return db.query(User).filter(
            or_(
                User.username.ilike(search_term),
                User.email.ilike(search_term),
                User.full_name.ilike(search_term)
            )
        ).offset(skip).limit(limit).all()
    
    def get_active_users(
        self, 
        db: Session, 
        *, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[User]:
        """
        Obtener usuarios activos.
        
        Args:
            db: Sesión de base de datos
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de usuarios activos
        """
        return db.query(User).filter(
            User.is_active == True
        ).offset(skip).limit(limit).all()
    
    def create_user(self, db: Session, *, user: UserCreate) -> User:
        """
        Crear un nuevo usuario con validaciones adicionales.
        
        Args:
            db: Sesión de base de datos
            user: Datos del usuario a crear
            
        Returns:
            Usuario creado
            
        Raises:
            ValueError: Si el email o username ya existen
        """
        # Verificar si el email ya existe
        if self.get_by_email(db, email=user.email):
            raise ValueError("El email ya está registrado")
        
        # Verificar si el username ya existe
        if self.get_by_username(db, username=user.username):
            raise ValueError("El nombre de usuario ya está registrado")
        
        return self.create(db, obj_in=user)
    
    def update_user(
        self, 
        db: Session, 
        *, 
        db_obj: User, 
        obj_in: UserUpdate
    ) -> User:
        """
        Actualizar usuario con validaciones adicionales.
        
        Args:
            db: Sesión de base de datos
            db_obj: Usuario existente
            obj_in: Datos de actualización
            
        Returns:
            Usuario actualizado
            
        Raises:
            ValueError: Si el nuevo email o username ya existen
        """
        # Verificar email si se está actualizando
        if obj_in.email and obj_in.email != db_obj.email:
            existing_user = self.get_by_email(db, email=obj_in.email)
            if existing_user and existing_user.id != db_obj.id:
                raise ValueError("El email ya está registrado")
        
        # Verificar username si se está actualizando
        if obj_in.username and obj_in.username != db_obj.username:
            existing_user = self.get_by_username(db, username=obj_in.username)
            if existing_user and existing_user.id != db_obj.id:
                raise ValueError("El nombre de usuario ya está registrado")
        
        return self.update(db, db_obj=db_obj, obj_in=obj_in)
    
    def deactivate_user(self, db: Session, *, user_id: int) -> User:
        """
        Desactivar un usuario (soft delete).
        
        Args:
            db: Sesión de base de datos
            user_id: ID del usuario a desactivar
            
        Returns:
            Usuario desactivado
        """
        user = self.get(db, id=user_id)
        if user:
            user.is_active = False
            db.commit()
            db.refresh(user)
        return user

# Instancia global
user = CRUDUser(User)
```

### 2. CRUD de Categorías (`app/crud/categories.py`)

```python
from typing import Optional, List
from sqlalchemy.orm import Session
from sqlalchemy import func

from app.crud.base import CRUDBase
from app.models.category import Category
from app.schemas.category import CategoryCreate, CategoryUpdate

class CRUDCategory(CRUDBase[Category, CategoryCreate, CategoryUpdate]):
    """
    Operaciones CRUD específicas para categorías.
    """
    
    def get_by_name(self, db: Session, *, name: str) -> Optional[Category]:
        """
        Obtener categoría por nombre.
        
        Args:
            db: Sesión de base de datos
            name: Nombre de la categoría
            
        Returns:
            Categoría encontrada o None
        """
        return db.query(Category).filter(
            func.lower(Category.name) == func.lower(name)
        ).first()
    
    def search_categories(
        self, 
        db: Session, 
        *, 
        query: str, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Category]:
        """
        Buscar categorías por nombre o descripción.
        
        Args:
            db: Sesión de base de datos
            query: Término de búsqueda
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de categorías que coinciden
        """
        search_term = f"%{query}%"
        return db.query(Category).filter(
            Category.name.ilike(search_term) |
            Category.description.ilike(search_term)
        ).offset(skip).limit(limit).all()
    
    def get_categories_with_item_count(
        self, 
        db: Session, 
        *, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[tuple]:
        """
        Obtener categorías con el conteo de artículos.
        
        Args:
            db: Sesión de base de datos
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de tuplas (Category, item_count)
        """
        from app.models.item import Item
        
        return db.query(
            Category, 
            func.count(Item.id).label('item_count')
        ).outerjoin(Item).group_by(Category.id).offset(skip).limit(limit).all()
    
    def create_category(self, db: Session, *, category: CategoryCreate) -> Category:
        """
        Crear una nueva categoría con validaciones.
        
        Args:
            db: Sesión de base de datos
            category: Datos de la categoría
            
        Returns:
            Categoría creada
            
        Raises:
            ValueError: Si el nombre ya existe
        """
        # Verificar si el nombre ya existe
        if self.get_by_name(db, name=category.name):
            raise ValueError("Ya existe una categoría con ese nombre")
        
        return self.create(db, obj_in=category)
    
    def update_category(
        self, 
        db: Session, 
        *, 
        db_obj: Category, 
        obj_in: CategoryUpdate
    ) -> Category:
        """
        Actualizar categoría con validaciones.
        
        Args:
            db: Sesión de base de datos
            db_obj: Categoría existente
            obj_in: Datos de actualización
            
        Returns:
            Categoría actualizada
            
        Raises:
            ValueError: Si el nuevo nombre ya existe
        """
        # Verificar nombre si se está actualizando
        if obj_in.name and obj_in.name != db_obj.name:
            existing_category = self.get_by_name(db, name=obj_in.name)
            if existing_category and existing_category.id != db_obj.id:
                raise ValueError("Ya existe una categoría con ese nombre")
        
        return self.update(db, db_obj=db_obj, obj_in=obj_in)
    
    def can_delete(self, db: Session, *, category_id: int) -> bool:
        """
        Verificar si una categoría puede ser eliminada.
        
        Args:
            db: Sesión de base de datos
            category_id: ID de la categoría
            
        Returns:
            True si puede ser eliminada, False si no
        """
        from app.models.item import Item
        
        item_count = db.query(Item).filter(
            Item.category_id == category_id
        ).count()
        
        return item_count == 0

# Instancia global
category = CRUDCategory(Category)
```

### 3. CRUD de Artículos (`app/crud/items.py`)

```python
from typing import Optional, List
from sqlalchemy.orm import Session, joinedload
from sqlalchemy import or_, and_

from app.crud.base import CRUDBase
from app.models.item import Item, ItemStatus
from app.schemas.item import ItemCreate, ItemUpdate, ItemSearch

class CRUDItem(CRUDBase[Item, ItemCreate, ItemUpdate]):
    """
    Operaciones CRUD específicas para artículos.
    """
    
    def get_by_serial_number(
        self, 
        db: Session, 
        *, 
        serial_number: str
    ) -> Optional[Item]:
        """
        Obtener artículo por número de serie.
        
        Args:
            db: Sesión de base de datos
            serial_number: Número de serie del artículo
            
        Returns:
            Artículo encontrado o None
        """
        return db.query(Item).filter(
            Item.serial_number == serial_number
        ).first()
    
    def get_with_category(self, db: Session, *, item_id: int) -> Optional[Item]:
        """
        Obtener artículo con información de categoría cargada.
        
        Args:
            db: Sesión de base de datos
            item_id: ID del artículo
            
        Returns:
            Artículo con categoría cargada o None
        """
        return db.query(Item).options(
            joinedload(Item.category)
        ).filter(Item.id == item_id).first()
    
    def get_available_items(
        self, 
        db: Session, 
        *, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Item]:
        """
        Obtener artículos disponibles para préstamo.
        
        Args:
            db: Sesión de base de datos
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de artículos disponibles
        """
        return db.query(Item).filter(
            Item.status == ItemStatus.AVAILABLE
        ).offset(skip).limit(limit).all()
    
    def get_by_category(
        self, 
        db: Session, 
        *, 
        category_id: int, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Item]:
        """
        Obtener artículos por categoría.
        
        Args:
            db: Sesión de base de datos
            category_id: ID de la categoría
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de artículos de la categoría
        """
        return db.query(Item).filter(
            Item.category_id == category_id
        ).offset(skip).limit(limit).all()
    
    def search_items(
        self, 
        db: Session, 
        *, 
        search: ItemSearch, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Item]:
        """
        Buscar artículos con múltiples criterios.
        
        Args:
            db: Sesión de base de datos
            search: Criterios de búsqueda
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de artículos que coinciden
        """
        query = db.query(Item)
        
        # Filtrar por nombre
        if search.name:
            query = query.filter(Item.name.ilike(f"%{search.name}%"))
        
        # Filtrar por categoría
        if search.category_id:
            query = query.filter(Item.category_id == search.category_id)
        
        # Filtrar por estado
        if search.status:
            query = query.filter(Item.status == search.status)
        
        # Filtrar por número de serie
        if search.serial_number:
            query = query.filter(
                Item.serial_number.ilike(f"%{search.serial_number}%")
            )
        
        return query.offset(skip).limit(limit).all()
    
    def create_item(self, db: Session, *, item: ItemCreate) -> Item:
        """
        Crear un nuevo artículo con validaciones.
        
        Args:
            db: Sesión de base de datos
            item: Datos del artículo
            
        Returns:
            Artículo creado
            
        Raises:
            ValueError: Si el número de serie ya existe o la categoría no existe
        """
        # Verificar si el número de serie ya existe
        if self.get_by_serial_number(db, serial_number=item.serial_number):
            raise ValueError("Ya existe un artículo con ese número de serie")
        
        # Verificar si la categoría existe
        from app.crud.categories import category
        if not category.get(db, id=item.category_id):
            raise ValueError("La categoría especificada no existe")
        
        return self.create(db, obj_in=item)
    
    def update_item(
        self, 
        db: Session, 
        *, 
        db_obj: Item, 
        obj_in: ItemUpdate
    ) -> Item:
        """
        Actualizar artículo con validaciones.
        
        Args:
            db: Sesión de base de datos
            db_obj: Artículo existente
            obj_in: Datos de actualización
            
        Returns:
            Artículo actualizado
            
        Raises:
            ValueError: Si el nuevo número de serie ya existe
        """
        # Verificar número de serie si se está actualizando
        if obj_in.serial_number and obj_in.serial_number != db_obj.serial_number:
            existing_item = self.get_by_serial_number(
                db, serial_number=obj_in.serial_number
            )
            if existing_item and existing_item.id != db_obj.id:
                raise ValueError("Ya existe un artículo con ese número de serie")
        
        # Verificar categoría si se está actualizando
        if obj_in.category_id and obj_in.category_id != db_obj.category_id:
            from app.crud.categories import category
            if not category.get(db, id=obj_in.category_id):
                raise ValueError("La categoría especificada no existe")
        
        return self.update(db, db_obj=db_obj, obj_in=obj_in)
    
    def update_status(self, db: Session, *, item_id: int, status: ItemStatus) -> Item:
        """
        Actualizar solo el estado de un artículo.
        
        Args:
            db: Sesión de base de datos
            item_id: ID del artículo
            status: Nuevo estado
            
        Returns:
            Artículo actualizado
        """
        item = self.get(db, id=item_id)
        if item:
            item.status = status
            db.commit()
            db.refresh(item)
        return item
    
    def can_delete(self, db: Session, *, item_id: int) -> bool:
        """
        Verificar si un artículo puede ser eliminado.
        
        Args:
            db: Sesión de base de datos
            item_id: ID del artículo
            
        Returns:
            True si puede ser eliminado, False si no
        """
        from app.models.loan import Loan, LoanStatus
        
        # No se puede eliminar si tiene préstamos activos
        active_loans = db.query(Loan).filter(
            and_(
                Loan.item_id == item_id,
                Loan.status == LoanStatus.ACTIVE
            )
        ).count()
        
        return active_loans == 0

# Instancia global
item = CRUDItem(Item)
```

### 4. CRUD de Préstamos (`app/crud/loans.py`)

```python
from typing import Optional, List
from datetime import datetime, timedelta
from sqlalchemy.orm import Session, joinedload
from sqlalchemy import and_, or_

from app.crud.base import CRUDBase
from app.models.loan import Loan, LoanStatus
from app.models.item import ItemStatus
from app.schemas.loan import LoanCreate, LoanUpdate, LoanReturn

class CRUDLoan(CRUDBase[Loan, LoanCreate, LoanUpdate]):
    """
    Operaciones CRUD específicas para préstamos.
    """
    
    def get_with_details(self, db: Session, *, loan_id: int) -> Optional[Loan]:
        """
        Obtener préstamo con detalles de usuario y artículo.
        
        Args:
            db: Sesión de base de datos
            loan_id: ID del préstamo
            
        Returns:
            Préstamo con detalles cargados o None
        """
        return db.query(Loan).options(
            joinedload(Loan.user),
            joinedload(Loan.item).joinedload(Item.category)
        ).filter(Loan.id == loan_id).first()
    
    def get_active_loans(
        self, 
        db: Session, 
        *, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Loan]:
        """
        Obtener préstamos activos.
        
        Args:
            db: Sesión de base de datos
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de préstamos activos
        """
        return db.query(Loan).filter(
            Loan.status == LoanStatus.ACTIVE
        ).offset(skip).limit(limit).all()
    
    def get_overdue_loans(
        self, 
        db: Session, 
        *, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Loan]:
        """
        Obtener préstamos vencidos.
        
        Args:
            db: Sesión de base de datos
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de préstamos vencidos
        """
        now = datetime.utcnow()
        return db.query(Loan).filter(
            and_(
                Loan.status == LoanStatus.ACTIVE,
                Loan.due_date < now
            )
        ).offset(skip).limit(limit).all()
    
    def get_by_user(
        self, 
        db: Session, 
        *, 
        user_id: int, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Loan]:
        """
        Obtener préstamos de un usuario.
        
        Args:
            db: Sesión de base de datos
            user_id: ID del usuario
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de préstamos del usuario
        """
        return db.query(Loan).filter(
            Loan.user_id == user_id
        ).offset(skip).limit(limit).all()
    
    def get_by_item(
        self, 
        db: Session, 
        *, 
        item_id: int, 
        skip: int = 0, 
        limit: int = 100
    ) -> List[Loan]:
        """
        Obtener préstamos de un artículo.
        
        Args:
            db: Sesión de base de datos
            item_id: ID del artículo
            skip: Registros a saltar
            limit: Límite de registros
            
        Returns:
            Lista de préstamos del artículo
        """
        return db.query(Loan).filter(
            Loan.item_id == item_id
        ).offset(skip).limit(limit).all()
    
    def create_loan(self, db: Session, *, loan: LoanCreate) -> Loan:
        """
        Crear un nuevo préstamo con validaciones.
        
        Args:
            db: Sesión de base de datos
            loan: Datos del préstamo
            
        Returns:
            Préstamo creado
            
        Raises:
            ValueError: Si el artículo no está disponible o el usuario no existe
        """
        from app.crud.users import user as crud_user
        from app.crud.items import item as crud_item
        
        # Verificar que el usuario existe
        if not crud_user.get(db, id=loan.user_id):
            raise ValueError("El usuario especificado no existe")
        
        # Verificar que el artículo existe y está disponible
        item_obj = crud_item.get(db, id=loan.item_id)
        if not item_obj:
            raise ValueError("El artículo especificado no existe")
        
        if item_obj.status != ItemStatus.AVAILABLE:
            raise ValueError("El artículo no está disponible para préstamo")
        
        # Verificar que el usuario no tenga préstamos vencidos
        overdue_count = db.query(Loan).filter(
            and_(
                Loan.user_id == loan.user_id,
                Loan.status == LoanStatus.ACTIVE,
                Loan.due_date < datetime.utcnow()
            )
        ).count()
        
        if overdue_count > 0:
            raise ValueError(
                "El usuario tiene préstamos vencidos y no puede solicitar nuevos préstamos"
            )
        
        # Crear el préstamo
        db_loan = self.create(db, obj_in=loan)
        
        # Actualizar estado del artículo
        crud_item.update_status(db, item_id=loan.item_id, status=ItemStatus.LOANED)
        
        return db_loan
    
    def return_loan(
        self, 
        db: Session, 
        *, 
        loan_id: int, 
        return_data: LoanReturn
    ) -> Loan:
        """
        Procesar la devolución de un préstamo.
        
        Args:
            db: Sesión de base de datos
            loan_id: ID del préstamo
            return_data: Datos de la devolución
            
        Returns:
            Préstamo actualizado
            
        Raises:
            ValueError: Si el préstamo no existe o ya fue devuelto
        """
        from app.crud.items import item as crud_item
        
        # Obtener el préstamo
        loan = self.get(db, id=loan_id)
        if not loan:
            raise ValueError("El préstamo especificado no existe")
        
        if loan.status != LoanStatus.ACTIVE:
            raise ValueError("El préstamo ya fue devuelto o cancelado")
        
        # Actualizar el préstamo
        loan.return_date = return_data.return_date
        loan.status = LoanStatus.RETURNED
        if return_data.notes:
            loan.notes = return_data.notes
        
        db.commit()
        db.refresh(loan)
        
        # Actualizar estado del artículo
        crud_item.update_status(
            db, 
            item_id=loan.item_id, 
            status=ItemStatus.AVAILABLE
        )
        
        return loan
    
    def extend_loan(
        self, 
        db: Session, 
        *, 
        loan_id: int, 
        new_due_date: datetime
    ) -> Loan:
        """
        Extender la fecha de vencimiento de un préstamo.
        
        Args:
            db: Sesión de base de datos
            loan_id: ID del préstamo
            new_due_date: Nueva fecha de vencimiento
            
        Returns:
            Préstamo actualizado
            
        Raises:
            ValueError: Si el préstamo no puede ser extendido
        """
        loan = self.get(db, id=loan_id)
        if not loan:
            raise ValueError("El préstamo especificado no existe")
        
        if loan.status != LoanStatus.ACTIVE:
            raise ValueError("Solo se pueden extender préstamos activos")
        
        if new_due_date <= datetime.utcnow():
            raise ValueError("La nueva fecha debe ser futura")
        
        if new_due_date <= loan.due_date:
            raise ValueError("La nueva fecha debe ser posterior a la actual")
        
        # Límite máximo de extensión (ej: 30 días desde hoy)
        max_extension = datetime.utcnow() + timedelta(days=30)
        if new_due_date > max_extension:
            raise ValueError("La extensión no puede ser mayor a 30 días")
        
        loan.due_date = new_due_date
        db.commit()
        db.refresh(loan)
        
        return loan
    
    def update_overdue_status(self, db: Session) -> int:
        """
        Actualizar el estado de préstamos vencidos.
        
        Args:
            db: Sesión de base de datos
            
        Returns:
            Número de préstamos actualizados
        """
        now = datetime.utcnow()
        
        # Obtener préstamos activos vencidos
        overdue_loans = db.query(Loan).filter(
            and_(
                Loan.status == LoanStatus.ACTIVE,
                Loan.due_date < now
            )
        ).all()
        
        # Actualizar estado
        for loan in overdue_loans:
            loan.status = LoanStatus.OVERDUE
        
        db.commit()
        
        return len(overdue_loans)

# Instancia global
loan = CRUDLoan(Loan)
```

## Uso en endpoints

### Ejemplo de uso en router

```python
# app/routers/items.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List

from app.crud.items import item as crud_item
from app.schemas.item import ItemCreate, ItemResponse, ItemUpdate
from app.database.database import get_db

router = APIRouter()

@router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
def create_item(item: ItemCreate, db: Session = Depends(get_db)):
    """
    Crear un nuevo artículo.
    """
    try:
        return crud_item.create_item(db=db, item=item)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.get("/", response_model=List[ItemResponse])
def read_items(
    skip: int = 0,
    limit: int = 100,
    db: Session = Depends(get_db)
):
    """
    Obtener lista de artículos.
    """
    return crud_item.get_multi(db, skip=skip, limit=limit)

@router.get("/{item_id}", response_model=ItemResponse)
def read_item(item_id: int, db: Session = Depends(get_db)):
    """
    Obtener un artículo por ID.
    """
    item = crud_item.get_with_category(db, item_id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    return item

@router.put("/{item_id}", response_model=ItemResponse)
def update_item(
    item_id: int,
    item_update: ItemUpdate,
    db: Session = Depends(get_db)
):
    """
    Actualizar un artículo.
    """
    item = crud_item.get(db, id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    
    try:
        return crud_item.update_item(db=db, db_obj=item, obj_in=item_update)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.delete("/{item_id}")
def delete_item(item_id: int, db: Session = Depends(get_db)):
    """
    Eliminar un artículo.
    """
    if not crud_item.can_delete(db, item_id=item_id):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="No se puede eliminar el artículo porque tiene préstamos activos"
        )
    
    item = crud_item.remove(db, id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    
    return {"message": "Artículo eliminado exitosamente"}
```

## Mejores prácticas

### 1. Validaciones en CRUD
- **Validar existencia** de registros relacionados
- **Verificar unicidad** de campos únicos
- **Validar reglas de negocio** antes de operaciones

### 2. Manejo de errores
- **Usar excepciones específicas** para diferentes errores
- **Mensajes descriptivos** para facilitar debugging
- **Validar antes de modificar** la base de datos

### 3. Performance
- **Usar joinedload** para cargar relaciones necesarias
- **Paginación** en consultas que retornan listas
- **Índices** en columnas frecuentemente consultadas

### 4. Transacciones
- **Operaciones atómicas** para cambios relacionados
- **Rollback automático** en caso de errores
- **Commit explícito** después de validaciones

## Próximos pasos

En el siguiente tema aprenderemos sobre la creación de endpoints y routers, donde utilizaremos estas operaciones CRUD para construir nuestra API REST.

---

**💡 Tips importantes**:

1. **Separar lógica de negocio** - mantener en CRUD, no en endpoints
2. **Validaciones tempranas** - verificar antes de modificar datos
3. **Operaciones atómicas** - usar transacciones para cambios relacionados
4. **Mensajes de error claros** - facilita el debugging y UX
5. **Instancias globales** - reutilizar objetos CRUD en toda la aplicación

**🔗 Enlaces útiles**:
- [SQLAlchemy ORM Querying](https://docs.sqlalchemy.org/en/14/orm/query.html)
- [FastAPI Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [Pydantic Models](https://pydantic-docs.helpmanual.io/usage/models/)