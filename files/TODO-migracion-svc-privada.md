# TODO — Migración svc-privada al monorepo + mejoras

Checklist vivo del esfuerzo. Marcar `[x]` a medida que se completa cada paso.
Specs: `spec-migracion-svc-privada.md` (approved v1.0.0) + los 5 specs hijos (`draft`).
ADRs: ADR-008..ADR-016 en `arquitectura.md`. Fasado completo en `roadmap.md` (Etapa 2-bis).

> **Regla:** el contrato `/api/v1/privada/**` y `frontend/src/modules/privada/` deben seguir
> operativos en todo momento hasta el cutover. Nada de esto se aplica contra producción sin
> pasar por Cloud Shell + `cloud-sql-proxy` desde `services/svc-privada/`.

---

## Fase 0 — Decisiones y specs ✅ (2026-08-31)

- [x] ADR-008..ADR-016 escritos en `docs/files/arquitectura.md`
- [x] `spec-migracion-svc-privada.md` promovido a `approved` v1.0.0 con deltas §0–§14
- [x] 5 specs hijos creados en `draft`
- [x] `roadmap.md` actualizado (Etapa 2 ✅ + Etapa 2-bis)
- [ ] Reunión de relevamiento con Secretaría Privada — ventana de cutover, retención del proyecto
      viejo, revisión del alta de usuarios (17 en `usuarios_roles`, 4 gateway/test con `Admin`)
- [x] **Anexo A** — `A_v_informe_cooperativas.sql` (informe de Cooperativas, 10 temas por regex —
      NO transversal)
- [x] **Anexo A2** — `A2_resumen_territorial.sql` (Resumen Territorial: no hay vista; replicar el
      patrón de `resumen_territorial` de svc-vivienda) + `A2_rollup_territorial_baseline` (ETL)
- [x] **Anexo B** — `B_cat_*.json` (6 catálogos; `cat_estado.id == nombre`, orden 10..60)
- [x] **Anexo C** — `C_usuarios_roles.csv` (17 filas), `C_usuario_modulos.csv` **vacío** → D-3
      confirmado (nadie con acceso parcial), `C_usuarios_eventos.json`
- [ ] **Anexo D** — BLOQUEADO: (a) necesita token Firebase; (b) **bug**: los 4
      `/api/v1/privada/informe/cooperativas/*` no están montados con el prefijo `/api/v1/privada`
      en el `main.py` viejo → 404 por el gateway. El frontend nuevo no los usa (Looker iframe).
      Capturar los demás endpoints igual; los de informe se validan directo contra el Cloud Run
      viejo (`/informe/cooperativas/*` legacy) o se difieren al Tablero nativo.
- [x] **Anexo F** — `F_derivado_a_distinct.json` (~27 valores, cola larga),
      `F_organismo_id_distinct.json`, `F_cat_ministerio_agencia.json`. **`F_metadata_derivado_a` =
      `[]`** → no hay traza histórica de derivaciones (afecta E3, ver `spec-privada-flujo-derivaciones.md`)
- [x] **Anexo G** — `G_por_tipo_evento.json`, `G_muestras.json`. Sólo **166 eventos** / 2123
      gestiones; `metadata_json.derivado_a` siempre null
- [x] Línea base ETL — `ETL_baseline.json` (2123 gestiones / 1987 activas / 110 finalizadas),
      `ETL_fecha_finalizacion_gap.json` (**103/103** FINALIZADA sin fecha → RE-9 100%),
      `schema_*.json` (14 tablas)
- [x] `scripts/generar_anexos.sh` — script empaquetado y re-ejecutable (delta del cutover)

## Fase 1 — Scaffold + schema

- [x] `services/svc-privada/` — estructura (Dockerfile, pyproject.toml, alembic.ini,
      docker-compose.dev.yml, .env.example, README.md) copiada de svc-vivienda
- [x] `app/` — `main.py`, `config.py`, `database.py`, `auth.py` (ADR-015: lookup vía endpoint
      interno IAM, degrada a `invitado` en dev), `audit.py`
