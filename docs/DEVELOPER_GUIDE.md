# Guía del Desarrollador: Helisa Innovate Services

## Tabla de Contenidos
- [Requisitos Previos](#requisitos-previos)
- [Setup Inicial](#setup-inicial)
- [Desarrollo Local](#desarrollo-local)
- [Workflows de Desarrollo](#workflows-de-desarrollo)
- [Troubleshooting](#troubleshooting)
- [Recursos Adicionales](#recursos-adicionales)

---

## Requisitos Previos

### Software Requerido

| Software | Versión Mínima | Instalación |
|----------|---------------|-------------|
| **Git** | 2.40+ | https://git-scm.com/downloads |
| **Node.js** | 20.x LTS | https://nodejs.org/ |
| **Python** | 3.12+ | https://www.python.org/downloads/ |
| **Docker** | 24.0+ | https://www.docker.com/get-started |
| **Docker Compose** | 2.20+ | Incluido en Docker Desktop |
| **make** | 4.0+ | Linux/Mac: preinstalado, Windows: choco install make |
| **psql** (opcional) | 16+ | Cliente PostgreSQL |
| **Redis CLI** (opcional) | 7+ | Cliente Redis |

### Verificar Instalaciones

```bash
# Verificar versiones
git --version                # Git 2.40.0 o superior
node --version               # v20.11.0 o superior
npm --version                # 10.2.0 o superior
python --version             # Python 3.12.0 o superior
docker --version             # Docker 24.0.0 o superior
docker-compose --version     # Docker Compose 2.20.0 o superior
make --version               # GNU Make 4.0 o superior
```

### Credenciales de Helisa

Solicitar a tu líder técnico:
- `HELISA_CLIENT_ID`: ID de cliente asignado
- `HELISA_SECRET_KEY`: Clave secreta para firma HMAC-SHA256
- `HELISA_API_URL`: URL de la API (dev/staging/prod)

---

## Setup Inicial

### 1. Clonar Repositorio

```bash
# Clonar repo
git clone https://github.com/innovate-nutrition/helisa-innovate-services.git
cd helisa-innovate-services

# Verificar branch
git branch
# Deberías estar en: main o develop

# Ver estructura
ls -la
# .github/  apps/  docs/  infrastructure/  libs/  Makefile  README.md
```

### 2. Configurar Variables de Entorno

```bash
# Copiar template de variables de entorno
cp .env.example .env

# Editar con tus valores
nano .env
# o usa tu editor preferido: code .env, vim .env, etc.
```

**Archivo `.env` (ejemplo)**:
```bash
# ===== Database =====
POSTGRES_USER=helisa_user
POSTGRES_PASSWORD=dev_password_123_CHANGE_ME
POSTGRES_DB=helisa_innovate

# ===== Redis =====
REDIS_PASSWORD=

# ===== RabbitMQ =====
RABBITMQ_USER=helisa
RABBITMQ_PASS=dev_password_123_CHANGE_ME

# ===== Helisa API (OBTENER DE LIDER TECNICO) =====
HELISA_API_URL=https://helisa-dev.com/api/KansasWS
HELISA_CLIENT_ID=TU_CLIENT_ID_AQUI
HELISA_SECRET_KEY=TU_SECRET_KEY_AQUI

# ===== JWT =====
JWT_SECRET_KEY=dev_jwt_secret_key_CHANGE_IN_PRODUCTION_min_32_chars

# ===== Slack (opcional) =====
SLACK_WEBHOOK_URL=

# ===== Environment =====
NODE_ENV=development
PYTHON_ENV=development
```

**⚠️ IMPORTANTE**: Nunca commitees el archivo `.env` a git. Está en `.gitignore`.

### 3. Instalar Dependencias

```bash
# Usar Makefile para instalar todo
make install

# Esto ejecuta:
# - npm install en apps/frontend
# - poetry install en todos los servicios Python
# - Puede tardar 5-10 minutos en primera ejecución
```

**Si prefieres instalación manual**:

```bash
# Frontend (Angular)
cd apps/frontend
npm install
cd ../..

# BFF Service
cd apps/bff-service
pip install poetry
poetry install
cd ../..

# Helisa Adapter
cd apps/helisa-adapter
poetry install
cd ../..

# Sync Service
cd apps/sync-service
poetry install
cd ../..

# Audit Service
cd apps/audit-service
poetry install
cd ../..
```

### 4. Levantar Infraestructura

```bash
# Levantar PostgreSQL, Redis, RabbitMQ, OpenSearch
docker-compose up -d postgres redis rabbitmq opensearch

# Verificar que estén running
docker-compose ps

# Deberías ver:
# NAME                STATUS
# helisa-postgres     Up (healthy)
# helisa-redis        Up (healthy)
# helisa-rabbitmq     Up (healthy)
# helisa-opensearch   Up (healthy)
```

**Esperar health checks** (30-60 segundos):
```bash
# Verificar logs
docker-compose logs -f postgres

# Esperar mensaje: "database system is ready to accept connections"
# Ctrl+C para salir de logs
```

### 5. Aplicar Migraciones de Base de Datos

```bash
# Crear esquema inicial
make db-migrate

# Esto ejecuta Alembic en cada servicio
# Verás output como:
# INFO  [alembic.runtime.migration] Running upgrade -> abc123, Initial schema
# INFO  [alembic.runtime.migration] Running upgrade abc123 -> def456, Add audit logs
```

**Si falla**, verificar:
```bash
# Conectar manualmente a PostgreSQL
docker exec -it helisa-postgres psql -U helisa_user -d helisa_innovate

# Verificar tablas
\dt

# Debería mostrar: users, accounts, accounting_entries, etc.
```

### 6. Seed de Datos (Opcional)

```bash
# Cargar datos de prueba
make db-seed

# Esto crea:
# - Usuario admin: admin@innovatenutrition.com / Admin123!
# - 5 usuarios de prueba
# - 50 cuentas contables
# - 10 centros de costo
# - 20 productos
```

### 7. Verificar Instalación

```bash
# Test backend services
cd apps/bff-service
poetry run pytest tests/unit -v

cd ../helisa-adapter
poetry run pytest tests/unit -v

# Test frontend
cd ../frontend
npm run test:ci
```

**Si todos los tests pasan**: ✅ Setup completo!

---

## Desarrollo Local

### Modo 1: Todo con Docker

```bash
# Levantar stack completo (infra + servicios)
make docker-up

# Ver logs
make docker-logs

# Acceder:
# - Frontend: http://localhost:4200
# - BFF API: http://localhost:8000
# - OpenSearch Dashboards: http://localhost:5601
# - RabbitMQ Management: http://localhost:15672 (user: helisa, pass: dev_password_123)

# Parar todo
make docker-down
```

### Modo 2: Infra Docker + Servicios Local (Recomendado para desarrollo)

```bash
# Terminal 1: Infra
docker-compose up -d postgres redis rabbitmq opensearch

# Terminal 2: Backend services
make dev-backend

# Terminal 3: Frontend
cd apps/frontend
npm start

# Acceder:
# - Frontend: http://localhost:4200
# - BFF: http://localhost:8000
# - Helisa Adapter: http://localhost:8001
# - Sync Service: http://localhost:8002
# - Audit Service: http://localhost:8003
```

**Ventajas Modo 2**:
- ✅ Hot reload en todos los servicios
- ✅ Debugging en IDE
- ✅ Logs en consola
- ✅ Más rápido para iterar

### Modo 3: Solo Backend

```bash
# Solo backend (sin frontend)
docker-compose up -d postgres redis rabbitmq

cd apps/bff-service
poetry run uvicorn app.main:app --reload --port 8000

# Acceder a docs interactivas:
# http://localhost:8000/docs (Swagger UI)
# http://localhost:8000/redoc (ReDoc)
```

### Modo 4: Solo Frontend (con backend mock)

```bash
cd apps/frontend

# Levantar con backend mock
npm run start:mock

# Esto usa interceptores HTTP para mockear respuestas
# Útil para desarrollo de UI sin backend
```

---

## Workflows de Desarrollo

### Flujo de Trabajo Típico

```bash
# 1. Actualizar rama develop
git checkout develop
git pull origin develop

# 2. Crear rama de feature
git checkout -b feature/ABC-123-nombre-descriptivo
# Patrón: feature/TICKET-descripcion o fix/TICKET-descripcion

# 3. Hacer cambios...
# Editar código, agregar tests, etc.

# 4. Ejecutar tests
make test

# 5. Ejecutar linter
make lint

# 6. Auto-formatear código
make format

# 7. Commit
git add .
git commit -m "feat(accounting): add entry validation

- Validate debit = credit
- Add custom error messages
- Update tests

Closes ABC-123"

# Formato: tipo(scope): descripción
# Tipos: feat, fix, docs, style, refactor, test, chore

# 8. Push
git push -u origin feature/ABC-123-nombre-descriptivo

# 9. Crear Pull Request en GitHub
# Ir a https://github.com/innovate-nutrition/helisa-innovate-services/pulls
# Clic en "New Pull Request"
```

### Crear Nuevo Endpoint (Backend)

**Ejemplo**: Agregar endpoint GET `/api/v1/products/categories`

```bash
# 1. Crear archivo de endpoint
touch apps/bff-service/app/api/v1/endpoints/products.py

# 2. Implementar endpoint
```

```python
# apps/bff-service/app/api/v1/endpoints/products.py
from fastapi import APIRouter, Depends
from typing import List
from ....core.security import get_current_user, TokenData
from ....schemas.products import CategoryResponse

router = APIRouter(prefix="/products", tags=["Productos"])

@router.get("/categories", response_model=List[CategoryResponse])
async def list_categories(
    current_user: TokenData = Depends(get_current_user)
):
    """
    Lista todas las categorías de productos
    """
    # TODO: Implementar lógica
    categories = [
        {"id": "1", "name": "Proteínas", "count": 25},
        {"id": "2", "name": "Vitaminas", "count": 15}
    ]
    return categories
```

```bash
# 3. Registrar router
# En apps/bff-service/app/api/v1/router.py

from .endpoints import products

api_router = APIRouter()
api_router.include_router(products.router)
```

```bash
# 4. Crear schema
touch apps/bff-service/app/schemas/products.py
```

```python
# apps/bff-service/app/schemas/products.py
from pydantic import BaseModel

class CategoryResponse(BaseModel):
    id: str
    name: str
    count: int
```

```bash
# 5. Crear test
touch apps/bff-service/tests/integration/test_products.py
```

```python
# apps/bff-service/tests/integration/test_products.py
import pytest

@pytest.mark.asyncio
async def test_list_categories(async_client, auth_token):
    response = await async_client.get(
        "/api/v1/products/categories",
        headers={"Authorization": f"Bearer {auth_token}"}
    )

    assert response.status_code == 200
    data = response.json()
    assert isinstance(data, list)
    assert len(data) > 0
    assert "id" in data[0]
    assert "name" in data[0]
```

```bash
# 6. Ejecutar test
cd apps/bff-service
poetry run pytest tests/integration/test_products.py -v

# 7. Verificar en Swagger UI
# http://localhost:8000/docs
# Debería aparecer el nuevo endpoint
```

### Crear Nuevo Componente (Frontend)

**Ejemplo**: Componente de lista de productos

```bash
cd apps/frontend

# Generar componente con Angular CLI
ng generate component features/inventory/components/product-list --standalone

# Esto crea:
# - product-list.component.ts
# - product-list.component.html
# - product-list.component.scss
# - product-list.component.spec.ts
```

```typescript
// apps/frontend/src/app/features/inventory/components/product-list/product-list.component.ts
import { Component, OnInit, inject, signal } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ProductsService } from '../../services/products.service';
import { Product } from '../../models/product.model';

@Component({
  selector: 'app-product-list',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './product-list.component.html',
  styleUrls: ['./product-list.component.scss']
})
export class ProductListComponent implements OnInit {
  private productsService = inject(ProductsService);

  products = signal<Product[]>([]);
  loading = signal(false);
  error = signal<string | null>(null);

  ngOnInit() {
    this.loadProducts();
  }

  async loadProducts() {
    this.loading.set(true);
    this.error.set(null);

    try {
      const products = await this.productsService.getProducts();
      this.products.set(products);
    } catch (err) {
      this.error.set('Error al cargar productos');
      console.error(err);
    } finally {
      this.loading.set(false);
    }
  }
}
```

```html
<!-- apps/frontend/src/app/features/inventory/components/product-list/product-list.component.html -->
<div class="product-list">
  <h2>Lista de Productos</h2>

  @if (loading()) {
    <p>Cargando...</p>
  }

  @if (error()) {
    <div class="error-message">{{ error() }}</div>
  }

  @if (products().length > 0) {
    <table>
      <thead>
        <tr>
          <th>Código</th>
          <th>Nombre</th>
          <th>Precio</th>
          <th>Stock</th>
        </tr>
      </thead>
      <tbody>
        @for (product of products(); track product.code) {
          <tr>
            <td>{{ product.code }}</td>
            <td>{{ product.name }}</td>
            <td>{{ product.price | currency:'COP':'symbol-narrow' }}</td>
            <td>{{ product.stock }}</td>
          </tr>
        }
      </tbody>
    </table>
  }
</div>
```

```bash
# Ejecutar tests del componente
npm run test -- --include='**/product-list.component.spec.ts'

# Ver en navegador
npm start
# Navegar a http://localhost:4200/inventory/products
```

### Migración de Base de Datos

```bash
cd apps/bff-service

# Crear migración automática (detecta cambios en models)
poetry run alembic revision --autogenerate -m "Add user preferences table"

# Esto crea archivo en alembic/versions/abc123_add_user_preferences.py

# Revisar migración generada
cat alembic/versions/*_add_user_preferences.py

# Aplicar migración
poetry run alembic upgrade head

# Si algo sale mal, hacer rollback
poetry run alembic downgrade -1

# Ver historial de migraciones
poetry run alembic history

# Ver estado actual
poetry run alembic current
```

### Debugging

#### Backend (Python)

```python
# Opción 1: Agregar breakpoints en código
import ipdb; ipdb.set_trace()  # Debugger interactivo

# O con built-in
import pdb; pdb.set_trace()

# Ejecutar con debugger
poetry run uvicorn app.main:app --reload
```

**VSCode Launch Config**:
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: FastAPI",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": [
        "app.main:app",
        "--reload",
        "--host", "0.0.0.0",
        "--port", "8000"
      ],
      "cwd": "${workspaceFolder}/apps/bff-service",
      "env": {
        "DATABASE_URL": "postgresql+asyncpg://helisa_user:dev_password_123@localhost:5432/helisa_innovate"
      }
    }
  ]
}
```

#### Frontend (Angular)

```typescript
// Usar console en desarrollo
console.log('Debug:', this.products());
console.table(this.products());  // Tabla en consola

// Debugger nativo
debugger;  // Pausa ejecución en DevTools
```

**VSCode Launch Config**:
```json
{
  "name": "Angular: Chrome",
  "type": "chrome",
  "request": "launch",
  "url": "http://localhost:4200",
  "webRoot": "${workspaceFolder}/apps/frontend",
  "sourceMapPathOverrides": {
    "webpack:/*": "${webRoot}/*"
  }
}
```

### Ver Logs

```bash
# Logs de servicios Docker
docker-compose logs -f bff-service
docker-compose logs -f helisa-adapter
docker-compose logs -f postgres

# Logs de servicios locales (si están en terminales)
# Simplemente ver la terminal correspondiente

# Logs de PostgreSQL (queries lentas)
docker exec -it helisa-postgres tail -f /var/log/postgresql/postgresql-16-main.log

# Logs de NGINX
docker exec -it helisa-nginx tail -f /var/log/nginx/access.log
docker exec -it helisa-nginx tail -f /var/log/nginx/error.log
```

---

## Troubleshooting

### Problema: PostgreSQL no inicia

```bash
# Verificar logs
docker-compose logs postgres

# Común: Puerto 5432 ya en uso
# Solución: Parar PostgreSQL local
sudo systemctl stop postgresql
# O en Mac: brew services stop postgresql

# O cambiar puerto en docker-compose.yml
ports:
  - "5433:5432"  # Usar 5433 externamente
```

### Problema: Migraciones fallan

```bash
# Reset completo de base de datos (⚠️ BORRA TODO)
docker-compose down -v
docker-compose up -d postgres
# Esperar 30 segundos
make db-migrate
make db-seed
```

### Problema: Tests fallan por timeout

```bash
# Aumentar timeout en pytest.ini
[pytest]
asyncio_mode = auto
timeout = 300  # 5 minutos

# O en test específico
@pytest.mark.timeout(60)
async def test_slow_operation():
    ...
```

### Problema: Frontend no carga

```bash
# Limpiar cache de node_modules
cd apps/frontend
rm -rf node_modules package-lock.json
npm install

# Limpiar cache de Angular
npm run clean
npm run build

# Verificar puerto
lsof -i :4200  # Mac/Linux
netstat -ano | findstr :4200  # Windows

# Matar proceso si hay conflicto
kill -9 <PID>
```

### Problema: Helisa API retorna error 1001 (firma inválida)

```bash
# Verificar variables de entorno
echo $HELISA_SECRET_KEY
echo $HELISA_CLIENT_ID

# Test de firma en Python
cd apps/helisa-adapter
poetry run python -c "
from app.services.signature import HelisaSignatureService
import os

service = HelisaSignatureService(
    secret_key=os.getenv('HELISA_SECRET_KEY'),
    client_id=os.getenv('HELISA_CLIENT_ID')
)

data = {'test': 'data'}
result = service.generate_signature(data)
print('Signature generated:', result['sign'])
"

# Si muestra error, revisar:
# 1. Secret key correcta (sin espacios extras)
# 2. Client ID correcto
# 3. JSON encoding (debe ser minificado, sin espacios)
```

### Problema: Docker consume mucha RAM

```bash
# Ver uso de recursos
docker stats

# Limpiar imágenes y contenedores no usados
docker system prune -a

# Limitar memoria en docker-compose.yml
services:
  postgres:
    mem_limit: 512m
    memswap_limit: 512m
```

### Problema: Hot reload no funciona

**Backend**:
```bash
# Verificar que estés usando --reload
poetry run uvicorn app.main:app --reload

# Verificar que el volumen esté montado en docker-compose.yml
volumes:
  - ./apps/bff-service:/app
```

**Frontend**:
```bash
# Reiniciar dev server
npm start

# Verificar angular.json
"options": {
  "poll": 2000  # Polling cada 2 segundos (útil en Docker)
}
```

---

## Recursos Adicionales

### Documentación Interna
- [Arquitectura de Alto Nivel](./ARCHITECTURE_DESIGN.md)
- [Especificación Técnica](./TECHNICAL_SPECIFICATION.md)
- [Guía de Implementación](./IMPLEMENTATION_GUIDE.md)
- [API de Helisa](./HELISA_API_ARCHITECTURE.md)
- [Firma de Seguridad](./HELISA_SECURITY_SIGNATURE.md)

### Documentación Externa
- [FastAPI Docs](https://fastapi.tiangolo.com/)
- [Angular Docs](https://angular.dev/)
- [PostgreSQL 16](https://www.postgresql.org/docs/16/)
- [SQLAlchemy 2.0](https://docs.sqlalchemy.org/en/20/)
- [Pydantic](https://docs.pydantic.dev/)
- [Docker Docs](https://docs.docker.com/)
- [Kubernetes Docs](https://kubernetes.io/docs/)

### Comandos Útiles (Cheat Sheet)

```bash
# ===== General =====
make help                    # Ver todos los comandos disponibles
make install                 # Instalar todas las dependencias
make dev                     # Levantar entorno completo
make test                    # Ejecutar todos los tests
make lint                    # Linter en todo el código
make format                  # Auto-formatear código

# ===== Docker =====
make docker-up               # Levantar stack completo
make docker-down             # Bajar stack
make docker-logs             # Ver logs de todos los contenedores
make docker-rebuild          # Rebuild de imágenes

# ===== Base de Datos =====
make db-migrate              # Aplicar migraciones
make db-rollback             # Rollback última migración
make db-seed                 # Cargar datos de prueba
psql -h localhost -U helisa_user -d helisa_innovate  # Conectar a DB

# ===== Testing =====
make test                    # Todos los tests
make test-coverage           # Tests con coverage
cd apps/bff-service && poetry run pytest tests/unit -v  # Tests unitarios
cd apps/frontend && npm run test:ci  # Tests frontend

# ===== Servicios Individuales =====
cd apps/bff-service && poetry run uvicorn app.main:app --reload
cd apps/helisa-adapter && poetry run uvicorn app.main:app --reload --port 8001
cd apps/frontend && npm start

# ===== Git =====
git checkout -b feature/ABC-123-descripcion
git add .
git commit -m "feat(scope): descripción"
git push -u origin feature/ABC-123-descripcion

# ===== Debugging =====
docker-compose logs -f <service>
docker exec -it <container> bash
docker stats  # Ver uso de recursos
```

### Tips y Trucos

1. **Alias útiles** (agregar a `.bashrc` o `.zshrc`):
```bash
alias dcu='docker-compose up -d'
alias dcd='docker-compose down'
alias dcl='docker-compose logs -f'
alias dcr='docker-compose restart'
alias dps='docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"'
```

2. **Pre-commit hooks** (validación automática antes de commit):
```bash
# Instalar pre-commit
pip install pre-commit

# Setup hooks
pre-commit install

# Ahora en cada commit:
# - Se ejecuta linter
# - Se ejecuta formatter
# - Se verifican secrets
# - Se ejecutan tests rápidos
```

3. **VS Code Extensions Recomendadas**:
- Python (Microsoft)
- Pylance
- Angular Language Service
- Docker
- GitLens
- Thunder Client (alternativa a Postman)
- SQLTools (para PostgreSQL)

4. **Hot reload más rápido**:
```bash
# Backend: usar watchfiles en lugar de watchgod
poetry add --dev watchfiles

# Usar en uvicorn:
poetry run uvicorn app.main:app --reload --reload-include='*.py'
```

---

## Contacto y Soporte

**Canal de Slack**: #helisa-dev
**Email del equipo**: dev-helisa@innovatenutrition.com
**Documentación**: https://docs.innovatenutrition.com/helisa/
**Jira Board**: https://innovatenutrition.atlassian.net/helisa/

**Horario de soporte**: Lunes a Viernes, 9:00 AM - 6:00 PM (COT)

---

**Happy Coding! 🚀**

**Versión**: 1.0
**Última actualización**: 2024-11-08
