# Sistema Integral de Gestión Ministerial — Contexto para Claude Code

## Qué es este proyecto

Sistema de gestión pública para el Ministerio de Cooperativas y Mutuales de la Provincia de Córdoba. Centraliza múltiples subsistemas independientes (uno por secretaría/dirección) bajo un portal unificado que permite al ministro y autoridades acceder a información consolidada de todas las áreas.

## Principio de desarrollo

**Spec Driven Development**: ningún componente, endpoint, modelo o pantalla se implementa sin un spec aprobado previo. Toda decisión de diseño debe poder trazarse a un spec en `docs/files/`.

**Conflictos con ADRs o specs**: si una instrucción del usuario contradice una decisión registrada en `docs/files/arquitectura.md`, señalarlo explícitamente antes de implementar y esperar confirmación.

## Stack tecnológico

| Capa | Tecnología |
|------|-----------|
| Frontend | React 18 + Vite + TypeScript + TanStack Query v5 + Tailwind CSS v4 |
| Hosting | Firebase Hosting |
| Auth | Firebase Authentication (Google Sign-In) |
| API Gateway | GCP API Gateway (OpenAPI 2.0 / Swagger spec) |
| Backend | FastAPI (Python 3.12) — un servicio por área |
| Runtime | Google Cloud Run (contenedores Docker) |
| Base de datos principal | Cloud SQL — PostgreSQL 15 (asyncpg + SQLAlchemy 2.0 async) |
| Analítica ejecutiva | BigQuery (solo svc-privada existente) |
| Mensajería | Cloud Pub/Sub |
| Secretos | GCP Secret Manager |
| CI/CD | Cloud Build (trigger en push a main del repo backend) |
| Registro de imágenes | Artifact Registry (`southamerica-east1-docker.pkg.dev`) |

## Estructura de secretarías y servicios

Servicios backend (uno por secretaría):
- `svc-vivienda` ✅ **EN PRODUCCIÓN** — Secretaría de Vivienda + módulo portal (gestión de usuarios)
- `svc-infraestructura` — Secretaría de Gestión y Vinculación de Infraestructura (pendiente)
- `svc-territorial` — Secretaría de Planificación y Articulación Territorial (pendiente)
- `svc-desarrollo` — Secretaría de Desarrollo (pendiente)
- `svc-gasifera` — Secretaría de Infraestructura Gasífera (pendiente)
- `svc-privada` ✅ **EN PRODUCCIÓN** — Sistema existente en proyecto GCP `essential-haiku-482815-u4` con BigQuery

## Convenciones de código

### Python / FastAPI
- Estructura por módulo funcional: `router.py`, `models.py`, `schemas.py`, `service.py`, `repository.py`
- Type hints obligatorios en todas las funciones
- Pydantic v2 para schemas de entrada/salida
- SQLAlchemy 2.0 async para acceso a base de datos
- Prefijo de ruta: `/api/v1/{secretaria}/{recurso}`
- Repository pattern obligatorio — nunca queries SQL directas en routers o services