- [x] `infra/cloudsql-setup.sh` — `privada` agregado a `DBS`; create de db/usuario idempotente
- [x] `alembic/env.py` con import explícito de todos los modelos `priv_*`
- [x] Migración `0001` — schema `priv_*` completo (§4 del spec):
      `priv_gestiones`, `priv_gestiones_eventos`, `priv_localidades_info`, `priv_departamentos_info`,
      `priv_geo_localidades`, `priv_cat_*` (6), `priv_audit_log`. Verificada: `alembic upgrade head --sql`
      emite DDL limpio; `pytest` (2 tests, SQLite) en verde
- [x] Modelos reconciliados con los `schema_*.json` reales: `gestiones_eventos.usuario` NOT NULL,
      `localidades_info`/`departamentos_info` con `created_at`/`created_by`, `costo_estimado`
      `Numeric(18,2)`, free-text a `Text`, `geo_localidades` depto/localidad NOT NULL
- [x] `services/cloudbuild.yaml` — ya es parametrizado por `_SERVICE`; se le sumó la substitución
      `_SVC_VIVIENDA_INTERNAL_URL` (default vacío) para el deploy de privada. Deploy:
      `gcloud builds submit --config=cloudbuild.yaml --substitutions=_SERVICE=svc-privada,_SVC_VIVIENDA_INTERNAL_URL=<url-svc-vivienda>`
- [x] Provisionado en prod (Fase A del runbook): `db_privada`, `user_privada`, secreto
      `svc-privada-db-url` (password hex URL-safe). SA `svc-privada@` con `cloudsql.client` +
      `secretmanager.secretAccessor`
- [x] Cloud Run desplegado (2026-09-01): `https://svc-privada-iwni7vc2qq-rj.a.run.app`; `GET /health` OK
- [x] `alembic upgrade head` contra `db_privada` de prod → `alembic current` = `0001`

## Fase 2 — Endpoints a paridad

Implementado en `services/svc-privada/app/` (panel-module, SQLAlchemy async, sin BigQuery).
Verificado con **48 tests** (SQLite) incl. **tests de contrato** contra los fixtures de `anexos/D/`
(key sets == sistema viejo). Cobertura ~67% (el gate >80% es Fase 5).

- [x] `app/gestiones/` — `GET /gestiones` (+ `/`) con filtros (`estado/ministerio/categoria/
      departamento/localidad/tipo_gestion/canal_origen/q`) + paginación; 15 campos por item
- [x] `GET /gestiones/{id}` (acepta UUID o `id_legacy`; 32 campos + `is_deleted`),
      `GET /gestiones/{id}/eventos` (`metadata_json` como **objeto**)
- [x] `POST /gestiones` — geo lookup + evento `CREACION`; `origen="APP"`, `estado="INGRESADO"`
- [x] `POST /gestiones/{id}/cambiar-estado` — lock optimista **opcional** `updated_at` → `409`;
      `FINALIZADA` setea `fecha_finalizacion` (RE-9); evento `CAMBIO_ESTADO` + `ACTUALIZA_DATO` por campo
- [x] `PATCH /gestiones/{id}` — edición separada (nuevo en v1) + eventos `ACTUALIZA_DATO`
- [x] `DELETE /gestiones/{id}` — soft delete `deleted_at` + evento `ARCHIVO`
- [x] `GET /gestiones/resumen-territorial` (2 scopes, `departamento` obligatorio) — patrón A2;
      `metadata_json` de eventos va como **string** acá (inconsistencia del viejo, replicada)
- [x] `GET /gestiones/rollup-territorial` — **nuevo**, rollup global por (depto, localidad)
- [x] `GET /localidades-info` / `PUT /localidades-info` (el PUT sólo escribe los 4 campos, como hoy)
- [x] `GET /departamentos-info` — **nuevo**, read-only
- [x] `GET /catalogos/{estados,urgencias,ministerios,categorias,tipos-gestion,canales-origen}` +
      `/departamentos` + `/localidades?departamento=` + `/geo?departamento=&localidad=`. Sin `/modulos`.
- [x] `GET /informe/cooperativas/{resumen,temporal,por-departamento,puntos}` — clasificación
      regex de `v_informe_cooperativas` portada a Python (`app/informe/clasificacion.py`, 10
      prioridades). Montado bajo `/api/v1/privada`
