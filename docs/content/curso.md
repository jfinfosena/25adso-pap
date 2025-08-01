# Curso de FastAPI: Estructura de 16 Clases

Este curso de **FastAPI** de 16 clases está diseñado para enseñar desde los fundamentos hasta temas avanzados, con un enfoque práctico centrado en el desarrollo de una API de gestión de proyectos. Cada clase combina teoría, práctica y trabajo en un proyecto continuo, asumiendo un nivel inicial-intermedio en Python. Duración aproximada por clase: 1.5-2 horas.

---

## Módulo 1: Fundamentos y Primeros Pasos

### Clase 1: Introducción a FastAPI y APIs REST
- **Objetivo**: Comprender FastAPI y configurar el entorno.
- **Contenido**:
  - ¿Qué es FastAPI? Ventajas y comparación con otros frameworks.
  - Conceptos de APIs REST: métodos HTTP, endpoints, JSON.
  - Instalación de FastAPI, Uvicorn y Python.
- **Práctica**: Crear una API con un endpoint `/projects` que devuelva una lista de proyectos.
- **Tarea**: Configurar el entorno local y probar la API.

### Clase 2: Rutas y Parámetros
- **Objetivo**: Crear endpoints dinámicos con parámetros.
- **Contenido**:
  - Endpoints con GET, POST, PUT, DELETE.
  - Parámetros de ruta (`/projects/{id}`) y query parameters (`/projects?status=active`).
  - Validación básica de datos.
- **Práctica**: Implementar endpoints para crear y obtener proyectos.
- **Tarea**: Crear un endpoint para eliminar un proyecto.

### Clase 3: Modelos con Pydantic
- **Objetivo**: Usar Pydantic para validar datos.
- **Contenido**:
  - Definición de modelos Pydantic.
  - Validación de entradas y salidas.
  - Uso de `response_model`.
- **Práctica**: Crear un modelo para validar datos de proyectos (nombre, descripción, fecha).
- **Tarea**: Implementar un endpoint que valide la creación de un proyecto.

### Clase 4: Documentación y Estructura del Proyecto
- **Objetivo**: Documentar la API y organizar el proyecto.
- **Contenido**:
  - Swagger UI y ReDoc para documentación automática.
  - Estructura de carpetas y buenas prácticas.
  - Introducción al proyecto: API de gestión de proyectos.
- **Práctica**: Documentar endpoints con tags y descripciones.
- **Tarea**: Diseñar una estructura de carpetas para el proyecto.

---

## Módulo 2: Funcionalidades Intermedias y Bases de Datos

### Clase 5: Bases de Datos con SQLAlchemy
- **Objetivo**: Conectar la API a una base de datos.
- **Contenido**:
  - Configuración de SQLAlchemy (SQLite/PostgreSQL).
  - Modelos de base de datos para proyectos.
  - Operaciones CRUD básicas.
- **Práctica**: Almacenar y recuperar proyectos en una base de datos.
- **Tarea**: Crear un endpoint para actualizar proyectos.

### Clase 6: Relaciones en Bases de Datos
- **Objetivo**: Manejar relaciones en la base de datos.
- **Contenido**:
  - Relaciones uno-a-muchos (proyectos y tareas).
  - Configuración en SQLAlchemy.
  - Consultas para datos relacionados.
- **Práctica**: Asociar tareas a un proyecto.
- **Tarea**: Listar tareas de un proyecto.

### Clase 7: Autenticación con OAuth2
- **Objetivo**: Implementar autenticación con JWT.
- **Contenido**:
  - Autenticación con OAuth2 y JWT.
  - Endpoints protegidos con `OAuth2PasswordBearer`.
  - Generación y verificación de tokens.
- **Práctica**: Proteger endpoints de proyectos.
- **Tarea**: Crear un endpoint de login.

### Clase 8: Manejo de Errores
- **Objetivo**: Gestionar errores de forma robusta.
- **Contenido**:
  - Uso de `HTTPException`.
  - Respuestas de error estructuradas.
  - Middleware para errores globales.
