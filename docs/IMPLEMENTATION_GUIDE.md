# Guía de Implementación: Helisa Innovate Services

## Tabla de Contenidos
- [Configuración Docker](#configuración-docker)
- [Código MVP](#código-mvp)
- [CI/CD con GitHub Actions](#cicd-con-github-actions)
- [Kubernetes (Opcional)](#kubernetes-opcional)
- [Estrategia de Testing](#estrategia-de-testing)
- [Métricas y Monitoreo](#métricas-y-monitoreo)

---

## Configuración Docker

### docker-compose.yml - Desarrollo Local

```yaml
version: '3.9'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: helisa-postgres
    environment:
      POSTGRES_DB: helisa_innovate
      POSTGRES_USER: ${POSTGRES_USER:-helisa_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-dev_password_123}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infrastructure/docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U helisa_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: helisa-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # RabbitMQ Message Broker
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: helisa-rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER:-helisa}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS:-dev_password_123}
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  # OpenSearch
  opensearch:
    image: opensearchproject/opensearch:2.11.0
    container_name: helisa-opensearch
    environment:
      - discovery.type=single-node
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
      - DISABLE_SECURITY_PLUGIN=true  # Solo para desarrollo
    ports:
      - "9200:9200"
      - "9600:9600"
    volumes:
      - opensearch_data:/usr/share/opensearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # OpenSearch Dashboards (Kibana)
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:2.11.0
    container_name: helisa-opensearch-dashboards
    ports:
      - "5601:5601"
    environment:
      OPENSEARCH_HOSTS: '["http://opensearch:9200"]'
      DISABLE_SECURITY_DASHBOARDS_PLUGIN: "true"
    depends_on:
      - opensearch

  # Keycloak (Opcional)
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    container_name: helisa-keycloak
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: ${POSTGRES_USER:-helisa_user}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD:-dev_password_123}
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin123
    ports:
      - "8080:8080"
    command: start-dev
    depends_on:
      - postgres
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 30s
      timeout: 3s
      retries: 20

  # BFF Service
  bff-service:
    build:
      context: ./apps/bff-service
      dockerfile: Dockerfile
    container_name: helisa-bff
    environment:
      DATABASE_URL: postgresql+asyncpg://helisa_user:dev_password_123@postgres:5432/helisa_innovate
      REDIS_URL: redis://redis:6379/0
      HELISA_ADAPTER_URL: http://helisa-adapter:8001
      SYNC_SERVICE_URL: http://sync-service:8002
      AUDIT_SERVICE_URL: http://audit-service:8003
      JWT_SECRET_KEY: ${JWT_SECRET_KEY:-dev_secret_key_change_in_production}
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./apps/bff-service:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  # Helisa Adapter Service
  helisa-adapter:
    build:
      context: ./apps/helisa-adapter
      dockerfile: Dockerfile
    container_name: helisa-adapter
    environment:
      DATABASE_URL: postgresql+asyncpg://helisa_user:dev_password_123@postgres:5432/helisa_innovate
      REDIS_URL: redis://redis:6379/1
      RABBITMQ_URL: amqp://helisa:dev_password_123@rabbitmq:5672/
      HELISA_API_URL: ${HELISA_API_URL:-https://helisa.com/api/KansasWS}
      HELISA_CLIENT_ID: ${HELISA_CLIENT_ID}
      HELISA_SECRET_KEY: ${HELISA_SECRET_KEY}
    ports:
      - "8001:8001"
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    volumes:
      - ./apps/helisa-adapter:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload

  # Sync Service
  sync-service:
    build:
      context: ./apps/sync-service
      dockerfile: Dockerfile
    container_name: helisa-sync-service
    environment:
      DATABASE_URL: postgresql+asyncpg://helisa_user:dev_password_123@postgres:5432/helisa_innovate
      HELISA_ADAPTER_URL: http://helisa-adapter:8001
      SLACK_WEBHOOK_URL: ${SLACK_WEBHOOK_URL:-}
    ports:
      - "8002:8002"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./apps/sync-service:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8002 --reload

  # Audit Service
  audit-service:
    build:
      context: ./apps/audit-service
      dockerfile: Dockerfile
    container_name: helisa-audit-service
    environment:
      DATABASE_URL: postgresql+asyncpg://helisa_user:dev_password_123@postgres:5432/helisa_innovate
      OPENSEARCH_URL: http://opensearch:9200
      RABBITMQ_URL: amqp://helisa:dev_password_123@rabbitmq:5672/
    ports:
      - "8003:8003"
    depends_on:
      postgres:
        condition: service_healthy
      opensearch:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    volumes:
      - ./apps/audit-service:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8003 --reload

  # NGINX Reverse Proxy
  nginx:
    build:
      context: ./infrastructure/docker/nginx
      dockerfile: Dockerfile
    container_name: helisa-nginx
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - bff-service
    volumes:
      - ./apps/frontend/dist:/usr/share/nginx/html:ro
      - ./infrastructure/docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  opensearch_data:
```

### Dockerfiles

#### BFF Service Dockerfile

```dockerfile
# apps/bff-service/Dockerfile
FROM python:3.12-slim as base

WORKDIR /app

# Dependencias del sistema
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Instalar Poetry
RUN pip install poetry==1.7.1

# Configurar Poetry
ENV POETRY_NO_INTERACTION=1 \
    POETRY_VIRTUALENVS_IN_PROJECT=1 \
    POETRY_VIRTUALENVS_CREATE=1 \
    POETRY_CACHE_DIR=/tmp/poetry_cache

# Copiar archivos de dependencias
COPY pyproject.toml poetry.lock ./

# Instalar dependencias
RUN poetry install --no-root && rm -rf $POETRY_CACHE_DIR

# Stage de producción
FROM python:3.12-slim as production

WORKDIR /app

# Copiar virtual env desde base
COPY --from=base /app/.venv /app/.venv

# Copiar código
COPY ./app ./app

# Usuario no-root
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Variables de entorno
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Comando por defecto
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### Frontend Dockerfile (Multi-stage)

```dockerfile
# apps/frontend/Dockerfile
FROM node:20-alpine as build

WORKDIR /app

# Copiar package files
COPY package*.json ./

# Instalar dependencias
RUN npm ci --prefer-offline --no-audit

# Copiar código
COPY . .

# Build de producción
RUN npm run build --configuration=production

# Stage de producción con NGINX
FROM nginx:1.25-alpine

# Copiar build
COPY --from=build /app/dist/browser /usr/share/nginx/html

# Copiar configuración NGINX
COPY nginx.conf /etc/nginx/nginx.conf

# Usuario no-root
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## Código MVP

### 1. Angular - Servicio de Conexión API

```typescript
// apps/frontend/src/app/core/http/api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams, HttpHeaders } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry, map } from 'rxjs/operators';
import { environment } from '../../../environments/environment';

export interface ApiResponse<T> {
  data: T;
  message?: string;
  errors?: string[];
}

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private http = inject(HttpClient);
  private readonly baseUrl = environment.apiUrl;

  /**
   * GET request genérico
   */
  get<T>(endpoint: string, params?: Record<string, any>): Observable<T> {
    const httpParams = this.buildParams(params);

    return this.http.get<ApiResponse<T>>(`${this.baseUrl}${endpoint}`, {
      params: httpParams
    }).pipe(
      retry(1),
      map(response => response.data),
      catchError(this.handleError)
    );
  }

  /**
   * POST request genérico
   */
  post<T>(endpoint: string, body: any): Observable<T> {
    return this.http.post<ApiResponse<T>>(`${this.baseUrl}${endpoint}`, body).pipe(
      map(response => response.data),
      catchError(this.handleError)
    );
  }

  /**
   * PUT request genérico
   */
  put<T>(endpoint: string, body: any): Observable<T> {
    return this.http.put<ApiResponse<T>>(`${this.baseUrl}${endpoint}`, body).pipe(
      map(response => response.data),
      catchError(this.handleError)
    );
  }

  /**
   * DELETE request genérico
   */
  delete<T>(endpoint: string): Observable<T> {
    return this.http.delete<ApiResponse<T>>(`${this.baseUrl}${endpoint}`).pipe(
      map(response => response.data),
      catchError(this.handleError)
    );
  }

  /**
   * Construye HttpParams desde objeto
   */
  private buildParams(params?: Record<string, any>): HttpParams {
    let httpParams = new HttpParams();

    if (params) {
      Object.keys(params).forEach(key => {
        const value = params[key];
        if (value !== null && value !== undefined) {
          httpParams = httpParams.set(key, String(value));
        }
      });
    }

    return httpParams;
  }

  /**
   * Manejo centralizado de errores
   */
  private handleError(error: any): Observable<never> {
    let errorMessage = 'Ocurrió un error desconocido';

    if (error.error instanceof ErrorEvent) {
      // Error del cliente
      errorMessage = `Error: ${error.error.message}`;
    } else {
      // Error del servidor
      errorMessage = error.error?.error?.message ||
                    `Error ${error.status}: ${error.statusText}`;
    }

    console.error('API Error:', errorMessage, error);
    return throwError(() => new Error(errorMessage));
  }
}
```

### 2. Angular - Servicio de Contabilidad

```typescript
// apps/frontend/src/app/features/accounting/services/accounting.service.ts
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';
import { ApiService } from '../../../core/http/api.service';
import { AccountingEntry, AccountingEntryCreate, Account } from '../models/accounting.model';

@Injectable({
  providedIn: 'root'
})
export class AccountingService {
  private api = inject(ApiService);
  private readonly endpoint = '/accounting';

  /**
   * Listar comprobantes contables
   */
  getEntries(filters?: {
    page?: number;
    pageSize?: number;
    documentType?: string;
    dateFrom?: string;
    dateTo?: string;
    status?: string;
  }): Observable<{ items: AccountingEntry[]; total: number; page: number; pageSize: number }> {
    return this.api.get(`${this.endpoint}/entries`, filters);
  }

  /**
   * Obtener comprobante por ID
   */
  getEntry(id: string): Observable<AccountingEntry> {
    return this.api.get(`${this.endpoint}/entries/${id}`);
  }

  /**
   * Crear comprobante contable
   */
  createEntry(entry: AccountingEntryCreate): Observable<AccountingEntry> {
    // Validar balance antes de enviar
    if (!this.validateBalance(entry)) {
      throw new Error('El comprobante no está balanceado');
    }

    return this.api.post(`${this.endpoint}/entries`, entry);
  }

  /**
   * Obtener catálogo de cuentas
   */
  getAccounts(accountingType: 'local' | 'NIIF' = 'local'): Observable<Account[]> {
    return this.api.get(`${this.endpoint}/accounts`, { accountingType });
  }

  /**
   * Validar que débitos = créditos
   */
  private validateBalance(entry: AccountingEntryCreate): boolean {
    const totalDebit = entry.details
      .filter(d => d.nature === 'D')
      .reduce((sum, d) => sum + d.value, 0);

    const totalCredit = entry.details
      .filter(d => d.nature === 'C')
      .reduce((sum, d) => sum + d.value, 0);

    return Math.abs(totalDebit - totalCredit) < 0.01; // Tolerancia de 1 centavo
  }
}
```

### 3. FastAPI - BFF Endpoint

```python
# apps/bff-service/app/api/v1/endpoints/accounting.py
from fastapi import APIRouter, Depends, HTTPException, status
from typing import List, Optional
from uuid import UUID

from ....core.security import get_current_user, TokenData
from ....services.helisa_client import HelisaClient
from ....services.cache_service import CacheService
from ....schemas.accounting import (
    AccountingEntryResponse,
    AccountingEntryCreate,
    AccountingEntryList,
    AccountResponse
)

router = APIRouter(prefix="/accounting", tags=["Contabilidad"])

@router.get("/entries", response_model=AccountingEntryList)
async def list_accounting_entries(
    page: int = 1,
    page_size: int = 20,
    document_type: Optional[str] = None,
    date_from: Optional[str] = None,
    date_to: Optional[str] = None,
    status: Optional[str] = None,
    current_user: TokenData = Depends(get_current_user),
    helisa_client: HelisaClient = Depends()
):
    """
    Lista comprobantes contables con filtros
    """
    try:
        entries = await helisa_client.get_accounting_entries(
            page=page,
            page_size=page_size,
            document_type=document_type,
            date_from=date_from,
            date_to=date_to,
            status=status
        )
        return entries
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Error al obtener comprobantes: {str(e)}"
        )

@router.post("/entries", response_model=AccountingEntryResponse, status_code=status.HTTP_201_CREATED)
async def create_accounting_entry(
    entry: AccountingEntryCreate,
    current_user: TokenData = Depends(get_current_user),
    helisa_client: HelisaClient = Depends()
):
    """
    Crea un nuevo comprobante contable
    """
    # Validar balance
    total_debit = sum(d.value for d in entry.details if d.nature == 'D')
    total_credit = sum(d.value for d in entry.details if d.nature == 'C')

    if abs(total_debit - total_credit) > 0.01:
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            detail=f"Comprobante desbalanceado: Débito={total_debit}, Crédito={total_credit}"
        )

    try:
        # Enviar a Helisa Adapter
        created_entry = await helisa_client.create_accounting_entry(
            entry=entry,
            user_id=current_user.user_id
        )
        return created_entry
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Error al crear comprobante: {str(e)}"
        )

@router.get("/accounts", response_model=List[AccountResponse])
async def list_accounts(
    accounting_type: str = "local",
    current_user: TokenData = Depends(get_current_user),
    helisa_client: HelisaClient = Depends(),
    cache: CacheService = Depends()
):
    """
    Obtiene catálogo de cuentas (con caché)
    """
    cache_key = f"accounts:{accounting_type}"

    # Intentar obtener de caché
    cached = await cache.get(cache_key)
    if cached:
        return cached

    # Si no está en caché, obtener de Helisa
    accounts = await helisa_client.get_accounts(accounting_type)

    # Guardar en caché por 1 hora
    await cache.set(cache_key, accounts, ttl=3600)

    return accounts
```

### 4. FastAPI - Helisa Adapter (Firma HMAC)

```python
# apps/helisa-adapter/app/services/signature.py
import hmac
import hashlib
import json
from typing import Dict, Any

class HelisaSignatureService:
    """
    Servicio para generar firmas HMAC-SHA256 para Helisa API
    """

    def __init__(self, secret_key: str, client_id: str):
        self.secret_key = secret_key
        self.client_id = client_id

    def generate_signature(self, data: Dict[str, Any]) -> Dict[str, str]:
        """
        Genera firma HMAC-SHA256 para petición a Helisa

        Args:
            data: Datos a enviar

        Returns:
            Dict con id, json y sign
        """
        # Convertir a JSON string (minificado)
        json_string = json.dumps(data, separators=(',', ':'), ensure_ascii=False)

        # Generar HMAC-SHA256
        signature = hmac.new(
            self.secret_key.encode('utf-8'),
            json_string.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()

        return {
            "id": self.client_id,
            "json": json_string,
            "sign": signature
        }

    def verify_signature(self, json_string: str, signature: str) -> bool:
        """
        Verifica una firma (útil para testing)
        """
        expected_signature = hmac.new(
            self.secret_key.encode('utf-8'),
            json_string.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()

        # Comparación en tiempo constante
        return hmac.compare_digest(signature, expected_signature)
```

```python
# apps/helisa-adapter/app/services/helisa_api.py
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential
from circuitbreaker import circuit
from typing import Dict, Any, Optional
from ..core.config import settings
from .signature import HelisaSignatureService

class HelisaAPIClient:
    """
    Cliente HTTP para API de Helisa con retry, circuit breaker y firma
    """

    def __init__(self):
        self.base_url = settings.HELISA_API_URL
        self.signature_service = HelisaSignatureService(
            secret_key=settings.HELISA_SECRET_KEY,
            client_id=settings.HELISA_CLIENT_ID
        )
        self.client = httpx.AsyncClient(
            timeout=30.0,
            limits=httpx.Limits(max_keepalive_connections=5, max_connections=10)
        )

    @circuit(failure_threshold=5, recovery_timeout=60)
    @retry(
        stop=stop_after_attempt(4),
        wait=wait_exponential(multiplier=1, min=2, max=16)
    )
    async def post(self, endpoint: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """
        POST request a Helisa con firma HMAC-SHA256

        Args:
            endpoint: Endpoint relativo (ej: /accountingEntry)
            data: Datos a enviar

        Returns:
            Response data

        Raises:
            HelisaAPIError: Si la petición falla
        """
        # Generar firma
        signed_payload = self.signature_service.generate_signature(data)

        # Enviar petición
        url = f"{self.base_url}{endpoint}"
        response = await self.client.post(
            url,
            data=signed_payload,
            headers={"Content-Type": "application/x-www-form-urlencoded"}
        )

        # Validar respuesta
        if response.status_code == 401:
            raise HelisaAPIError("Firma de seguridad inválida", code=1001)
        elif response.status_code >= 400:
            error_data = response.json()
            raise HelisaAPIError(
                error_data.get("error", {}).get("message", "Error desconocido"),
                code=error_data.get("error", {}).get("code")
            )

        return response.json()

    async def get(self, endpoint: str, params: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        """
        GET request a Helisa con firma

        Args:
            endpoint: Endpoint relativo
            params: Query parameters

        Returns:
            Response data
        """
        # Para GET, los params van en la firma también
        signed_payload = self.signature_service.generate_signature(params or {})

        url = f"{self.base_url}{endpoint}"
        response = await self.client.get(
            url,
            params=signed_payload
        )

        if response.status_code >= 400:
            raise HelisaAPIError(f"Error en GET: {response.text}")

        return response.json()

class HelisaAPIError(Exception):
    """Excepción personalizada para errores de Helisa API"""

    def __init__(self, message: str, code: Optional[int] = None):
        self.message = message
        self.code = code
        super().__init__(self.message)
```

---

## CI/CD con GitHub Actions

### Workflow - Frontend CI

```yaml
# .github/workflows/ci-frontend.yml
name: Frontend CI

on:
  push:
    branches: [main, develop]
    paths:
      - 'apps/frontend/**'
      - 'libs/typescript-shared/**'
  pull_request:
    branches: [main, develop]
    paths:
      - 'apps/frontend/**'

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [20.x]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: apps/frontend/package-lock.json

      - name: Install dependencies
        working-directory: apps/frontend
        run: npm ci

      - name: Lint
        working-directory: apps/frontend
        run: npm run lint

      - name: Unit Tests
        working-directory: apps/frontend
        run: npm run test:ci

      - name: Build
        working-directory: apps/frontend
        run: npm run build --configuration=production

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: frontend-dist
          path: apps/frontend/dist

      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            http://localhost:4200
          uploadArtifacts: true
```

### Workflow - Backend CI

```yaml
# .github/workflows/ci-backend.yml
name: Backend CI

on:
  push:
    branches: [main, develop]
    paths:
      - 'apps/bff-service/**'
      - 'apps/helisa-adapter/**'
      - 'apps/sync-service/**'
      - 'apps/audit-service/**'
      - 'libs/python-shared/**'
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    strategy:
      matrix:
        python-version: ['3.12']
        service:
          - bff-service
          - helisa-adapter
          - sync-service
          - audit-service

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install Poetry
        uses: snok/install-poetry@v1
        with:
          version: 1.7.1
          virtualenvs-create: true
          virtualenvs-in-project: true

      - name: Load cached venv
        id: cached-poetry-dependencies
        uses: actions/cache@v3
        with:
          path: apps/${{ matrix.service }}/.venv
          key: venv-${{ runner.os }}-${{ matrix.python-version }}-${{ hashFiles('**/poetry.lock') }}

      - name: Install dependencies
        working-directory: apps/${{ matrix.service }}
        if: steps.cached-poetry-dependencies.outputs.cache-hit != 'true'
        run: poetry install --no-interaction --no-root

      - name: Lint with ruff
        working-directory: apps/${{ matrix.service }}
        run: |
          poetry run ruff check app/
          poetry run black --check app/

      - name: Type check with mypy
        working-directory: apps/${{ matrix.service }}
        run: poetry run mypy app/

      - name: Run tests with pytest
        working-directory: apps/${{ matrix.service }}
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
        run: |
          poetry run pytest --cov=app --cov-report=xml --cov-report=html

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: apps/${{ matrix.service }}/coverage.xml
          flags: ${{ matrix.service }}
```

### Workflow - CD Production

```yaml
# .github/workflows/cd-production.yml
name: Deploy to Production

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker images
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.ref_name }}
        run: |
          # BFF Service
          docker build -t $ECR_REGISTRY/helisa-bff:$IMAGE_TAG apps/bff-service
          docker push $ECR_REGISTRY/helisa-bff:$IMAGE_TAG

          # Helisa Adapter
          docker build -t $ECR_REGISTRY/helisa-adapter:$IMAGE_TAG apps/helisa-adapter
          docker push $ECR_REGISTRY/helisa-adapter:$IMAGE_TAG

          # Sync Service
          docker build -t $ECR_REGISTRY/helisa-sync:$IMAGE_TAG apps/sync-service
          docker push $ECR_REGISTRY/helisa-sync:$IMAGE_TAG

          # Audit Service
          docker build -t $ECR_REGISTRY/helisa-audit:$IMAGE_TAG apps/audit-service
          docker push $ECR_REGISTRY/helisa-audit:$IMAGE_TAG

      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            infrastructure/kubernetes/apps/
          images: |
            ${{ steps.login-ecr.outputs.registry }}/helisa-bff:${{ github.ref_name }}
            ${{ steps.login-ecr.outputs.registry }}/helisa-adapter:${{ github.ref_name }}
          namespace: helisa-innovate-prod
```

---

## Kubernetes (Opcional)

### BFF Deployment

```yaml
# infrastructure/kubernetes/apps/bff-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bff-service
  namespace: helisa-innovate
  labels:
    app: bff-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bff-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: bff-service
        version: v1
    spec:
      containers:
        - name: bff
          image: helisa-bff:latest
          ports:
            - containerPort: 8000
              name: http
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: helisa-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: helisa-config
                  key: redis-url
            - name: JWT_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: helisa-secrets
                  key: jwt-secret
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: bff-service
  namespace: helisa-innovate
spec:
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
      protocol: TCP
      name: http
  selector:
    app: bff-service

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bff-hpa
  namespace: helisa-innovate
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bff-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## Estrategia de Testing

### 1. Testing Pyramid

```
           /\
          /  \
         / E2E\         10% - Tests End-to-End
        /______\
       /        \
      /Integration\    30% - Tests de Integración
     /____________\
    /              \
   /  Unit Tests    \  60% - Tests Unitarios
  /__________________\
```

### 2. Tests Unitarios (Python)

```python
# apps/bff-service/tests/unit/test_signature_service.py
import pytest
from app.services.signature import HelisaSignatureService

def test_generate_signature():
    """Test generación de firma HMAC-SHA256"""
    service = HelisaSignatureService(
        secret_key="test_secret_key",
        client_id="TEST_CLIENT"
    )

    data = {
        "documentType": "CE",
        "date": {"day": 15, "month": 11, "year": 2024}
    }

    result = service.generate_signature(data)

    assert "id" in result
    assert "json" in result
    assert "sign" in result
    assert result["id"] == "TEST_CLIENT"
    assert len(result["sign"]) == 64  # SHA-256 hex = 64 chars

def test_verify_signature():
    """Test verificación de firma"""
    service = HelisaSignatureService(
        secret_key="test_secret_key",
        client_id="TEST_CLIENT"
    )

    json_string = '{"test":"data"}'
    signature = service.generate_signature({"test": "data"})["sign"]

    assert service.verify_signature(json_string, signature) is True
    assert service.verify_signature(json_string, "invalid_signature") is False
```

### 3. Tests de Integración (Python)

```python
# apps/bff-service/tests/integration/test_accounting_endpoints.py
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_create_accounting_entry_success(async_client: AsyncClient, auth_token: str):
    """Test crear comprobante contable"""
    payload = {
        "document_type": "CE",
        "date": {"day": 15, "month": 11, "year": 2024},
        "description": "Test comprobante",
        "details": [
            {
                "account_code": "110505",
                "nature": "C",
                "value": 100000,
                "description": "Crédito caja"
            },
            {
                "account_code": "510506",
                "nature": "D",
                "value": 100000,
                "description": "Débito gasto"
            }
        ]
    }

    response = await async_client.post(
        "/api/v1/accounting/entries",
        json=payload,
        headers={"Authorization": f"Bearer {auth_token}"}
    )

    assert response.status_code == 201
    data = response.json()
    assert "id" in data
    assert data["document_type"] == "CE"
    assert data["status"] == "PENDING"

@pytest.mark.asyncio
async def test_create_accounting_entry_unbalanced(async_client: AsyncClient, auth_token: str):
    """Test comprobante desbalanceado debe fallar"""
    payload = {
        "document_type": "CE",
        "date": {"day": 15, "month": 11, "year": 2024},
        "description": "Test comprobante",
        "details": [
            {
                "account_code": "110505",
                "nature": "C",
                "value": 100000,
                "description": "Crédito"
            },
            {
                "account_code": "510506",
                "nature": "D",
                "value": 50000,  # Desbalanceado!
                "description": "Débito"
            }
        ]
    }

    response = await async_client.post(
        "/api/v1/accounting/entries",
        json=payload,
        headers={"Authorization": f"Bearer {auth_token}"}
    )

    assert response.status_code == 422
    assert "desbalanceado" in response.json()["detail"].lower()
```

### 4. Tests E2E (Playwright)

```typescript
// apps/frontend/e2e/accounting.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Módulo de Contabilidad', () => {
  test.beforeEach(async ({ page }) => {
    // Login
    await page.goto('http://localhost:4200/login');
    await page.fill('input[name="email"]', 'test@innovatenutrition.com');
    await page.fill('input[name="password"]', 'TestPassword123');
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL('http://localhost:4200/dashboard');
  });

  test('Crear comprobante contable balanceado', async ({ page }) => {
    // Navegar a contabilidad
    await page.click('text=Contabilidad');
    await page.click('text=Nuevo Comprobante');

    // Llenar formulario
    await page.selectOption('select[name="documentType"]', 'CE');
    await page.fill('input[name="description"]', 'Comprobante de prueba E2E');

    // Agregar detalle débito
    await page.click('button:has-text("Agregar Detalle")');
    await page.fill('input[name="accountCode_0"]', '110505');
    await page.selectOption('select[name="nature_0"]', 'D');
    await page.fill('input[name="value_0"]', '100000');

    // Agregar detalle crédito
    await page.click('button:has-text("Agregar Detalle")');
    await page.fill('input[name="accountCode_1"]', '510506');
    await page.selectOption('select[name="nature_1"]', 'C');
    await page.fill('input[name="value_1"]', '100000');

    // Verificar balance
    await expect(page.locator('.balance-indicator')).toHaveText('Balanceado ✓');

    // Enviar
    await page.click('button[type="submit"]');

    // Verificar éxito
    await expect(page.locator('.success-message')).toBeVisible();
    await expect(page).toHaveURL(/\/accounting\/entries\/[a-f0-9-]+/);
  });

  test('Validar comprobante desbalanceado', async ({ page }) => {
    await page.click('text=Contabilidad');
    await page.click('text=Nuevo Comprobante');

    // Crear comprobante desbalanceado
    await page.fill('input[name="accountCode_0"]', '110505');
    await page.selectOption('select[name="nature_0"]', 'D');
    await page.fill('input[name="value_0"]', '100000');

    await page.fill('input[name="accountCode_1"]', '510506');
    await page.selectOption('select[name="nature_1"]', 'C');
    await page.fill('input[name="value_1"]', '50000');  // Desbalanceado

    // Verificar alerta
    await expect(page.locator('.balance-indicator')).toHaveText(/Desbalanceado/);
    await expect(page.locator('button[type="submit"]')).toBeDisabled();
  });
});
```

---

## Métricas y Monitoreo

### Prometheus Metrics (FastAPI)

```python
# apps/bff-service/app/middleware/metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from fastapi import Request
import time

# Métricas
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint']
)

active_requests = Gauge(
    'http_requests_active',
    'Active HTTP requests',
    ['method', 'endpoint']
)

helisa_api_requests_total = Counter(
    'helisa_api_requests_total',
    'Total requests to Helisa API',
    ['endpoint', 'status']
)

async def metrics_middleware(request: Request, call_next):
    """Middleware para recolectar métricas"""
    method = request.method
    endpoint = request.url.path

    active_requests.labels(method=method, endpoint=endpoint).inc()
    start_time = time.time()

    try:
        response = await call_next(request)
        status = response.status_code
    except Exception as e:
        status = 500
        raise
    finally:
        duration = time.time() - start_time
        active_requests.labels(method=method, endpoint=endpoint).dec()
        http_requests_total.labels(method=method, endpoint=endpoint, status=status).inc()
        http_request_duration_seconds.labels(method=method, endpoint=endpoint).observe(duration)

    return response

# Endpoint de métricas
@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

### Checklist de Seguridad y Hardening

```markdown
## Seguridad - Pre-Production Checklist

### Autenticación y Autorización
- [ ] JWT con expiración corta (15-60 min)
- [ ] Refresh tokens rotados
- [ ] Rate limiting en /auth/* endpoints
- [ ] Validación de dominio @innovatenutrition.com
- [ ] RBAC implementado (roles y permisos)
- [ ] 2FA opcional disponible

### Datos y Comunicación
- [ ] HTTPS obligatorio en producción
- [ ] TLS 1.2+ únicamente
- [ ] Certificados SSL válidos
- [ ] Headers de seguridad (HSTS, CSP, etc.)
- [ ] CORS configurado correctamente
- [ ] Claves secretas en vault/secrets manager

### Base de Datos
- [ ] Conexiones con SSL
- [ ] Passwords seguros (> 16 chars)
- [ ] Principio de mínimo privilegio (usuarios DB)
- [ ] Backups automáticos diarios
- [ ] Encriptación at-rest habilitada
- [ ] Logs de auditoría append-only

### Código y Dependencias
- [ ] Dependencias actualizadas (npm audit, safety)
- [ ] No secrets en código ni git
- [ ] Input validation en todos los endpoints
- [ ] SQL injection protegido (ORM)
- [ ] XSS protegido (sanitización)
- [ ] CSRF protegido (tokens)

### Infraestructura
- [ ] Contenedores no-root
- [ ] Security scanning de imágenes Docker
- [ ] Firewalls configurados
- [ ] VPC con subnets privadas
- [ ] Logs centralizados
- [ ] Alertas configuradas

### Monitoreo
- [ ] Prometheus + Grafana funcionando
- [ ] Alertas críticas definidas
- [ ] Logs en OpenSearch/ELK
- [ ] Uptime monitoring
- [ ] Error tracking (Sentry opcional)
```

---

**Versión**: 1.0
**Fecha**: 2024-11-08
**Próximo**: Ver DEVELOPER_GUIDE.md para setup local paso a paso
