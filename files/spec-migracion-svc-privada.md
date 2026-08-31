# Spec: Migración de svc-privada al monorepo (BigQuery → PostgreSQL)

**Estado**: approved
**Versión**: 1.0.0
**Aprobado**: 2026-08-31 (Pedro Bonafe) — decisiones D-1..D-4 tomadas; ADR-008..ADR-016 registrados.
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-08-31
**Servicio**: nuevo `services/svc-privada/` (reemplaza el Cloud Run externo en `essential-haiku-482815-u4`)

> **Supersede ADR-006.** ADR-008 (`docs/files/arquitectura.md`) revierte la decisión de no migrar.
> Ver también ADR-009 (patrón panel-module), ADR-012 (datos territoriales propiedad de svc-privada),
> ADR-015 (auth al portal + endpoint interno IAM), ADR-016 (`resumen_territorial` federación
> server-side).
>
> **Este spec es sólo migración-paridad.** Las mejoras funcionales pedidas en la misma sesión
> (2026-08-31) — categorías/programas/áreas editables, DAG de flujo, ficha de localidad, campos
> Ok Gobernador/Ok Ministro, Tablero nativo — son **specs hijos posteriores al cutover** (ver §2).
>
> **Decisiones tomadas (2026-08-31):**
> - **D-3** — No hay (o no se conoce) acceso módulo-parcial en Privada: roles 1:1 con el portal,
>   acceso = secretaría `"privada"`. Si el relevamiento revela lo contrario → enmienda a ADR-015.
> - **D-4** — El Tablero se reconstruye **nativo en React** (ADR-014). Sin mirror de BigQuery.
> - Instancia Cloud SQL: `db_privada` en la instancia compartida `ministerio-postgres` (§3.2).
> - `GET /api/v1/privada/me` se mantiene como alias de `/api/v1/portal/me` durante v1.
> - `PATCH /gestiones/{id}` separado se expone en v1 (hoy la edición va embebida en `cambiar-estado`).
>
> **A confirmar en la reunión de relevamiento** (no bloquea Fases 0–6): ventana de mantenimiento
> del cutover y responsable del smoke test; retención del proyecto viejo (suspender vs borrar).

---

## 0. Origen

Pedido del usuario (sesión 2026-08-31): migrar el sistema de gestión de demandas de la Secretaría
Privada del Ministro dentro del monorepo, con el mismo stack que el resto de los servicios
(FastAPI + Cloud SQL PostgreSQL + Alembic + API Gateway), para unificar autenticación, permisos,
CI/CD y operación, y para dejar de depender del proyecto GCP personal `essential-haiku-482815-u4`.

La misma sesión pidió además 5 mejoras funcionales (ficha de localidad en Resumen Territorial;
seguimiento de gestión como DAG de áreas; categorías editables en runtime; lógica de carga
Categoría→Programa→Área con desplegables ampliables desde administración; campos Ok Gobernador /
Ok Ministro). Esas mejoras **no** son parte de este spec — la migración *habilita* su implementación
posterior; ver §2 y los specs hijos.

Contexto previo:
- `docs/context/areas/Privada Ministro/sistema_gestion_demandas.md` ya anticipaba esta fase:
  *"Inicialmente lo integraremos al servicio general pero más adelante realizaremos un análisis
  integral del servicio, y propuesta de mejora/migración."*
- `proyecto_sistema_gestiones/plan_migracion_nueva_cuenta_gcp.md` es un plan de **réplica del
  sistema tal cual** (BigQuery → BigQuery en otra cuenta). Este spec es distinto: cambia el
  datastore a PostgreSQL y lo absorbe como módulo del monorepo.

## 1. Propósito

Reemplazar el backend externo `svc-privada` (FastAPI sobre BigQuery `infra_gestion`, proyecto
`essential-haiku-482815-u4`) por un microservicio `svc-privada` dentro de `gestorcooperativo`,
sirviendo **el mismo contrato HTTP** que ya consume el frontend React (`src/modules/privada/`),
de modo que el cambio sea transparente para la SPA salvo por la unificación de usuarios/roles
en el portal.

## 2. Alcance

### Incluido (v1)

- **Nuevo servicio `services/svc-privada/`** siguiendo el patrón "panel module" de svc-vivienda
  (router/models/schemas/service; queries inline en `service.py`, sin `repository.py`; sin
  Pub/Sub; audit log en toda escritura). Base de datos propia `db_privada` (ADR-001).
- **Migración de datos** de las 12 tablas de `infra_gestion` (BigQuery) a `db_privada`
  (PostgreSQL), con verificación de integridad (conteos y checksums por tabla).
- **Paridad de endpoints** con el sistema actual bajo el prefijo `/api/v1/privada/**` (ver §6),
  con respuestas byte-compatibles para no tocar `src/modules/privada/` salvo lo indicado.
- **Unificación de auth y permisos en el portal** (ADR-003 / ADR-007): los usuarios de Privada
  pasan a `portal_usuarios` con la secretaría `"privada"` asignada; se elimina la tabla propia
  `usuarios_roles` y el mecanismo `usuario_modulos`. Se elimina la ruta doble de autenticación
  (Google OAuth2 directo vs Firebase JWT) — queda sólo `app/auth.py` estándar.
- **Informe de Cooperativas** (`/api/v1/privada/informe/cooperativas/**`): se porta la vista
  BigQuery `v_informe_cooperativas` (clasificación de gestiones en ~10 "temas") como vista SQL
  o como agregación pura en `service.py`.
- **Datos enriquecidos de localidad/departamento** (`localidades_info`, `departamentos_info`,
  editables desde la UI): migran a `priv_localidades_info` / `priv_departamentos_info`.
- **Gateway**: repuntar `x-google-backend.address` de todos los paths `/api/v1/privada/**` al
  nuevo Cloud Run; nueva config `ministerio-config-v{YYYYMMDD}` y `gateway update`. Quitar la
  config IAM cross-project de ADR-006.
- **CI/CD**: entrada en `services/cloudbuild.yaml`, Dockerfile, secreto `svc-privada-db-url`.
- **Plan de cutover y rollback** (§8).
- **Desmantelamiento** del sistema viejo (§10): proyecto `essential-haiku-482815-u4`, OAuth
  client, frontend GitHub Pages, dashboard Looker.

