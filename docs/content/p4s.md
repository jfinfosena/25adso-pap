# ✅ Solución: Actividad – Gestión de Productos en una Tienda de Mascotas

> 🐾 API FastAPI completa con: CRUD, JSON, Query Parameters, entorno virtual y estructura de repositorio.

---

## 📁 Estructura del repositorio

```
tienda-mascotas-base/
│
├── main.py             # API FastAPI
├── info.json           # Información del estudiante
├── requirements.txt    # Dependencias
└── README.md           # Instrucciones (opcional)
```

---

### 1. `requirements.txt`

```txt
fastapi>=0.68.0
uvicorn[standard]>=0.15.0
pydantic>=1.8.0
```

> Este archivo permite instalar todas las dependencias con `pip install -r requirements.txt`.

---

### 2. `info.json` (ejemplo completado)

```json
{
  "nombre": "Ana Rodríguez Pérez",
  "codigo": "202310050",
  "correo": "ana.rodriguez@universidad.edu",
  "fecha_entrega": "2025-04-05",
  "descripcion_proyecto": "API REST para gestión de productos de tienda de mascotas con FastAPI"
}
```

---

### 3. `main.py` – Solución completa

```python
from fastapi import FastAPI, Query
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# Modelo: Producto completo
class Producto(BaseModel):
    nombre: str
    precio: float
    categoria: str
    stock: int

# Modelo: Producto parcial (para PATCH)
class ProductoParcial(BaseModel):
    nombre: Optional[str] = None
    precio: Optional[float] = None
    categoria: Optional[str] = None
    stock: Optional[int] = None

# Base de datos simulada
productos = [
    {
        "id": 1,
        "nombre": "Alimento para perros",
        "precio": 25.99,
        "categoria": "alimento",
        "stock": 100
    },
    {
        "id": 2,
        "nombre": "Juguete de peluche",
        "precio": 8.50,
        "categoria": "juguetes",
        "stock": 50
    },
    {
        "id": 3,
        "nombre": "Collar ajustable",
        "precio": 12.00,
        "categoria": "accesorios",
        "stock": 30
    }
]

# 🔹 GET: Listar todos los productos + filtros por nombre y categoría
@app.get("/productos/")
def obtener_productos(
    nombre: str = Query(None, description="Filtrar por nombre (búsqueda parcial)"),
    categoria: str = Query(None, description="Filtrar por categoría")
):
    resultado = productos

    if nombre:
        resultado = [p for p in resultado if nombre.lower() in p["nombre"].lower()]

    if categoria:
        resultado = [p for p in resultado if p["categoria"].lower() == categoria.lower()]

    return {"productos": resultado}

# 🔹 GET: Obtener producto por ID
@app.get("/productos/{producto_id}")
def obtener_producto(producto_id: int):
    for producto in productos:
        if producto["id"] == producto_id:
            return producto
    return {"error": "Producto no encontrado"}

# 🔷 POST: Crear nuevo producto
@app.post("/productos/")
def crear_producto(producto: Producto):
    nuevo_id = max([p["id"] for p in productos]) + 1 if productos else 1
    nuevo_producto = {
        "id": nuevo_id,
        **producto.dict()
    }
    productos.append(nuevo_producto)
    return {"mensaje": "Producto creado", "producto": nuevo_producto}

# 🔶 PUT: Actualizar producto completo
@app.put("/productos/{producto_id}")
def actualizar_producto(producto_id: int, producto: Producto):
    for p in productos:
        if p["id"] == producto_id:
            p.update(producto.dict())
            return {"mensaje": "Producto actualizado", "producto": p}
    return {"error": "Producto no encontrado"}

# 🟡 PATCH: Actualizar parcialmente un producto
@app.patch("/productos/{producto_id}")
def actualizar_producto_parcial(producto_id: int, producto_parcial: ProductoParcial):
    for p in productos:
        if p["id"] == producto_id:
            producto_parcial_dict = producto_parcial.dict(exclude_unset=True)  # Solo campos enviados
            p.update(producto_parcial_dict)
            return {"mensaje": "Producto actualizado parcialmente", "producto": p}
    return {"error": "Producto no encontrado"}

# 🔻 DELETE: Eliminar producto
@app.delete("/productos/{producto_id}")
def eliminar_producto(producto_id: int):
    for producto in productos:
        if producto["id"] == producto_id:
            productos.remove(producto)
            return {"mensaje": "Producto eliminado"}
    return {"error": "Producto no encontrado"}
```

---

## ✅ Explicación de elementos clave

### ✅ Uso de `exclude_unset=True`
```python
producto_parcial.dict(exclude_unset=True)
```
- Garantiza que solo se actualicen los campos que fueron enviados en el `PATCH`.
- Evita sobrescribir campos opcionales con `None`.

### ✅ Filtros con Query Parameters
- Soporta ambos filtros simultáneamente:  
  `?nombre=peluche&categoria=juguetes`
- Búsqueda insensible a mayúsculas con `.lower()`.

### ✅ Validación automática con Pydantic
- Si se envía un `precio` como `"abc"`, FastAPI responde con error 422.
- Asegura que los datos sean del tipo correcto.

---

## ▶️ Cómo probar la solución

1. Iniciar el servidor:
   ```bash
   uvicorn main:app --reload
   ```

2. Acceder a la documentación interactiva:
   ```
   http://127.0.0.1:8000/docs
   ```

3. Ejemplos de pruebas en Postman:

### 🟢 POST: Crear producto
- URL: `POST http://127.0.0.1:8000/productos/`
- Body (JSON):
  ```json
  {
    "nombre": "Arena para gatos",
    "precio": 18.0,
    "categoria": "higiene",
    "stock": 40
  }
  ```

### 🔍 GET con filtros
- `GET /productos/?categoria=alimento`
- `GET /productos/?nombre=col`
- `GET /productos/?nombre=peluche&categoria=juguetes`

### 🟡 PATCH: Actualizar solo stock
- URL: `PATCH /productos/1`
- Body:
  ```json
  { "stock": 90 }
  ```

---

## 🧪 Resultados esperados

| Acción | Respuesta esperada |
|-------|--------------------|
| `GET /productos/` | Lista de 3 productos |
| `GET /productos/1` | Producto con ID 1 |
| `GET /productos/?categoria=alimento` | Solo productos de alimento |
| `POST` con JSON válido | Mensaje "Producto creado" |
| `PATCH` con `{ "stock": 90 }` | Actualiza solo el stock |
| `DELETE /productos/1` | "Producto eliminado" |

---

