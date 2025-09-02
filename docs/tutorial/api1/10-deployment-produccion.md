# Deployment y Producción

## Introducción

En este tema aprenderemos cómo llevar nuestra API FastAPI a producción de manera segura y eficiente. Cubriremos:

- **Preparación** para producción
- **Configuración** de entornos
- **Deployment** en diferentes plataformas
- **Monitoreo** y logging
- **Optimización** de rendimiento
- **Seguridad** en producción

## Preparación para producción

### 1. Configuración de entornos

```python
# app/config.py
import os
from typing import List, Optional
from pydantic import BaseSettings, validator

class Settings(BaseSettings):
    """
    Configuración de la aplicación con soporte para múltiples entornos.
    """
    
    # Información básica
    APP_NAME: str = "Sistema de Inventario API"
    APP_VERSION: str = "1.0.0"
    ENVIRONMENT: str = "development"
    DEBUG: bool = True
    
    # Base de datos
    DATABASE_URL: str = "sqlite:///./inventory.db"
    DATABASE_POOL_SIZE: int = 5
    DATABASE_MAX_OVERFLOW: int = 10
    
    # Seguridad
    SECRET_KEY: str = "your-secret-key-here"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    
    # CORS
    ALLOWED_HOSTS: List[str] = ["*"]
    ALLOWED_ORIGINS: List[str] = ["http://localhost:3000"]
    
    # Rate limiting
    RATE_LIMIT_CALLS: int = 100
    RATE_LIMIT_PERIOD: int = 60
    
    # Logging
    LOG_LEVEL: str = "INFO"
    LOG_FILE: Optional[str] = None
    
    # Email (para notificaciones)
    SMTP_HOST: Optional[str] = None
    SMTP_PORT: int = 587
    SMTP_USERNAME: Optional[str] = None
    SMTP_PASSWORD: Optional[str] = None
    SMTP_USE_TLS: bool = True
    
    # Redis (para cache y sesiones)
    REDIS_URL: Optional[str] = None
    
    # Monitoring
    SENTRY_DSN: Optional[str] = None
    
    @validator('ENVIRONMENT')
    def validate_environment(cls, v):
        allowed = ['development', 'testing', 'staging', 'production']
        if v not in allowed:
            raise ValueError(f'Environment must be one of: {allowed}')
        return v
    
    @validator('SECRET_KEY')
    def validate_secret_key(cls, v, values):
        if values.get('ENVIRONMENT') == 'production' and v == 'your-secret-key-here':
            raise ValueError('Must set a secure SECRET_KEY for production')
        return v
    
    @validator('DEBUG')
    def validate_debug(cls, v, values):
        if values.get('ENVIRONMENT') == 'production' and v:
            raise ValueError('DEBUG must be False in production')
        return v
    
    @property
    def is_production(self) -> bool:
        return self.ENVIRONMENT == 'production'
    
    @property
    def is_development(self) -> bool:
        return self.ENVIRONMENT == 'development'
    
    @property
    def is_testing(self) -> bool:
        return self.ENVIRONMENT == 'testing'
    
    class Config:
        env_file = ".env"
        case_sensitive = True

# Instancia global de configuración
settings = Settings()
```

### 2. Variables de entorno

```bash
# .env.development
ENVIRONMENT=development
DEBUG=true
DATABASE_URL=sqlite:///./inventory_dev.db
SECRET_KEY=dev-secret-key
ALLOWED_ORIGINS=["http://localhost:3000", "http://localhost:8080"]
LOG_LEVEL=DEBUG

# .env.production
ENVIRONMENT=production
DEBUG=false
DATABASE_URL=postgresql://user:password@localhost:5432/inventory_prod
SECRET_KEY=super-secure-production-key
ALLOWED_ORIGINS=["https://yourdomain.com"]
LOG_LEVEL=INFO
SENTRY_DSN=https://your-sentry-dsn
REDIS_URL=redis://localhost:6379/0

# .env.testing
ENVIRONMENT=testing
DEBUG=false
DATABASE_URL=sqlite:///:memory:
SECRET_KEY=test-secret-key
LOG_LEVEL=WARNING
```