- [x] `GET /me` — alias con shape viejo `{email, nombre, rol, modulos: []}`
- [x] Audit log en toda escritura (`priv_audit_log`)
- [x] Paths nuevos agregados a `infra/gateway/openapi.yaml` y activos en el gateway (Fase 6, 2026-09-01)
- [ ] Pendiente: `POST /gestiones` / `cambiar-estado` — documentar forma exacta del payload viejo
      (los fixtures D no capturan escrituras)

## Fase 3 — Unificación de auth

- [x] `svc-vivienda`: `app/internal/router.py` — `GET /internal/portal/usuarios/{email}` (IAM-only,
      sin prefijo `/api/v1`; sólo usuarios activos → 404 si no; +1 test, 246 tests svc-vivienda OK)
- [x] `svc-privada`: `app/auth.py` valida Firebase JWT + `_fetch_portal_user` async (httpx.AsyncClient)
      contra el endpoint interno; fallback `invitado`. `require_privada(*roles)` chequea rol +
      secretaría `"privada"` (Admin exento). 5 tests de auth e2e
- [x] `svc-privada`: routers usan `require_privada` + tuplas `ROLES_*` compartidas
- [x] `svc-vivienda`: `"privada"` ya está en `portal/schemas.SECRETARIAS_VALIDAS` (nada que cambiar)
- [x] frontend: `DashboardPage.tsx` (id `privada`, activa) + `AdminUsuariosPage` ya incluyen `"privada"`
- [x] **deploy**: IAM — SA de `svc-privada` con `roles/run.invoker` sobre `svc-vivienda` (Fase E.1);
      SA `api-gateway-sa@` con `run.invoker` sobre `svc-privada` (Fase E.2)
- [x] **deploy**: env `SVC_VIVIENDA_INTERNAL_URL=https://svc-vivienda-iwni7vc2qq-rj.a.run.app`
      seteada en el `gcloud run deploy` de svc-privada

## Fase 4 — ETL + verificación

- [x] `services/svc-privada/scripts/migrar_desde_bigquery.py` — 12 tablas, transformaciones §5;
      `--dry-run` / `--truncate` / `--limit N`. Helpers `s`/`ts`/`d`/`num`/`meta_to_dict` con tests
- [x] `is_deleted`→`deleted_at` (fecha del último evento `ARCHIVO`, o `updated_at`, o now);
      `''`→`NULL`; `TIMESTAMP`→`TIMESTAMPTZ` UTC; `uuid4()` nuevo + mapa `id_legacy`→`id`;
      `metadata_json` STRING→dict; `lat_centro/lon_centro`→`lat/lon`
- [x] Backfill `fecha_finalizacion`: último `CAMBIO_ESTADO`→`FINALIZADA`, si no hay usa `fecha_estado`
- [x] Verificación `_verificar()`: conteos PG vs `anexos/ETL_baseline.json`; chequea `finalizadas_sin_fecha == 0`
- [x] Idempotente: `--truncate` hace `TRUNCATE priv_* RESTART IDENTITY CASCADE`
- [x] **Dry-run ejecutado contra BQ real** (2026-09-01): 2123 gestiones / 136 borradas / 166 eventos
      / 0 huérfanos / 426 localidades_info / 25 departamentos_info / 110 backfill de
      `fecha_finalizacion` — **coincide exacto con la línea base** (`ETL_baseline.json`)
- [x] **Bug encontrado y corregido**: primer `--truncate` real falló con
      `StringDataRightTruncationError: value too long for character varying(30)` — BQ es STRING
      sin límite y la muestra chica del Anexo D no reflejaba la diversidad real (datos desde 2004).
      Se ensancharon `origen/id_legacy/nro_expediente/estado/urgencia/ministerio_agencia_id/
      organismo_id/derivado_a_id/categoria_general_id/subcategoria_id/tipo_demanda_principal_id/
      geo_id/costo_moneda/tipo_gestion/canal_origen` (modelo + migración `0001`, no desplegada
      a prod todavía → edición in-place segura). 61 tests siguen en verde
- [x] **`--truncate` ejecutado OK contra Postgres local** (2026-09-01, 2ª corrida, schema
      ensanchado). Verificación: 2123 gestiones / 1987 activas / 110 finalizadas / **0 sin fecha**
      / 166 eventos / 426 / 25 / 551 — **coincide exacto con la línea base**. El culpable del
      `varchar(30)` era `geo_id` (valores de hasta 56 chars); `String(60)` alcanza. `origen` sólo
      tiene `APP/ACTUAL/Actual/Histórico`
