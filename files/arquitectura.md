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
> ⚠️ **SUPERSEDIDO por ADR-008 (2026-08-31).** Se decidió migrar `svc-privada` al monorepo sobre
> PostgreSQL (`db_privada`) y desmantelar `essential-haiku-482815-u4`. Este ADR se conserva como
> registro histórico; su razonamiento ("migrar implica riesgo sin beneficio inmediato") dejó de
> aplicar cuando el sistema pasó a requerir unificación de auth/CI-CD/operación, salir del proyecto
> GCP personal y habilitar la federación server-side de `resumen_territorial`.

**Decisión**: El sistema de gestión de demandas de la Secretaría Privada del Ministro (`svc-privada`) queda en su proyecto GCP original (`essential-haiku-482815-u4`) con BigQuery como BD. Se expone al nuevo API Gateway como backend externo.
**Razón**: El sistema tiene ~523 registros activos en producción. Migrar implica riesgo operativo sin beneficio inmediato. El API Gateway puede rutear cross-project sin necesidad de migración.
**Consecuencia**: `svc-privada` usa BigQuery (no PostgreSQL). El API Gateway necesita configuración IAM cross-project (Service Account del Gateway con `roles/run.invoker` sobre el Cloud Run externo).

### ADR-007: Módulo portal centralizado en svc-vivienda
**Decisión**: La gestión de usuarios del portal (roles, secretarías asignadas) vive en `svc-vivienda` como módulo `app/portal/`, no como un servicio separado.
**Razón**: Es el único servicio con Cloud SQL activo en producción al momento de implementar esta funcionalidad. Crear un servicio dedicado agrega complejidad operativa sin beneficio a la escala actual.
**Consecuencia**: `svc-vivienda` expone `/api/v1/portal/me` y `/api/v1/portal/admin/usuarios/*` además de sus rutas propias. Si en el futuro se decide separar, es un movimiento de módulo dentro del mismo patrón de código.

### ADR-008: Migrar svc-privada al monorepo (supersede ADR-006)
**Decisión**: Se construye `services/svc-privada/` como microservicio propio del monorepo, FastAPI + Cloud SQL PostgreSQL (base `db_privada` en la instancia compartida `ministerio-postgres`), y se migran las 12 tablas del dataset BigQuery `infra_gestion`. Al terminar el período de estabilización se desmantela el proyecto GCP personal `essential-haiku-482815-u4` (Cloud Run, BigQuery, OAuth client, frontend GitHub Pages, dashboard Looker). El contrato HTTP `/api/v1/privada/**` se mantiene byte-compatible; sólo se repunta el backend del API Gateway al nuevo Cloud Run.
**Razón**: Revierte ADR-006. El sistema pasó a requerir: unificación de autenticación/permisos con el portal (ADR-003/007), CI/CD y operación comunes, salir de un proyecto GCP de titularidad personal, y habilitar la federación server-side de `resumen_territorial` (ADR-016). El riesgo operativo de la migración se acota con un plan por fases, ETL verificable y cutover con rollback (ver `spec-migracion-svc-privada.md`).
**Consecuencia**: Un microservicio más con su base propia (ADR-001). Se elimina la configuración IAM cross-project del gateway. Detalle de fasado, ETL, cutover y decommission en `spec-migracion-svc-privada.md`. Las mejoras funcionales pedidas junto con la migración (categorías/programas/áreas editables, DAG de flujo, ficha de localidad, campos Ok Gobernador/Ok Ministro, Tablero nativo) se implementan como specs hijos **posteriores al cutover** y no forman parte de la migración-paridad.

