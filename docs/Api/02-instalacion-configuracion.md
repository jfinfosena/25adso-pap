# Instalación y Configuración

## Prerrequisitos

Antes de comenzar, asegúrate de tener instalado:

- **Python 3.7 o superior** (recomendado Python 3.9+)
- **pip** (gestor de paquetes de Python)
- **Editor de código** (VS Code, PyCharm, etc.)

### Verificar la instalación de Python

```bash
# Verificar versión de Python
python --version
# o
python3 --version

# Verificar pip
pip --version
```

## Configuración del entorno de desarrollo

### 1. Crear directorio del proyecto

```bash
# Crear directorio
mkdir fastapi-inventory
cd fastapi-inventory
```

### 2. Crear entorno virtual

Un entorno virtual aísla las dependencias de tu proyecto:

```bash
# Crear entorno virtual
python -m venv .venv

# Activar entorno virtual
# En Windows:
.venv\Scripts\activate

# En macOS/Linux:
source .venv/bin/activate
```

**💡 Tip**: Siempre activa el entorno virtual antes de trabajar en el proyecto.

### 3. Actualizar pip

```bash
# Actualizar pip a la última versión
pip install --upgrade pip
```

## Instalación de FastAPI

### Instalación básica

```bash
# Instalar FastAPI
pip install fastapi

# Instalar servidor ASGI (Uvicorn)
pip install "uvicorn[standard]"
```

### Instalación completa para el proyecto

Para nuestro proyecto de inventario, necesitamos dependencias adicionales:

```bash
# Instalar todas las dependencias necesarias
pip install fastapi uvicorn[standard] sqlalchemy aiosqlite pydantic[email] python-multipart python-dotenv
```

### Crear archivo requirements.txt

Es una buena práctica mantener un registro de las dependencias:

```bash
# Crear archivo requirements.txt
pip freeze > requirements.txt
```

El archivo `requirements.txt` debería verse así:

```txt
fastapi>=0.104.1
uvicorn[standard]>=0.24.0
sqlalchemy>=2.0.23
aiosqlite>=0.19.0
pydantic>=2.5.0
python-multipart>=0.0.6
python-dotenv>=1.0.0
```

## Primera aplicación FastAPI

### 1. Crear archivo main.py

Crea un archivo llamado `main.py` en el directorio raíz:

```python
from fastapi import FastAPI

# Crear instancia de FastAPI
app = FastAPI(
    title="Inventory API",
    description="API REST para gestión de inventario",
    version="1.0.0"
)

# Ruta raíz
@app.get("/")
def read_root():
    return {
        "message": "¡Bienvenido a la API de Inventario!",
        "version": "1.0.0",
        "docs": "/docs"
    }

# Ruta de ejemplo
@app.get("/health")
def health_check():
    return {"status": "OK", "message": "API funcionando correctamente"}
```

### 2. Ejecutar la aplicación

```bash
# Ejecutar servidor de desarrollo
uvicorn main:app --reload
```

**Parámetros explicados**:
- `main:app`: archivo `main.py` y variable `app`
- `--reload`: reinicia automáticamente cuando detecta cambios

### 3. Verificar que funciona

Abre tu navegador y visita:

- **API**: http://localhost:8000
- **Documentación Swagger**: http://localhost:8000/docs
- **Documentación ReDoc**: http://localhost:8000/redoc

## Configuración avanzada

### 1. Variables de entorno

Crea un archivo `.env` para configuraciones:

```env
# .env
DATABASE_URL=sqlite:///./inventory.db
DEBUG=True
SECRET_KEY=your-secret-key-here
```

### 2. Archivo de configuración