- **Práctica**: Agregar manejo de errores a endpoints.
- **Tarea**: Crear un middleware para registrar errores.

---

## Módulo 3: Optimización y Funcionalidades Avanzadas

### Clase 9: Paginación y Filtrado
- **Objetivo**: Optimizar listas de datos.
- **Contenido**:
  - Paginación para proyectos o tareas.
  - Filtrado y ordenamiento con query parameters.
  - Optimización de consultas.
- **Práctica**: Crear un endpoint paginado para proyectos.
- **Tarea**: Implementar filtros por estado o fecha.

### Clase 10: Programación Asíncrona
- **Objetivo**: Usar asincronía para mejorar el rendimiento.
- **Contenido**:
  - `def` vs. `async def`.
  - Bases de datos asíncronas (`asyncpg`).
  - Beneficios de la asincronía.
- **Práctica**: Convertir un endpoint a asíncrono.
- **Tarea**: Comparar rendimiento síncrono vs. asíncrono.

### Clase 11: Dependencias Avanzadas
- **Objetivo**: Crear dependencias reutilizables.
- **Contenido**:
  - Uso de `Depends` para autenticación y validaciones.
  - Dependencias anidadas.
  - Ejemplos de dependencias complejas.
- **Práctica**: Verificar roles de usuario con una dependencia.
- **Tarea**: Limitar acceso por rol.

### Clase 12: Testing Automatizado
- **Objetivo**: Escribir pruebas para la API.
- **Contenido**:
  - `pytest` y `TestClient`.
  - Pruebas unitarias e integración.
  - Pruebas con base de datos.
- **Práctica**: Escribir pruebas para endpoints.
- **Tarea**: Validar errores de autenticación.

---

## Módulo 4: Producción y Características Avanzadas

### Clase 13: WebSockets
- **Objetivo**: Implementar comunicación en tiempo real.
- **Contenido**:
  - WebSockets en FastAPI.
  - Notificaciones para proyectos (ej. tarea asignada).
  - Manejo de conexiones.
- **Práctica**: Crear un WebSocket para notificaciones.
- **Tarea**: Notificar nuevos proyectos.

### Clase 14: Integración con Servicios Externos
- **Objetivo**: Conectar con APIs externas.
- **Contenido**:
  - Uso de `httpx` para consumir APIs.
  - Respuestas asíncronas.
  - Ejemplo: notificaciones por correo.
- **Práctica**: Integrar un servicio externo.
- **Tarea**: Probar una API externa relevante.

### Clase 15: Despliegue en Producción
- **Objetivo**: Llevar la API a producción.
- **Contenido**:
  - Uvicorn y Gunicorn.
  - Despliegue en Heroku, AWS o DigitalOcean.
  - Variables de entorno y seguridad.
- **Práctica**: Desplegar la API.
- **Tarea**: Configurar HTTPS.

### Clase 16: Proyecto Final y Cierre
- **Objetivo**: Completar y presentar el proyecto.
- **Contenido**:
  - Finalización de la API de gestión de proyectos.
  - Revisión de código y documentación.
  - Presentación de proyectos.
- **Práctica**: Desplegar y probar el proyecto final.
- **Tarea**: Compartir repositorio y un informe.

---

## Notas Adicionales
- **Proyecto**: Los estudiantes desarrollan una API de gestión de proyectos (proyectos, tareas, usuarios) a lo largo del curso.
- **Metodología**: 30% teoría, 50% práctica, 20% resolución de problemas.
- **Requisitos previos**: Python básico y conceptos de web.
- **Recursos**:
  - [Documentación de FastAPI](https://fastapi.tiangolo.com/)
  - Tutoriales de SQLAlchemy, Pydantic, JWT.
  - Repositorio de GitHub con ejemplos.
- **Evaluación**: Proyecto (50%), tareas (30%), participación (20%).