### ADR-009: svc-privada usa el patrón panel-module, no el generic-core
**Decisión**: El código interno de `svc-privada` sigue el patrón de `app/cordon_cuneta/` / `app/informes/` de svc-vivienda: queries inline en `service.py` (sin `repository.py`), sin Pub/Sub, `log_audit` en toda escritura, y máquina de estados **laxa** (columna `estado VARCHAR` + `CHECK`, sin tabla de transiciones válidas ni validación de transición). Explícitamente **no** se porta el patrón generic-core de `app/expedientes/` (máquina de estados formal `TRANSICIONES_VALIDAS` + `publish_event`).
**Razón**: El sistema de gestiones de Privada es un ABM con historial de eventos, no un flujo de expediente con transiciones controladas. El sistema viejo nunca validó transiciones. Copiar el generic-core agregaría complejidad (repositorio, eventos Pub/Sub, máquina formal) sin requerimiento que lo justifique. El "Repository pattern obligatorio" de `CLAUDE.md` ya tiene esta excepción sancionada en `cordon_cuneta`/`informes`/`resumen_territorial`.
**Consecuencia**: `svc-privada` no publica eventos Pub/Sub. Correcciones al portar el sistema viejo: el lock optimista frágil (`WHERE fecha_ingreso = @old`) pasa a `updated_at` + `409`; `estado = FINALIZADA` sí setea `fecha_finalizacion` (bug del sistema viejo, se corrige y se backfillea en el ETL).

### ADR-010: Categoría, Programa y Área en svc-privada son tres catálogos editables independientes
**Decisión**: `priv_categorias`, `priv_programas` y `priv_areas` son tablas-catálogo con CRUD en runtime desde un panel de administración (mismo patrón que `viv_cc_estados`: PK bigint client-generada, `label`, `orden`, `activo`; `bg`/`text_color` sólo donde aporte). En el alta/edición de una gestión el usuario elige libremente de cada desplegable (con opción "+ nueva" inline); `priv_gestiones` gana FKs nullable `categoria_id`, `programa_id`, `area_id`. **No** hay tabla de mapeo obligatoria Categoría→Programa→Área (opcional/diferido: una "sugerencia" por categoría). El `DELETE` de un catálogo tiene guard de integridad → `409 {"code":"...EN_USO"}` contando referencias en `priv_gestiones`, en el historial y en el mapa de clasificación del informe.
**Razón**: El área pidió explícitamente que Programa y Área sean desplegables ampliables desde administración, no un mapa cargado. La clasificación actual del informe (`v_informe_cooperativas`) es por regex sobre `LOWER(detalle)` + `categoria_general_id` legacy — frágil; moverla a `categoria_id` estructurado es una decisión de integridad de datos con consecuencias de migración/backfill (ver RE-1 en el spec hijo).
**Consecuencia**: Migración `0002+` en `db_privada` agrega las 3 tablas + columnas + backfill desde `categoria_general_id`. La columna legacy `categoria_general_id` (y `tipo_gestion`/`canal_origen`) se conservan verbatim durante la migración-paridad; el spec hijo las supersede. Spec: `spec-privada-categorias-programas.md`.

### ADR-011: svc-privada tiene su propio priv_programas editable (no reusa viv_programas)
**Decisión**: El catálogo de programas de Privada es `priv_programas` en `db_privada`, con CRUD desde el panel de administración. No se importa ni se comparte `viv_programas` de svc-vivienda. La correlación con los programas de Vivienda (Córdoba Hogar, Mi Lugar, Cordón Cuneta) es string-keyed por `codigo`/`label`, sólo para agregación — nunca una FK entre servicios.
**Razón**: ADR-001 (database-per-service). `viv_programas` no tiene endpoint `POST` (sólo se siembra) ni concepto de `area`; reusarlo forzaría un camino de escritura cross-service. Fija la regla general "correlacionar programas entre servicios por string, nunca por FK".
**Consecuencia**: Puede haber `codigo` de programa que colisione o se solape con los de Vivienda — se documentan los códigos reservados y se normaliza `codigo` con unicidad. `POST /api/v1/privada/programas` restringido a `ROLES_TRANSICION` (no Operador) para acotar el sprawl de catálogo.

