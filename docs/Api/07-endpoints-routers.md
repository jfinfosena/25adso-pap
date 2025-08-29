# Endpoints y Routers

## Introducción

Los endpoints son las URLs específicas donde tu API puede recibir solicitudes. En FastAPI, los routers nos permiten organizar estos endpoints de manera modular y escalable.

## ¿Qué son los Routers?

Un router en FastAPI es una forma de agrupar endpoints relacionados. Esto nos permite:

- **Organizar código** por funcionalidad
- **Reutilizar configuraciones** comunes
- **Aplicar middleware** específico
- **Mantener el código** más limpio y mantenible

### Ventajas de usar Routers

1. **Modularidad**: Cada router maneja una entidad específica
2. **Escalabilidad**: Fácil agregar nuevas funcionalidades
3. **Mantenimiento**: Código organizado y fácil de encontrar
4. **Reutilización**: Configuraciones comunes compartidas
5. **Testing**: Fácil testear endpoints específicos

## Estructura de Routers

### Organización recomendada

```
app/routers/
├── __init__.py
├── users.py        # Endpoints de usuarios
├── categories.py   # Endpoints de categorías
├── items.py        # Endpoints de artículos
└── loans.py        # Endpoints de préstamos
```

## Implementación de Routers

### 1. Router de Usuarios (`app/routers/users.py`)

```python
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from app.crud.users import user as crud_user
from app.schemas.user import (
    UserCreate, 
    UserResponse, 
    UserUpdate, 
    UserSearch
)
from app.database.database import get_db
from app.schemas.common import PaginatedResponse

# Crear el router
router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Usuario no encontrado"}}
)

@router.post(
    "/", 
    response_model=UserResponse, 
    status_code=status.HTTP_201_CREATED,
    summary="Crear usuario",
    description="Crear un nuevo usuario en el sistema"
)
def create_user(
    user: UserCreate, 
    db: Session = Depends(get_db)
):
    """
    Crear un nuevo usuario.
    
    - **username**: Nombre de usuario único
    - **email**: Email único del usuario
    - **full_name**: Nombre completo del usuario
    - **phone**: Teléfono del usuario (opcional)
    """
    try:
        return crud_user.create_user(db=db, user=user)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.get(
    "/", 
    response_model=PaginatedResponse[UserResponse],
    summary="Listar usuarios",
    description="Obtener lista paginada de usuarios"
)
def read_users(
    skip: int = Query(0, ge=0, description="Número de registros a saltar"),
    limit: int = Query(100, ge=1, le=1000, description="Número máximo de registros"),
    active_only: bool = Query(False, description="Solo usuarios activos"),
    db: Session = Depends(get_db)
):
    """
    Obtener lista de usuarios con paginación.
    
    - **skip**: Número de registros a saltar para paginación
    - **limit**: Número máximo de registros a retornar
    - **active_only**: Si es True, solo retorna usuarios activos
    """
    if active_only:
        users = crud_user.get_active_users(db, skip=skip, limit=limit)
    else:
        users = crud_user.get_multi(db, skip=skip, limit=limit)
    
    total = crud_user.count(db)
    
    return PaginatedResponse(
        items=users,
        total=total,
        skip=skip,
        limit=limit
    )

@router.get(
    "/search",
    response_model=List[UserResponse],
    summary="Buscar usuarios",
    description="Buscar usuarios por nombre, email o username"
)
def search_users(
    q: str = Query(..., min_length=2, description="Término de búsqueda"),
    skip: int = Query(0, ge=0),
    limit: int = Query(50, ge=1, le=100),
    db: Session = Depends(get_db)
):
    """
    Buscar usuarios por diferentes criterios.
    
    - **q**: Término de búsqueda (mínimo 2 caracteres)
    - **skip**: Registros a saltar
    - **limit**: Límite de registros
    """
    return crud_user.search_users(db, query=q, skip=skip, limit=limit)

@router.get(
    "/{user_id}",
    response_model=UserResponse,
    summary="Obtener usuario",
    description="Obtener un usuario específico por ID"
)
def read_user(
    user_id: int, 
    db: Session = Depends(get_db)
):
    """
    Obtener un usuario específico por ID.
    
    - **user_id**: ID único del usuario
    """
    user = crud_user.get(db, id=user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuario no encontrado"
        )
    return user

@router.put(
    "/{user_id}",
    response_model=UserResponse,
    summary="Actualizar usuario",
    description="Actualizar información de un usuario existente"
)
def update_user(
    user_id: int,
    user_update: UserUpdate,
    db: Session = Depends(get_db)
):
    """
    Actualizar un usuario existente.
    
    - **user_id**: ID del usuario a actualizar
    - Solo se actualizarán los campos proporcionados
    """
    user = crud_user.get(db, id=user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuario no encontrado"
        )
    
    try:
        return crud_user.update_user(db=db, db_obj=user, obj_in=user_update)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.delete(
    "/{user_id}",
    summary="Desactivar usuario",
    description="Desactivar un usuario (soft delete)"
)
def deactivate_user(
    user_id: int,
    db: Session = Depends(get_db)
):
    """
    Desactivar un usuario (no se elimina físicamente).
    
    - **user_id**: ID del usuario a desactivar
    """
    user = crud_user.get(db, id=user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuario no encontrado"
        )
    
    crud_user.deactivate_user(db, user_id=user_id)
    return {"message": "Usuario desactivado exitosamente"}

@router.get(
    "/{user_id}/loans",
    response_model=List[dict],  # Usaremos dict por simplicidad
    summary="Préstamos del usuario",
    description="Obtener todos los préstamos de un usuario"
)
def get_user_loans(
    user_id: int,
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener préstamos de un usuario específico.
    
    - **user_id**: ID del usuario
    - **skip**: Registros a saltar
    - **limit**: Límite de registros
    """
    # Verificar que el usuario existe
    user = crud_user.get(db, id=user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuario no encontrado"
        )
    
    from app.crud.loans import loan as crud_loan
    return crud_loan.get_by_user(db, user_id=user_id, skip=skip, limit=limit)
```