Crea `app/config.py`:

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Settings:
    # Base de datos
    DATABASE_URL: str = os.getenv("DATABASE_URL", "sqlite:///./inventory.db")
    
    # Configuración de la aplicación
    DEBUG: bool = os.getenv("DEBUG", "False").lower() == "true"
    SECRET_KEY: str = os.getenv("SECRET_KEY", "your-secret-key")
    
    # Configuración de CORS
    ALLOWED_ORIGINS: list = [
        "http://localhost:3000",
        "http://localhost:8080",
    ]
    
    # Configuración de paginación
    DEFAULT_PAGE_SIZE: int = 20
    MAX_PAGE_SIZE: int = 100

settings = Settings()
```

### 3. Configuración de CORS

Para permitir peticiones desde el frontend:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings

app = FastAPI(
    title="Inventory API",
    description="API REST para gestión de inventario",
    version="1.0.0"
)

# Configurar CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Estructura de directorios recomendada

```
fastapi-inventory/
├── .venv/                 # Entorno virtual
├── app/                   # Código de la aplicación
│   ├── __init__.py
│   ├── main.py           # Aplicación principal
│   ├── config.py         # Configuraciones
│   ├── database/         # Configuración de BD
│   ├── models/           # Modelos SQLAlchemy
│   ├── schemas/          # Esquemas Pydantic
│   ├── crud/             # Operaciones CRUD
│   ├── routers/          # Rutas de la API
│   └── utils/            # Utilidades
├── .env                  # Variables de entorno
├── .gitignore           # Archivos a ignorar en Git
├── requirements.txt     # Dependencias
└── README.md           # Documentación
```

## Comandos útiles para desarrollo

### Ejecutar servidor

```bash
# Desarrollo (con recarga automática)
uvicorn app.main:app --reload

# Especificar host y puerto
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# Producción (sin recarga)
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### Gestión de dependencias

```bash
# Instalar desde requirements.txt
pip install -r requirements.txt

# Actualizar requirements.txt
pip freeze > requirements.txt

# Instalar nueva dependencia
pip install nueva-dependencia
pip freeze > requirements.txt
```

## Configuración del editor (VS Code)

### Extensiones recomendadas

- **Python** (Microsoft)
- **Pylance** (Microsoft)
- **Python Docstring Generator**
- **REST Client** (para probar APIs)

### Configuración de workspace

Crea `.vscode/settings.json`:

```json
{
    "python.defaultInterpreterPath": "./.venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "python.formatting.provider": "black",
    "python.sortImports.args": ["--profile", "black"],
    "editor.formatOnSave": true
}
```

## Solución de problemas comunes

### Error: "uvicorn: command not found"

```bash
# Asegúrate de que el entorno virtual esté activado
source .venv/bin/activate  # macOS/Linux
.venv\Scripts\activate     # Windows

# Reinstalar uvicorn
pip install uvicorn[standard]
```

### Error: "ModuleNotFoundError"

```bash
# Verificar que estés en el directorio correcto
pwd

# Verificar que el entorno virtual esté activado
which python  # macOS/Linux
where python  # Windows

# Reinstalar dependencias
pip install -r requirements.txt
```

### Puerto en uso

```bash
# Usar otro puerto
uvicorn app.main:app --port 8001 --reload

# Encontrar proceso usando el puerto (Linux/macOS)
lsof -i :8000

# Matar proceso (Linux/macOS)
kill -9 <PID>
```

## Próximos pasos

Ahora que tienes FastAPI instalado y configurado, en el siguiente tema aprenderemos sobre la estructura del proyecto y cómo organizar el código de manera eficiente.

---

**💡 Tips importantes**:

1. **Siempre usa entornos virtuales** para evitar conflictos de dependencias
2. **Mantén actualizado requirements.txt** para facilitar la colaboración
3. **Usa variables de entorno** para configuraciones sensibles
4. **Aprovecha la documentación automática** en `/docs` para probar tu API

**🔗 Enlaces útiles**:
- [Documentación de Uvicorn](https://www.uvicorn.org/)
- [Guía de entornos virtuales](https://docs.python.org/3/tutorial/venv.html)
- [Variables de entorno con python-dotenv](https://pypi.org/project/python-dotenv/)