### Fuera de alcance (v1) — cubierto por specs hijos posteriores al cutover

- **Categorías / Programas / Áreas editables + panel de administración + lógica de carga
  Categoría→Programa→Área** (mejoras 4 y 5, D-1) → `spec-privada-categorias-programas.md`.
  v1 conserva `categoria_general_id`/`tipo_gestion`/`canal_origen` verbatim (§3.11); el spec hijo
  agrega `priv_categorias`/`priv_programas`/`priv_areas` + FKs nullable en `priv_gestiones` +
  backfill.
- **Campos `Ok Gobernador` / `Ok Ministro`** + cableado de `derivado_a` / `acciones_implementadas`
  en el modal de cambiar-estado (mejora 6) → mismo spec de categorías o
  `spec-privada-campos-gestion.md`.
- **DAG de flujo de la gestión** + `priv_gestion_derivaciones` + backfill desde `metadata_json`
  (mejora 3, ADR-013, D-2) → `spec-privada-flujo-derivaciones.md`.
- **Reclasificación del informe de cooperativas sobre el modelo estructurado** (mejora 4, cierre)
  → `spec-privada-informe-cooperativas-v2.md` (o enmienda al de categorías). v1 porta la
  clasificación por regex de `v_informe_cooperativas` **tal cual** (§4.6).
- **Ficha de localidad en Resumen Territorial** + federación server-side de Privada (mejora 2,
  ADR-016) → `spec-resumen-territorial-ficha-localidad.md` (o enmienda v0.3.0 a
  `spec-resumen-territorial.md`). v1 sólo entrega el endpoint `rollup-territorial` que ese spec
  consumirá (§3.9, §6).
- **Tablero React nativo** (D-4, ADR-014) → `spec-privada-tablero.md`. Es prerequisito del
  *decommission* (§10), no del cutover.

### Fuera de alcance (v1) — sin spec hijo

- **Cambiar el frontend de Privada de paradigma**: `GestionesListPage.tsx` seguirá siendo
  server-side paginado (ya lo es).
- **Emisión de eventos Pub/Sub** desde Privada (consistente con CC/CH/ML, ADR-009).
- **Migrar `geo_localidades`** a un recurso compartido: se copia a `priv_geo_localidades` local
  (ADR-012); la consolidación con el GeoJSON de `informes/` queda para después.

---

## 3. Decisiones de arquitectura

### 3.1 Servicio nuevo, no módulo de svc-vivienda

Privada es una secretaría con su propio dominio de datos y su propio equipo/operación. Va como
**servicio independiente** (`services/svc-privada/`, `db_privada`), igual que el resto de las
secretarías del roadmap — **no** como módulo de svc-vivienda. La excepción de ADR-007 (portal
dentro de svc-vivienda) fue por no tener otra base productiva entonces; no aplica acá.

Patrón interno: "panel module" (como `cordon_cuneta`), no "generic core" — **ADR-009**. Sin
`repository.py`, sin máquina de estados formal, sin Pub/Sub, audit log en cada escritura.

### 3.2 Datastore: PostgreSQL `db_privada` en la instancia `ministerio-postgres`

- ADR-001 (database per service) se cumple a nivel **base de datos**: `db_privada` nueva en la
  instancia compartida `ministerio-postgres` (misma que `db_vivienda`, `db_infraestructura`,
  etc.). No se crea una instancia Cloud SQL nueva.
- Se agrega `db_privada` + `user_privada` a `infra/cloudsql-setup.sh` (hoy itera sobre 5
  servicios) y el secreto `svc-privada-db-url` en Secret Manager.
- SQLAlchemy 2.0 async + Alembic, idéntico a svc-vivienda. La migración `0001` crea todo el
  schema `priv_*`.

### 3.3 Modelo de IDs: UUID nuevo + columna legacy

BigQuery usa `id_gestion STRING` (identificador propio, no UUID canónico). En PostgreSQL:
- PK `id UUID DEFAULT gen_random_uuid()`.
- `id_legacy VARCHAR` con UNIQUE — conserva el `id_gestion` de BigQuery para trazabilidad y
  para no romper enlaces/rutas existentes durante el cutover.
- **El frontend hoy usa el string id en la URL del drawer y en las llamadas** (`gestiones/{id}`).
  Decisión: durante v1 el endpoint `GET /gestiones/{id}` **acepta ambos** (UUID o `id_legacy`);
  el `list` devuelve el UUID nuevo como `id`. Se elimina el doble soporte en v2.

### 3.4 Historial de cambios: `priv_gestiones_eventos` (igual que hoy)

`gestiones_eventos` ya es un log append-only de cambios (estado y campo a campo). Se migra 1:1 a
`priv_gestiones_eventos`. Toda escritura (`crear`, `cambiar-estado`, `editar`, `eliminar`)
inserta su fila de evento dentro de la **misma transacción** que la mutación (hoy en BigQuery
son dos statements sin transacción — mejora implícita, no cambia el contrato).

> **Nota forward (ADR-013).** `priv_gestiones_eventos` queda como log de auditoría. El spec hijo
> `spec-privada-flujo-derivaciones.md` agrega una tabla estructurada `priv_gestion_derivaciones`
> como fuente del DAG y hace que `cambiar-estado` escriba además una fila de derivación. La
> migración-paridad **sólo** porta los eventos.

### 3.5 Máquina de estados: se mantiene laxa (sin validación de transiciones)

El `cambiar-estado` actual **no valida transiciones**: cualquier estado → cualquier estado entre
los 6 (`INGRESADO`, `DERIVADO A SUAC`, `LISTA PARA INNAUGURAR`, `FINALIZADA`, `NO REMITE SUAC`,
`ARCHIVADO`). v1 conserva ese comportamiento (registra el evento, no bloquea) — ADR-009. Formalizar
la máquina es un spec posterior. `estado` se persiste como `VARCHAR` + `CHECK` sobre la lista, no
como `ENUM` de Postgres (para poder agregar estados sin migración de tipo).