### 3. Configuración de logging

```python
# app/core/logging.py
import logging
import sys
from pathlib import Path
from typing import Optional

from app.config import settings

def setup_logging(log_file: Optional[str] = None) -> None:
    """
    Configurar logging para la aplicación.
    
    Args:
        log_file: Archivo de log opcional
    """
    # Configuración básica
    log_level = getattr(logging, settings.LOG_LEVEL.upper())
    
    # Formato de logs
    log_format = (
        "%(asctime)s - %(name)s - %(levelname)s - "
        "%(filename)s:%(lineno)d - %(message)s"
    )
    
    # Configurar handlers
    handlers = []
    
    # Handler para consola
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(logging.Formatter(log_format))
    handlers.append(console_handler)
    
    # Handler para archivo (si se especifica)
    if log_file or settings.LOG_FILE:
        file_path = Path(log_file or settings.LOG_FILE)
        file_path.parent.mkdir(parents=True, exist_ok=True)
        
        file_handler = logging.FileHandler(file_path)
        file_handler.setFormatter(logging.Formatter(log_format))
        handlers.append(file_handler)
    
    # Configurar logging
    logging.basicConfig(
        level=log_level,
        format=log_format,
        handlers=handlers
    )
    
    # Configurar loggers específicos
    if settings.is_production:
        # En producción, reducir logs de librerías externas
        logging.getLogger("uvicorn").setLevel(logging.WARNING)
        logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)
    else:
        # En desarrollo, mostrar más información
        logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)

# Logger para la aplicación
logger = logging.getLogger("inventory_api")
```

### 4. Configuración de base de datos para producción

```python
# app/database/database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import QueuePool

from app.config import settings
from app.core.logging import logger

# Configuración del engine según el entorno
if settings.is_production:
    # Configuración optimizada para producción
    engine = create_engine(
        settings.DATABASE_URL,
        poolclass=QueuePool,
        pool_size=settings.DATABASE_POOL_SIZE,
        max_overflow=settings.DATABASE_MAX_OVERFLOW,
        pool_pre_ping=True,  # Verificar conexiones
        pool_recycle=3600,   # Reciclar conexiones cada hora
        echo=False           # No mostrar SQL en logs
    )
else:
    # Configuración para desarrollo
    engine = create_engine(
        settings.DATABASE_URL,
        echo=settings.DEBUG  # Mostrar SQL en desarrollo
    )

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def get_db():
    """
    Dependencia para obtener sesión de base de datos.
    """
    db = SessionLocal()
    try:
        yield db
    except Exception as e:
        logger.error(f"Database error: {e}")
        db.rollback()
        raise
    finally:
        db.close()
```

## Configuración del servidor ASGI

### 1. Configuración de Uvicorn

```python
# app/main.py
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.gzip import GZipMiddleware

from app.config import settings
from app.core.logging import setup_logging, logger
from app.database.database import engine
from app.database.base import BaseModel
from app.middleware.logging import LoggingMiddleware
from app.middleware.rate_limit import RateLimitMiddleware
from app.routers import auth, users, categories, items, loans

# Configurar logging
setup_logging()

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Gestionar el ciclo de vida de la aplicación.
    """
    # Startup
    logger.info(f"Starting {settings.APP_NAME} v{settings.APP_VERSION}")
    logger.info(f"Environment: {settings.ENVIRONMENT}")
    
    # Crear tablas si no existen
    BaseModel.metadata.create_all(bind=engine)
    logger.info("Database tables created")
    
    yield
    
    # Shutdown
    logger.info("Shutting down application")

# Crear aplicación FastAPI
app = FastAPI(
    title=settings.APP_NAME,
    version=settings.APP_VERSION,
    description="API REST para gestión de inventario y préstamos",
    debug=settings.DEBUG,
    lifespan=lifespan,
    # Configuración para producción
    docs_url="/docs" if not settings.is_production else None,
    redoc_url="/redoc" if not settings.is_production else None,
    openapi_url="/openapi.json" if not settings.is_production else None
)

# Middleware de seguridad (solo en producción)
if settings.is_production:
    app.add_middleware(
        TrustedHostMiddleware,
        allowed_hosts=settings.ALLOWED_HOSTS
    )

# Middleware de compresión
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Middleware personalizado
app.add_middleware(LoggingMiddleware)
app.add_middleware(
    RateLimitMiddleware,
    calls=settings.RATE_LIMIT_CALLS,
    period=settings.RATE_LIMIT_PERIOD
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Incluir routers
app.include_router(auth.router, prefix="/api/v1")
app.include_router(users.router, prefix="/api/v1")
app.include_router(categories.router, prefix="/api/v1")
app.include_router(items.router, prefix="/api/v1")
app.include_router(loans.router, prefix="/api/v1")

@app.get("/")
def read_root():
    return {
        "message": f"Welcome to {settings.APP_NAME}",
        "version": settings.APP_VERSION,
        "environment": settings.ENVIRONMENT
    }

@app.get("/health")
def health_check():
    """
    Endpoint de health check para monitoreo.
    """
    return {
        "status": "healthy",
        "version": settings.APP_VERSION,
        "environment": settings.ENVIRONMENT
    }
```