- [x] **Validación end-to-end del informe**: `app/informe/service.resumen()` sobre los datos
      migrados reproduce **exacto** `informe_resumen.json` del sistema viejo (10 temas, todos los
      conteos, total 847) → el port del regex de `v_informe_cooperativas` es correcto
- [x] **Ejecutado contra Cloud SQL de prod** (2026-09-01, `--truncate` contra `db_privada`):
      2123 gestiones / 1987 activas / 110 finalizadas / **0 sin fecha** / 166 eventos /
      426 localidades_info / 25 departamentos_info / 551 geo — coincide exacto con la línea base.
      Grant temporal `bigquery.jobUser`+`dataViewer` a `infraestructura.coop@gmail.com` sobre
      `essential-haiku-482815-u4` (revocar en el decommission; se conserva para el ETL delta)

## Fase 5 — Tests

- [x] Suite de tests de contrato (`test_contrato.py`): key sets de cada respuesta nueva vs los
      fixtures de `anexos/D/` (se saltea si `anexos/D/` no está)
- [x] Tests unitarios `pytest` — **64 tests, cobertura 91%** (`gestiones/service` 92%,
      `informe/service` 90%, `catalogos/service` 97%, `clasificacion` 100%). `auth.py` 65% (sin
      testear: minteo del ID token de la SA + fetch de JWKS — glue de infra; la lógica de
      `require_privada`/`get_current_user` sí está cubierta en `test_auth.py`)
- [x] `pyproject.toml`: `[tool.coverage.run] concurrency = ["greenlet"]` (sin esto coverage no ve
      el código sync ejecutado dentro del greenlet de SQLAlchemy async)
- [ ] Pendiente: capturar la forma exacta del payload de `POST /gestiones` y `cambiar-estado` del
      sistema viejo (los fixtures D no cubren escrituras) — o documentarla del código

## Fase 6 — Deploy + gateway config (sin activar)

- [x] **`infra/gateway/CUTOVER-svc-privada.md`** — spec de todos los cambios de `openapi.yaml`
      para aplicar de una en el cutover: repuntar `address`/`jwt_audience` de `/api/v1/privada/**`
      a `<SVC_PRIVADA_URL>`; borrar los 4 paths `/usuarios/**`; agregar `PATCH /gestiones/{id}`,
      `rollup-territorial`, `departamentos-info` y los 4 `/informe/cooperativas/*` (con bloques YAML
      listos para pegar); comandos de deploy + rollback + IAM + smoke
- [x] `openapi.yaml` — aplicado `CUTOVER-svc-privada.md` (2026-09-01, commit `b71e6d3` en
      `panel.infra`): 60 refs al backend viejo → `https://svc-privada-iwni7vc2qq-rj.a.run.app`;
      borrados los 4 paths `/api/v1/privada/usuarios/**`; `modulos` fuera del enum de `/catalogos`;
      agregados `PATCH /gestiones/{id}`, `rollup-territorial`, `departamentos-info` y los 4
      `/informe/cooperativas/*` (todos con `options:` + `security: []`). 77 paths, YAML válido
- [x] `ministerio-config-v20260901` creada **y activada** (`gateways update`) — se optó por cutover
      directo, no config latente. Rollback disponible: `--api-config=ministerio-config-v20260716b`

## Cutover

- [x] frontend `privada`: no usa `/usuarios/**` ni `/catalogos/modulos` (eso era del Vanilla JS
      viejo; el admin React vive en `modules/admin/`). `me()` pega a `/api/v1/privada/me` (alias).
      El lock `updated_at`/`409` es **opt-in** → sin enviarlo, `cambiar-estado` se comporta igual.
      `list`/`get`/`eventos` byte-compatibles (tests de contrato). `updated_at` + `derivado_a`/
      `acciones_implementadas` quedan para E1/E2.
- [x] **Alta de gestiones en el frontend nuevo** (2026-09-01) — hueco de paridad detectado en el
      cutover: el módulo React no tenía UI de alta (`POST /gestiones` existía sin invocador).
      Agregado `AgregarGestionModal.tsx` + `gestionesApi.crear` + botón "+ Nueva gestión" en
      `GestionesListPage` (gate `canModify`). Registrado en `spec-migracion §3.10`. Commit `abe3aac`
      en `panel.front`. **Desplegado a Firebase Hosting y verificado en prod** (2026-09-01)
