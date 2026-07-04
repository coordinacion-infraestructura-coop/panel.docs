# Arquitectura técnica — Decisiones y patrones

## Visión general

Sistema de microservicios desplegado en Google Cloud Platform. Cada secretaría tiene su propio servicio FastAPI independiente con su propia base de datos PostgreSQL. Un portal React unificado consume todos los servicios via un único API Gateway.

## Diagrama de capas

```
[Usuarios: funcionarios / ministro / ciudadanos]
          ↓ HTTPS
[Firebase Hosting — React SPA — CDN global]
          ↓ fetch() / axios
[GCP API Gateway — validación JWT, rate limiting, ruteo]
          ↓ HTTP interno
[Cloud Run — microservicios FastAPI]
    ├── svc-vivienda       → Cloud SQL db_vivienda
    ├── svc-infraestructura → Cloud SQL db_infraestructura
    ├── svc-territorial    → Cloud SQL db_territorial
    ├── svc-desarrollo     → Cloud SQL db_desarrollo + UTN API
    ├── svc-gasifera       → Cloud SQL db_gasifera
    └── svc-[transversales] → Firestore / Cloud Storage
          ↓ eventos
[Cloud Pub/Sub → BigQuery — dashboard ejecutivo]
```

## Decisiones de arquitectura registradas

### ADR-001: Database per service
**Decisión**: Cada microservicio tiene su propia instancia de Cloud SQL PostgreSQL.
**Razón**: Aislamiento de datos, deploys independientes, equipos pueden evolucionar sus schemas sin coordinar con otros.
**Consecuencia**: Los reportes ejecutivos se alimentan via eventos a BigQuery, no por joins entre bases.

### ADR-002: API Gateway como único punto de entrada
**Decisión**: GCP API Gateway delante de todos los microservicios.
**Razón**: Centraliza auth, logging, rate limiting. Los Cloud Run services no son públicos.
**Consecuencia**: Todo cambio de API pública requiere actualizar el spec OpenAPI del gateway y crear una nueva versión de config (las configs son inmutables).

### ADR-003: Permisos DB-first (revisado 2026-07-02)
**Decisión original**: JWT con custom claims `{role, secretaria}` — el backend leería el rol del token sin consultar la BD.
**Decisión implementada**: BD-first — tras validar el JWT (Firebase), se consulta `portal_usuarios` en Cloud SQL para obtener rol + secretarías asignadas al usuario. Los custom claims de Firebase no se usan para permisos.
**Razón del cambio**: El usuario requirió un panel de administración en React con Cloud SQL como fuente de verdad. A la escala del sistema (decenas de usuarios), el overhead de un SELECT por PK por request es despreciable (~5ms). La revocación de acceso es inmediata (no hay espera de expiración del JWT). El gateway no necesita conocer roles para enrutar.
**Consecuencia**: Los cambios de rol tienen efecto inmediato en el próximo request. Cada request autenticado hace un SELECT en `portal_usuarios`. Tabla `portal_usuarios` + `portal_usuario_secretarias` en `svc-vivienda` es el único lugar donde se gestionan permisos del nuevo portal. Los cambios de roles NO requieren renovación de token.
**Implementación**: `app/portal/` en svc-vivienda, tablas migración 0004. El `app/auth.py` hace la consulta DB después de validar el JWT; el lookup tiene try/except — si falla, el usuario recibe `role="invitado"` (sin acceso) en lugar de 500.

### ADR-004: Integración UTN via adapter service
**Decisión**: `svc-desarrollo` envuelve el sistema UTN con una capa adaptadora.
**Razón**: El sistema UTN es externo, su API puede cambiar. El adaptador aísla ese riesgo.
**Consecuencia**: Los datos de gestión de cooperativas (UTN) se sincronizan a BigQuery via Pub/Sub para el dashboard ejecutivo.

### ADR-005: Soft delete universal
**Decisión**: Todas las entidades tienen `deleted_at TIMESTAMP NULL`. Nunca se borra físicamente.
**Razón**: Trazabilidad y auditoría requerida en sistemas de gobierno.
**Consecuencia**: Todos los queries deben filtrar `WHERE deleted_at IS NULL` por defecto.

### ADR-006: svc-privada coexiste en proyecto GCP separado con BigQuery
**Decisión**: El sistema de gestión de demandas de la Secretaría Privada del Ministro (`svc-privada`) queda en su proyecto GCP original (`essential-haiku-482815-u4`) con BigQuery como BD. Se expone al nuevo API Gateway como backend externo.
**Razón**: El sistema tiene ~523 registros activos en producción. Migrar implica riesgo operativo sin beneficio inmediato. El API Gateway puede rutear cross-project sin necesidad de migración.
**Consecuencia**: `svc-privada` usa BigQuery (no PostgreSQL). El API Gateway necesita configuración IAM cross-project (Service Account del Gateway con `roles/run.invoker` sobre el Cloud Run externo).

### ADR-007: Módulo portal centralizado en svc-vivienda
**Decisión**: La gestión de usuarios del portal (roles, secretarías asignadas) vive en `svc-vivienda` como módulo `app/portal/`, no como un servicio separado.
**Razón**: Es el único servicio con Cloud SQL activo en producción al momento de implementar esta funcionalidad. Crear un servicio dedicado agrega complejidad operativa sin beneficio a la escala actual.
**Consecuencia**: `svc-vivienda` expone `/api/v1/portal/me` y `/api/v1/portal/admin/usuarios/*` además de sus rutas propias. Si en el futuro se decide separar, es un movimiento de módulo dentro del mismo patrón de código.