**Corrección de bug del sistema viejo (desviación deliberada de paridad, RE-9):** el
`cambiar-estado` viejo **nunca** setea `fecha_finalizacion`, ni siquiera cuando
`nuevo_estado == FINALIZADA`. El port **sí** setea `fecha_finalizacion` cuando la gestión pasa a
`FINALIZADA` (y la limpia si sale de `FINALIZADA`). El ETL backfillea `fecha_finalizacion` de las
gestiones ya finalizadas desde la fecha del último evento `CAMBIO_ESTADO` a `FINALIZADA`.

### 3.6 Control de concurrencia

El `UPDATE ... WHERE fecha_ingreso = @old_fecha_ingreso` actual es un lock optimista pobre
(y frágil: `fecha_ingreso` es editable). Se reemplaza por optimista sobre `updated_at`
(o `xmin`): el `PATCH`/`cambiar-estado` envía el `updated_at` que leyó; si no coincide → `409
CONFLICTO_ESTADO`. Cambio de contrato menor — coordinar con frontend (§6).

### 3.7 Auth y roles: todo al portal (ADR-015)

| Sistema viejo | Destino |
|---|---|
| Tabla `usuarios_roles` en BigQuery (email, nombre, rol, activo) | `portal_usuarios` (Cloud SQL, módulo `portal` de svc-vivienda) con `"privada"` en `secretarias` |
| Roles `Admin` / `Supervisor` / `Operador` / `Consulta` | **1:1** con los roles del portal (mismos nombres) — D-3 |
| `usuario_modulos` / `cat_modulos` (permiso por módulo) | Se descarta. El acceso a Privada se expresa como la secretaría `"privada"` asignada al usuario. **Condición D-3**: si el relevamiento revela usuarios con acceso a *sólo una parte* del sistema, se agrega un rol acotado estilo `TecnicoDGV` (enmienda a ADR-015) |
| `require_roles(...)` local (lee de BigQuery) | `Depends(get_current_user)` estándar + chequeo de secretaría, con las tuplas `ROLES_LECTURA`/`ROLES_ESCRITURA`/`ROLES_ELIMINACION` de `app/auth.py` |
| Doble auth: Firebase JWT (`X-Forwarded-Authorization`, aud `gestorcooperativo`) **o** Google OAuth2 directo (Vanilla JS legacy) | Sólo el flujo estándar del gateway (`X-Forwarded-Authorization`). Se elimina el path OAuth2 directo |
| `GET /api/v1/privada/me` | **Se mantiene** como alias de `GET /api/v1/portal/me` durante v1 (evita tocar `src/modules/privada/api/gestiones.api.ts` en el cutover). Se retira en v2 |
| `/api/v1/privada/usuarios/**` (CRUD usuarios) | **Se elimina.** La gestión de usuarios de Privada pasa a `AdminUsuariosPage` del portal (`/api/v1/portal/admin/usuarios`) |

`"privada"` se agrega como secretaría válida en el portal (backend `portal/schemas.py`
`ROLES_VALIDOS`/lista de secretarías + frontend `DashboardPage.tsx` `SECRETARIAS` y
`AdminUsuariosPage`). Ya existe la entrada de módulo frontend `src/modules/privada/` con
`activa:true` de facto (tiene páginas), sólo hay que formalizar el permiso.

`svc-privada` lee `portal_usuarios` (que vive en `db_vivienda`) llamando
`GET /internal/portal/usuarios/{email}` en `svc-vivienda` — endpoint IAM-only, **nunca** conexión
directa cross-DB (ADR-015, §7).

### 3.8 El tab "Tablero" (Looker Studio) — resuelto: nativo (ADR-014, D-4)

Hoy `TableroPage.tsx` embebe un informe Looker Studio que lee BigQuery `infra_gestion`
directamente. Al apagar BigQuery se rompe. **Decisión (D-4 / ADR-014): opción A — reconstruir
nativo** en React sobre `/api/v1/privada/informe/**`, sin mirror de BigQuery. El Tablero nativo es
un spec hijo (`spec-privada-tablero.md`) y es **prerequisito del decommission** (apagar BigQuery),
**no** del cutover.

Alternativas rechazadas: **B** — mirror BQ sólo-lectura (job Cloud Scheduler → dataset BQ mínimo en
`gestorcooperativo`, Looker reconectado): arrastra un pipeline nuevo permanente. **C** — descartar
el tab: es funcionalidad activa del área.

### 3.9 Habilita la federación server-side de `resumen_territorial` (ADR-016)

Con Privada en el mismo proyecto GCP, el SA de svc-vivienda puede invocar el Cloud Run de
svc-privada (IAM `run.invoker`) y svc-privada expone un rollup global por
(departamento, localidad) sin `departamento` obligatorio. Eso destraba el "plan A" descrito y
descartado en `spec-resumen-territorial.md §3.3`. **v1 de esta migración entrega el endpoint
`GET /api/v1/privada/gestiones/rollup-territorial`** (obligatorio, §6). El *switch* de
`resumen_territorial` de federación-en-el-browser a federación server-side es
**ADR-016** y se implementa en `spec-resumen-territorial-ficha-localidad.md`, no acá.

### 3.10 Frontend GitHub Pages legacy: se retira

El frontend Vanilla JS en `labotech-analytics.github.io` ya fue reemplazado por
`src/modules/privada/` en el portal. Tras el cutover se despublica (o queda con un aviso de
redirección). El CORS `allow_origins` del backend viejo deja de importar.

### 3.11 Las columnas legacy de clasificación se conservan verbatim

La migración de `priv_gestiones` **conserva sin renombrar ni dropear** las columnas
`categoria_general_id`, `tipo_gestion`, `canal_origen` (y las columnas muertas `subcategoria_id`,
`tipo_demanda_principal_id`). El spec hijo `spec-privada-categorias-programas.md` agrega
`categoria_id`/`programa_id`/`area_id` (FKs nullable a los catálogos nuevos) y **backfillea** desde
`categoria_general_id`; recién después de confirmar paridad del informe (RE-1) se congela/retira la
columna legacy. v1 no toca este terreno.

### 3.12 `priv_departamentos_info` recibe endpoint de lectura en v1

