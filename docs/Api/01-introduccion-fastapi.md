# Introducción a FastAPI

## ¿Qué es FastAPI?

FastAPI es un framework web moderno y de alto rendimiento para construir APIs con Python 3.7+ basado en las anotaciones de tipo estándar de Python.

### Características principales

- **Rápido**: Muy alto rendimiento, a la par con NodeJS y Go (gracias a Starlette y Pydantic)
- **Rápido de codificar**: Aumenta la velocidad de desarrollo entre 200% y 300%
- **Menos errores**: Reduce aproximadamente un 40% de errores inducidos por humanos
- **Intuitivo**: Gran soporte del editor con autocompletado en todas partes
- **Fácil**: Diseñado para ser fácil de usar y aprender
- **Corto**: Minimiza la duplicación de código
- **Robusto**: Obtén código listo para producción con documentación automática interactiva
- **Basado en estándares**: Basado en (y totalmente compatible con) los estándares abiertos para APIs: OpenAPI y JSON Schema

## ¿Por qué FastAPI?

### 1. Documentación Automática
FastAPI genera automáticamente documentación interactiva para tu API usando:
- **Swagger UI** (disponible en `/docs`)
- **ReDoc** (disponible en `/redoc`)

### 2. Validación de Datos
Utiliza Pydantic para validación automática de datos de entrada y salida:
- Validación de tipos
- Validación de formato
- Mensajes de error claros

### 3. Tipado Estático
Aprovecha las anotaciones de tipo de Python para:
- Mejor soporte del IDE
- Detección temprana de errores
- Código más legible y mantenible

### 4. Rendimiento
- Uno de los frameworks Python más rápidos disponibles
- Comparable con frameworks de Node.js y Go
- Basado en Starlette para la parte web y Pydantic para la parte de datos

## Comparación con otros frameworks

| Característica | FastAPI | Flask | Django REST |
|---|---|---|---|
| Rendimiento | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Documentación automática | ✅ | ❌ | ❌ |
| Validación automática | ✅ | ❌ | ✅ |
| Tipado estático | ✅ | ❌ | ❌ |
| Curva de aprendizaje | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| Async/await nativo | ✅ | ✅ | ✅ |

## Conceptos clave

### 1. Path Operations
Las operaciones de ruta son las funciones que manejan las peticiones HTTP:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

### 2. Path Parameters
Parámetros que forman parte de la URL:

```python
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}
```

### 3. Query Parameters
Parámetros que van después del `?` en la URL:

```python
@app.get("/items/")
def read_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}
```

### 4. Request Body
Datos enviados en el cuerpo de la petición:

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = False

@app.post("/items/")
def create_item(item: Item):
    return item
```

## Estructura básica de una aplicación FastAPI

```python
from fastapi import FastAPI

# Crear instancia de la aplicación
app = FastAPI(
    title="Mi API",
    description="Una API de ejemplo",
    version="1.0.0"
)

# Definir rutas
@app.get("/")
def read_root():
    return {"message": "¡Hola Mundo!"}

# Ejecutar con: uvicorn main:app --reload
```

## Ventajas para el desarrollo

### 1. Desarrollo rápido
- Menos código repetitivo
- Validación automática
- Documentación automática
- Excelente soporte del IDE

### 2. Mantenibilidad
- Código más limpio y legible
- Tipado estático reduce errores
- Estructura clara y organizada

### 3. Escalabilidad
- Soporte nativo para async/await
- Alto rendimiento
- Fácil de testear

## ¿Cuándo usar FastAPI?

FastAPI es ideal para:

- **APIs REST modernas**: Especialmente cuando necesitas documentación automática
- **Microservicios**: Por su rendimiento y facilidad de despliegue
- **Prototipado rápido**: Desarrollo rápido con validación automática
- **APIs con validación compleja**: Pydantic hace la validación muy fácil
- **Proyectos que requieren alto rendimiento**: Comparable con frameworks de otros lenguajes

## Próximos pasos

En el siguiente tema aprenderemos cómo instalar y configurar FastAPI para comenzar a desarrollar nuestra API de inventario.

---

**💡 Tip**: FastAPI está basado en estándares abiertos como OpenAPI y JSON Schema, lo que significa que tu API será compatible con muchas herramientas y servicios existentes.

**🔗 Enlaces útiles**:
- [Documentación oficial de FastAPI](https://fastapi.tiangolo.com/)
- [Tutorial oficial](https://fastapi.tiangolo.com/tutorial/)
- [Repositorio en GitHub](https://github.com/tiangolo/fastapi)