## Sistema de roles del portal

| Rol | Descripción |
|-----|-------------|
| `Admin` | Acceso total — puede ver y editar todas las secretarías asignadas, eliminar registros, gestionar usuarios del portal |
| `Supervisor` | Puede ver, crear, editar y eliminar dentro de sus secretarías asignadas |
| `Operador` | Puede ver, crear y editar pero no eliminar |
| `Consulta` | Solo lectura |

Los roles se almacenan en `portal_usuarios.rol` en Cloud SQL. Son independientes de los roles del sistema existente (svc-privada, que tiene su propia gestión de usuarios con BigQuery).

Constantes en `app/auth.py`:
```python
ROLES_LECTURA    = ("Admin", "Supervisor", "Operador", "Consulta")
ROLES_ESCRITURA  = ("Admin", "Supervisor", "Operador")
ROLES_TRANSICION = ("Admin", "Supervisor")
ROLES_ELIMINACION = ("Admin",)
ROLES_ADMIN       = ("Admin",)
```

## Estructura de directorios por servicio backend

```
svc-{nombre}/
├── Dockerfile
├── pyproject.toml
├── alembic.ini
├── alembic/
│   └── versions/
├── app/
│   ├── main.py              # FastAPI app, middlewares, lifespan
│   ├── config.py            # Settings con pydantic-settings
│   ├── database.py          # Engine async, get_db()
│   ├── auth.py              # JWT validation + DB lookup de permisos
│   ├── audit.py             # Audit log middleware
│   └── {modulo}/
│       ├── __init__.py
│       ├── router.py        # APIRouter, endpoints
│       ├── models.py        # SQLAlchemy ORM models
│       ├── schemas.py       # Pydantic schemas (Request/Response)
│       ├── service.py       # Lógica de negocio
│       └── repository.py    # Queries a la base de datos
└── tests/
    ├── conftest.py
    └── test_{modulo}.py
```

## Estructura del frontend React

```
frontend/src/
├── App.tsx                  # Rutas y providers
├── pages/
│   ├── LoginPage.tsx
│   └── DashboardPage.tsx    # Panel central — filtra secretarías por permisos del usuario
├── shared/
│   ├── auth/                # Firebase auth (AuthContext, ProtectedRoute)
│   ├── api/                 # axios client con interceptores JWT
│   ├── components/
│   │   └── Layout.tsx       # Nav contextual por secretaría activa
│   └── hooks/
│       └── usePortalUser.ts # Hook react-query → GET /api/v1/portal/me
└── modules/
    ├── vivienda/            # Programas, Beneficiarios, Expedientes, Cordón Cuneta
    ├── privada/             # Gestiones (lista+drawer+modal), Tablero (Looker Studio)
    └── admin/
        └── AdminUsuariosPage.tsx  # CRUD de usuarios del portal (solo Admin)
```

## Convenciones de API REST

```
GET    /api/v1/{secretaria}/{recurso}           → listar (paginado)
POST   /api/v1/{secretaria}/{recurso}           → crear
GET    /api/v1/{secretaria}/{recurso}/{id}      → detalle
PUT    /api/v1/{secretaria}/{recurso}/{id}      → reemplazar
PATCH  /api/v1/{secretaria}/{recurso}/{id}      → actualizar parcial
DELETE /api/v1/{secretaria}/{recurso}/{id}      → soft delete

GET    /api/v1/{secretaria}/{recurso}/{id}/historial → audit log del recurso
GET    /api/v1/portal/me                        → perfil del usuario autenticado (rol + secretarías)
GET    /api/v1/portal/admin/usuarios            → admin: listar todos los usuarios
POST   /api/v1/portal/admin/usuarios            → admin: crear usuario
GET    /health                                  → health check (sin auth)
```

Parámetros de paginación estándar:
```
?limit=20&offset=0&order_by=created_at&order=desc
```

## Manejo de errores

```json
{
  "error": {
    "code": "RECURSO_NO_ENCONTRADO",
    "message": "El expediente solicitado no existe",
    "detail": "expediente_id=123 no encontrado en db_vivienda",
    "timestamp": "2025-01-15T10:30:00Z",
    "trace_id": "abc-123-def"
  }
}
```

Códigos de error propios (además de HTTP status):
- `AUTH_TOKEN_INVALIDO` — JWT inválido o expirado
- `PERMISO_INSUFICIENTE` — rol sin acceso al recurso
- `USUARIO_NO_REGISTRADO` — usuario Firebase válido pero sin entrada en `portal_usuarios`
- `RECURSO_NO_ENCONTRADO` — 404 tipado
- `VALIDACION_FALLIDA` — error de schema Pydantic
- `CONFLICTO_ESTADO` — operación inválida para el estado actual
- `EMAIL_DUPLICADO` — intento de crear usuario con email ya existente

## Eventos Pub/Sub

Todos los servicios publican eventos a tópicos de Pub/Sub para:
1. Alimentar BigQuery (dashboard ejecutivo)
2. Disparar notificaciones
3. Sincronización eventual entre servicios

Formato de evento:
```json
{
  "event_id": "uuid-v4",
  "event_type": "vivienda.expediente.creado",
  "service": "svc-vivienda",
  "timestamp": "2025-01-15T10:30:00Z",
  "actor": {
    "user_id": "uid-firebase",
    "role": "Operador"
  },
  "payload": { ... }
}
```

Naming de event types: `{secretaria}.{recurso}.{accion}`