### ADR-012: Los datos de referencia territorial son propiedad de svc-privada, expuestos read-only vía gateway
**Decisión**: El padrón demográfico/electoral (`priv_localidades_info` ~551, `priv_departamentos_info` 26, `priv_geo_localidades` ~551) vive en `db_privada` y lo mantiene el UPSERT existente de Privada (sólo 4 campos editables: `habitantes`, `electores`, `intendente_jefe_comunal`, `partido_politico`; `tipo_localidad` y `color_semaforo` son read-only, cargados por un proceso Excel one-off). Otros servicios (p. ej. `resumen_territorial` en svc-vivienda) lo consumen **read-only** vía `GET /api/v1/privada/localidades-info` y `GET /api/v1/privada/departamentos-info`. No se crea un `svc-referencia` compartido ahora, ni se duplica la demografía en `db_vivienda`.
**Razón**: La demografía la usa hoy sólo Privada y mañana la ficha de localidad del panel transversal. Crear un servicio de referencia o duplicar la tabla es sobre-ingeniería a la escala actual. Dejar explícito que es fuente única de verdad evita que cada secretaría futura se arme su copia.
**Consecuencia**: `resumen_territorial` hace una llamada HTTP read-only a `svc-privada` para la ficha (no un join cross-DB). Riesgo de staleness de `color_semaforo`/`tipo_localidad` (nadie los refresca) — se documenta el procedimiento de recarga. Spec: `spec-resumen-territorial-ficha-localidad.md`.

### ADR-013: El flujo/DAG de una gestión se apoya en una tabla de derivaciones estructurada
**Decisión**: `priv_gestion_derivaciones` (append-only: `gestion_id`, `area_desde_id`, `area_hacia_id` → `priv_areas`, `estado`, `fecha`, `usuario`, `evento_id`, `origen` ∈ {`backfill`, `runtime`}, `confianza`) es la fuente del diagrama de flujo. `priv_gestiones_eventos` se conserva 1:1 como log de auditoría append-only. El histórico de derivaciones que hoy vive dentro de `gestiones_eventos.metadata_json` (`derivado_a` texto libre) se backfillea best-effort a la tabla estructurada, normalizando contra `priv_areas` + una tabla de alias, con `confianza` por fila y un nodo centinela "área desconocida". Los nodos del DAG son `priv_areas` (sembrada híbrido: valores distintos observados en `derivado_a`/`organismo_id` + curado a mano, editable en runtime).
**Razón**: No existe hoy ninguna entidad de flujo ni taxonomía de áreas; `derivado_a` es texto libre. Un DAG legible necesita un set de nodos estable. Parsear `metadata_json` en cada request no da nodos consistentes.
**Consecuencia**: `cambiar-estado`/`derivar` escriben una fila de derivación en la misma transacción. La UI muestra un badge "reconstruido" en los tramos de baja confianza. Spec: `spec-privada-flujo-derivaciones.md`.

### ADR-014: El Tablero de Privada se reconstruye nativo en React (se retira el iframe Looker Studio)
**Decisión**: `TableroPage.tsx` reemplaza el iframe embebido de Looker Studio (que lee BigQuery directo) por un dashboard React nativo sobre los endpoints `/api/v1/privada/informe/**`. No se mantiene ningún mirror de BigQuery.
**Razón**: El iframe Looker es la única dependencia del frontend con BigQuery; apagar BQ (ADR-008) lo rompe. Un mirror BQ (opción B) o retirar el tab (opción C) fueron descartados: el mirror arrastra un pipeline nuevo permanente, y el tab es funcionalidad activa que el área usa.
**Consecuencia**: El Tablero nativo es prerequisito del apagado de BigQuery (gate del decommission, no del cutover). Spec: `spec-privada-tablero.md`.