### 2. Router de Categorías (`app/routers/categories.py`)

```python
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List

from app.crud.categories import category as crud_category
from app.schemas.category import (
    CategoryCreate, 
    CategoryResponse, 
    CategoryUpdate,
    CategoryWithItemCount
)
from app.database.database import get_db

router = APIRouter(
    prefix="/categories",
    tags=["categories"],
    responses={404: {"description": "Categoría no encontrada"}}
)

@router.post(
    "/", 
    response_model=CategoryResponse, 
    status_code=status.HTTP_201_CREATED,
    summary="Crear categoría",
    description="Crear una nueva categoría de artículos"
)
def create_category(
    category: CategoryCreate, 
    db: Session = Depends(get_db)
):
    """
    Crear una nueva categoría.
    
    - **name**: Nombre único de la categoría
    - **description**: Descripción de la categoría (opcional)
    """
    try:
        return crud_category.create_category(db=db, category=category)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.get(
    "/", 
    response_model=List[CategoryResponse],
    summary="Listar categorías",
    description="Obtener lista de todas las categorías"
)
def read_categories(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener lista de categorías.
    
    - **skip**: Número de registros a saltar
    - **limit**: Número máximo de registros
    """
    return crud_category.get_multi(db, skip=skip, limit=limit)

@router.get(
    "/with-counts",
    response_model=List[CategoryWithItemCount],
    summary="Categorías con conteo",
    description="Obtener categorías con el número de artículos en cada una"
)
def read_categories_with_counts(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener categorías con el conteo de artículos.
    
    Útil para mostrar estadísticas en dashboards.
    """
    categories_with_counts = crud_category.get_categories_with_item_count(
        db, skip=skip, limit=limit
    )
    
    return [
        CategoryWithItemCount(
            **category.__dict__,
            item_count=count
        )
        for category, count in categories_with_counts
    ]

@router.get(
    "/search",
    response_model=List[CategoryResponse],
    summary="Buscar categorías",
    description="Buscar categorías por nombre o descripción"
)
def search_categories(
    q: str = Query(..., min_length=2, description="Término de búsqueda"),
    skip: int = Query(0, ge=0),
    limit: int = Query(50, ge=1, le=100),
    db: Session = Depends(get_db)
):
    """
    Buscar categorías por nombre o descripción.
    
    - **q**: Término de búsqueda (mínimo 2 caracteres)
    """
    return crud_category.search_categories(db, query=q, skip=skip, limit=limit)

@router.get(
    "/{category_id}",
    response_model=CategoryResponse,
    summary="Obtener categoría",
    description="Obtener una categoría específica por ID"
)
def read_category(
    category_id: int, 
    db: Session = Depends(get_db)
):
    """
    Obtener una categoría específica por ID.
    
    - **category_id**: ID único de la categoría
    """
    category = crud_category.get(db, id=category_id)
    if not category:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Categoría no encontrada"
        )
    return category

@router.put(
    "/{category_id}",
    response_model=CategoryResponse,
    summary="Actualizar categoría",
    description="Actualizar información de una categoría existente"
)
def update_category(
    category_id: int,
    category_update: CategoryUpdate,
    db: Session = Depends(get_db)
):
    """
    Actualizar una categoría existente.
    
    - **category_id**: ID de la categoría a actualizar
    - Solo se actualizarán los campos proporcionados
    """
    category = crud_category.get(db, id=category_id)
    if not category:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Categoría no encontrada"
        )
    
    try:
        return crud_category.update_category(
            db=db, db_obj=category, obj_in=category_update
        )
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.delete(
    "/{category_id}",
    summary="Eliminar categoría",
    description="Eliminar una categoría (solo si no tiene artículos)"
)
def delete_category(
    category_id: int,
    db: Session = Depends(get_db)
):
    """
    Eliminar una categoría.
    
    Solo se puede eliminar si no tiene artículos asociados.
    
    - **category_id**: ID de la categoría a eliminar
    """
    category = crud_category.get(db, id=category_id)
    if not category:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Categoría no encontrada"
        )
    
    if not crud_category.can_delete(db, category_id=category_id):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="No se puede eliminar la categoría porque tiene artículos asociados"
        )
    
    crud_category.remove(db, id=category_id)
    return {"message": "Categoría eliminada exitosamente"}

@router.get(
    "/{category_id}/items",
    response_model=List[dict],  # Simplificado
    summary="Artículos de la categoría",
    description="Obtener todos los artículos de una categoría"
)
def get_category_items(
    category_id: int,
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener artículos de una categoría específica.
    
    - **category_id**: ID de la categoría
    - **skip**: Registros a saltar
    - **limit**: Límite de registros
    """
    # Verificar que la categoría existe
    category = crud_category.get(db, id=category_id)
    if not category:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Categoría no encontrada"
        )
    
    from app.crud.items import item as crud_item
    return crud_item.get_by_category(
        db, category_id=category_id, skip=skip, limit=limit
    )
```