- [ ] Avisar a los usuarios de Privada que dejen de usar el frontend viejo de GitHub Pages para
      cargar/editar (escribe directo al Cloud Run viejo → BigQuery → bifurca). El alta ya funciona
      en el portal nuevo. Ideal: poner el viejo en sólo-lectura o con aviso de redirección
- [ ] **Validar paridad frontend viejo vs nuevo antes de despublicar** — ver checklist abajo

### Paridad frontend viejo (`app.js`) vs nuevo (`modules/privada/`)

Vista **Gestiones** — **paridad completa al 2026-09-01**, desplegada a Firebase Hosting y
verificada en prod (commits `abe3aac`, `1d81886`, `0294ec7` en `panel.front`):
- [x] Nueva gestión (`AgregarGestionModal`)
- [x] Exportar tabla a **Excel** según filtros — pagina todo el resultado filtrado y arma `.xlsx`
- [x] Exportar tabla a **PDF** — `jspdf` + `jspdf-autotable` (lazy-import, no pesa en el bundle
      inicial), encabezado con filtros activos + total, horizontal
- [x] Columna Nro expediente + botón copiar; botón **Copiar ID** por fila
- [x] **Selector de columnas** — 14 columnas, Min/Todo/Reset, persistido en `localStorage` del
      navegador (antes 7 fijas). Columnas nuevas: Departamento, Ministerio, Categoría, Tipo, Canal,
      Costo, Días transcurridos, ID
- [x] **Ordenar por columna** (click en cabecera). ⚠️ *Limitación*: ordena la **página cargada**
      (50 filas), no el dataset completo — para eso falta un parámetro `sort` en `GET /gestiones`
      (backend, requiere redeploy). Documentado, no bloquea
- [x] `cambiar-estado` (`CambiarEstadoModal`): campos **Derivado a** y **Acciones implementadas**
      (el backend ya los aceptaba — era el pedacito de E2 que faltaba cablear) + editar
      **Departamento/Localidad** (cascada, prefill desde el detalle, se envían sólo si cambian)
- [x] Filtros, paginación, drawer detalle con timeline, eliminar

Vista **Resumen territorial** (en el nuevo es el módulo transversal `resumen-territorial`, no vive
dentro de privada):
- [x] **Ficha de localidad** (E5b, drawer, 2026-09-01) — `FichaDemografica` en el `DetailDrawer`:
      habitantes, electores, semáforo (chip), intendente + partido, tipo de localidad, legisladores
      + electores/habitantes del departamento. On-demand con el token del usuario a
      `GET /api/v1/privada/{localidades,departamentos}-info`. `spec-resumen-territorial-ficha-localidad.md` v0.2.0
- [ ] Ficha en el **export Excel** y la **vista de impresión** — diferido: necesita endpoint bulk
      de `localidades-info` en svc-privada o el embebido server-side de E5a
- [ ] Edición inline de la ficha desde el panel transversal — fuera de alcance (el `PUT` de
      svc-privada sigue siendo la vía; `tipo_localidad`/`color_semaforo` son read-only)
- [ ] Exportar resumen a PDF (el nuevo exporta a Excel) — el resumen territorial viejo tenía PDF

### E5a — Federación server-side de Privada (ADR-016) — código listo, falta deploy

- [x] svc-privada: `GET /internal/privada/rollup-territorial` (IAM-only) + tests
- [x] svc-vivienda: `config.svc_privada_internal_url`, `fetch_privada_lineas` vía endpoint interno,
      `_map_privada_payload` entiende el rollup
- [x] `cloudbuild.yaml`: `_PRIVADA_FETCH_ENABLED` / `_SVC_PRIVADA_INTERNAL_URL`
- [x] frontend: flag `VITE_PRIVADA_CLIENT_FEDERATION (opt-in de rollback; OFF por default)` (RE-7 — merge cliente un release detrás)
- [x] **Desplegado y verificado en prod** (2026-09-02): IAM `run.invoker` de `svc-vivienda@` sobre
      svc-privada · svc-privada con `/internal/privada/rollup-territorial` · svc-vivienda con
      `PRIVADA_FETCH_ENABLED=true` + `SVC_PRIVADA_INTERNAL_URL` · frontend con el merge client-side
      OFF por default (`ba70889`). `POST /resumen-territorial/actualizar` → `generado_para_areas`
      = `[vivienda, privada]`; en el navegador las líneas de Privada aparecen una sola vez.
      El scheduler `resumen-territorial-refresh` (*/30) mantiene el snapshot.
