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
- [ ] `services/cloudbuild.yaml` — paso de build+deploy para `svc-privada`
- [ ] Ejecutar `infra/cloudsql-setup.sh` en prod (crea `db_privada`, `user_privada`, secreto
      `svc-privada-db-url`)
- [ ] Cloud Run desplegado; `GET /health` OK
- [ ] `alembic upgrade head` contra `db_privada` (Cloud Shell + proxy, desde `services/svc-privada/`);
      `alembic current` = `0001`

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
- [ ] Pendiente: agregar los paths nuevos a `infra/gateway/openapi.yaml` (Fase 6)
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
- [ ] **deploy**: IAM — SA de `svc-privada` con `roles/run.invoker` sobre `svc-vivienda`
- [ ] **deploy**: env `SVC_VIVIENDA_INTERNAL_URL` = URL del Cloud Run de svc-vivienda (sin `/api/v1`)

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
- [ ] **Re-ejecutar `--truncate`** contra Postgres local (Docker, `:5433`) con el schema ensanchado
      y confirmar `finalizadas_sin_fecha: 0` en el reporte de verificación
- [ ] Ejecutar contra Cloud SQL de prod (Cloud Shell + `cloud-sql-proxy`, una vez provisionada
      `db_privada` — Fase 1 pendiente de deploy)

## Fase 5 — Tests

- [ ] Suite de tests de contrato: respuesta vieja (Anexo D) vs nueva, por endpoint marcado `=`
- [ ] Tests unitarios `pytest` en `services/svc-privada/` — cobertura > 80%

## Fase 6 — Deploy + gateway config (sin activar)

- [ ] `infra/gateway/openapi.yaml` — repuntar `x-google-backend.address` de `/api/v1/privada/**`;
      quitar `/usuarios/**` y `/catalogos/modulos`; agregar `rollup-territorial`, `departamentos-info`,
      `PATCH /gestiones/{id}` y los **4 `/informe/cooperativas/*`** (hoy 404 por el gateway);
      `options:` para cada path nuevo
- [ ] `ministerio-config-v{YYYYMMDD}` creada (NO activada)
- [ ] Smoke end-to-end por URL directa de Cloud Run
- [ ] Rollback ensayado (revertir a `ministerio-config-v20260716b`)

## Cutover

- [ ] frontend `privada` v1: enviar `updated_at` + manejar `409`; quitar usos de `/usuarios/**` y
      `catalogos/modulos`; permisos vía `usePortalUser` + `"privada"`
- [ ] Ventana: sistema viejo en sólo-lectura
- [ ] ETL delta final
- [ ] `gcloud api-gateway gateways update` → nueva config
- [ ] Alta de usuarios de Privada en `portal_usuarios` (desde el CSV del Anexo C)
- [ ] Smoke desde el portal: login → lista → detalle → cambiar-estado → informe
- [ ] Monitoreo T+1..T+30

---

## Mejoras (specs hijos — post-cutover, aditivas, no bloquean el cutover)

### E1 — Categorías / Programas / Áreas editables (`spec-privada-categorias-programas.md`)

- [ ] Migración `0002`: `priv_categorias`, `priv_programas`, `priv_areas` + columnas
      `categoria_id`/`programa_id`/`area_id`/`ok_gobernador`/`ok_ministro`/`acciones_implementadas`
- [ ] Seed de las 9 categorías + backfill `categoria_id` desde `categoria_general_id` (Anexo A del spec hijo)
- [ ] CRUD de los 3 catálogos + guard 409 (`*_EN_USO`)
- [ ] Schemas `POST/PATCH /gestiones` aditivos
- [ ] Frontend: modal(es) "Gestionar…" + 3 desplegables con "+ nueva opción" + filtros `Ok Gob/Min`

### E2 — Campos de gestión (fusionable con E1)

- [ ] `CambiarEstadoModal` cablea `derivado_a` + `acciones_implementadas`
- [ ] `acciones_implementadas` se persiste en `priv_gestiones` (no sólo en `metadata_json`)

### E3 — Flujo / DAG (`spec-privada-flujo-derivaciones.md`)

- [ ] `priv_gestion_derivaciones` + `priv_area_alias`
- [ ] Escritura runtime en `cambiar-estado`
- [ ] Job de backfill desde `metadata_json` (confianza + nodo centinela)
- [ ] `GET /gestiones/{id}/flujo`
- [ ] Vista "Flujo" (DAG) en `GestionDetalleDrawer`, junto al timeline

### E4 — Informe v2 (`spec-privada-informe-cooperativas-v2.md`)

- [ ] Mapa `categoria_id` → `tema_informe`
- [ ] Reimplementar la clasificación sin regex
- [ ] Doble corrida regex vs estructurada + reporte de diffs + sign-off del área
- [ ] Congelar/dropear `categoria_general_id`

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