El sistema viejo expone `GET /localidades-info` pero no un GET análogo de departamento. v1 agrega
`GET /api/v1/privada/departamentos-info?departamento=` (read-only) — la ficha de localidad
(`spec-resumen-territorial-ficha-localidad.md`) lo necesita, la tabla ya migra, es aditivo y de
bajo riesgo. `priv_departamentos_info` se mantiene read-only desde la app (carga por Excel).

---

## 4. Modelo de datos destino (`db_privada`, prefijo `priv_`)

> DDL definitiva en la migración Alembic `0001`. Tipos orientativos.

### 4.1 `priv_gestiones`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | UUID | PK, `gen_random_uuid()` |
| `id_legacy` | VARCHAR | UNIQUE, `id_gestion` de BigQuery |
| `nro_expediente` | VARCHAR | nullable |
| `origen` | VARCHAR | nullable |
| `estado` | VARCHAR | NOT NULL, `CHECK` en los 6 valores; default `INGRESADO` |
| `fecha_ingreso` | DATE | NOT NULL |
| `fecha_estado` | TIMESTAMPTZ | marca del último cambio de estado |
| `fecha_finalizacion` | DATE | nullable |
| `urgencia` | VARCHAR | `Alta` / `Media` / `Baja`; default `Media` |
| `ministerio_agencia_id` | VARCHAR | FK lógica → `priv_cat_ministerio_agencia` |
| `organismo_id` | VARCHAR | nullable |
| `derivado_a_id` | VARCHAR | nullable |
| `categoria_general_id` | VARCHAR | FK lógica → `priv_cat_categoria_general` |
| `subcategoria_id` | VARCHAR | nullable |
| `tipo_demanda_principal_id` | VARCHAR | nullable |
| `subtipo_detalle` | VARCHAR | nullable |
| `detalle` | TEXT | NOT NULL |
| `observaciones` | TEXT | nullable |
| `geo_id` | VARCHAR | nullable |
| `departamento` | VARCHAR | NOT NULL |
| `localidad` | VARCHAR | NOT NULL |
| `direccion` | VARCHAR | nullable |
| `lat` / `lon` | NUMERIC | nullable |
| `costo_estimado` | NUMERIC(16,2) | nullable |
| `costo_moneda` | VARCHAR | nullable |
| `tipo_gestion` | VARCHAR | nullable (`TG_*`) |
| `canal_origen` | VARCHAR | nullable (`CO_*`) |
| `created_at` / `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() |
| `created_by` / `updated_by` | VARCHAR | email del actor |
| `deleted_at` | TIMESTAMPTZ | NULL — soft delete (ADR-005), reemplaza `is_deleted BOOL` |

Índices: `(departamento, localidad)`, `(estado)`, `(fecha_ingreso DESC)`, `(nro_expediente)`,
GIN/trigram para el buscador `q` si hace falta.

> `categoria_general_id`, `subcategoria_id`, `tipo_demanda_principal_id`, `tipo_gestion`,
> `canal_origen` se migran **verbatim** (§3.11). Las columnas de mejora (`categoria_id`,
> `programa_id`, `area_id`, `ok_gobernador`, `ok_ministro`) **no** las crea la migración `0001` —
> las agregan las migraciones `0002+` de los specs hijos, con `DEFAULT` para no requerir backfill
> en el mismo release.

### 4.2 `priv_gestiones_eventos`

| Columna | Tipo |
|---|---|
| `id` | UUID PK |
| `gestion_id` | UUID FK → `priv_gestiones.id` |
| `fecha_evento` | TIMESTAMPTZ |
| `usuario` | VARCHAR (email) |
| `rol_usuario` | VARCHAR |
| `tipo_evento` | VARCHAR (`ALTA` / `CAMBIO_ESTADO` / `EDICION` / `BAJA`) |
| `estado_anterior` / `estado_nuevo` | VARCHAR nullable |
| `campo_modificado` / `valor_anterior` / `valor_nuevo` | VARCHAR/TEXT nullable |
| `comentario` | TEXT nullable |
| `metadata_json` | JSONB nullable |

### 4.3 `priv_localidades_info` / `priv_departamentos_info`

Mismas columnas que hoy (habitantes, electores, intendente/legisladores, partido político,
`color_semaforo`, `updated_at`, `updated_by`). PK: `(departamento, localidad)` y `(departamento)`
respectivamente. Upsert (hoy `MERGE`) → `INSERT ... ON CONFLICT DO UPDATE`.

### 4.4 `priv_geo_localidades`

`id_geo`, `departamento`, `localidad`, `lat`, `lon`, `activo`. Sólo lectura desde la app.

### 4.5 Catálogos `priv_cat_*`

`priv_cat_estado`, `priv_cat_urgencia`, `priv_cat_ministerio_agencia`,
`priv_cat_categoria_general`, `priv_cat_tipo_gestion`, `priv_cat_canal_origen`.
Columnas: `id VARCHAR PK`, `nombre`, `orden INT`, `activo BOOL`, `descripcion`.
Se pueden migrar como tablas o como seed en la migración; se migran como **tablas** (el sistema
viejo permite altas de catálogo vía BigQuery). Sin CRUD de catálogos en v1 salvo el ya existente.

> `priv_cat_categoria_general` es el catálogo **legacy**. El spec hijo
> `spec-privada-categorias-programas.md` introduce `priv_categorias` (catálogo editable en runtime,
> patrón `viv_cc_estados`) que lo supersede para la clasificación; la tabla legacy se conserva
> durante la ventana de compatibilidad del informe (RE-1).

### 4.6 Vista / agregación de informe

`v_informe_cooperativas` de BigQuery clasifica cada gestión en un `tema_informe` mediante un
`CASE WHEN` de **10 prioridades** sobre `categoria_general_id` + `REGEXP_CONTAINS(LOWER(detalle), …)`
(DDL disponible: `proyecto_sistema_gestiones/informe/bq_views/v_informe_cooperativas.sql` — ver
**Anexo A**). Temas: `Cordón Cuneta + Adoquinado`, `Kits Solares`, `Luces LED`, `Gas`,
`Bombeo Solar`, `Vivienda`, `Lotes`, `Infraestructura Eléctrica`, `Préstamos y Fortalecimiento`,
`Otras Obras`, else `NULL`.

**v1 porta esa lógica TAL CUAL** (regex incluido), como función pura en
`app/privada/informe_service.py` (patrón `informes/aggregations.py`; 523 filas, cómputo trivial en
request) o como vista SQL `priv_v_informe_cooperativas`. **No** se re-apunta a un campo estructurado
en la migración-paridad — eso es `spec-privada-informe-cooperativas-v2.md` (mueve la clasificación a
`categoria_id` con un mapa de compatibilidad nueva-categoría→`tema_informe` y sign-off del área;
RE-1).

### 4.7 Sin tabla de usuarios

`usuarios_roles`, `usuarios_eventos`, `usuario_modulos`, `cat_modulos` **no se migran**
(ver §3.7). Se exporta su contenido a un CSV de referencia para el alta manual/scripted en
`portal_usuarios`.

---

## 5. Mapeo BigQuery → PostgreSQL

| Tabla BigQuery (`infra_gestion.*`) | Tabla PostgreSQL (`db_privada`) | Conversión |
|---|---|---|
| `gestiones` | `priv_gestiones` | `id_gestion`→`id_legacy` (+UUID nuevo); `is_deleted BOOL`→`deleted_at TIMESTAMPTZ` (`TRUE`→`now()` del último evento de baja, o `now()`); `TIMESTAMP`→`TIMESTAMPTZ` (UTC); `NUMERIC`→`NUMERIC(16,2)` |
| `gestiones_eventos` | `priv_gestiones_eventos` | `id_gestion`→`gestion_id` (resuelto contra el UUID nuevo por `id_legacy`); `metadata_json` STRING/JSON→`JSONB` |
| `localidades_info` | `priv_localidades_info` | directo |
| `departamentos_info` | `priv_departamentos_info` | directo |
| `geo_localidades` | `priv_geo_localidades` | `lat_centro`/`lon_centro`→`lat`/`lon` |
| `cat_estado` | `priv_cat_estado` | directo |
| `cat_urgencia` | `priv_cat_urgencia` | directo |
| `cat_ministerio_agencia` | `priv_cat_ministerio_agencia` | directo |
| `cat_categoria_general` | `priv_cat_categoria_general` | directo |
| `cat_tipo_gestion` | `priv_cat_tipo_gestion` | directo |
| `cat_canal_origen` | `priv_cat_canal_origen` | directo |
| `usuarios_roles` | — (a `portal_usuarios`, manual/scripted) | export CSV de referencia |
| `usuarios_eventos` | — (descartado; audit del portal cubre lo nuevo) | export CSV de archivo |
| `usuario_modulos`, `cat_modulos` | — (descartado) | — |
| vista `v_informe_cooperativas` | lógica en `informe_service.py` | extraer DDL con `bq show --view` |

Notas de conversión:
- Fechas: BigQuery `DATE` → `DATE`; `TIMESTAMP` (UTC) → `TIMESTAMPTZ`.
- `NUMERIC` de BQ (lat/lon, costos) → `NUMERIC` de Postgres, sin pérdida.
- Strings vacíos `''` en columnas nullable → `NULL` (normalización).
- `metadata_json`: si viene como STRING con JSON, `PARSE_JSON` → `JSONB`; si ya es JSON, directo.

---

## 6. Endpoints — matriz de compatibilidad

Prefijo `/api/v1/privada`. "=" ⇒ contrato idéntico; el frontend no cambia.

| Endpoint | Roles | v1 | Nota |
|---|---|---|---|
| `GET /me` | todos | ⚠️ alias | Se mantiene como alias de `/api/v1/portal/me` en v1; se retira en v2 |
| `GET /gestiones` / `GET /gestiones/` | Lectura | = | paginado `limit`/`offset` + filtros `q, estado, ministerio, categoria, tipo_gestion, canal_origen, departamento, localidad` |
| `GET /gestiones/{id}` | Lectura | = | acepta UUID o `id_legacy` en v1 (§3.3) |
| `GET /gestiones/{id}/eventos` | Lectura | = | historial desc |
| `POST /gestiones` / `POST /gestiones/` | Escritura | = | `201`; genera evento `ALTA` |
| `POST /gestiones/{id}/cambiar-estado` | Escritura | ✏️ | + campo `updated_at` para lock optimista (§3.6); sin validación de transición (§3.5) |
| `PATCH /gestiones/{id}` | Escritura | ➕ v1 | edición de campos separada del cambio de estado (hoy van juntos en `cambiar-estado`); mismo lock optimista `updated_at` |
| `DELETE /gestiones/{id}` | Eliminación (`Admin`,`Supervisor`) | = | soft delete → `deleted_at` |
| `GET /gestiones/resumen-territorial` | Lectura | = | `departamento` obligatorio (se mantiene el comportamiento actual) |
| `GET /gestiones/rollup-territorial` | Lectura | ➕ v1 | rollup global por (depto, localidad) sin `departamento` obligatorio — lo consume ADR-016 (§3.9) |
| `GET /localidades-info` / `PUT /localidades-info` | Lectura / Escritura | = | upsert (el `PUT` sólo escribe `habitantes`/`electores`/`intendente_jefe_comunal`/`partido_politico` — igual que hoy) |
| `GET /departamentos-info` | Lectura | ➕ v1 | read-only; `?departamento=` (§3.12) |
| `GET /catalogos/{catalogo}` | todos | = | `estados, urgencias, ministerios, categorias, tipos-gestion, canales-origen, departamentos, localidades?departamento=, geo` |
| `GET /informe/cooperativas/resumen` | Lectura | = | KPIs por tema |
| `GET /informe/cooperativas/temporal` | Lectura | = | evolución mensual |
| `GET /informe/cooperativas/por-departamento` | Lectura | = | tema × departamento |
| `GET /informe/cooperativas/puntos` | Lectura | = | puntos lat/lon para Leaflet |
| `GET/POST/PUT/DELETE /usuarios/**` | Admin | ❌ eliminado | pasa a `/api/v1/portal/admin/usuarios` (§3.7) |
| `GET/POST/DELETE /usuarios/{email}/modulos/**` | Admin | ❌ eliminado | mecanismo descartado |
| `GET /catalogos/modulos` | todos | ❌ eliminado | — |
| `POST /internal/privada/actualizar-informe` | IAM | ➕ opcional | sólo si el informe se cachea con snapshot; ver §4.6 |
| `GET/POST/PATCH/DELETE /categorias` | Lectura / `ROLES_TRANSICION` | ➕ spec hijo | `spec-privada-categorias-programas.md` |
| `GET/POST/PATCH/DELETE /programas` | Lectura / `ROLES_TRANSICION` | ➕ spec hijo | ídem; **`POST` existe** (a diferencia de `viv_programas`) |
| `GET/POST/PATCH/DELETE /areas` | Lectura / `ROLES_TRANSICION` | ➕ spec hijo | ídem; comparte tabla con el DAG (ADR-013) |
| `GET /gestiones/{id}/flujo` | Lectura | ➕ spec hijo | `spec-privada-flujo-derivaciones.md` — nodos/aristas/actual del DAG |

En `POST /gestiones` y `PATCH /gestiones/{id}` los specs hijos agregan (aditivo, no-breaking):
`categoria_id`, `programa_id`, `area_id`, `ok_gobernador`, `ok_ministro`, `derivado_a`,
`acciones_implementadas`. `cambiar-estado` (spec hijo): además de `updated_at`, escribe una fila
`priv_gestion_derivaciones` y persiste `acciones_implementadas` en la gestión (el sistema viejo sólo
lo guardaba en `metadata_json` — RE-10).

**Cambios en `src/modules/privada/` (frontend)**:
- **v1 (cutover)**: `api/gestiones.api.ts` envía `updated_at` en `cambiarEstado` y maneja `409`;
  se quita todo uso de `/api/v1/privada/usuarios/**` y `catalogos/modulos`; permisos vía
  `usePortalUser` + secretaría `"privada"` en vez de `me.rol` propio.
- **specs hijos (post-cutover)**: modal(es) "Gestionar categorías/programas/áreas" (copia de
  `GestionarEstadosModal` de `CordonCunetaPage.tsx`), 3 desplegables con "+ nueva opción" inline en
  el alta/edición, inputs `derivado_a`/`acciones_implementadas` en `CambiarEstadoModal`, vista
  "Flujo" (DAG) en `GestionDetalleDrawer` junto al timeline actual, y `TableroPage.tsx` nativo.

---

## 7. Auth y permisos — detalle

- `app/auth.py` de `svc-privada` = copia del de svc-vivienda: valida Firebase JWT desde
  `X-Forwarded-Authorization`, luego `SELECT` en `portal_usuarios` (**cross-service read**: ver
  §11 riesgo R-4) para rol + `secretarias`. Fallback `invitado` en cualquier error.
- Acceso al módulo: el usuario debe tener `"privada"` en `secretarias` (o rol `Admin`).
  Cada router declara el chequeo; se reutilizan `ROLES_LECTURA`/`ROLES_ESCRITURA`/
  `ROLES_ELIMINACION` (Privada mapea 1:1, así que **sí** entran en las tuplas compartidas, a
  diferencia de `TecnicoDGV`/`Autoridad`).
- `portal_usuarios` vive en `db_vivienda`. **Decisión (ADR-015): opción (b)** — `svc-vivienda`
  expone `GET /internal/portal/usuarios/{email}` (router `app/internal/`, sin prefijo `/api/v1`,
  IAM-only) y `svc-privada` lo consulta con el ID token de su SA (`roles/run.invoker` sobre
  `svc-vivienda`), cacheando por request. **Nunca** conexión directa a `db_vivienda` (rechaza (a));
  no se mueve `portal` a lib/servicio compartido ahora (rechaza (c)).

---

## 8. Migración de datos y cutover

### 8.1 ETL (one-shot, repetible)

Script `services/svc-privada/scripts/migrar_desde_bigquery.py`:
1. Lee cada tabla de `infra_gestion` con el cliente BigQuery (SA con `bigquery.dataViewer` en
   `essential-haiku-482815-u4`).
2. Transforma según §5 (normaliza fechas, `''`→`NULL`, `is_deleted`→`deleted_at`, genera UUIDs
   y el mapa `id_legacy`→`id`).
3. Inserta en `db_privada` vía la instancia (proxy), en orden: catálogos → geo →
   localidades/departamentos_info → gestiones → gestiones_eventos.
4. Verificación: conteo por tabla + `SUM`/`MIN`/`MAX` de columnas clave (fechas, costos) BQ vs
   PG; reporte de diffs.
5. Idempotente: `TRUNCATE` + recarga, o `ON CONFLICT DO NOTHING` con clave `id_legacy`.

### 8.2 Cutover

1. **T-7d**: infra lista (db, secreto, Cloud Run desplegado, gateway config nueva **creada
   pero no activada**), ETL corrido contra datos de prod → validación funcional del nuevo
   backend vía una URL directa de Cloud Run (no gateway).
2. **T-0 (ventana de mantenimiento, ~30 min)**:
   a. Poner el sistema viejo en **sólo-lectura** (feature flag / quitar permisos de escritura
      en `usuarios_roles`, o bajar el Cloud Run viejo a 0 y avisar).
   b. Correr el ETL final (delta desde el último corte).
   c. `gcloud api-gateway gateways update` → apuntar a `ministerio-config-v{YYYYMMDD}` (paths
      `/api/v1/privada/**` → nuevo Cloud Run).
   d. Alta/actualización de los usuarios de Privada en `portal_usuarios` con secretaría
      `"privada"` (script a partir del CSV de `usuarios_roles`).
   e. Smoke test end-to-end desde el portal (login → lista → detalle → cambiar estado → informe).
3. **T+1..T+30d**: sistema viejo apagado pero **conservado** (BigQuery intacto) para rollback.
4. **T+30d**: si todo ok, §10 (desmantelamiento).

### 8.3 Rollback

Mientras BigQuery siga intacto y el Cloud Run viejo exista: revertir el `gateway update` a
`ministerio-config-v20260716b` (o la previa vigente) y reactivar escrituras en el sistema
viejo. Ventana de rollback barato = hasta T+30d. Las escrituras hechas en el sistema nuevo
durante ese lapso habría que re-exportarlas manualmente (bajo volumen esperado).

---

## 9. Infra / deploy

- `infra/cloudsql-setup.sh`: agregar `privada` a la lista de servicios (crea `db_privada`,
  `user_privada`, secreto `svc-privada-db-url`).
- `services/svc-privada/`: `Dockerfile`, `pyproject.toml`, `alembic/`, `app/`, `tests/` — mismo
  esqueleto que svc-vivienda (ver `docs/files/arquitectura.md` §"Estructura de directorios").
- `services/cloudbuild.yaml`: nuevo paso de build+deploy para `svc-privada` (trigger ya es
  push a `services/`).
- IAM: SA de `svc-privada` con `roles/cloudsql.client`; SA del gateway con `run.invoker` sobre
  el nuevo Cloud Run; SA del ETL con `bigquery.dataViewer` en el proyecto viejo (temporal).
- Gateway: `infra/gateway/openapi.yaml` — los 12 paths `/api/v1/privada/**` ya existen; sólo
  cambia `x-google-backend.address`. Quitar los paths `/usuarios/**` y `/catalogos/modulos`.
  Nueva config `ministerio-config-v{YYYYMMDD}` + `gateway update` (configs inmutables, ADR-002).
- Seguir el skill `/deploy-servicio` paso a paso.
- Migraciones contra prod: desde Cloud Shell con `cloud-sql-proxy`, desde
  `services/svc-privada/`, nunca desde la raíz.

---

## 10. Desmantelamiento del sistema viejo (post T+30d)

- [ ] **Tablero React nativo en producción** (ADR-014 / `spec-privada-tablero.md`) — gate del
      apagado de BigQuery: el frontend no debe tener ninguna lectura de BQ.
- [ ] Exportar `infra_gestion` completo a GCS (backup frío, retención 1 año).
- [ ] Apagar el Cloud Run `infraestructura-gestioninterna` en `essential-haiku-482815-u4`.
- [ ] Despublicar el frontend GitHub Pages `labotech-analytics.github.io/SistemaGestiones_*`
      (o dejar redirect al portal).
- [ ] Retirar el informe Looker Studio `f9dc4a4e-a174-45a8-938c-385f4121f689` (reemplazado por el
      Tablero nativo).
- [ ] OAuth Client ID `354063050046-fkp06ao8...`: revocar.
- [ ] ADR-008, ADR-014, ADR-015, ADR-016 registrados en `docs/files/arquitectura.md` (hecho
      2026-08-31; verificar que reflejan el estado final).
- [ ] Actualizar `docs/files/CLAUDE.md` y `CLAUDE.md` raíz (sección *Services / status* y ADR-006),
      `docs/files/roadmap.md`, `docs/context/areas/Privada Ministro/arquitectura_actual.md`.
- [ ] Borrar el proyecto `essential-haiku-482815-u4` (o dejarlo suspendido) una vez confirmado
      el backup.

---

## 11. Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| R-1 | Lógica de `v_informe_cooperativas` (clasificación en temas) no está en el repo | Extraer el DDL de la vista con `bq show --view` **antes** de aprobar; incluirlo como anexo del spec |
| R-2 | Respuestas no byte-compatibles rompen el frontend en silencio | Tests de contrato: capturar respuestas reales del sistema viejo y compararlas contra el nuevo por endpoint |
| R-3 | `id_gestion` string embebido en URLs/bookmarks/estado del frontend | `id_legacy` + endpoint que acepta ambos en v1 |
| R-4 | `svc-privada` leyendo `portal_usuarios` de `db_vivienda` acopla servicios | Endpoint interno IAM en svc-vivienda (§7 opción b), no conexión directa a la otra base |
| R-5 | Divergencia de datos durante la ventana de cutover | Ventana corta + sistema viejo en sólo-lectura + ETL delta final |
| R-6 | Tablero Looker se rompe al apagar BigQuery | Decidir §3.8 antes del cutover; si es opción A, el tablero nativo es prerequisito del T-0 |
| R-7 | Roles/campos del área no cubiertos por el modelo actual | Reunión §12 antes de `approved` (ya estaba pendiente en `arquitectura_actual.md`) |
| R-8 | Migración marcada como aplicada en Cloud Shell con código incompleto | Commit del código de la migración **antes** de `alembic upgrade` en prod (lección registrada en memoria del proyecto) |
| R-9 | Pérdida del histórico `usuarios_eventos` al descartarlo | Export CSV a `docs/` o a GCS antes de apagar |

Los riesgos específicos de las mejoras (RE-1..RE-11: reclasificación del informe, backfill de
derivaciones sin taxonomía de áreas, sprawl de `priv_programas`, `fecha_finalizacion`, flip de
federación de Resumen Territorial, staleness de `color_semaforo`, etc.) se rastrean en los specs
hijos, no acá.

---

## 12. Decisiones

### Resueltas (2026-08-31)

| # | Decisión |
|---|---|
| Roles 1:1 | Confirmado como hipótesis (D-3). Si el relevamiento revela lo contrario → enmienda a ADR-015 (rol acotado tipo `TecnicoDGV`). |
| `usuario_modulos` | Se descarta; acceso = secretaría `"privada"` (D-3). Sujeto a la misma condición. |
| Tablero | Nativo en React (D-4 / ADR-014). Sin mirror BQ. |
| `GET /me` | Se mantiene como alias de `/portal/me` en v1; se retira en v2. |
| `PATCH /gestiones/{id}` | Se expone separado de `cambiar-estado` en v1 (§6). |
| Instancia Cloud SQL | `db_privada` en la instancia compartida `ministerio-postgres` (§3.2). |
| Frontend GitHub Pages | Baja con redirect al portal (§3.10). |

### A confirmar en la reunión de relevamiento con Secretaría Privada (no bloquea Fases 0–6)

1. **Ventana de mantenimiento**: fecha/hora del cutover; quién valida el smoke test.
2. **Retención del proyecto viejo**: ¿suspender o borrar tras el backup? ¿owner del billing hoy?
3. **`usuario_modulos`**: confirmar que ningún usuario tiene acceso módulo-parcial (condición de D-3).

### En los specs hijos (no en este spec)

Mapa/valores de las categorías, taxonomía de áreas para el DAG, semántica exacta de
`Ok Gobernador`/`Ok Ministro`, si `tema_informe` adopta las categorías nuevas, edición de
`departamentos_info`, aceptación de la reclasificación del informe (RE-1). Defaults propuestos en
`agente Plan` / plan de trabajo.

---

## 13. Criterios de aceptación

- [ ] `services/svc-privada/` desplegado en Cloud Run (`gestorcooperativo`), `GET /health` OK.
- [ ] `db_privada` con schema `priv_*` vía Alembic `0001`; `alembic current` = head en prod.
- [ ] ETL migra las 12 tablas; verificación de conteos e importes BQ vs PG sin diffs.
- [ ] Todos los endpoints del §6 marcados "=" responden byte-compatibles con el sistema viejo
      (suite de tests de contrato en verde).
- [ ] Usuarios de Privada en `portal_usuarios` con secretaría `"privada"`; login desde el portal
      llega a `GestionesListPage` con permisos correctos por rol.
- [ ] `cambiar-estado` registra evento en `priv_gestiones_eventos` en la misma transacción;
      `409` ante `updated_at` desactualizado.
- [ ] Soft delete: `DELETE` marca `deleted_at`, los listados lo excluyen.
- [ ] Informe de Cooperativas: los 4 endpoints devuelven los mismos totales que el Looker/BQ
      para un rango de fechas de control.
- [ ] Gateway `ministerio-config-v{YYYYMMDD}` activa; `/api/v1/privada/**` resuelve al nuevo
      backend; paths `/usuarios/**` y `/catalogos/modulos` retirados.
- [ ] Endpoints `GET /gestiones/rollup-territorial` y `GET /departamentos-info` disponibles
      (aunque `resumen_territorial` todavía no los consuma).
- [ ] Columnas legacy `categoria_general_id`, `tipo_gestion`, `canal_origen` presentes y pobladas
      en `priv_gestiones` (insumo del backfill de los specs hijos).
- [ ] `estado = FINALIZADA` setea `fecha_finalizacion`; ETL backfilleó las gestiones ya finalizadas.
- [ ] `GET /internal/portal/usuarios/{email}` operativo en `svc-vivienda` (IAM-only) y consumido
      por `svc-privada`.
- [ ] Tests backend con cobertura > 80% (`pytest` en `services/svc-privada/`).
- [ ] Plan de rollback documentado y probado en staging/local (revertir `gateway update`).
- [ ] ADR-008/014/015/016 registrados; `CLAUDE.md`, `roadmap.md` y `arquitectura_actual.md`
      actualizados (en el decommission).
- [ ] Backup frío de `infra_gestion` en GCS antes de apagar nada.

---

## 14. Plan de trabajo por fases

Fasado completo (Fases 0–8 de migración + E1–E5 de mejoras) en el plan de trabajo del proyecto.
Resumen de la migración-paridad:

| Fase | Contenido | Prerequisito |
|---|---|---|
| 0 | ADR-008..016 + este spec `approved`; reunión de relevamiento; Anexos A–G | — |
| 1 | Scaffold `services/svc-privada/`; `db_privada` en infra; secreto; Alembic `0001` (schema `priv_*`) | Fase 0 |
| 2 | Portar routers/servicios: gestiones (CRUD + `PATCH` + eventos + `cambiar-estado`), catálogos, localidades-info, `departamentos-info`, informe (4), `rollup-territorial` | Fase 1 |
| 3 | Auth: `app/auth.py` de svc-privada + `GET /internal/portal/usuarios/{email}` en svc-vivienda; `"privada"` en `ROLES_VALIDOS`/`SECRETARIAS`/`AdminUsuariosPage` | Fase 1 |
| 4 | ETL `migrar_desde_bigquery.py` + verificación (conteos + `SUM`/`MIN`/`MAX`) + backfill `fecha_finalizacion` | Fase 1 |
| 5 | Tests de contrato (viejo vs nuevo, Anexo D) + unitarios > 80% | Fases 2–4 |
| 6 | Deploy Cloud Run + gateway config nueva (creada, no activada); validación por URL directa; rollback ensayado | Fases 2–5 |
| 7 | Frontend v1: `updated_at`/`409`, quitar `/usuarios/**` y `modulos`, permisos vía `usePortalUser` | Fase 6 |
| 8 | Cutover (§8.2): sólo-lectura viejo → ETL delta → `gateway update` → alta usuarios → smoke test | Fases 6–7 |
| — | (post-cutover) Tablero nativo → gate del decommission; specs hijos E1–E5 | Fase 8 |
| — | Ventana de rollback T+30d → Desmantelamiento (§10) + actualización de docs | — |

---

## Anexos

- **Anexo A**: DDL de `v_informe_cooperativas` — disponible en
  `proyecto_sistema_gestiones/informe/bq_views/v_informe_cooperativas.sql` (10 prioridades, regex
  sobre `LOWER(detalle)`). Copiar al spec al arrancar la Fase 2.
- **Anexo B**: dump de `cat_*` (valores actuales de catálogos) para el seed de `priv_cat_*` —
  `bq extract` o `SELECT *` por tabla.
- **Anexo C**: export de `usuarios_roles` (para el alta en `portal_usuarios`) + export de
  `usuarios_eventos` (backup de archivo, R-9).
- **Anexo D**: capturas de respuestas del sistema viejo por endpoint (fixtures de los tests de
  contrato, R-2). Capturar durante la Fase 5 contra el sistema viejo en producción.
- **Anexo E** *(insumo de `spec-privada-categorias-programas.md`)*: valores, `orden` y colores de
  las 9 categorías pedidas + sugerencia inicial de programa/área por categoría. A completar con el
  área.
- **Anexo F** *(insumo de `spec-privada-flujo-derivaciones.md`)*: taxonomía de áreas (`priv_areas`)
  — lista curada + valores distintos observados en `derivado_a`/`organismo_id` + los 17
  `cat_ministerio_agencia`, con tabla de alias. A construir con un `SELECT DISTINCT` sobre
  `gestiones` + curado del área.
- **Anexo G** *(insumo del parser de backfill de derivaciones)*: payloads de muestra de
  `gestiones_eventos.metadata_json` para los 4 `tipo_evento` (`CREACION`, `CAMBIO_ESTADO`,
  `ACTUALIZA_DATO`, `ARCHIVO`).
