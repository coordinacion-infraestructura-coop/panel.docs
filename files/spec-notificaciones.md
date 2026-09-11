# Spec: Panel de notificaciones internas

**Estado**: approved (backend + frontend + gateway implementados 2026-09-10; pendiente de deploy)
**Versión**: 0.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-10
**Servicio**: `svc-vivienda` (módulo nuevo `app/notificaciones/`, sin servicio nuevo) + frontend `src/modules/notificaciones/`
**Depende de**: ADR-007 (módulo transversal en svc-vivienda) · `spec-resumen-territorial.md` §3.4 (mismo criterio de ubicación)
**ADRs**: ADR-019 (panel de notificaciones internas)

---

## 0. Origen

Pedido del usuario (2026-09-10): "crear un panel de notificaciones para que lleguen alertas
internas del sistema que puedan visualizarse por allí. Lo creamos y luego lo iremos alimentando."

En esta etapa se construye el **almacenamiento + la vista + el canal de alta interno**. La lógica
que *genera* notificaciones automáticamente (fallos de sync, vencimientos, etc.) es un paso
posterior.

Prior art: sólo una línea de backlog en `docs/context/tasks/todo.md` ("Módulo de notificaciones
(Pub/Sub → Cloud Scheduler → email/push)", "cuando haya demanda real") y un `svc-notificaciones`
dedicado descartado en `docs/context/organigrama.md`. No había spec ni ADR.

## 1. Propósito

Un lugar único donde cualquier usuario del portal ve las alertas internas del sistema, con
historial y estado leído/no leído. Reemplaza (para este tipo de aviso) el banner rojo efímero y
local que hoy usa cada pantalla.

## 2. Alcance

### Incluido (v1)

- **Feed global** de notificaciones (`portal_notificaciones`) — una fila por evento del sistema.
- **Estado de lectura por usuario** (`portal_notificacion_lecturas`) — la ausencia de fila = no
  leída. No se duplica la notificación por destinatario.
- **Destino opcional** por notificación: `global` (todos), `rol` (`destino_valor` = nombre de rol)
  o `secretaria` (`destino_valor` = id de secretaría). El filtro por visibilidad se aplica en la
  lectura; `Admin`/`Autoridad` ven todo (mismo criterio que `resumen_territorial`).
- **Página `/notificaciones`** — lista con filtro "solo no leídas", "marcar leída" por fila y
  "marcar todas como leídas".
- **Campana con badge** de no leídas en la barra superior (poll cada 60 s, patrón de
  `CordonCunetaPage` sync-estado). Un click lleva a `/notificaciones`.
- **Alta interna**: `POST /internal/notificaciones` (IAM-only, no expuesto por el Gateway — patrón
  ADR-015). Es el punto de integración para jobs (Cloud Scheduler) u otros servicios.
- Migración `0028` siembra 2 filas de ejemplo para que el panel no arranque vacío.

### Fuera de alcance (v1)

- Emisión automática de notificaciones desde jobs / eventos Pub/Sub / reglas de negocio.
- Alta / edición / borrado de notificaciones desde la UI.
- Dropdown de la campana con las últimas N (v1: la campana sólo navega a la página).
- Notificaciones por email / push.
- Segmentación por usuario individual (sólo `global` / `rol` / `secretaria`).

## 3. Decisiones de arquitectura

- **Ubicación (ADR-007 / ADR-019)**: módulo `app/notificaciones/` en `svc-vivienda`, montado con
  prefijo `/api/v1` (no `/api/v1/vivienda`), igual que `app/portal/` y `app/resumen_territorial/`.
  `svc-vivienda` es el único servicio con Cloud SQL activo; si más adelante se separa, es un
  movimiento de módulo dentro del mismo patrón.
- **Feed global, no bandeja por-usuario**: una fila por evento. El emisor no hace fan-out. La única
  parte por-persona es la marca de leída.
- **Patrón panel-module** (ADR-009): queries inline en `service.py`, sin `repository.py`,
  `log_audit` en las escrituras, sin Pub/Sub.
- **`VARCHAR` + `CHECK`, nunca ENUM** para `nivel` y `destino_tipo` (convención del proyecto).
- **Prefijo de tabla `portal_*`**: funcionalidad transversal adyacente a la identidad de usuario
  (precedente `portal_usuarios`), no `viv_*`.
- **Emisión desacoplada por endpoint interno IAM** (ADR-015): `POST /internal/notificaciones` no se
  declara en `infra/gateway/openapi.yaml`; sólo lo invocan principals con `roles/run.invoker`.

## 4. Modelo de datos

Migración `alembic/versions/20260910_0028_notificaciones.py` (`down_revision = "0027"`).

### `portal_notificaciones`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `VARCHAR(36)` PK | uuid4 |
| `titulo` | `VARCHAR(200)` NOT NULL | |
| `mensaje` | `TEXT` NOT NULL | |
| `nivel` | `VARCHAR(20)` NOT NULL default `'info'` | CHECK `IN ('info','exito','advertencia','error')` |
| `origen` | `VARCHAR(60)` NOT NULL default `'sistema'` | texto libre: sistema de origen (`sync-checklist`, `gestiones`, …) |
| `enlace` | `VARCHAR(500)` NULL | ruta in-app opcional (p. ej. `/privada/gestiones?estado=...`) |
| `destino_tipo` | `VARCHAR(20)` NOT NULL default `'global'` | CHECK `IN ('global','rol','secretaria')` |
| `destino_valor` | `VARCHAR(60)` NULL | nombre de rol o id de secretaría si `destino_tipo != 'global'` |
| `created_at` | `TIMESTAMPTZ` NOT NULL | |
| `created_by` | `VARCHAR(200)` NULL | `actor.email` / `cloud-scheduler` / `migracion-0028` |
| `deleted_at` | `TIMESTAMPTZ` NULL | soft delete (sin UI en v1) |

Índice `ix_portal_notificaciones_created_at (created_at)`.

### `portal_notificacion_lecturas`

| Columna | Tipo | Notas |
|---|---|---|
| `notificacion_id` | `VARCHAR(36)` | PK compuesta · FK → `portal_notificaciones.id` `ON DELETE CASCADE` |
| `usuario_email` | `VARCHAR(200)` | PK compuesta |
| `leida_at` | `TIMESTAMPTZ` NOT NULL | |

Índice `ix_portal_notif_lecturas_usuario (usuario_email)`.

## 5. Endpoints

Router `app/notificaciones/router.py`, tupla local
`ROLES_NOTIF = ROLES_LECTURA + ("Autoridad", "TecnicoDGV")` (todos los roles reales del portal;
`invitado` queda afuera). No se toca `app/auth.py`.

| Método | Path | Roles | Descripción |
|---|---|---|---|
| GET | `/api/v1/notificaciones?solo_no_leidas=&limit=&offset=` | `ROLES_NOTIF` | Feed visible + `total` + `no_leidas`; cada item con `leida` calculado para el usuario |
| GET | `/api/v1/notificaciones/no-leidas/contar` | `ROLES_NOTIF` | `{ "no_leidas": N }` — lo consume el badge de la campana |
| POST | `/api/v1/notificaciones/{id}/marcar-leida` | `ROLES_NOTIF` | Upsert de la marca de leída del usuario (idempotente). 404 si no existe/no es visible |
| POST | `/api/v1/notificaciones/marcar-todas-leidas` | `ROLES_NOTIF` | Marca todas las visibles no leídas del usuario → `{ "marcadas": N }` |
| POST | `/internal/notificaciones` | IAM (`run.invoker`) | Alta. Body `NotificacionIn`. **No** declarado en el Gateway |

`NotificacionIn`: `titulo`, `mensaje`, `nivel='info'`, `origen='sistema'`, `enlace=None`,
`destino_tipo='global'`, `destino_valor=None`.

## 6. Frontend — `src/modules/notificaciones/`

- `types/notificaciones.types.ts` — espejo de `schemas.py` (`Notificacion`, `NotificacionesListResponse`).
- `api/notificaciones.api.ts` — `notificacionesApi` con `list` / `contarNoLeidas` / `marcarLeida` /
  `marcarTodasLeidas` (patrón `<recurso>Api` + `apiClient`).
- `pages/NotificacionesPage.tsx` — lista (plantilla `AdminUsuariosPage` + rama de error estilo
  `TableroPage`). Encabezado con toggle "Solo no leídas" + "Marcar todas como leídas". Fila: pill de
  `nivel` (sky/green/amber/red), título, mensaje, `origen` + tiempo relativo, "Marcar leída" por
  fila no leída, "Ver" si hay `enlace`. Las mutaciones invalidan `['notificaciones']` y
  `['notificaciones-contador']`.
- `src/shared/components/NotificacionesCampana.tsx` — `useQuery(['notificaciones-contador'], …, {
  refetchInterval: 60_000, enabled: !!portalUser })`; botón campana (SVG inline) + badge naranja
  (`gov-orange`, tope "9+"); `onClick` → `navigate('/notificaciones')`.
- `src/shared/components/Layout.tsx` — `<NotificacionesCampana />` en el header (antes del bloque
  de usuario) + link "Notificaciones" en la nav transversal (junto a "Resumen Territorial"), ambos
  gated por `portalUser`.
- `src/App.tsx` — `<Route path="notificaciones" element={<NotificacionesPage />} />` sin
  `ProtectedRoute roles={…}` interno (el backend aplica el gate real).

## 7. Infraestructura

- `infra/gateway/openapi.yaml` — 4 paths nuevos (`get`/`post` + `options` CORS), `x-google-backend`
  → Cloud Run de `svc-vivienda`, `path_translation: APPEND_PATH_TO_ADDRESS`. `/internal/notificaciones`
  **no** se agrega. Requiere nueva config `ministerio-config-v{YYYYMMDD}` + `gateways update`.
- Registro de modelos: `app/main.py` y `alembic/env.py` importan `Notificacion, NotificacionLectura`.
- Sin Cloud Scheduler nuevo en v1 (no hay job de emisión todavía).

## 8. Riesgos

- **RN-1** — La emisión (lo que llena el feed) no existe en v1; el panel arranca sólo con las 2
  filas sembradas. Riesgo de que quede percibido como "vacío/inútil" hasta el paso siguiente.
  Mitigación: es explícitamente el plan del usuario ("lo iremos alimentando").
- **RN-2** — Sin baja/expiración, `portal_notificaciones` crece sin techo. A la escala actual es
  irrelevante; si el volumen sube, agregar retención (job que hace soft delete de > N meses).
- **RN-3** — `destino_valor` es texto libre; un rol/secretaría mal escrito hace que la
  notificación no le llegue a nadie (sólo a `Admin`/`Autoridad`). Es error de visibilidad, no de
  integridad. Mitigación: validar contra `ROLES_VALIDOS` / `SECRETARIAS_VALIDAS` cuando exista UI
  de alta.
- **RN-4** — El poll de 60 s de la campana agrega una request por usuario/minuto a `svc-vivienda`.
  El endpoint es un `COUNT` barato con índice; aceptable. Revisar si el nº de usuarios crece mucho.

## 9. Decisiones abiertas

- ¿Dropdown de la campana con las últimas 5 (sin ir a la página)? — diferido.
- ¿Alta manual de un aviso por un Admin desde la UI? — diferido (hoy sólo `/internal`).
- ¿La emisión automática se hará por endpoint interno (job que barre y postea) o por consumo de
  eventos Pub/Sub? — se decide en el spec del paso siguiente.
- ¿Notificaciones por email para `nivel = 'error'`? — fuera de v1.

## 10. Criterios de aceptación

- [x] `alembic upgrade head` crea `portal_notificaciones` + `portal_notificacion_lecturas` y
      siembra 2 filas de ejemplo.
- [x] `GET /api/v1/notificaciones` devuelve el feed visible para el usuario con `leida` por-usuario
      + `total` + `no_leidas`.
- [x] Una notificación `destino_tipo='rol'`/`'secretaria'` no la ve un usuario que no coincide;
      `Admin`/`Autoridad` la ven siempre.
- [x] `marcar-leida` es idempotente y por-usuario (otro usuario sigue viéndola como no leída);
      `marcar-todas-leidas` deja `no_leidas = 0`.
- [x] `POST /internal/notificaciones` crea una fila y aparece en el feed.
- [x] `invitado` → 403 en todos los `/api/v1/notificaciones/*`.
- [x] Frontend: `npm run build` pasa; la campana muestra el badge y decrementa al marcar leída; el
      toggle "solo no leídas" filtra.
- [ ] Deploy: config nueva del gateway activa + `curl` a `/api/v1/notificaciones` por el gateway
      devuelve el feed; `OPTIONS` responde CORS.
- [x] `pytest tests/test_notificaciones.py` en verde (12 casos) + suite completa sin regresiones
      (272 en verde).