### 2. Configuración de Gunicorn

```python
# gunicorn.conf.py
import multiprocessing
import os

# Configuración del servidor
bind = f"0.0.0.0:{os.getenv('PORT', '8000')}"
workers = int(os.getenv('WEB_CONCURRENCY', multiprocessing.cpu_count() * 2 + 1))
worker_class = "uvicorn.workers.UvicornWorker"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 100

# Timeouts
timeout = 30
keepalive = 2
graceful_timeout = 30

# Logging
accesslog = "-"
errorlog = "-"
loglevel = os.getenv('LOG_LEVEL', 'info').lower()
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# Seguridad
user = os.getenv('USER', None)
group = os.getenv('GROUP', None)

# Preload
preload_app = True

# Hooks
def on_starting(server):
    server.log.info("Starting Gunicorn server")

def on_reload(server):
    server.log.info("Reloading Gunicorn server")

def worker_int(worker):
    worker.log.info("Worker received INT or QUIT signal")

def pre_fork(server, worker):
    server.log.info(f"Worker spawned (pid: {worker.pid})")

def post_fork(server, worker):
    server.log.info(f"Worker spawned (pid: {worker.pid})")

def worker_abort(worker):
    worker.log.info(f"Worker received SIGABRT signal (pid: {worker.pid})")
```

## Deployment en diferentes plataformas

### 1. Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Configurar variables de entorno
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app

# Crear usuario no-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Crear directorio de trabajo
WORKDIR /app

# Copiar archivos de dependencias
COPY requirements.txt .

# Instalar dependencias de Python
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copiar código de la aplicación
COPY . .

# Cambiar propietario de archivos
RUN chown -R appuser:appuser /app

# Cambiar a usuario no-root
USER appuser

# Exponer puerto
EXPOSE 8000