### ADR-015: La autenticación de svc-privada se unifica en portal_usuarios; la lectura cross-service va por endpoint interno IAM
**Decisión**: Los usuarios de Privada pasan a `portal_usuarios` (en `db_vivienda`) con la secretaría `"privada"` asignada. Se descartan las tablas propias `usuarios_roles`, `usuarios_eventos`, `usuario_modulos`, `cat_modulos` y el camino de autenticación Google-OAuth2 directo (queda sólo el flujo estándar del gateway con `X-Forwarded-Authorization`). Los roles del sistema viejo (`Admin`/`Supervisor`/`Operador`/`Consulta`) mapean 1:1 al portal, así que `svc-privada` **sí** reutiliza las tuplas compartidas `ROLES_LECTURA`/`ROLES_ESCRITURA`/`ROLES_ELIMINACION` de `app/auth.py` (a diferencia de `TecnicoDGV`/`Autoridad`). `svc-privada` lee `portal_usuarios` llamando `GET /internal/portal/usuarios/{email}` en `svc-vivienda` (endpoint IAM-only, sin prefijo `/api/v1`, SA de `svc-privada` con `roles/run.invoker`), **nunca** conectándose directo a `db_vivienda`.
**Razón**: Extiende ADR-003/ADR-007 a un segundo servicio. Un endpoint interno IAM evita acoplar `svc-privada` al esquema de `db_vivienda` y respeta database-per-service. Fija la regla "para leer datos de otro servicio, endpoint interno IAM, no conexión cross-DB".
**Consecuencia**: Se retiran del gateway los paths `/api/v1/privada/usuarios/**` y `/api/v1/privada/catalogos/modulos`; la gestión de usuarios de Privada pasa a `AdminUsuariosPage` del portal. **Condición**: si el relevamiento con el área revela usuarios con acceso a *sólo una parte* del sistema (uso real de `usuario_modulos`), se agrega un rol acotado estilo `TecnicoDGV` en vez del 1:1 puro — se registra como enmienda a este ADR.

### ADR-016: resumen_territorial pasa a federación server-side de Privada (supersede la decisión "plan B" de spec-resumen-territorial.md §3.3)
**Decisión**: Con `svc-privada` en el mismo proyecto GCP, `resumen_territorial` calcula las líneas de Privada en el servidor: `settings.privada_fetch_enabled = True`, llamando `GET /api/v1/privada/gestiones/rollup-territorial` (endpoint nuevo, rollup global por `(departamento, localidad)` sin `departamento` obligatorio) con el ID token de la SA de `svc-vivienda`. Se elimina la federación en el browser (`frontend/src/modules/resumen-territorial/api/privadaGestiones.ts`).
**Razón**: `spec-resumen-territorial.md §3.3` había resuelto federar en el frontend ("plan B") porque el `svc-privada` externo rechazaba el token de la SA y su endpoint exigía `departamento`. Ambos bloqueos desaparecen con el nuevo backend en el mismo proyecto + el endpoint de rollup. La federación server-side elimina la paginación N+1 en el navegador y la regla de visibilidad duplicada client-side.
**Consecuencia**: Cambia el modo de falla: una caída de Privada produce un snapshot parcial para todos hasta el próximo recálculo (antes era un aviso por-usuario). Se conserva la semántica de `generado_para_areas` y se deja el path browser detrás de un flag un release. Spec: `spec-resumen-territorial-ficha-localidad.md` (o enmienda v0.3.0 a `spec-resumen-territorial.md`).

## Sistema de roles del portal

| Rol | Descripción |
|-----|-------------|
| `Admin` | Acceso total — puede ver y editar todas las secretarías asignadas, eliminar registros, gestionar usuarios del portal |
| `Supervisor` | Puede ver, crear, editar y eliminar dentro de sus secretarías asignadas |
| `Operador` | Puede ver, crear y editar pero no eliminar |
| `Consulta` | Solo lectura |

**Roles fuera de la jerarquía** (no son un rango; son alcance acotado a un módulo). No se
agregan a las constantes compartidas de `app/auth.py` — cada router que los admite declara su
propia tupla local (patrón introducido en `spec-checklist-tecnico-dgv.md §8`):

| Rol | Alcance |
|-----|---------|
| `TecnicoDGV` | Sólo el módulo `checklist_tecnico` + el Tablero de `programas`, dentro de `vivienda` |
| `Autoridad` | Lectura consolidada cross-área en el módulo `resumen_territorial` (ve todas las áreas, igual que `Admin`). Puede además tener secretarías asignadas para los paneles operativos. Ver `spec-resumen-territorial.md §7` |

Los roles se almacenan en `portal_usuarios.rol` en Cloud SQL. Tras la migración de `svc-privada`
(ADR-008/ADR-015), los usuarios de Privada también viven en `portal_usuarios` con la secretaría
`"privada"` asignada y roles 1:1 con la jerarquía — Privada **no** es un rol acotado; reutiliza las
tuplas compartidas de `app/auth.py`.

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