- [x] `cloudbuild.yaml` defaults en E5a-on (`a7d1794`) — un redeploy con `--set-env-vars` desde el
      yaml borraba las env seteadas a mano y el Resumen Territorial perdía Privada hasta el próximo
      `gcloud run services update`. ⚠️ un deploy que pase `--set-env-vars` **sin** incluir
      `PRIVADA_FETCH_ENABLED`/`SVC_PRIVADA_INTERNAL_URL` sigue pudiendo pisarlo — usar el yaml.

### Tablero nativo (Fase 7 / spec-privada-tablero.md) — GATE del decommission de BigQuery

- [x] `TableroPage.tsx` nativo: KPIs, donut por tema, barras por departamento, evolución mensual,
      mapa de puntos, filtros fecha + tema. Sobre los 4 endpoints `informe/cooperativas/*`.
      Sin `<iframe>`, `grep` de `lookerstudio`/`bigquery` = 0. `panel.front` `6791757`
- [ ] Deploy del frontend + **paridad numérica vs el Looker `f9dc4a4e-…`** para un rango de control
      (criterio de aceptación antes de apagar BigQuery)
- [ ] Marcar "Tablero nativo en producción" en `spec-migracion §10`

Vista **Usuarios**: cubierta por `modules/admin/AdminUsuariosPage`. El "panel de módulos por
usuario" del viejo se descartó a propósito (ADR-015 / D-3: todos los de Privada ven todo).

Vista **Tablero**: iframe Looker igual en ambos → **Fase 7** lo reemplaza por nativo (gate del
decommission).
- [ ] Ventana: sistema viejo en sólo-lectura *(sigue online read-write; se apaga en el
      decommission. El ETL fue `--truncate` completo, no delta)*
- [x] ETL contra `db_privada` de prod (ver Fase 4 — corrida completa 2026-09-01, no delta)
- [x] `CUTOVER-svc-privada.md` aplicado → `ministerio-config-v20260901` → `gateways update` (2026-09-01)
- [x] Alta de usuarios (2026-09-01): `scripts/fase_g_usuarios.sql` — 13 usuarios reales en
      `portal_usuarios`/`portal_usuario_secretarias` con secretaría `privada`, rol 1:1. Excluidas
      las 3 cuentas gateway/test Admin + `prueba@gmail.com` (inactivo). Pendiente decisión:
      `aguirrevictoriamariela` / `vanetoranzo` figuran Operador en el portal vs Supervisor en el viejo
- [x] Smoke por el gateway (2026-09-01, token Admin): `/me`, `/gestiones`, `/informe/cooperativas/*`
      (4), `/gestiones/rollup-territorial`, `/departamentos-info?departamento=`, `PATCH /gestiones/{id}`
      → todos 200. Preflight CORS (con `Access-Control-Request-Method`) → 200
- [x] Smoke desde el portal (navegador): lista → detalle → cambiar-estado → **OK**
- [ ] Monitoreo T+1..T+30

---

## Mejoras (specs hijos — post-cutover, aditivas, no bloquean el cutover)

### E1 — Categorías / Programas / Áreas editables (`spec-privada-categorias-programas.md` **approved v1.0.0**)

- [x] Migración `0002` (2026-09-02): `priv_categorias`/`priv_programas`/`priv_areas` + columnas
      `categoria_id`/`programa_id`/`area_id`/`ok_gobernador`/`ok_ministro`/`acciones_implementadas` en `priv_gestiones`
- [x] Seed de las 9 categorías (orden/colores por defecto, editables) + arranque programas/áreas
- [x] CRUD `/api/v1/privada/{categorias,programas,areas}` (`app/catalogos_editables/`) + guard 409
      (`*_EN_USO`, `CODIGO_DUPLICADO`) + roles (GET lectura / escritura `ROLES_TRANSICION`)
