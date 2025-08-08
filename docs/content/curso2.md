# 📘 Tutorial Completo y Profundo de **Pydantic + FastAPI** en Español  
## 🔥 Desarrollo de APIs REST Robustas, Seguras y Documentadas

---

## 📚 Tabla de Contenidos

- [📘 Tutorial Completo y Profundo de **Pydantic + FastAPI** en Español](#-tutorial-completo-y-profundo-de-pydantic--fastapi-en-español)
  - [🔥 Desarrollo de APIs REST Robustas, Seguras y Documentadas](#-desarrollo-de-apis-rest-robustas-seguras-y-documentadas)
  - [📚 Tabla de Contenidos](#-tabla-de-contenidos)
  - [1. Introducción: ¿Por qué Pydantic + FastAPI?](#1-introducción-por-qué-pydantic--fastapi)
    - [🔑 Ventajas del combo:](#-ventajas-del-combo)
  - [2. Instalación y Entorno](#2-instalación-y-entorno)
  - [3. Conceptos Clave de FastAPI](#3-conceptos-clave-de-fastapi)
    - [Estructura básica](#estructura-básica)
  - [4. Modelos Pydantic: Validación y Serialización](#4-modelos-pydantic-validación-y-serialización)
    - [Modelo básico](#modelo-básico)
    - [Validación de tipos y restricciones](#validación-de-tipos-y-restricciones)
  - [5. Tipos de Modelos: Entrada, Salida, Común](#5-tipos-de-modelos-entrada-salida-común)
  - [6. Validaciones Avanzadas con Pydantic](#6-validaciones-avanzadas-con-pydantic)
    - [Validadores de campo](#validadores-de-campo)
    - [Validador de modelo completo](#validador-de-modelo-completo)
  - [7. Rutas y Parámetros en FastAPI](#7-rutas-y-parámetros-en-fastapi)
    - [Parámetros de ruta, consulta y cuerpo](#parámetros-de-ruta-consulta-y-cuerpo)
    - [Sub-dependencias](#sub-dependencias)
  - [8. Manejo de Errores y Excepciones Personalizadas](#8-manejo-de-errores-y-excepciones-personalizadas)
    - [Excepciones HTTP](#excepciones-http)
    - [Manejadores globales](#manejadores-globales)
  - [9. Autenticación y Seguridad (JWT, OAuth2)](#9-autenticación-y-seguridad-jwt-oauth2)
    - [Configuración de JWT](#configuración-de-jwt)
    - [Ruta de login](#ruta-de-login)
    - [Proteger rutas](#proteger-rutas)
  - [10. Conexión con Base de Datos (SQLAlchemy + Pydantic)](#10-conexión-con-base-de-datos-sqlalchemy--pydantic)
    - [Modelo SQLAlchemy](#modelo-sqlalchemy)
    - [Uso en rutas](#uso-en-rutas)
  - [11. Documentación Automática (Swagger y ReDoc)](#11-documentación-automática-swagger-y-redoc)
    - [Mejora la documentación con descripciones](#mejora-la-documentación-con-descripciones)
  - [12. Pruebas Unitarias con Pydantic y FastAPI](#12-pruebas-unitarias-con-pydantic-y-fastapi)
  - [13. Despliegue Básico (Docker, Uvicorn)](#13-despliegue-básico-docker-uvicorn)
    - [`Dockerfile`](#dockerfile)
    - [`docker-compose.yml`](#docker-composeyml)
  - [14. Ejemplo Completo: API de Usuarios y Publicaciones](#14-ejemplo-completo-api-de-usuarios-y-publicaciones)
    - [Estructura del proyecto](#estructura-del-proyecto)
    - [Funcionalidades incluidas:](#funcionalidades-incluidas)
  - [15. Mejores Prácticas y Consejos Profesionales](#15-mejores-prácticas-y-consejos-profesionales)
  - [16. Recursos y Enlaces Útiles](#16-recursos-y-enlaces-útiles)

---

## 1. Introducción: ¿Por qué Pydantic + FastAPI?

**FastAPI** es uno de los frameworks más modernos y rápidos para construir APIs en Python, y su gran ventaja es su integración nativa con **Pydantic**.

### 🔑 Ventajas del combo:
- ✅ **Validación automática** de datos de entrada/salida.
- ✅ **Documentación automática** (Swagger UI y ReDoc).
- ✅ **Alto rendimiento**, comparable a Node.js y Go.
- ✅ **Basado en estándares**: OpenAPI y JSON Schema.
- ✅ **Soporte para async/await** (ideal para operaciones I/O).
- ✅ **Integración perfecta con editores** (autocompletado, chequeo de tipos).

Este tutorial te llevará desde cero hasta construir una API profesional completa, segura, bien estructurada y lista para producción.

---

## 2. Instalación y Entorno

Crea un entorno virtual y instala las dependencias:

```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
# o
venv\Scripts\activate     # Windows

pip install "fastapi[all]"  # incluye Uvicorn, Starlette, Pydantic, etc.
pip install sqlalchemy  # para base de datos
pip install python-jose[cryptography]  # para JWT
pip install passlib[bcrypt]  # para hashing de contraseñas
```

> ✅ `fastapi[all]` instala FastAPI con todas sus dependencias opcionales (como el servidor Uvicorn).

Inicia el servidor con:

```bash
uvicorn main:app --reload
```

---

## 3. Conceptos Clave de FastAPI

FastAPI está construido sobre **Starlette** (para endpoints ASGI) y **Pydantic** (para validación).

### Estructura básica

```python
# main.py
from fastapi import FastAPI

app = FastAPI(title="Mi API", version="1.0")

@app.get("/")
def saludar():
    return {"mensaje": "¡Hola desde FastAPI!"}
```

Accede a:
- `http://localhost:8000` → JSON
- `http://localhost:8000/docs` → Swagger UI
- `http://localhost:8000/redoc` → ReDoc

---

## 4. Modelos Pydantic: Validación y Serialización

### Modelo básico

```python
from pydantic import BaseModel

class UsuarioBase(BaseModel):
    email: str
    nombre: str
```

### Validación de tipos y restricciones

```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional

class UsuarioCrear(UsuarioBase):
    password: str = Field(..., min_length=6)
    email: EmailStr  # Validación de formato de email

class UsuarioSalida(UsuarioBase):
    id: int
    activo: bool = True

    class Config:
        from_attributes = True  # Antes: orm_mode = True (v1)
```

> ✅ `from_attributes = True` permite que FastAPI lea datos desde objetos ORM (como SQLAlchemy).

---

## 5. Tipos de Modelos: Entrada, Salida, Común

Organiza tus modelos para evitar fugas de datos.

```python
# models.py
from pydantic import BaseModel, EmailStr

# Modelo base compartido
class UsuarioBase(BaseModel):
    email: EmailStr
    nombre: str

# Para crear (entrada)
class UsuarioCrear(UsuarioBase):
    password: str

# Para respuesta (salida)
class UsuarioSalida(UsuarioBase):
    id: int
    activo: bool

    class Config:
        from_attributes = True

# Para actualización (opcional en todos los campos)
class UsuarioActualizar(BaseModel):
    nombre: Optional[str] = None
    activo: Optional[bool] = None
```

---

## 6. Validaciones Avanzadas con Pydantic

### Validadores de campo

```python
from pydantic import field_validator

class Producto(BaseModel):
    nombre: str
    precio: float
    categoria: str

    @field_validator('precio')
    @classmethod
    def precio_positivo(cls, v):
        if v <= 0:
            raise ValueError('El precio debe ser mayor que 0')
        return v

    @field_validator('categoria')
    @classmethod
    def categoria_permitida(cls, v):
        categorias_validas = ['tecnología', 'ropa', 'hogar']
        if v.lower() not in categorias_validas:
            raise ValueError(f'Categoría debe ser una de: {", ".join(categorias_validas)}')
        return v.lower()
```

### Validador de modelo completo

```python
from pydantic import model_validator

class Pedido(BaseModel):
    producto: str
    cantidad: int
    precio_unitario: float
    precio_total: float = None

    @model_validator(mode='after')
    def calcular_total(cls, model):
        model.precio_total = model.cantidad * model.precio_unitario
        return model
```

---

## 7. Rutas y Parámetros en FastAPI

### Parámetros de ruta, consulta y cuerpo

```python
from fastapi import FastAPI, Path, Query, Body

@app.post("/productos/{producto_id}")
def crear_producto(
    producto_id: int = Path(..., gt=0),
    cantidad: int = Query(1, ge=1, le=100),
    producto: Producto = Body(...)  # cuerpo esperado
):
    return {
        "producto_id": producto_id,
        "cantidad": cantidad,
        "producto": producto
    }
```

### Sub-dependencias

```python
def pagination(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items/")
def listar_items(pag: dict = Depends(pagination)):
    return {"items": [], "paginación": pag}
```

---

## 8. Manejo de Errores y Excepciones Personalizadas

### Excepciones HTTP

```python
from fastapi import HTTPException

@app.get("/usuarios/{user_id}")
def obtener_usuario(user_id: int):
    if user_id not in usuarios_db:
        raise HTTPException(
            status_code=404,
            detail="Usuario no encontrado",
            headers={"X-Error": "No existe"}
        )
    return usuarios_db[user_id]
```

### Manejadores globales

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(ValueError)
async def value_error_handler(request: Request, exc: ValueError):
    return JSONResponse(
        status_code=400,
        content={"error": "Valor inválido", "detalle": str(exc)}
    )
```

---

## 9. Autenticación y Seguridad (JWT, OAuth2)

### Configuración de JWT

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext

SECRET_KEY = "tu_clave_secreta_muy_segura"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

### Ruta de login

```python
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/token")
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = buscar_usuario(form_data.username)
    if not user or not verify_password(form_data.password, user.password):
        raise HTTPException(status_code=401, detail="Credenciales inválidas")
    
    token = create_access_token({"sub": user.email})
    return {"access_token": token, "token_type": "bearer"}
```

### Proteger rutas

```python
from fastapi import Depends, Security

def get_current_user(token: str = Security(get_token_header)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None:
            raise HTTPException(status_code=401, detail="Token inválido")
        return buscar_usuario_por_email(email)
    except JWTError:
        raise HTTPException(status_code=401, detail="Token inválido")

@app.get("/perfil")
def perfil_usuario(usuario: UsuarioSalida = Depends(get_current_user)):
    return usuario
```

---

## 10. Conexión con Base de Datos (SQLAlchemy + Pydantic)

### Modelo SQLAlchemy

```python
# database.py
from sqlalchemy import create_engine, Column, Integer, String, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

class UsuarioDB(Base):
    __tablename__ = "usuarios"
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    nombre = Column(String)
    hashed_password = Column(String)
    activo = Column(Boolean, default=True)
```

### Uso en rutas

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/usuarios/", response_model=UsuarioSalida)
def crear_usuario(usuario: UsuarioCrear, db: Session = Depends(get_db)):
    hashed = hash_password(usuario.password)
    db_usuario = UsuarioDB(**usuario.model_dump(), hashed_password=hashed)
    db.add(db_usuario)
    db.commit()
    db.refresh(db_usuario)
    return db_usuario
```

---

## 11. Documentación Automática (Swagger y ReDoc)

FastAPI genera automáticamente:

- ✅ `/docs` → Swagger UI (interfaz interactiva)
- ✅ `/redoc` → ReDoc (documentación elegante)

### Mejora la documentación con descripciones

```python
class UsuarioCrear(BaseModel):
    email: EmailStr = Field(..., description="Correo electrónico del usuario")
    nombre: str = Field(..., min_length=2, description="Nombre completo")
    password: str = Field(..., min_length=6, description="Contraseña segura")
```

---

## 12. Pruebas Unitarias con Pydantic y FastAPI

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_saludo():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"mensaje": "¡Hola desde FastAPI!"}

def test_crear_usuario():
    response = client.post("/usuarios/", json={
        "email": "test@example.com",
        "nombre": "Test User",
        "password": "secret123"
    })
    assert response.status_code == 200
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "id" in data
```

Ejecuta pruebas:

```bash
pytest test_main.py -v
```

---

## 13. Despliegue Básico (Docker, Uvicorn)

### `Dockerfile`

```Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```

### `docker-compose.yml`

```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8000:80"
    environment:
      - ENV=production
```

---

## 14. Ejemplo Completo: API de Usuarios y Publicaciones

👉 Descarga el código completo en: [GitHub - fastapi-pydantic-demo](https://github.com/tu-usuario/fastapi-pydantic-demo)

### Estructura del proyecto

```
proyecto/
├── main.py
├── models.py
├── schemas.py
├── database.py
├── auth.py
├── crud.py
└── requirements.txt
```

### Funcionalidades incluidas:
- ✅ Registro y login de usuarios
- ✅ Crear, leer, actualizar y eliminar publicaciones
- ✅ Autenticación con JWT
- ✅ Validación con Pydantic
- ✅ Paginación
- ✅ Manejo de errores
- ✅ Pruebas unitarias
- ✅ Documentación automática

---

## 15. Mejores Prácticas y Consejos Profesionales

| Práctica | Recomendación |
|--------|---------------|
| 📁 Estructura modular | Separa `routers`, `models`, `schemas`, `crud`, `auth` |
| 🔐 Nunca expongas contraseñas | Usa modelos de salida sin `password` |
| 🧩 Usa `from_attributes = True` | Para compatibilidad con ORM |
| 🔄 Evita `dict()` en validación | Usa `model_validate()` (v2) |
| 🧪 Escribe pruebas | Asegura que tus validaciones funcionen |
| 📈 Usa `limit` y `skip` | Para paginación y evitar sobrecarga |
| 🔍 Usa `exclude_unset` o `exclude_none` | Al serializar para JSON limpio |
| 🧠 Usa `TypeVar` y `Generic` | Para respuestas genéricas |

---

## 16. Recursos y Enlaces Útiles

- 📘 [Documentación Oficial de FastAPI](https://fastapi.tiangolo.com/)
- 📘 [Documentación de Pydantic v2](https://docs.pydantic.dev/latest/)
- 🐳 [Docker + FastAPI](https://fastapi.tiangolo.com/deployment/docker/)
- 🔐 [Seguridad en FastAPI](https://fastapi.tiangolo.com/tutorial/security/)
- 🧪 [Testing en FastAPI](https://fastapi.tiangolo.com/tutorial/testing/)
- 📺 [Curso Gratuito en YouTube (en español)](https://www.youtube.com/results?search_query=fastapi+pydantic+espa%C3%B1ol)
- 💬 [Comunidad en Discord de FastAPI](https://discord.gg/fastapi)

---

