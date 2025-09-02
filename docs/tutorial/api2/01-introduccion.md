# 1. Introducción y Conceptos Básicos

## ¿Qué es una API REST?

Una **API REST** (Application Programming Interface - Representational State Transfer) es un conjunto de reglas y convenciones para crear servicios web que permiten la comunicación entre diferentes aplicaciones.

### Características principales de REST:

- **Stateless (Sin estado)**: Cada petición contiene toda la información necesaria
- **Cacheable**: Las respuestas pueden ser almacenadas en caché
- **Cliente-Servidor**: Separación clara entre cliente y servidor
- **Interfaz uniforme**: Uso consistente de métodos HTTP

### Métodos HTTP más comunes:

| Método | Propósito | Ejemplo |
|--------|-----------|----------|
| GET | Obtener datos | `GET /users` - Obtener lista de usuarios |
| POST | Crear nuevos recursos | `POST /users` - Crear un nuevo usuario |
| PUT | Actualizar recursos completos | `PUT /users/1` - Actualizar usuario con ID 1 |
| DELETE | Eliminar recursos | `DELETE /users/1` - Eliminar usuario con ID 1 |

## ¿Qué es FastAPI?

**FastAPI** es un framework web moderno y rápido para construir APIs con Python. Fue creado por Sebastián Ramirez y se ha vuelto muy popular por sus características únicas.

### Ventajas de FastAPI:

1. **🚀 Velocidad**: Uno de los frameworks más rápidos disponibles
2. **📝 Documentación automática**: Genera documentación interactiva automáticamente
3. **🔍 Validación de datos**: Validación automática usando Python type hints
4. **🐍 Python moderno**: Aprovecha las características más recientes de Python
5. **📚 Fácil de aprender**: Sintaxis intuitiva y bien documentada

### Comparación con otros frameworks:

| Framework | Velocidad | Documentación | Validación | Curva de aprendizaje |
|-----------|-----------|---------------|------------|---------------------|
| FastAPI | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Flask | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Django | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |

## Conceptos Fundamentales

### 1. Endpoint
Un **endpoint** es una URL específica donde tu API puede recibir peticiones.

Ejemplo:
```
GET http://localhost:8000/users/1
```

### 2. Request (Petición)
Los datos que el cliente envía al servidor.

```json
{
  "name": "Juan Pérez",
  "email": "juan@email.com"
}
```

### 3. Response (Respuesta)
Los datos que el servidor devuelve al cliente.

```json
{
  "id": 1,
  "name": "Juan Pérez",
  "email": "juan@email.com"
}
```

### 4. Status Codes (Códigos de Estado)
Números que indican el resultado de la petición:

- **200**: OK - Petición exitosa
- **201**: Created - Recurso creado exitosamente
- **400**: Bad Request - Error en la petición del cliente
- **404**: Not Found - Recurso no encontrado
- **500**: Internal Server Error - Error del servidor

## Arquitectura de Nuestro Proyecto

Nuestro proyecto seguirá una arquitectura en capas:

```
📁 Proyecto API
├── 🗄️ Capa de Datos (models/)
├── 🔧 Capa de Lógica (crud/)
├── 🌐 Capa de Presentación (routers/)
├── ✅ Capa de Validación (schemas/)
└── 🔌 Capa de Configuración (database/)
```

### Beneficios de esta arquitectura:

- **Separación de responsabilidades**: Cada capa tiene una función específica
- **Mantenibilidad**: Fácil de modificar y extender
- **Testabilidad**: Cada capa se puede probar independientemente
- **Reutilización**: Componentes reutilizables en diferentes partes

## Las 3 Entidades de Nuestro Proyecto

### 1. 👤 Usuario (User)
- **Propósito**: Representar a los usuarios del sistema
- **Campos**: ID, nombre, email
- **Operaciones**: Crear, listar, obtener por ID

### 2. 📦 Producto (Product)
- **Propósito**: Representar productos disponibles
- **Campos**: ID, nombre, precio
- **Operaciones**: Crear, listar, obtener por ID

### 3. 🛒 Pedido (Order)
- **Propósito**: Representar pedidos de usuarios
- **Campos**: ID, usuario_id, producto_id, cantidad
- **Operaciones**: Crear, listar, obtener por ID

## Flujo de Trabajo Típico

1. **Cliente hace petición** → `GET /users`
2. **Router recibe petición** → Valida parámetros
3. **CRUD ejecuta operación** → Consulta base de datos
4. **Modelo devuelve datos** → Datos en formato Python
5. **Schema serializa datos** → Convierte a JSON
6. **Cliente recibe respuesta** → JSON con los datos

## ¿Por Qué Esta Estructura?

- **Escalabilidad**: Fácil agregar nuevas funcionalidades
- **Mantenimiento**: Código organizado y fácil de entender
- **Colaboración**: Múltiples desarrolladores pueden trabajar en paralelo
- **Estándares**: Sigue las mejores prácticas de la industria

---

**Siguiente:** [Configuración del Entorno](02-configuracion-entorno.md)

**Anterior:** [Índice Principal](README.md)