### 3. Router de Artículos (`app/routers/items.py`)

```python
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List, Optional

from app.crud.items import item as crud_item
from app.schemas.item import (
    ItemCreate, 
    ItemResponse, 
    ItemUpdate, 
    ItemSearch,
    ItemStatus
)
from app.database.database import get_db

router = APIRouter(
    prefix="/items",
    tags=["items"],
    responses={404: {"description": "Artículo no encontrado"}}
)

@router.post(
    "/", 
    response_model=ItemResponse, 
    status_code=status.HTTP_201_CREATED,
    summary="Crear artículo",
    description="Crear un nuevo artículo en el inventario"
)
def create_item(
    item: ItemCreate, 
    db: Session = Depends(get_db)
):
    """
    Crear un nuevo artículo.
    
    - **name**: Nombre del artículo
    - **description**: Descripción detallada
    - **serial_number**: Número de serie único
    - **category_id**: ID de la categoría
    - **purchase_price**: Precio de compra (opcional)
    - **purchase_date**: Fecha de compra (opcional)
    """
    try:
        return crud_item.create_item(db=db, item=item)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.get(
    "/", 
    response_model=List[ItemResponse],
    summary="Listar artículos",
    description="Obtener lista de artículos con filtros opcionales"
)
def read_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    category_id: Optional[int] = Query(None, description="Filtrar por categoría"),
    status: Optional[ItemStatus] = Query(None, description="Filtrar por estado"),
    available_only: bool = Query(False, description="Solo artículos disponibles"),
    db: Session = Depends(get_db)
):
    """
    Obtener lista de artículos con filtros opcionales.
    
    - **skip**: Registros a saltar
    - **limit**: Límite de registros
    - **category_id**: Filtrar por categoría específica
    - **status**: Filtrar por estado específico
    - **available_only**: Solo artículos disponibles para préstamo
    """
    if available_only:
        return crud_item.get_available_items(db, skip=skip, limit=limit)
    elif category_id:
        return crud_item.get_by_category(
            db, category_id=category_id, skip=skip, limit=limit
        )
    else:
        return crud_item.get_multi(db, skip=skip, limit=limit)

@router.get(
    "/search",
    response_model=List[ItemResponse],
    summary="Buscar artículos",
    description="Buscar artículos con múltiples criterios"
)
def search_items(
    name: Optional[str] = Query(None, description="Buscar por nombre"),
    serial_number: Optional[str] = Query(None, description="Buscar por número de serie"),
    category_id: Optional[int] = Query(None, description="Filtrar por categoría"),
    status: Optional[ItemStatus] = Query(None, description="Filtrar por estado"),
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Buscar artículos con múltiples criterios.
    
    Permite combinar diferentes filtros para búsquedas específicas.
    """
    search_criteria = ItemSearch(
        name=name,
        serial_number=serial_number,
        category_id=category_id,
        status=status
    )
    
    return crud_item.search_items(
        db, search=search_criteria, skip=skip, limit=limit
    )

@router.get(
    "/available",
    response_model=List[ItemResponse],
    summary="Artículos disponibles",
    description="Obtener solo artículos disponibles para préstamo"
)
def read_available_items(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener artículos disponibles para préstamo.
    
    Útil para mostrar opciones en formularios de préstamo.
    """
    return crud_item.get_available_items(db, skip=skip, limit=limit)

@router.get(
    "/{item_id}",
    response_model=ItemResponse,
    summary="Obtener artículo",
    description="Obtener un artículo específico por ID"
)
def read_item(
    item_id: int, 
    db: Session = Depends(get_db)
):
    """
    Obtener un artículo específico por ID.
    
    Incluye información de la categoría asociada.
    
    - **item_id**: ID único del artículo
    """
    item = crud_item.get_with_category(db, item_id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    return item

@router.put(
    "/{item_id}",
    response_model=ItemResponse,
    summary="Actualizar artículo",
    description="Actualizar información de un artículo existente"
)
def update_item(
    item_id: int,
    item_update: ItemUpdate,
    db: Session = Depends(get_db)
):
    """
    Actualizar un artículo existente.
    
    - **item_id**: ID del artículo a actualizar
    - Solo se actualizarán los campos proporcionados
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

@router.patch(
    "/{item_id}/status",
    response_model=ItemResponse,
    summary="Actualizar estado",
    description="Actualizar solo el estado de un artículo"
)
def update_item_status(
    item_id: int,
    new_status: ItemStatus,
    db: Session = Depends(get_db)
):
    """
    Actualizar solo el estado de un artículo.
    
    - **item_id**: ID del artículo
    - **new_status**: Nuevo estado del artículo
    """
    item = crud_item.get(db, id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    
    return crud_item.update_status(db, item_id=item_id, status=new_status)

@router.delete(
    "/{item_id}",
    summary="Eliminar artículo",
    description="Eliminar un artículo (solo si no tiene préstamos activos)"
)
def delete_item(
    item_id: int,
    db: Session = Depends(get_db)
):
    """
    Eliminar un artículo.
    
    Solo se puede eliminar si no tiene préstamos activos.
    
    - **item_id**: ID del artículo a eliminar
    """
    item = crud_item.get(db, id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    
    if not crud_item.can_delete(db, item_id=item_id):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="No se puede eliminar el artículo porque tiene préstamos activos"
        )
    
    crud_item.remove(db, id=item_id)
    return {"message": "Artículo eliminado exitosamente"}

@router.get(
    "/{item_id}/loans",
    response_model=List[dict],  # Simplificado
    summary="Historial de préstamos",
    description="Obtener historial de préstamos de un artículo"
)
def get_item_loans(
    item_id: int,
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener historial de préstamos de un artículo.
    
    - **item_id**: ID del artículo
    - **skip**: Registros a saltar
    - **limit**: Límite de registros
    """
    # Verificar que el artículo existe
    item = crud_item.get(db, id=item_id)
    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Artículo no encontrado"
        )
    
    from app.crud.loans import loan as crud_loan
    return crud_loan.get_by_item(db, item_id=item_id, skip=skip, limit=limit)
```

