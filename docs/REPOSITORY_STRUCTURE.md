# Estructura del Monorepo: Helisa Innovate Services

## Tabla de Contenidos
- [Visión General del Monorepo](#visión-general-del-monorepo)
- [Árbol Completo](#árbol-completo)
- [Descripción de Carpetas](#descripción-de-carpetas)
- [Convenciones de Código](#convenciones-de-código)
- [Scripts y Comandos](#scripts-y-comandos)

---

## Visión General del Monorepo

### ¿Por qué Monorepo?

**Ventajas**:
- **Código compartido**: Librerías comunes entre frontend y backend
- **Refactoring atómico**: Cambios en múltiples servicios en un solo commit
- **CI/CD unificado**: Pipeline único para todo el proyecto
- **Versionado sincronizado**: Todos los servicios compatibles

**Herramientas**:
- **pnpm workspaces** (frontend/shared)
- **Poetry** (Python services)
- **Docker Compose** (desarrollo local)
- **Makefile** (comandos comunes)

---

## Árbol Completo

```
helisa-innovate-services/
├── .github/
│   ├── workflows/
│   │   ├── ci-frontend.yml              # CI para Angular
│   │   ├── ci-backend.yml               # CI para Python services
│   │   ├── cd-staging.yml               # Deploy a staging
│   │   └── cd-production.yml            # Deploy a producción
│   ├── dependabot.yml
│   └── CODEOWNERS
│
├── apps/
│   ├── frontend/                        # Angular 18+ App
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── core/
│   │   │   │   │   ├── auth/
│   │   │   │   │   │   ├── guards/
│   │   │   │   │   │   │   ├── auth.guard.ts
│   │   │   │   │   │   │   └── domain.guard.ts
│   │   │   │   │   │   ├── interceptors/
│   │   │   │   │   │   │   ├── auth.interceptor.ts
│   │   │   │   │   │   │   ├── error.interceptor.ts
│   │   │   │   │   │   │   └── logging.interceptor.ts
│   │   │   │   │   │   ├── services/
│   │   │   │   │   │   │   ├── auth.service.ts
│   │   │   │   │   │   │   ├── token.service.ts
│   │   │   │   │   │   │   └── user.service.ts
│   │   │   │   │   │   └── models/
│   │   │   │   │   │       ├── user.model.ts
│   │   │   │   │   │       └── auth.model.ts
│   │   │   │   │   ├── http/
│   │   │   │   │   │   ├── api.service.ts
│   │   │   │   │   │   └── http-config.ts
│   │   │   │   │   └── config/
│   │   │   │   │       ├── app.config.ts
│   │   │   │   │       └── environment.service.ts
│   │   │   │   ├── features/
│   │   │   │   │   ├── auth/                    # Login/Register
│   │   │   │   │   │   ├── login/
│   │   │   │   │   │   │   ├── login.component.ts
│   │   │   │   │   │   │   ├── login.component.html
│   │   │   │   │   │   │   ├── login.component.scss
│   │   │   │   │   │   │   └── login.component.spec.ts
│   │   │   │   │   │   └── register/
│   │   │   │   │   │       └── ...
│   │   │   │   │   ├── dashboard/               # Home dashboard
│   │   │   │   │   │   ├── dashboard.component.ts
│   │   │   │   │   │   ├── components/
│   │   │   │   │   │   │   ├── sales-widget/
│   │   │   │   │   │   │   ├── inventory-widget/
│   │   │   │   │   │   │   └── alerts-widget/
│   │   │   │   │   │   └── services/
│   │   │   │   │   │       └── dashboard.service.ts
│   │   │   │   │   ├── accounting/              # Módulo contable
│   │   │   │   │   │   ├── accounting-entries/
│   │   │   │   │   │   │   ├── list/
│   │   │   │   │   │   │   ├── create/
│   │   │   │   │   │   │   └── detail/
│   │   │   │   │   │   ├── accounts/
│   │   │   │   │   │   │   ├── list/
│   │   │   │   │   │   │   └── detail/
│   │   │   │   │   │   ├── cost-centers/
│   │   │   │   │   │   ├── services/
│   │   │   │   │   │   │   ├── accounting.service.ts
│   │   │   │   │   │   │   └── accounts.service.ts
│   │   │   │   │   │   └── models/
│   │   │   │   │   │       ├── accounting-entry.model.ts
│   │   │   │   │   │       └── account.model.ts
│   │   │   │   │   ├── inventory/               # Módulo inventario
│   │   │   │   │   │   ├── products/
│   │   │   │   │   │   │   ├── list/
│   │   │   │   │   │   │   ├── create/
│   │   │   │   │   │   │   └── detail/
│   │   │   │   │   │   ├── movements/
│   │   │   │   │   │   ├── warehouses/
│   │   │   │   │   │   ├── services/
│   │   │   │   │   │   │   ├── products.service.ts
│   │   │   │   │   │   │   └── inventory.service.ts
│   │   │   │   │   │   └── models/
│   │   │   │   │   │       └── product.model.ts
│   │   │   │   │   ├── sales/                   # Ventas
│   │   │   │   │   │   ├── orders/
│   │   │   │   │   │   ├── invoices/
│   │   │   │   │   │   └── customers/
│   │   │   │   │   ├── purchases/               # Compras
│   │   │   │   │   │   ├── orders/
│   │   │   │   │   │   └── suppliers/
│   │   │   │   │   └── reports/                 # Reportes
│   │   │   │   │       ├── financial-statements/
│   │   │   │   │       ├── inventory-reports/
│   │   │   │   │       └── audit-logs/
│   │   │   │   ├── shared/
│   │   │   │   │   ├── components/
│   │   │   │   │   │   ├── data-table/
│   │   │   │   │   │   │   ├── data-table.component.ts
│   │   │   │   │   │   │   ├── data-table.component.html
│   │   │   │   │   │   │   └── data-table.component.scss
│   │   │   │   │   │   ├── form-controls/
│   │   │   │   │   │   │   ├── date-picker/
│   │   │   │   │   │   │   ├── autocomplete/
│   │   │   │   │   │   │   └── currency-input/
│   │   │   │   │   │   ├── modals/
│   │   │   │   │   │   │   ├── confirm-dialog/
│   │   │   │   │   │   │   └── alert-dialog/
│   │   │   │   │   │   └── layout/
│   │   │   │   │   │       ├── header/
│   │   │   │   │   │       ├── sidebar/
│   │   │   │   │   │       └── footer/
│   │   │   │   │   ├── directives/
│   │   │   │   │   │   ├── has-permission.directive.ts
│   │   │   │   │   │   └── currency-format.directive.ts
│   │   │   │   │   ├── pipes/
│   │   │   │   │   │   ├── currency.pipe.ts
│   │   │   │   │   │   ├── date-format.pipe.ts
│   │   │   │   │   │   └── search-highlight.pipe.ts
│   │   │   │   │   ├── validators/
│   │   │   │   │   │   ├── email-domain.validator.ts
│   │   │   │   │   │   └── accounting-balance.validator.ts
│   │   │   │   │   └── utils/
│   │   │   │   │       ├── date.utils.ts
│   │   │   │   │       └── format.utils.ts
│   │   │   │   ├── app.component.ts
│   │   │   │   ├── app.routes.ts                # Standalone routing
│   │   │   │   └── app.config.ts                # App config
│   │   │   ├── assets/
│   │   │   │   ├── images/
│   │   │   │   ├── icons/
│   │   │   │   └── i18n/
│   │   │   │       ├── es.json
│   │   │   │       └── en.json
│   │   │   ├── environments/
│   │   │   │   ├── environment.ts
│   │   │   │   ├── environment.development.ts
│   │   │   │   └── environment.production.ts
│   │   │   ├── styles/
│   │   │   │   ├── _variables.scss
│   │   │   │   ├── _mixins.scss
│   │   │   │   └── styles.scss
│   │   │   ├── index.html
│   │   │   └── main.ts
│   │   ├── angular.json
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── tsconfig.app.json
│   │   ├── tsconfig.spec.json
│   │   ├── .eslintrc.json
│   │   └── README.md
│   │
│   ├── bff-service/                     # Backend for Frontend
│   │   ├── app/
│   │   │   ├── api/
│   │   │   │   ├── v1/
│   │   │   │   │   ├── endpoints/
│   │   │   │   │   │   ├── __init__.py
│   │   │   │   │   │   ├── dashboard.py
│   │   │   │   │   │   ├── accounting.py
│   │   │   │   │   │   ├── inventory.py
│   │   │   │   │   │   └── sync.py
│   │   │   │   │   ├── dependencies.py
│   │   │   │   │   └── router.py
│   │   │   │   └── __init__.py
│   │   │   ├── core/
│   │   │   │   ├── config.py                    # Pydantic Settings
│   │   │   │   ├── security.py                  # JWT validation
│   │   │   │   ├── cache.py                     # Redis client
│   │   │   │   └── logging.py                   # Structured logging
│   │   │   ├── services/
│   │   │   │   ├── helisa_client.py             # HTTP client to Helisa Adapter
│   │   │   │   ├── sync_client.py               # HTTP client to Sync Service
│   │   │   │   ├── aggregator.py                # Data aggregation logic
│   │   │   │   └── cache_service.py             # Cache strategies
│   │   │   ├── schemas/
│   │   │   │   ├── dashboard.py
│   │   │   │   ├── accounting.py
│   │   │   │   ├── inventory.py
│   │   │   │   └── common.py                    # Shared schemas
│   │   │   ├── middleware/
│   │   │   │   ├── auth.py
│   │   │   │   ├── cors.py
│   │   │   │   └── rate_limit.py
│   │   │   ├── main.py                          # FastAPI app
│   │   │   └── __init__.py
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── conftest.py
│   │   ├── alembic/                             # DB migrations
│   │   │   ├── versions/
│   │   │   ├── env.py
│   │   │   └── alembic.ini
│   │   ├── Dockerfile
│   │   ├── pyproject.toml                       # Poetry
│   │   ├── poetry.lock
│   │   ├── .env.example
│   │   └── README.md
│   │
│   ├── helisa-adapter/                  # Helisa Integration Service
│   │   ├── app/
│   │   │   ├── api/
│   │   │   │   └── v1/
│   │   │   │       ├── endpoints/
│   │   │   │       │   ├── accounts.py
│   │   │   │       │   ├── products.py
│   │   │   │       │   ├── accounting_entries.py
│   │   │   │       │   └── inventory.py
│   │   │   │       └── router.py
│   │   │   ├── core/
│   │   │   │   ├── config.py
│   │   │   │   ├── database.py                  # SQLAlchemy async
│   │   │   │   ├── security.py                  # HMAC signature
│   │   │   │   └── exceptions.py
│   │   │   ├── services/
│   │   │   │   ├── helisa_api.py                # Helisa HTTP client
│   │   │   │   ├── signature.py                 # HMAC-SHA256 generator
│   │   │   │   ├── retry.py                     # Retry logic
│   │   │   │   └── circuit_breaker.py           # Circuit breaker
│   │   │   ├── models/
│   │   │   │   ├── accounting.py                # SQLAlchemy models
│   │   │   │   ├── inventory.py
│   │   │   │   └── sync_log.py
│   │   │   ├── schemas/
│   │   │   │   ├── helisa_request.py
│   │   │   │   ├── helisa_response.py
│   │   │   │   └── internal.py
│   │   │   ├── workers/
│   │   │   │   ├── sync_worker.py               # RabbitMQ consumer
│   │   │   │   └── retry_worker.py
│   │   │   ├── main.py
│   │   │   └── __init__.py
│   │   ├── tests/
│   │   ├── alembic/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   └── README.md
│   │
│   ├── sync-service/                    # Synchronization Service
│   │   ├── app/
│   │   │   ├── api/
│   │   │   │   └── v1/
│   │   │   │       ├── endpoints/
│   │   │   │       │   ├── sync_jobs.py
│   │   │   │       │   └── reconciliation.py
│   │   │   │       └── router.py
│   │   │   ├── core/
│   │   │   │   ├── config.py
│   │   │   │   ├── database.py
│   │   │   │   └── scheduler.py                 # APScheduler
│   │   │   ├── jobs/
│   │   │   │   ├── sync_catalogs.py             # Cron jobs
│   │   │   │   ├── reconcile_accounts.py
│   │   │   │   └── retry_failed.py
│   │   │   ├── services/
│   │   │   │   ├── diff_service.py              # Data diffing
│   │   │   │   ├── notification.py              # Slack/Email
│   │   │   │   └── metrics.py
│   │   │   ├── models/
│   │   │   │   ├── sync_log.py
│   │   │   │   └── reconciliation.py
│   │   │   ├── main.py
│   │   │   └── __init__.py
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   └── README.md
│   │
│   └── audit-service/                   # Audit & Logging Service
│       ├── app/
│       │   ├── api/
│       │   │   └── v1/
│       │   │       ├── endpoints/
│       │   │       │   ├── audit_logs.py
│       │   │       │   └── search.py
│       │   │       └── router.py
│       │   ├── core/
│       │   │   ├── config.py
│       │   │   ├── database.py
│       │   │   └── opensearch.py                # OpenSearch client
│       │   ├── services/
│       │   │   ├── audit_logger.py
│       │   │   ├── indexer.py                   # OpenSearch indexing
│       │   │   └── search.py
│       │   ├── models/
│       │   │   └── audit_log.py
│       │   ├── workers/
│       │   │   └── audit_consumer.py            # RabbitMQ consumer
│       │   ├── main.py
│       │   └── __init__.py
│       ├── tests/
│       ├── Dockerfile
│       ├── pyproject.toml
│       └── README.md
│
├── libs/
│   ├── python-shared/                   # Shared Python code
│   │   ├── helisa_shared/
│   │   │   ├── __init__.py
│   │   │   ├── models/
│   │   │   │   ├── base.py
│   │   │   │   └── common.py
│   │   │   ├── utils/
│   │   │   │   ├── date.py
│   │   │   │   ├── validation.py
│   │   │   │   └── crypto.py
│   │   │   ├── schemas/
│   │   │   │   └── base_schemas.py
│   │   │   └── exceptions/
│   │   │       └── custom_exceptions.py
│   │   ├── tests/
│   │   ├── pyproject.toml
│   │   └── README.md
│   │
│   └── typescript-shared/               # Shared TypeScript code
│       ├── src/
│       │   ├── models/
│       │   │   └── api-models.ts
│       │   ├── utils/
│       │   │   └── validators.ts
│       │   └── constants/
│       │       └── app-constants.ts
│       ├── package.json
│       ├── tsconfig.json
│       └── README.md
│
├── infrastructure/
│   ├── docker/
│   │   ├── nginx/
│   │   │   ├── Dockerfile
│   │   │   ├── nginx.conf
│   │   │   └── ssl/
│   │   ├── postgres/
│   │   │   ├── Dockerfile
│   │   │   └── init.sql
│   │   ├── keycloak/
│   │   │   ├── Dockerfile
│   │   │   └── realm-export.json
│   │   └── opensearch/
│   │       ├── Dockerfile
│   │       └── opensearch.yml
│   │
│   ├── kubernetes/
│   │   ├── base/
│   │   │   ├── namespace.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── secrets.yaml
│   │   ├── apps/
│   │   │   ├── frontend-deployment.yaml
│   │   │   ├── bff-deployment.yaml
│   │   │   ├── helisa-adapter-deployment.yaml
│   │   │   ├── sync-service-deployment.yaml
│   │   │   └── audit-service-deployment.yaml
│   │   ├── services/
│   │   │   ├── postgres-statefulset.yaml
│   │   │   ├── redis-deployment.yaml
│   │   │   ├── rabbitmq-statefulset.yaml
│   │   │   └── opensearch-statefulset.yaml
│   │   ├── ingress/
│   │   │   ├── ingress.yaml
│   │   │   └── certificate.yaml
│   │   └── monitoring/
│   │       ├── prometheus/
│   │       └── grafana/
│   │
│   ├── terraform/                       # IaC (opcional)
│   │   ├── modules/
│   │   │   ├── eks/
│   │   │   ├── rds/
│   │   │   └── vpc/
│   │   ├── environments/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── scripts/
│       ├── deploy.sh
│       ├── rollback.sh
│       ├── backup-db.sh
│       └── seed-data.sh
│
├── docs/
│   ├── HELISA_API_ARCHITECTURE.md       # Ya creado
│   ├── HELISA_SECURITY_SIGNATURE.md     # Ya creado
│   ├── ARCHITECTURE_DESIGN.md           # Ya creado
│   ├── REPOSITORY_STRUCTURE.md          # Este archivo
│   ├── API_CONTRACTS.md                 # Por crear
│   ├── DATABASE_SCHEMA.md               # Por crear
│   ├── SECURITY_DESIGN.md               # Por crear
│   ├── DEPLOYMENT.md                    # Por crear
│   ├── DEVELOPER_GUIDE.md               # Por crear
│   ├── TESTING_STRATEGY.md              # Por crear
│   └── EXECUTIVE_SUMMARY.md             # Por crear
│
├── .github/
│   ├── workflows/
│   └── PULL_REQUEST_TEMPLATE.md
│
├── .gitignore
├── .dockerignore
├── .editorconfig
├── .env.example
├── docker-compose.yml                   # Desarrollo local
├── docker-compose.prod.yml              # Producción
├── Makefile                             # Comandos comunes
├── README.md                            # Raíz del proyecto
└── CONTRIBUTING.md
```

---

## Descripción de Carpetas

### `/apps` - Aplicaciones

Contiene todos los servicios deployables independientes.

#### `/apps/frontend` - Angular 18+
- **Standalone components**: Sin NgModules
- **Signals**: Estado reactivo nativo
- **Lazy loading**: Por feature module
- **Estructura por features**: Cada módulo de negocio aislado

**Comandos**:
```bash
cd apps/frontend
npm install
npm run start              # Dev server
npm run build              # Production build
npm run test               # Unit tests
npm run lint               # ESLint
```

#### `/apps/bff-service` - Backend for Frontend
- **FastAPI**: Async por defecto
- **Agregación**: Múltiples servicios → 1 respuesta
- **Cache**: Redis para optimización
- **OpenAPI**: Auto-documentación

**Comandos**:
```bash
cd apps/bff-service
poetry install
poetry run uvicorn app.main:app --reload  # Dev server
poetry run pytest                          # Tests
poetry run alembic upgrade head           # Migrations
```

#### `/apps/helisa-adapter` - Helisa Integration
- **HMAC-SHA256**: Firma de seguridad Helisa
- **Circuit breaker**: Protección contra Helisa down
- **Retry logic**: Exponential backoff
- **Queue workers**: RabbitMQ consumers

#### `/apps/sync-service` - Synchronization
- **APScheduler**: Cron jobs programados
- **Diff detection**: Comparación de datos
- **Reconciliation**: Corrección de inconsistencias
- **Notifications**: Alertas Slack/Email

#### `/apps/audit-service` - Audit & Logging
- **Append-only**: Logs inmutables
- **OpenSearch**: Indexación de logs
- **Dashboards**: Kibana integrado
- **Alerting**: Reglas de seguridad

### `/libs` - Librerías Compartidas

#### `/libs/python-shared`
Código reutilizable entre microservicios Python:
- **Models**: Pydantic base models
- **Utils**: Validaciones, fechas, crypto
- **Exceptions**: Excepciones custom
- **Schemas**: Schemas comunes

**Uso**:
```python
# En cualquier servicio
from helisa_shared.models import BaseModel
from helisa_shared.utils.date import helisa_date_format
```

#### `/libs/typescript-shared`
Código reutilizable para Angular:
- **Models**: Interfaces TypeScript
- **Utils**: Validators, formatters
- **Constants**: URLs, configs

**Uso**:
```typescript
// En Angular
import { ApiResponse } from '@helisa/shared/models';
import { validateEmail } from '@helisa/shared/utils';
```

### `/infrastructure` - Infraestructura

#### `/infrastructure/docker`
Dockerfiles y configuraciones por servicio:
- **Multi-stage builds**: Optimización de tamaño
- **Health checks**: Liveness/readiness probes
- **Non-root users**: Seguridad

#### `/infrastructure/kubernetes`
Manifiestos K8s:
- **Base**: Namespace, ConfigMaps, Secrets
- **Apps**: Deployments de servicios
- **Services**: Networking interno
- **Ingress**: Exposición externa

#### `/infrastructure/terraform` (opcional)
Infrastructure as Code:
- **Modules**: Componentes reutilizables
- **Environments**: Dev, staging, prod
- **State**: S3 backend (remote state)

### `/docs` - Documentación

Toda la documentación técnica:
- **Markdown**: Formato estándar
- **Mermaid**: Diagramas embebidos
- **Ejemplos**: Código copiable
- **Versionado**: En git con el código

---

## Convenciones de Código

### Python (PEP 8 + Type Hints)

```python
# Imports ordenados
import os
from typing import Optional, List

from fastapi import FastAPI, Depends
from pydantic import BaseModel

# Type hints obligatorios
def create_entry(
    entry: AccountingEntry,
    user: User = Depends(get_current_user)
) -> AccountingEntryResponse:
    """
    Crea un comprobante contable.

    Args:
        entry: Datos del comprobante
        user: Usuario autenticado

    Returns:
        Respuesta con el ID creado

    Raises:
        ValidationError: Si débitos != créditos
    """
    pass

# Pydantic models
class AccountingEntry(BaseModel):
    document_type: str
    date: HelisaDate
    description: str
    details: List[EntryDetail]

    class Config:
        orm_mode = True
```

### TypeScript/Angular (Strict Mode)

```typescript
// Interfaces explícitas
export interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
}

// Services con inyección de dependencias
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(
    private http: HttpClient,
    private auth: AuthService
  ) {}

  getUser(id: string): Observable<User> {
    return this.http.get<User>(`/api/v1/users/${id}`).pipe(
      catchError(this.handleError)
    );
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    // Error handling
    return throwError(() => new Error(error.message));
  }
}

// Components con Signals
@Component({
  selector: 'app-user-profile',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div>
      <h2>{{ user()?.firstName }}</h2>
    </div>
  `
})
export class UserProfileComponent {
  user = signal<User | null>(null);

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUser('123').subscribe(
      user => this.user.set(user)
    );
  }
}
```

### Naming Conventions

```yaml
Python:
  Files: snake_case (accounting_service.py)
  Classes: PascalCase (AccountingService)
  Functions: snake_case (create_accounting_entry)
  Constants: UPPER_SNAKE_CASE (MAX_RETRY_ATTEMPTS)
  Private: _prefix (_generate_signature)

TypeScript:
  Files: kebab-case (accounting.service.ts)
  Classes: PascalCase (AccountingService)
  Interfaces: PascalCase (AccountingEntry)
  Functions: camelCase (createAccountingEntry)
  Constants: UPPER_SNAKE_CASE (MAX_RETRY_ATTEMPTS)

Git:
  Branches: feature/ABC-123-description
  Commits: "feat(accounting): add entry validation"
  Tags: v1.2.3 (semantic versioning)
```

---

## Scripts y Comandos

### Makefile (Raíz del proyecto)

```makefile
# Makefile
.PHONY: help install dev test build clean

help:                              ## Mostrar esta ayuda
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

install:                           ## Instalar todas las dependencias
	@echo "Instalando frontend..."
	cd apps/frontend && npm install
	@echo "Instalando servicios Python..."
	cd apps/bff-service && poetry install
	cd apps/helisa-adapter && poetry install
	cd apps/sync-service && poetry install
	cd apps/audit-service && poetry install

dev:                               ## Levantar entorno de desarrollo
	docker-compose up -d postgres redis rabbitmq opensearch
	@echo "Esperando servicios..."
	sleep 10
	@echo "Levantando backend..."
	make dev-backend &
	@echo "Levantando frontend..."
	cd apps/frontend && npm run start

dev-backend:                       ## Solo backend services
	cd apps/bff-service && poetry run uvicorn app.main:app --reload --port 8000 &
	cd apps/helisa-adapter && poetry run uvicorn app.main:app --reload --port 8001 &
	cd apps/sync-service && poetry run uvicorn app.main:app --reload --port 8002 &
	cd apps/audit-service && poetry run uvicorn app.main:app --reload --port 8003 &

test:                              ## Ejecutar todos los tests
	@echo "Tests frontend..."
	cd apps/frontend && npm run test:ci
	@echo "Tests backend..."
	cd apps/bff-service && poetry run pytest
	cd apps/helisa-adapter && poetry run pytest
	cd apps/sync-service && poetry run pytest
	cd apps/audit-service && poetry run pytest

test-coverage:                     ## Tests con coverage
	cd apps/bff-service && poetry run pytest --cov=app --cov-report=html
	cd apps/helisa-adapter && poetry run pytest --cov=app --cov-report=html

lint:                              ## Linter en todo el código
	cd apps/frontend && npm run lint
	cd apps/bff-service && poetry run ruff check app/
	cd apps/helisa-adapter && poetry run ruff check app/

format:                            ## Auto-formatear código
	cd apps/frontend && npm run format
	cd apps/bff-service && poetry run black app/
	cd apps/helisa-adapter && poetry run black app/

build:                             ## Build de producción
	@echo "Building frontend..."
	cd apps/frontend && npm run build
	@echo "Building Docker images..."
	docker-compose -f docker-compose.prod.yml build

db-migrate:                        ## Aplicar migraciones de BD
	cd apps/bff-service && poetry run alembic upgrade head
	cd apps/helisa-adapter && poetry run alembic upgrade head
	cd apps/sync-service && poetry run alembic upgrade head
	cd apps/audit-service && poetry run alembic upgrade head

db-rollback:                       ## Rollback última migración
	cd apps/bff-service && poetry run alembic downgrade -1

db-seed:                           ## Seed de datos de prueba
	./infrastructure/scripts/seed-data.sh

clean:                             ## Limpiar archivos generados
	find . -type d -name "__pycache__" -exec rm -rf {} +
	find . -type d -name ".pytest_cache" -exec rm -rf {} +
	find . -type d -name "node_modules" -exec rm -rf {} +
	find . -type d -name "dist" -exec rm -rf {} +

docker-up:                         ## Levantar stack completo con Docker
	docker-compose up -d

docker-down:                       ## Bajar stack
	docker-compose down

docker-logs:                       ## Ver logs de todos los contenedores
	docker-compose logs -f

docker-rebuild:                    ## Rebuild de todos los servicios
	docker-compose build --no-cache

k8s-deploy:                        ## Deploy a Kubernetes
	kubectl apply -f infrastructure/kubernetes/base/
	kubectl apply -f infrastructure/kubernetes/apps/
	kubectl apply -f infrastructure/kubernetes/services/
	kubectl apply -f infrastructure/kubernetes/ingress/

k8s-status:                        ## Ver estado del cluster
	kubectl get pods -n helisa-innovate
	kubectl get services -n helisa-innovate
	kubectl get ingress -n helisa-innovate

k8s-logs:                          ## Ver logs de pods
	kubectl logs -f -l app=bff-service -n helisa-innovate

backup-db:                         ## Backup de PostgreSQL
	./infrastructure/scripts/backup-db.sh

restore-db:                        ## Restore de PostgreSQL
	./infrastructure/scripts/restore-db.sh $(DB_BACKUP_FILE)
```

### Uso del Makefile

```bash
# Ver todos los comandos disponibles
make help

# Instalar dependencias
make install

# Desarrollo local
make dev

# Solo backend
make dev-backend

# Tests
make test
make test-coverage

# Linting y formato
make lint
make format

# Build de producción
make build

# Migraciones
make db-migrate
make db-rollback
make db-seed

# Docker
make docker-up
make docker-logs
make docker-down

# Kubernetes
make k8s-deploy
make k8s-status
make k8s-logs
```

---

## Próximos Pasos

1. Ver [API_CONTRACTS.md](./API_CONTRACTS.md) para especificaciones OpenAPI
2. Ver [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) para esquemas PostgreSQL
3. Ver [DEPLOYMENT.md](./DEPLOYMENT.md) para Docker y Kubernetes
4. Ver [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md) para setup local

---

**Versión**: 1.0
**Fecha**: 2024-11-08