# Comando por defecto
CMD ["gunicorn", "app.main:app", "-c", "gunicorn.conf.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/inventory
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=inventory
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
```

### 2. Nginx como reverse proxy

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:8000;
    }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    # Configuración SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    server {
        listen 80;
        server_name yourdomain.com;
        
        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name yourdomain.com;
        
        # SSL certificates
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        
        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        
        # Gzip compression
        gzip on;
        gzip_vary on;
        gzip_min_length 1024;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        
        location / {
            # Rate limiting
            limit_req zone=api burst=20 nodelay;
            
            # Proxy settings
            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # Buffer settings
            proxy_buffering on;
            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;
        }
        
        # Health check endpoint
        location /health {
            access_log off;
            proxy_pass http://api/health;
        }
        
        # Static files (if any)
        location /static/ {
            alias /app/static/;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

### 3. Heroku

```python
# Procfile
web: gunicorn app.main:app -c gunicorn.conf.py
release: python -m app.database.init_db
```

```python
# runtime.txt
python-3.11.0
```

```python
# app.json
{
  "name": "inventory-api",
  "description": "Sistema de Inventario API",
  "image": "heroku/python",
  "addons": [
    "heroku-postgresql:hobby-dev",
    "heroku-redis:hobby-dev"
  ],
  "env": {
    "ENVIRONMENT": "production",
    "SECRET_KEY": {
      "generator": "secret"
    },
    "LOG_LEVEL": "INFO"
  },
  "formation": {
    "web": {
      "quantity": 1,
      "size": "hobby"
    }
  },
  "stack": "heroku-22"
}
```

### 4. AWS (usando ECS)

```yaml
# docker-compose.aws.yml
version: '3.8'

services:
  api:
    image: your-account.dkr.ecr.region.amazonaws.com/inventory-api:latest
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SECRET_KEY=${SECRET_KEY}
    logging:
      driver: awslogs
      options:
        awslogs-group: /ecs/inventory-api
        awslogs-region: us-east-1
        awslogs-stream-prefix: ecs
```

```json
# task-definition.json
{
  "family": "inventory-api",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::account:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "your-account.dkr.ecr.region.amazonaws.com/inventory-api:latest",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENVIRONMENT",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:database-url"
        },
        {
          "name": "SECRET_KEY",
          "valueFrom": "arn:aws:secretsmanager:region:account:secret:secret-key"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/inventory-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## Monitoreo y observabilidad

### 1. Configuración de Sentry

```python
# app/core/monitoring.py
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from sentry_sdk.integrations.logging import LoggingIntegration

from app.config import settings

def setup_sentry():
    """
    Configurar Sentry para monitoreo de errores.
    """
    if settings.SENTRY_DSN:
        sentry_sdk.init(
            dsn=settings.SENTRY_DSN,
            environment=settings.ENVIRONMENT,
            release=settings.APP_VERSION,
            integrations=[
                FastApiIntegration(auto_enabling_integrations=False),
                SqlalchemyIntegration(),
                LoggingIntegration(
                    level=logging.INFO,
                    event_level=logging.ERROR
                ),
            ],
            traces_sample_rate=0.1 if settings.is_production else 1.0,
            send_default_pii=False,
            attach_stacktrace=True,
            before_send=filter_sensitive_data,
        )

def filter_sensitive_data(event, hint):
    """
    Filtrar datos sensibles antes de enviar a Sentry.
    """
    # Remover información sensible
    if 'request' in event:
        request = event['request']
        
        # Filtrar headers sensibles
        if 'headers' in request:
            sensitive_headers = ['authorization', 'cookie', 'x-api-key']
            for header in sensitive_headers:
                if header in request['headers']:
                    request['headers'][header] = '[Filtered]'
        
        # Filtrar datos del cuerpo
        if 'data' in request:
            sensitive_fields = ['password', 'token', 'secret']
            for field in sensitive_fields:
                if field in request['data']:
                    request['data'][field] = '[Filtered]'
    
    return event
```

### 2. Métricas con Prometheus

```python
# app/middleware/metrics.py
import time
from typing import Callable
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

# Métricas
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint']
)

ACTIVE_REQUESTS = Counter(
    'http_requests_active',
    'Active HTTP requests'
)

class MetricsMiddleware(BaseHTTPMiddleware):
    """
    Middleware para recopilar métricas de la aplicación.
    """
    
    async def dispatch(self, request: Request, call_next: Callable) -> Response:
        # Incrementar requests activos
        ACTIVE_REQUESTS.inc()
        
        # Obtener información del request
        method = request.method
        path = request.url.path
        
        # Medir tiempo de respuesta
        start_time = time.time()
        
        try:
            response = await call_next(request)
            status_code = response.status_code
        except Exception as e:
            status_code = 500
            raise
        finally:
            # Decrementar requests activos
            ACTIVE_REQUESTS.dec()
            
            # Registrar métricas
            duration = time.time() - start_time
            
            REQUEST_COUNT.labels(
                method=method,
                endpoint=path,
                status_code=status_code
            ).inc()
            
            REQUEST_DURATION.labels(
                method=method,
                endpoint=path
            ).observe(duration)
        
        return response

# Endpoint para métricas
def metrics_endpoint():
    """
    Endpoint para exponer métricas de Prometheus.
    """
    return Response(
        generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )
```

### 3. Health checks avanzados

```python
# app/core/health.py
import asyncio
from typing import Dict, Any
from sqlalchemy import text
from sqlalchemy.orm import Session

from app.database.database import SessionLocal, engine
from app.config import settings

class HealthChecker:
    """
    Verificador de salud de la aplicación.
    """
    
    @staticmethod
    def check_database() -> Dict[str, Any]:
        """
        Verificar conexión a la base de datos.
        """
        try:
            db = SessionLocal()
            db.execute(text("SELECT 1"))
            db.close()
            return {
                "status": "healthy",
                "message": "Database connection successful"
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "message": f"Database connection failed: {str(e)}"
            }
    
    @staticmethod
    def check_redis() -> Dict[str, Any]:
        """
        Verificar conexión a Redis.
        """
        if not settings.REDIS_URL:
            return {
                "status": "skipped",
                "message": "Redis not configured"
            }
        
        try:
            import redis
            r = redis.from_url(settings.REDIS_URL)
            r.ping()
            return {
                "status": "healthy",
                "message": "Redis connection successful"
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "message": f"Redis connection failed: {str(e)}"
            }
    
    @staticmethod
    def check_disk_space() -> Dict[str, Any]:
        """
        Verificar espacio en disco.
        """
        try:
            import shutil
            total, used, free = shutil.disk_usage("/")
            
            free_percent = (free / total) * 100
            
            if free_percent < 10:
                status = "unhealthy"
                message = f"Low disk space: {free_percent:.1f}% free"
            elif free_percent < 20:
                status = "warning"
                message = f"Disk space warning: {free_percent:.1f}% free"
            else:
                status = "healthy"
                message = f"Disk space OK: {free_percent:.1f}% free"
            
            return {
                "status": status,
                "message": message,
                "free_space_percent": round(free_percent, 1)
            }
        except Exception as e:
            return {
                "status": "unknown",
                "message": f"Could not check disk space: {str(e)}"
            }
    
    @classmethod
    def get_health_status(cls) -> Dict[str, Any]:
        """
        Obtener estado de salud completo.
        """
        checks = {
            "database": cls.check_database(),
            "redis": cls.check_redis(),
            "disk": cls.check_disk_space()
        }
        
        # Determinar estado general
        overall_status = "healthy"
        for check in checks.values():
            if check["status"] == "unhealthy":
                overall_status = "unhealthy"
                break
            elif check["status"] == "warning" and overall_status == "healthy":
                overall_status = "warning"
        
        return {
            "status": overall_status,
            "version": settings.APP_VERSION,
            "environment": settings.ENVIRONMENT,
            "checks": checks
        }

# Endpoint de health check
def detailed_health_check():
    """
    Endpoint detallado de health check.
    """
    return HealthChecker.get_health_status()
```

## Optimización de rendimiento

### 1. Cache con Redis

```python
# app/core/cache.py
import json
import pickle
from typing import Any, Optional, Union
from functools import wraps

import redis
from app.config import settings

class CacheManager:
    """
    Gestor de cache con Redis.
    """
    
    def __init__(self):
        if settings.REDIS_URL:
            self.redis = redis.from_url(settings.REDIS_URL)
        else:
            self.redis = None
    
    def get(self, key: str) -> Optional[Any]:
        """
        Obtener valor del cache.
        """
        if not self.redis:
            return None
        
        try:
            value = self.redis.get(key)
            if value:
                return pickle.loads(value)
        except Exception:
            pass
        
        return None
    
    def set(self, key: str, value: Any, ttl: int = 300) -> bool:
        """
        Guardar valor en el cache.
        
        Args:
            key: Clave del cache
            value: Valor a guardar
            ttl: Tiempo de vida en segundos
        """
        if not self.redis:
            return False
        
        try:
            serialized = pickle.dumps(value)
            return self.redis.setex(key, ttl, serialized)
        except Exception:
            return False
    
    def delete(self, key: str) -> bool:
        """
        Eliminar valor del cache.
        """
        if not self.redis:
            return False
        
        try:
            return bool(self.redis.delete(key))
        except Exception:
            return False
    
    def clear_pattern(self, pattern: str) -> int:
        """
        Eliminar todas las claves que coincidan con el patrón.
        """
        if not self.redis:
            return 0
        
        try:
            keys = self.redis.keys(pattern)
            if keys:
                return self.redis.delete(*keys)
        except Exception:
            pass
        
        return 0

# Instancia global
cache = CacheManager()

def cached(ttl: int = 300, key_prefix: str = ""):
    """
    Decorator para cachear resultados de funciones.
    
    Args:
        ttl: Tiempo de vida del cache en segundos
        key_prefix: Prefijo para la clave del cache
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generar clave del cache
            cache_key = f"{key_prefix}:{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            # Intentar obtener del cache
            result = cache.get(cache_key)
            if result is not None:
                return result
            
            # Ejecutar función y guardar en cache
            result = func(*args, **kwargs)
            cache.set(cache_key, result, ttl)
            
            return result
        
        return wrapper
    return decorator
```

### 2. Optimización de consultas

```python
# app/crud/optimized.py
from typing import List, Optional
from sqlalchemy.orm import Session, joinedload, selectinload
from sqlalchemy import func

from app.models.item import Item
from app.models.category import Category
from app.models.loan import Loan
from app.core.cache import cached

class OptimizedQueries:
    """
    Consultas optimizadas para mejor rendimiento.
    """
    
    @staticmethod
    @cached(ttl=300, key_prefix="items")
    def get_items_with_category(db: Session, skip: int = 0, limit: int = 100) -> List[Item]:
        """
        Obtener artículos con categoría usando eager loading.
        """
        return (
            db.query(Item)
            .options(joinedload(Item.category))
            .offset(skip)
            .limit(limit)
            .all()
        )
    
    @staticmethod
    @cached(ttl=600, key_prefix="stats")
    def get_inventory_stats(db: Session) -> dict:
        """
        Obtener estadísticas del inventario.
        """
        total_items = db.query(func.count(Item.id)).scalar()
        total_categories = db.query(func.count(Category.id)).scalar()
        active_loans = db.query(func.count(Loan.id)).filter(
            Loan.status == "active"
        ).scalar()
        
        return {
            "total_items": total_items,
            "total_categories": total_categories,
            "active_loans": active_loans
        }
    
    @staticmethod
    def get_user_loans_optimized(db: Session, user_id: int) -> List[Loan]:
        """
        Obtener préstamos de usuario con eager loading.
        """
        return (
            db.query(Loan)
            .options(
                joinedload(Loan.item).joinedload(Item.category),
                joinedload(Loan.user)
            )
            .filter(Loan.user_id == user_id)
            .all()
        )
```

## Scripts de deployment

### 1. Script de deployment

```bash
#!/bin/bash
# deploy.sh

set -e

echo "Starting deployment..."

# Variables
APP_NAME="inventory-api"
DOCKER_IMAGE="$APP_NAME:latest"
CONTAINER_NAME="$APP_NAME-container"

# Construir imagen
echo "Building Docker image..."
docker build -t $DOCKER_IMAGE .

# Ejecutar tests
echo "Running tests..."
docker run --rm $DOCKER_IMAGE pytest

# Parar contenedor existente
echo "Stopping existing container..."
docker stop $CONTAINER_NAME || true
docker rm $CONTAINER_NAME || true

# Ejecutar migraciones
echo "Running database migrations..."
docker run --rm \
  --env-file .env.production \
  $DOCKER_IMAGE \
  python -m alembic upgrade head

# Iniciar nuevo contenedor
echo "Starting new container..."
docker run -d \
  --name $CONTAINER_NAME \
  --env-file .env.production \
  -p 8000:8000 \
  --restart unless-stopped \
  $DOCKER_IMAGE

# Verificar que el servicio está funcionando
echo "Checking service health..."
sleep 10
if curl -f http://localhost:8000/health; then
  echo "Deployment successful!"
else
  echo "Deployment failed - service not responding"
  exit 1
fi

echo "Deployment completed successfully!"
```

### 2. CI/CD con GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v3
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-test.txt
    
    - name: Run tests
      run: |
        pytest --cov=app --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build and push Docker image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: inventory-api
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
    
    - name: Deploy to ECS
      run: |
        aws ecs update-service \
          --cluster inventory-cluster \
          --service inventory-api-service \
          --force-new-deployment
```

## Seguridad en producción

### 1. Checklist de seguridad

- ✅ **HTTPS** habilitado con certificados válidos
- ✅ **Headers de seguridad** configurados
- ✅ **Rate limiting** implementado
- ✅ **Validación de entrada** en todos los endpoints
- ✅ **Autenticación** y autorización robustas
- ✅ **Secrets** gestionados de forma segura
- ✅ **Logs** sin información sensible
- ✅ **Base de datos** con acceso restringido
- ✅ **Firewall** configurado
- ✅ **Monitoreo** de seguridad activo

### 2. Configuración de seguridad

```python
# app/middleware/security.py
from fastapi import Request, HTTPException, status
from starlette.middleware.base import BaseHTTPMiddleware
import re

class SecurityMiddleware(BaseHTTPMiddleware):
    """
    Middleware de seguridad para proteger contra ataques comunes.
    """
    
    def __init__(self, app):
        super().__init__(app)
        
        # Patrones de ataques comunes
        self.sql_injection_patterns = [
            r"('|(\-\-)|(;)|(\||\|)|(\*|\*))",
            r"(union|select|insert|delete|update|drop|create|alter)"
        ]
        
        self.xss_patterns = [
            r"<script[^>]*>.*?</script>",
            r"javascript:",
            r"on\w+\s*="
        ]
    
    async def dispatch(self, request: Request, call_next):
        # Verificar User-Agent
        user_agent = request.headers.get("user-agent", "")
        if not user_agent or len(user_agent) > 500:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Invalid User-Agent"
            )
        
        # Verificar tamaño del request
        content_length = request.headers.get("content-length")
        if content_length and int(content_length) > 10 * 1024 * 1024:  # 10MB
            raise HTTPException(
                status_code=status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
                detail="Request too large"
            )
        
        # Verificar parámetros de consulta
        query_string = str(request.url.query)
        if self._contains_malicious_patterns(query_string):
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Malicious request detected"
            )
        
        response = await call_next(request)
        
        # Agregar headers de seguridad
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        
        return response
    
    def _contains_malicious_patterns(self, text: str) -> bool:
        """
        Verificar si el texto contiene patrones maliciosos.
        """
        text_lower = text.lower()
        
        # Verificar SQL injection
        for pattern in self.sql_injection_patterns:
            if re.search(pattern, text_lower, re.IGNORECASE):
                return True
        
        # Verificar XSS
        for pattern in self.xss_patterns:
            if re.search(pattern, text_lower, re.IGNORECASE):
                return True
        
        return False
```

## Conclusión

En este tema hemos cubierto todos los aspectos importantes para llevar una API FastAPI a producción:

1. **Configuración** adecuada para diferentes entornos
2. **Deployment** en múltiples plataformas
3. **Monitoreo** y observabilidad
4. **Optimización** de rendimiento
5. **Seguridad** en producción

Con estos conocimientos, puedes desplegar tu API de manera segura y eficiente en cualquier entorno de producción.

---

**💡 Tips importantes**:

1. **Siempre probar** en staging antes de producción
2. **Monitorear** constantemente el rendimiento
3. **Mantener** logs detallados pero seguros
4. **Actualizar** dependencias regularmente
5. **Tener** un plan de rollback

**🔗 Enlaces útiles**:
- [FastAPI Deployment](https://fastapi.tiangolo.com/deployment/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Nginx Configuration](https://nginx.org/en/docs/)
- [AWS ECS Guide](https://docs.aws.amazon.com/ecs/)
- [Heroku Python](https://devcenter.heroku.com/articles/python-support)