### 4. Router de Préstamos (`app/routers/loans.py`)

```python
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.orm import Session
from typing import List, Optional
from datetime import datetime

from app.crud.loans import loan as crud_loan
from app.schemas.loan import (
    LoanCreate, 
    LoanResponse, 
    LoanUpdate, 
    LoanReturn,
    LoanStatus
)
from app.database.database import get_db

router = APIRouter(
    prefix="/loans",
    tags=["loans"],
    responses={404: {"description": "Préstamo no encontrado"}}
)

@router.post(
    "/", 
    response_model=LoanResponse, 
    status_code=status.HTTP_201_CREATED,
    summary="Crear préstamo",
    description="Crear un nuevo préstamo de artículo"
)
def create_loan(
    loan: LoanCreate, 
    db: Session = Depends(get_db)
):
    """
    Crear un nuevo préstamo.
    
    - **user_id**: ID del usuario que solicita el préstamo
    - **item_id**: ID del artículo a prestar
    - **due_date**: Fecha de vencimiento del préstamo
    - **notes**: Notas adicionales (opcional)
    """
    try:
        return crud_loan.create_loan(db=db, loan=loan)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.get(
    "/", 
    response_model=List[LoanResponse],
    summary="Listar préstamos",
    description="Obtener lista de préstamos con filtros opcionales"
)
def read_loans(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    status: Optional[LoanStatus] = Query(None, description="Filtrar por estado"),
    user_id: Optional[int] = Query(None, description="Filtrar por usuario"),
    item_id: Optional[int] = Query(None, description="Filtrar por artículo"),
    overdue_only: bool = Query(False, description="Solo préstamos vencidos"),
    db: Session = Depends(get_db)
):
    """
    Obtener lista de préstamos con filtros opcionales.
    
    - **skip**: Registros a saltar
    - **limit**: Límite de registros
    - **status**: Filtrar por estado específico
    - **user_id**: Filtrar por usuario específico
    - **item_id**: Filtrar por artículo específico
    - **overdue_only**: Solo préstamos vencidos
    """
    if overdue_only:
        return crud_loan.get_overdue_loans(db, skip=skip, limit=limit)
    elif user_id:
        return crud_loan.get_by_user(db, user_id=user_id, skip=skip, limit=limit)
    elif item_id:
        return crud_loan.get_by_item(db, item_id=item_id, skip=skip, limit=limit)
    elif status:
        if status == LoanStatus.ACTIVE:
            return crud_loan.get_active_loans(db, skip=skip, limit=limit)
        else:
            # Para otros estados, usar filtro general
            return crud_loan.get_multi(db, skip=skip, limit=limit)
    else:
        return crud_loan.get_multi(db, skip=skip, limit=limit)

@router.get(
    "/active",
    response_model=List[LoanResponse],
    summary="Préstamos activos",
    description="Obtener solo préstamos activos"
)
def read_active_loans(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener préstamos activos.
    
    Útil para dashboards y reportes.
    """
    return crud_loan.get_active_loans(db, skip=skip, limit=limit)

@router.get(
    "/overdue",
    response_model=List[LoanResponse],
    summary="Préstamos vencidos",
    description="Obtener préstamos vencidos"
)
def read_overdue_loans(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000),
    db: Session = Depends(get_db)
):
    """
    Obtener préstamos vencidos.
    
    Importante para seguimiento y gestión de devoluciones.
    """
    return crud_loan.get_overdue_loans(db, skip=skip, limit=limit)

@router.get(
    "/{loan_id}",
    response_model=LoanResponse,
    summary="Obtener préstamo",
    description="Obtener un préstamo específico por ID"
)
def read_loan(
    loan_id: int, 
    db: Session = Depends(get_db)
):
    """
    Obtener un préstamo específico por ID.
    
    Incluye información detallada del usuario y artículo.
    
    - **loan_id**: ID único del préstamo
    """
    loan = crud_loan.get_with_details(db, loan_id=loan_id)
    if not loan:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Préstamo no encontrado"
        )
    return loan

@router.put(
    "/{loan_id}",
    response_model=LoanResponse,
    summary="Actualizar préstamo",
    description="Actualizar información de un préstamo"
)
def update_loan(
    loan_id: int,
    loan_update: LoanUpdate,
    db: Session = Depends(get_db)
):
    """
    Actualizar un préstamo existente.
    
    - **loan_id**: ID del préstamo a actualizar
    - Solo se actualizarán los campos proporcionados
    """
    loan = crud_loan.get(db, id=loan_id)
    if not loan:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Préstamo no encontrado"
        )
    
    return crud_loan.update(db=db, db_obj=loan, obj_in=loan_update)

@router.post(
    "/{loan_id}/return",
    response_model=LoanResponse,
    summary="Devolver préstamo",
    description="Procesar la devolución de un préstamo"
)
def return_loan(
    loan_id: int,
    return_data: LoanReturn,
    db: Session = Depends(get_db)
):
    """
    Procesar la devolución de un préstamo.
    
    - **loan_id**: ID del préstamo a devolver
    - **return_date**: Fecha de devolución
    - **notes**: Notas sobre la devolución (opcional)
    """
    try:
        return crud_loan.return_loan(
            db=db, loan_id=loan_id, return_data=return_data
        )
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.post(
    "/{loan_id}/extend",
    response_model=LoanResponse,
    summary="Extender préstamo",
    description="Extender la fecha de vencimiento de un préstamo"
)
def extend_loan(
    loan_id: int,
    new_due_date: datetime,
    db: Session = Depends(get_db)
):
    """
    Extender la fecha de vencimiento de un préstamo.
    
    - **loan_id**: ID del préstamo a extender
    - **new_due_date**: Nueva fecha de vencimiento
    """
    try:
        return crud_loan.extend_loan(
            db=db, loan_id=loan_id, new_due_date=new_due_date
        )
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.post(
    "/update-overdue-status",
    summary="Actualizar estados vencidos",
    description="Actualizar el estado de préstamos vencidos"
)
def update_overdue_status(
    db: Session = Depends(get_db)
):
    """
    Actualizar el estado de préstamos vencidos.
    
    Útil para ejecutar como tarea programada.
    """
    updated_count = crud_loan.update_overdue_status(db)
    return {
        "message": f"Se actualizaron {updated_count} préstamos vencidos",
        "updated_count": updated_count
    }
```

