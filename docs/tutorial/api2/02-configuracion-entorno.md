# 2. Configuración del Entorno

## Requisitos del Sistema

Antes de comenzar, asegúrate de tener instalado:

- **Python 3.8 o superior** (recomendado Python 3.9+)
- **pip** (gestor de paquetes de Python)
- **Editor de código** (VS Code, PyCharm, etc.)

### Verificar instalación de Python

Abre una terminal y ejecuta:

```bash
python --version
# o en algunos sistemas:
python3 --version
```

Deberías ver algo como: `Python 3.9.7`

## Paso 1: Crear el Directorio del Proyecto

```bash
# Crear directorio
mkdir api_simple
cd api_simple
```

## Paso 2: Crear Entorno Virtual

Un **entorno virtual** es un espacio aislado donde puedes instalar paquetes sin afectar tu sistema Python global.

### ¿Por qué usar entornos virtuales?

- **Aislamiento**: Evita conflictos entre proyectos
- **Reproducibilidad**: Mismas versiones en diferentes máquinas
- **Limpieza**: No contamina el Python del sistema

### Crear entorno virtual:

```bash
# Crear entorno virtual
python -m venv .venv

# En Windows:
python -m venv .venv

# En macOS/Linux:
python3 -m venv .venv
```

### Activar entorno virtual:

```bash
# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (Command Prompt)
.venv\Scripts\activate.bat

# macOS/Linux
source .venv/bin/activate
```

**💡 Tip**: Cuando el entorno esté activo, verás `(.venv)` al inicio de tu terminal.

## Paso 3: Crear archivo requirements.txt

Este archivo lista todas las dependencias de nuestro proyecto:

```txt
fastapi
uvicorn
sqlalchemy
```

### ¿Qué hace cada dependencia?

| Paquete | Propósito | ¿Por qué lo necesitamos? |
|---------|-----------|-------------------------|
| **fastapi** | Framework web | Core de nuestra API |
| **uvicorn** | Servidor ASGI | Ejecutar la aplicación |
| **sqlalchemy** | ORM | Interactuar con la base de datos |

## Paso 4: Instalar Dependencias

```bash
# Instalar todas las dependencias
pip install -r requirements.txt

# O instalar una por una:
pip install fastapi
pip install uvicorn
pip install sqlalchemy
```

### Verificar instalación:

```bash
pip list
```

Deberías ver las dependencias instaladas.

## Paso 5: Crear Estructura de Carpetas

```bash
# Crear estructura básica
mkdir app
mkdir app/models
mkdir app/schemas
mkdir app/crud
mkdir app/routers
mkdir app/database
mkdir docs
```

### Estructura resultante:

```
api_simple/
├── .venv/                 # Entorno virtual
├── app/                   # Código de la aplicación
│   ├── models/           # Modelos de base de datos
│   ├── schemas/          # Esquemas de validación
│   ├── crud/             # Operaciones de base de datos
│   ├── routers/          # Rutas y endpoints
│   └── database/         # Configuración de BD
├── docs/                 # Documentación
├── requirements.txt      # Dependencias
└── main.py              # Archivo principal (lo crearemos después)
```

## Paso 6: Crear archivos __init__.py

Python necesita estos archivos para reconocer las carpetas como paquetes:

```bash
# Crear archivos __init__.py vacíos
touch app/__init__.py
touch app/models/__init__.py
touch app/schemas/__init__.py
touch app/crud/__init__.py
touch app/routers/__init__.py
touch app/database/__init__.py

# En Windows, usa:
echo. > app/__init__.py
echo. > app/models/__init__.py
echo. > app/schemas/__init__.py
echo. > app/crud/__init__.py
echo. > app/routers/__init__.py
echo. > app/database/__init__.py
```

## Configuración del Editor

### VS Code (Recomendado)

1. **Instalar extensiones útiles**:
   - Python
   - Python Docstring Generator
   - REST Client

2. **Configurar intérprete de Python**:
   - `Ctrl+Shift+P` → "Python: Select Interpreter"
   - Seleccionar el intérprete de `.venv`

3. **Configuración recomendada** (`.vscode/settings.json`):

```json
{
    "python.defaultInterpreterPath": "./.venv/bin/python",
    "python.formatting.provider": "black",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true
}
```

## Verificación de la Configuración

Crea un archivo de prueba `test_setup.py`:

```python
# test_setup.py
import fastapi
import uvicorn
import sqlalchemy

print("✅ FastAPI version:", fastapi.__version__)
print("✅ Uvicorn version:", uvicorn.__version__)
print("✅ SQLAlchemy version:", sqlalchemy.__version__)
print("🎉 ¡Configuración exitosa!")
```

Ejecútalo:

```bash
python test_setup.py
```

Si ves las versiones sin errores, ¡todo está listo!

## Comandos Útiles

### Gestión del entorno virtual:

```bash
# Activar entorno
source .venv/bin/activate  # macOS/Linux
.venv\Scripts\activate     # Windows

# Desactivar entorno
deactivate

# Ver paquetes instalados
pip list

# Actualizar pip
pip install --upgrade pip
```

### Gestión de dependencias:

```bash
# Instalar nueva dependencia
pip install nombre_paquete

# Generar requirements.txt actualizado
pip freeze > requirements.txt

# Instalar desde requirements.txt
pip install -r requirements.txt
```

## Solución de Problemas Comunes

### Error: "python no se reconoce como comando"

**Solución**: Agregar Python al PATH del sistema o usar `python3`

### Error: "No se puede activar el entorno virtual"

**Windows PowerShell**:
```bash
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Error: "pip no encontrado"

**Solución**: Reinstalar Python con la opción "Add to PATH" marcada

### Error de permisos en macOS/Linux

**Solución**: Usar `python3 -m pip` en lugar de solo `pip`

## Mejores Prácticas

1. **Siempre usar entornos virtuales** para cada proyecto
2. **Mantener requirements.txt actualizado** con versiones específicas
3. **No subir .venv/ a control de versiones** (agregar a .gitignore)
4. **Documentar la configuración** para otros desarrolladores
5. **Usar nombres descriptivos** para carpetas y archivos

## Archivo .gitignore Recomendado

Crea un archivo `.gitignore` para excluir archivos innecesarios:

```gitignore
# Entorno virtual
.venv/
venv/
env/

# Cache de Python
__pycache__/
*.pyc
*.pyo
*.pyd

# Base de datos
*.db
*.sqlite3

# IDE
.vscode/
.idea/

# Logs
*.log

# Variables de entorno
.env
```

---

**Siguiente:** [Estructura del Proyecto](03-estructura-proyecto.md)

**Anterior:** [Introducción y Conceptos Básicos](01-introduccion.md)