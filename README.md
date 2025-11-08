# Helisa Innovate Services

Plataforma de integración empresarial entre **Innovate Nutrition** y el ERP **Helisa** para sincronización bidireccional de datos contables, inventario y ventas.

[![CI Status](https://github.com/innovate-nutrition/helisa-innovate-services/workflows/CI/badge.svg)](https://github.com/innovate-nutrition/helisa-innovate-services/actions)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/)
[![Node](https://img.shields.io/badge/node-20+-green.svg)](https://nodejs.org/)
[![Angular](https://img.shields.io/badge/angular-18+-red.svg)](https://angular.dev/)

---

## 🎯 Características Principales

- ✅ **Integración segura** con API de Helisa (firma HMAC-SHA256)
- ✅ **Sincronización bidireccional** de catálogos y transacciones
- ✅ **Dashboard ejecutivo** con métricas en tiempo real
- ✅ **Auditoría completa** de todas las operaciones
- ✅ **Arquitectura escalable** con microservicios y contenedores
- ✅ **Testing completo** (unitario, integración, E2E)

---

## 📚 Documentación

### Para Ejecutivos y Stakeholders
- 📊 **[Resumen Ejecutivo](docs/EXECUTIVE_SUMMARY.md)** - Visión general, ROI, roadmap, costos

### Para Arquitectos y Tech Leads
- 🏗️ **[Diseño de Arquitectura](docs/ARCHITECTURE_DESIGN.md)** - Diagramas, decisiones de diseño, stack tecnológico
- 📁 **[Estructura del Repositorio](docs/REPOSITORY_STRUCTURE.md)** - Organización del monorepo, convenciones
- 🔧 **[Especificación Técnica](docs/TECHNICAL_SPECIFICATION.md)** - APIs (OpenAPI), esquemas de BD, seguridad

### Para Desarrolladores
- 💻 **[Guía del Desarrollador](docs/DEVELOPER_GUIDE.md)** - Setup local paso a paso, workflows, troubleshooting
- 🚀 **[Guía de Implementación](docs/IMPLEMENTATION_GUIDE.md)** - Docker, CI/CD, código MVP, testing

### Documentación de Helisa
- 📖 **[API de Helisa](docs/HELISA_API_ARCHITECTURE.md)** - Endpoints, modelos de datos, ejemplos
- 🔒 **[Firma de Seguridad](docs/HELISA_SECURITY_SIGNATURE.md)** - HMAC-SHA256, ejemplos en 7 lenguajes

---

## 🚀 Quick Start

### Requisitos Previos
- **Docker** 24.0+
- **Node.js** 20.x LTS
- **Python** 3.12+
- **Git** 2.40+

### Instalación Rápida

```bash
# 1. Clonar repositorio
git clone https://github.com/innovate-nutrition/helisa-innovate-services.git
cd helisa-innovate-services

# 2. Configurar variables de entorno
cp .env.example .env
# Editar .env con tus credenciales de Helisa

# 3. Instalar dependencias
make install

# 4. Levantar infraestructura
docker-compose up -d postgres redis rabbitmq opensearch

# 5. Aplicar migraciones
make db-migrate

# 6. Seed de datos (opcional)
make db-seed

# 7. Levantar servicios
make dev
```

### Acceder a la Aplicación

- **Frontend**: http://localhost:4200
- **BFF API**: http://localhost:8000/docs (Swagger UI)
- **OpenSearch Dashboards**: http://localhost:5601
- **RabbitMQ Management**: http://localhost:15672 (helisa / dev_password_123)

**Usuario de prueba**:
- Email: `admin@innovatenutrition.com`
- Password: `Admin123!`

---

## 🏗️ Arquitectura

### Stack Tecnológico

**Frontend**:
- Angular 18 (Standalone components, Signals)
- Angular Material
- RxJS
- TypeScript 5.3+

**Backend**:
- FastAPI (Python 3.12)
- SQLAlchemy 2.0 (async)
- Pydantic v2
- asyncio / httpx

**Infraestructura**:
- PostgreSQL 16 (JSONB, full-text)
- Redis 7 (cache, queues)
- RabbitMQ 3.12 (message broker)
- OpenSearch 2.11 (logs, analytics)
- Docker + Kubernetes

### Diagrama de Alto Nivel

```
Angular App → NGINX → BFF Service → [Helisa Adapter, Sync Service, Audit Service]
                                            ↓
                                      PostgreSQL + Redis + RabbitMQ
                                            ↓
                                    Helisa API (External)
```

Ver diagrama completo en [ARCHITECTURE_DESIGN.md](docs/ARCHITECTURE_DESIGN.md).

---

## 📦 Estructura del Proyecto

```
helisa-innovate-services/
├── apps/
│   ├── frontend/              # Angular 18 app
│   ├── bff-service/           # Backend for Frontend (FastAPI)
│   ├── helisa-adapter/        # Helisa integration service
│   ├── sync-service/          # Synchronization jobs
│   └── audit-service/         # Audit logging
├── libs/
│   ├── python-shared/         # Shared Python code
│   └── typescript-shared/     # Shared TypeScript code
├── infrastructure/
│   ├── docker/                # Dockerfiles
│   ├── kubernetes/            # K8s manifests
│   └── scripts/               # Deploy scripts
├── docs/                      # Documentación completa
├── docker-compose.yml         # Desarrollo local
├── Makefile                   # Comandos comunes
└── README.md                  # Este archivo
```

Ver estructura completa en [REPOSITORY_STRUCTURE.md](docs/REPOSITORY_STRUCTURE.md).

---

## 🧪 Testing

```bash
# Todos los tests
make test

# Solo backend
cd apps/bff-service
poetry run pytest

# Solo frontend
cd apps/frontend
npm run test:ci

# E2E tests
cd apps/frontend
npm run e2e

# Coverage
make test-coverage
```

**Cobertura objetivo**: 80%+ en código crítico.

Ver estrategia completa en [IMPLEMENTATION_GUIDE.md](docs/IMPLEMENTATION_GUIDE.md#estrategia-de-testing).

---

## 🔒 Seguridad

- **Autenticación**: JWT + Keycloak (opcional)
- **Autorización**: RBAC (Role-Based Access Control)
- **Validación de dominios**: Por defecto `@innovatenutrition.com`
- **Comunicación**: HTTPS/TLS 1.2+
- **Helisa API**: Firma HMAC-SHA256 en todas las peticiones
- **Auditoría**: Logs inmutables de todas las operaciones

Ver detalles en [TECHNICAL_SPECIFICATION.md](docs/TECHNICAL_SPECIFICATION.md#arquitectura-de-seguridad).

---

## 🤝 Contribuir

1. Fork el repositorio
2. Crear branch: `git checkout -b feature/ABC-123-descripcion`
3. Commit: `git commit -m "feat(scope): descripción"`
4. Push: `git push origin feature/ABC-123-descripcion`
5. Crear Pull Request

**Convenciones**:
- Commits: [Conventional Commits](https://www.conventionalcommits.org/)
- Branches: `feature/TICKET-descripcion`, `fix/TICKET-descripcion`
- Code style: Black (Python), Prettier (TypeScript)

Ver [CONTRIBUTING.md](CONTRIBUTING.md) para más detalles.

---

## 📝 Comandos Útiles

```bash
# Ver todos los comandos
make help

# Desarrollo
make install          # Instalar dependencias
make dev              # Levantar entorno completo
make dev-backend      # Solo backend services
make dev-frontend     # Solo frontend

# Testing
make test             # Todos los tests
make test-coverage    # Con coverage
make lint             # Linter
make format           # Auto-format

# Base de Datos
make db-migrate       # Aplicar migraciones
make db-rollback      # Rollback última migración
make db-seed          # Cargar datos de prueba

# Docker
make docker-up        # Levantar stack completo
make docker-down      # Bajar stack
make docker-logs      # Ver logs
make docker-rebuild   # Rebuild imágenes

# Kubernetes
make k8s-deploy       # Deploy a K8s
make k8s-status       # Ver estado
make k8s-logs         # Ver logs
```

---

## 📊 Métricas y Monitoreo

- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000
- **OpenSearch Dashboards**: http://localhost:5601
- **RabbitMQ Management**: http://localhost:15672

**Dashboards incluidos**:
- API Performance
- Database Queries
- Helisa API Integration
- Business Metrics

---

## 🗺️ Roadmap

### Fase 1: MVP (Semanas 1-8) ✅
- [x] Setup infraestructura
- [x] Backend core (BFF, Helisa Adapter)
- [x] Frontend core (Login, Dashboard, Contabilidad)
- [x] Tests unitarios e integración

### Fase 2: Features Avanzadas (Semanas 9-14) 🚧
- [ ] Sync Service con cron jobs
- [ ] Módulo de inventario completo
- [ ] Módulo de ventas
- [ ] Observabilidad (Prometheus, Grafana)

### Fase 3: Producción (Semanas 15-18) ⏳
- [ ] Security audit
- [ ] Performance tuning
- [ ] Kubernetes setup
- [ ] Go-live

Ver roadmap completo en [EXECUTIVE_SUMMARY.md](docs/EXECUTIVE_SUMMARY.md#roadmap-de-implementación).

---

## 👥 Equipo

- **Tech Lead**: @tech-lead
- **Backend Developers**: @backend-team
- **Frontend Developer**: @frontend-dev
- **DevOps Engineer**: @devops
- **QA Engineer**: @qa

---

## 📄 Licencia

MIT License - Ver [LICENSE](LICENSE) para más detalles.

---

## 📞 Contacto

- **Slack**: #helisa-dev
- **Email**: dev-helisa@innovatenutrition.com
- **Jira**: [HELISA Board](https://innovatenutrition.atlassian.net/helisa)

---

## ⭐ Agradecimientos

- Equipo de Helisa por su API y soporte
- Innovate Nutrition por la confianza
- Contributors del proyecto

---

**Built with ❤️ by Innovate Nutrition Dev Team**

---

## 📚 Documentación Rápida por Rol

| Rol | Empieza aquí |
|-----|--------------|
| **CEO / CFO** | [Resumen Ejecutivo](docs/EXECUTIVE_SUMMARY.md) |
| **CTO / Arquitecto** | [Diseño de Arquitectura](docs/ARCHITECTURE_DESIGN.md) |
| **Tech Lead** | [Especificación Técnica](docs/TECHNICAL_SPECIFICATION.md) |
| **Backend Developer** | [Guía del Desarrollador](docs/DEVELOPER_GUIDE.md) |
| **Frontend Developer** | [Guía del Desarrollador](docs/DEVELOPER_GUIDE.md) + [Angular Components](apps/frontend/README.md) |
| **DevOps Engineer** | [Guía de Implementación](docs/IMPLEMENTATION_GUIDE.md) |
| **QA Engineer** | [Estrategia de Testing](docs/IMPLEMENTATION_GUIDE.md#estrategia-de-testing) |

---

**Última actualización**: 2024-11-08 | **Versión**: 1.0.0