## Configuración en main.py

### Registrar routers en la aplicación principal

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.routers import users, categories, items, loans
from app.database.database import engine
from app.database.base import Base
from app.config import settings

# Crear tablas
Base.metadata.create_all(bind=engine)

# Crear aplicación FastAPI
app = FastAPI(
    title="Sistema de Inventario API",
    description="API REST para gestión de inventario y préstamos",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc"
)

# Configurar CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_HOSTS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Incluir routers
app.include_router(users.router, prefix="/api/v1")
app.include_router(categories.router, prefix="/api/v1")
app.include_router(items.router, prefix="/api/v1")
app.include_router(loans.router, prefix="/api/v1")

# Endpoint raíz
@app.get("/")
def read_root():
    return {
        "message": "Sistema de Inventario API",
        "version": "1.0.0",
        "docs": "/docs",
        "redoc": "/redoc"
    }

# Endpoint de salud
@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

## Conceptos importantes

### 1. Códigos de estado HTTP

```python
# Códigos más comunes en APIs REST
status.HTTP_200_OK          # Éxito general
status.HTTP_201_CREATED     # Recurso creado
status.HTTP_204_NO_CONTENT  # Éxito sin contenido
status.HTTP_400_BAD_REQUEST # Error del cliente
status.HTTP_401_UNAUTHORIZED # No autenticado
status.HTTP_403_FORBIDDEN   # No autorizado
status.HTTP_404_NOT_FOUND   # Recurso no encontrado
status.HTTP_422_UNPROCESSABLE_ENTITY # Error de validación
status.HTTP_500_INTERNAL_SERVER_ERROR # Error del servidor
```