- [x] Schemas `POST/PATCH/cambiar-estado` aditivos + filtros `ok_gobernador`/`ok_ministro` en `GET /gestiones`
- [x] `scripts/backfill_categorias.py` (RE-1, `--dry-run`/`--force`/`--diff-informe`) — deriva de `tema_informe`
- [x] Gateway: 6 paths en `openapi.yaml` (nueva config pendiente)
- [x] Frontend: `CatalogoEditableSelect` (3 desplegables con "+ nueva opción" inline en el alta) +
      selects Ok Gob/Min + filtros de lista. `panel.front` `92422d5`
- [ ] **Deploy**: redeploy svc-privada + `alembic upgrade head` (0002) en prod + `backfill_categorias.py`
      (`--dry-run` → `--diff-informe` → aplicar) + nueva config de gateway + frontend
- [ ] Panel de administración **full** (editar orden/colores/activo, borrar con guard) — el "+ nueva"
      cubre el alta al vuelo; falta la UI de gestión completa
- [ ] **Con Secretaría Privada**: revisar orden/colores de los catálogos + validar el mapa de backfill
      (`--diff-informe`) antes de que E4 retire el regex

### E2 — Campos de gestión — **hecho** (parte con E1)

- [x] `CambiarEstadoModal` cablea `derivado_a` + `acciones_implementadas` (sprint del cutover)
- [x] `acciones_implementadas` se persiste en `priv_gestiones` (0002 + `cambiar_estado`)

### E3 — Flujo / DAG (`spec-privada-flujo-derivaciones.md`) — **BLOQUEADO** hasta el relevamiento

- [ ] Requiere la **taxonomía de áreas** definida con Secretaría Privada (RE-3) — `priv_areas` ya
      existe (0002) pero sólo con 3 valores de arranque + el centinela
- [ ] `metadata_json.derivado_a` es **null en todos los 166 eventos** (Anexo G) → el backfill del DAG
      no tiene datos de origen. Decidir: DAG sólo forward-only para gestiones nuevas
- [ ] `priv_gestion_derivaciones` + `priv_area_alias` + escritura en `cambiar-estado` + `GET /gestiones/{id}/flujo`
      + vista DAG en el drawer

### E4 — Informe v2 (`spec-privada-informe-cooperativas-v2.md`) — depende de E1 + **sign-off del área**

- [ ] `categoria_id` ya se puebla (E1 backfill). Falta el mapa `categoria_id → tema_informe`
- [ ] Doble corrida regex vs estructurada (`backfill_categorias.py --diff-informe`) + **sign-off de
      Secretaría Privada** — el spec no puede pasar a `approved` sin eso (RE-1)
- [ ] Recién con sign-off: reimplementar la clasificación sin regex + congelar/dropear `categoria_general_id`

### E5 — Resumen Territorial (`spec-resumen-territorial-ficha-localidad.md`)

- [ ] **E5a**: `privada_fetch_enabled=True`, `_map_privada_payload` al `rollup-territorial`,
      eliminar `privadaGestiones.ts` (tras flag un release)
- [ ] **E5b**: bloque `ficha` (demografía de localidad + departamento) en el payload / query lazy
- [ ] Frontend: tab/vista "Ficha de localidad" con electores, semáforo (chip), intendente+partido,
      legisladores; extender Excel + `@media print`

### Tablero nativo (`spec-privada-tablero.md`) — GATE del decommission

- [ ] Anexo A: inventario del informe Looker
- [ ] Endpoints de agregación faltantes
- [ ] `TableroPage.tsx` sin `<iframe>` ni URLs de `lookerstudio`/`bigquery`
- [ ] Paridad numérica vs Looker para un rango de control

---

## Decommission (post T+30d estable)

- [ ] Backup frío de `infra_gestion` a GCS (retención 1 año)
- [ ] Apagar Cloud Run `infraestructura-gestioninterna`
- [ ] Despublicar frontend GitHub Pages `labotech-analytics.github.io/SistemaGestiones_*`
- [ ] Retirar informe Looker `f9dc4a4e-a174-45a8-938c-385f4121f689`
- [ ] Revocar OAuth Client ID `354063050046-fkp06ao8...`
- [ ] Actualizar `docs/files/CLAUDE.md`, `CLAUDE.md` raíz, `arquitectura_actual.md`
- [ ] Suspender / borrar el proyecto `essential-haiku-482815-u4`