### React / Frontend
- Componentes funcionales con hooks únicamente
- Un directorio por subsistema en `src/modules/{secretaria}/`
- Módulo compartido en `src/shared/`
- TanStack Query v5 para data fetching (`keepPreviousData` → `placeholderData: keepPreviousData`)
- Paleta de colores gobierno: `gov-navy` (#172c3f), `gov-cyan` (#01aae3), `gov-blue` (#398ebd)
- Tailwind v4: el `@import` de Google Fonts DEBE ir ANTES de `@import "tailwindcss"`

### Base de datos
- Una base de datos PostgreSQL por servicio (database-per-service — ADR-001)
- Naming de tablas: `snake_case`, prefijadas por dominio (ej: `viv_programas`, `portal_usuarios`)
- Migraciones con Alembic — nunca modificar tablas manualmente
- Soft delete universal: todas las entidades tienen `deleted_at TIMESTAMP NULL`
- Correr migraciones SIEMPRE desde el directorio del servicio (`cd svc-vivienda/ && python -m alembic upgrade head`)

### API Gateway (openapi.yaml)
- Todo path nuevo necesita su operación `options:` con `security: []` (CORS preflight)
- Las configs de gateway son **inmutables** — cada cambio requiere nueva versión con nombre único
- Formato de nombre: `ministerio-config-v{YYYYMMDD}` o `ministerio-config-v{YYYYMMDD}{letra}`

## Sistema de roles y permisos

Los permisos se gestionan en Cloud SQL (tabla `portal_usuarios` en `svc-vivienda`), **no** en JWT custom claims (ADR-003 revisado).

| Rol | Puede hacer |
|-----|-------------|
| `Admin` | Todo — ver, crear, editar, eliminar, gestionar usuarios |
| `Supervisor` | Ver, crear, editar y eliminar dentro de sus secretarías |
| `Operador` | Ver, crear y editar (no eliminar) |
| `Consulta` | Solo lectura |

Cada usuario tiene una o más **secretarías asignadas**. Solo ve los módulos de sus secretarías en el dashboard. El `role="invitado"` es el fallback si el usuario Firebase no tiene entrada en `portal_usuarios`.

## Módulo portal (gestión de usuarios)

Vive en `svc-vivienda/app/portal/`:
- `GET /api/v1/portal/me` — perfil del usuario autenticado (403 si no está en el sistema)
- `GET /api/v1/portal/admin/usuarios` — lista usuarios (solo Admin)
- `POST /api/v1/portal/admin/usuarios` — crear usuario (solo Admin)
- `GET /api/v1/portal/admin/usuarios/{email}` — detalle usuario (solo Admin)
- `PUT /api/v1/portal/admin/usuarios/{email}` — actualizar usuario (solo Admin)

Tablas: `portal_usuarios` (PK: email) + `portal_usuario_secretarias` (email, secretaria).
Admins iniciales: `bonafepedro@gmail.com`, `infraestructura.coop@gmail.com` (seed en migración 0004).

## Repositorios GitHub

Organización: [coordinacion-infraestructura-coop](https://github.com/coordinacion-infraestructura-coop)

| Parte | Repositorio | Remote local |
|-------|-------------|--------------|
| Infraestructura / Gateway | panel.infra | `origin_infra` en `infra/` |
| Frontend | panel.front | `origin` en `frontend/` |
| Backend | panel.backend | `origin_back` en `services/` |

Cada carpeta tiene su propio `.git` independiente. `docs/` no tiene repositorio propio — es referencia local.

## URLs de producción

| Servicio | URL |
|----------|-----|
| Frontend | https://gestorcooperativo.web.app |
| API Gateway | https://ministerio-gateway-3j5k00ma.uc.gateway.dev |
| svc-vivienda (Cloud Run) | https://svc-vivienda-iwni7vc2qq-rj.a.run.app |
| svc-privada (Cloud Run) | https://infraestructura-gestioninterna-354063050046.southamerica-east1.run.app |

## Comandos frecuentes

```bash
# Correr migraciones en producción (Cloud Shell)
~/cloud-sql-proxy gestorcooperativo:southamerica-east1:ministerio-postgres --port 5432 > /tmp/proxy.log 2>&1 &
sleep 4 && cat /tmp/proxy.log   # verificar "Listening on 127.0.0.1:5432"
cd ~/gestorcooperativo/backend/svc-vivienda
DATABASE_URL="postgresql+asyncpg://user_vivienda:PASS@127.0.0.1:5432/db_vivienda" python -m alembic upgrade head
# Nota: el PASS tiene + y = que se URL-encodean como %2B y %3D

# Deploy backend (Cloud Shell)
cd ~/gestorcooperativo/backend/svc-vivienda
git pull origin main
gcloud run deploy svc-vivienda --source . --region=southamerica-east1 --project=gestorcooperativo

# Deploy gateway (Cloud Shell)
cd ~/panel.infra/gateway && git pull origin main
gcloud api-gateway api-configs create ministerio-config-v{FECHA} \
  --api=ministerio-api --openapi-spec=openapi.yaml \
  --project=gestorcooperativo \
  --backend-auth-service-account=api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com
gcloud api-gateway gateways update ministerio-gateway \
  --api=ministerio-api --api-config=ministerio-config-v{FECHA} \
  --location=us-central1 --project=gestorcooperativo

# Deploy frontend (local)
cd frontend && npm run build && firebase deploy --only hosting
```

## Patrones obligatorios

1. **Repository pattern** en todos los servicios backend
2. **Dependency injection** con FastAPI `Depends()`
3. **JWT validation** en cada endpoint via `get_current_user` (valida JWT + consulta portal_usuarios)
4. **Audit log** en toda operación de escritura (quién, cuándo, qué)
5. **Soft delete** en todas las entidades principales (`deleted_at`)
6. **Paginación** en todos los endpoints de listado (`limit`/`offset`)
7. **OPTIONS en gateway** para cada path nuevo (CORS preflight)

## Lo que NO hacer

- No implementar lógica de negocio en los routers; va en `service.py`
- No hacer queries SQL directas; siempre via repository
- No commitear secrets, `.env`, o archivos con credenciales
- No modificar la base de datos de otro servicio directamente
- No saltear specs: si no hay spec aprobado, no se implementa
- No hardcodear puerto en Dockerfile CMD (Cloud Run inyecta `PORT`)
- No usar `config.set_main_option()` en `alembic/env.py` (el `%` rompe configparser con passwords URL-encoded)
- No implementar sin leer los ADRs primero — señalar conflictos antes de actuar