### 2. Parámetros de consulta (Query Parameters)

```python
# Diferentes tipos de parámetros
@router.get("/items/")
def read_items(
    # Parámetro opcional con valor por defecto
    skip: int = Query(0, ge=0, description="Registros a saltar"),
    
    # Parámetro con validaciones
    limit: int = Query(100, ge=1, le=1000, description="Límite de registros"),
    
    # Parámetro opcional sin valor por defecto
    category_id: Optional[int] = Query(None, description="Filtrar por categoría"),
    
    # Parámetro requerido
    q: str = Query(..., min_length=2, description="Término de búsqueda"),
    
    # Parámetro booleano
    active_only: bool = Query(False, description="Solo activos")
):
    pass
```

### 3. Parámetros de ruta (Path Parameters)

```python
# Parámetros en la URL
@router.get("/users/{user_id}")
def read_user(
    user_id: int,  # Automáticamente validado como entero
    db: Session = Depends(get_db)
):
    pass

# Con validaciones adicionales
from fastapi import Path

@router.get("/users/{user_id}")
def read_user(
    user_id: int = Path(..., gt=0, description="ID del usuario"),
    db: Session = Depends(get_db)
):
    pass
```

### 4. Manejo de errores

```python
# Patrón estándar para manejo de errores
@router.get("/users/{user_id}")
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = crud_user.get(db, id=user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuario no encontrado"
        )
    return user

# Manejo de errores de validación
@router.post("/users/")
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    try:
        return crud_user.create_user(db=db, user=user)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )
```

### 5. Documentación automática

```python
# Agregar metadatos para documentación
@router.post(
    "/users/",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Crear usuario",                    # Título corto
    description="Crear un nuevo usuario en el sistema",  # Descripción
    response_description="Usuario creado exitosamente",  # Descripción de respuesta
    tags=["users"],                           # Agrupación
    responses={
        400: {"description": "Error de validación"},
        409: {"description": "Usuario ya existe"}
    }
)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    """
    Crear un nuevo usuario.
    
    - **username**: Nombre de usuario único
    - **email**: Email único del usuario
    - **full_name**: Nombre completo del usuario
    - **phone**: Teléfono del usuario (opcional)
    """
    pass
```

## Mejores prácticas

### 1. Organización de endpoints

- **Usar prefijos** consistentes (`/api/v1`)
- **Agrupar por entidad** (users, items, etc.)
- **Seguir convenciones REST** (GET, POST, PUT, DELETE)
- **Usar tags** para documentación

### 2. Validación de entrada

- **Validar parámetros** con Query y Path
- **Usar esquemas Pydantic** para request body
- **Manejar errores** de validación apropiadamente
- **Proporcionar mensajes** descriptivos

### 3. Respuestas consistentes

- **Usar response_model** para documentación
- **Códigos de estado** apropiados
- **Estructura consistente** en respuestas de error
- **Incluir metadatos** útiles (paginación, totales)

### 4. Performance

- **Paginación** en endpoints que retornan listas
- **Filtros opcionales** para reducir datos
- **Lazy loading** apropiado en relaciones
- **Límites razonables** en parámetros

## Próximos pasos

En el siguiente tema aprenderemos sobre middleware, autenticación y autorización para proteger nuestra API.

---

**💡 Tips importantes**:

1. **Documentación automática** - FastAPI genera docs automáticamente
2. **Validación automática** - Pydantic valida datos automáticamente
3. **Códigos de estado** - usar códigos HTTP apropiados
4. **Manejo de errores** - siempre manejar casos de error
5. **Paginación** - implementar en endpoints que retornan listas

**🔗 Enlaces útiles**:
- [FastAPI Path Operations](https://fastapi.tiangolo.com/tutorial/path-params/)
- [Query Parameters](https://fastapi.tiangolo.com/tutorial/query-params/)
- [Request Body](https://fastapi.tiangolo.com/tutorial/body/)
- [Response Model](https://fastapi.tiangolo.com/tutorial/response-model/)