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
      viejo, confirmar que ningún usuario tiene acceso módulo-parcial (`usuario_modulos`)
- [ ] **Anexo A** — copiar el DDL de `v_informe_cooperativas` al spec
      (`proyecto_sistema_gestiones/informe/bq_views/v_informe_cooperativas.sql`)
- [ ] **Anexo B** — dump de valores actuales de `cat_*` (para el seed de `priv_cat_*`)
- [ ] **Anexo C** — export de `usuarios_roles` + `usuarios_eventos` (alta en `portal_usuarios` + backup)
- [ ] **Anexo D** — capturar respuestas reales del sistema viejo por endpoint (fixtures de contrato)
- [ ] **Anexo F** — `SELECT DISTINCT derivado_a, organismo_id FROM gestiones` + curado → borrador de `priv_areas`
- [ ] **Anexo G** — muestras de `metadata_json` por `tipo_evento`

## Fase 1 — Scaffold + schema

- [ ] `services/svc-privada/` — estructura (Dockerfile, pyproject.toml, alembic.ini,
      docker-compose.dev.yml, .env.example, .gitignore) copiada de svc-vivienda
- [ ] `app/` — `main.py`, `config.py`, `database.py`, `auth.py`, `audit.py`
- [ ] `infra/cloudsql-setup.sh` — agregar `privada` (crea `db_privada`, `user_privada`, secreto
      `svc-privada-db-url`)
- [ ] `alembic/env.py` con import explícito de todos los modelos `priv_*`
- [ ] Migración `0001` — schema `priv_*` completo (§4 del spec):
      `priv_gestiones`, `priv_gestiones_eventos`, `priv_localidades_info`, `priv_departamentos_info`,
      `priv_geo_localidades`, `priv_cat_*` (6), `priv_audit_log`
- [ ] `services/cloudbuild.yaml` — paso de build+deploy para `svc-privada`
- [ ] Cloud Run desplegado; `GET /health` OK
- [ ] `alembic current` = head en la instancia de staging/local

## Fase 2 — Endpoints a paridad

- [ ] `app/gestiones/` — `GET /gestiones` (+ trailing slash) con filtros + paginación
- [ ] `GET /gestiones/{id}` (acepta UUID o `id_legacy`), `GET /gestiones/{id}/eventos`
- [ ] `POST /gestiones` (evento `ALTA`)
- [ ] `POST /gestiones/{id}/cambiar-estado` — lock optimista `updated_at` + `409`; `FINALIZADA`
      setea `fecha_finalizacion`; evento `CAMBIO_ESTADO` + `ACTUALIZA_DATO` por campo
- [ ] `PATCH /gestiones/{id}` — edición separada
- [ ] `DELETE /gestiones/{id}` — soft delete `deleted_at`
- [ ] `GET /gestiones/resumen-territorial` (`departamento` obligatorio, paridad)
- [ ] `GET /gestiones/rollup-territorial` — **nuevo**, rollup global sin `departamento`
- [ ] `GET /localidades-info` / `PUT /localidades-info` (el PUT sólo 4 campos, como hoy)
- [ ] `GET /departamentos-info` — **nuevo**, read-only
- [ ] `GET /catalogos/{catalogo}` (estados, urgencias, ministerios, categorias, tipos-gestion,
      canales-origen, departamentos, localidades, geo)
- [ ] `GET /informe/cooperativas/{resumen,temporal,por-departamento,puntos}` — porta la
      clasificación regex de `v_informe_cooperativas` **tal cual** (Anexo A)
- [ ] `GET /me` — alias de `/api/v1/portal/me`
- [ ] Audit log en toda escritura

## Fase 3 — Unificación de auth

- [ ] `svc-vivienda`: `app/internal/router.py` — `GET /internal/portal/usuarios/{email}` (IAM-only)
- [ ] `svc-privada`: `app/auth.py` valida Firebase JWT + consulta el endpoint interno; fallback `invitado`
- [ ] `svc-privada`: cada router chequea secretaría `"privada"` + tuplas `ROLES_*` compartidas
- [ ] `svc-vivienda`: `"privada"` en `portal/schemas.ROLES_VALIDOS` / lista de secretarías
- [ ] frontend: `DashboardPage.tsx` `SECRETARIAS` + `AdminUsuariosPage` incluyen `"privada"`
- [ ] IAM: SA de `svc-privada` con `roles/run.invoker` sobre `svc-vivienda`

## Fase 4 — ETL + verificación

- [ ] `services/svc-privada/scripts/migrar_desde_bigquery.py` — 12 tablas, transformaciones §5
- [ ] `is_deleted`→`deleted_at`; `''`→`NULL`; `TIMESTAMP`→`TIMESTAMPTZ`; UUID + mapa `id_legacy`→`id`
- [ ] Backfill `fecha_finalizacion` de gestiones ya finalizadas
- [ ] Verificación: conteo por tabla + `SUM`/`MIN`/`MAX` de columnas clave, BQ vs PG, reporte de diffs
- [ ] Idempotente (`TRUNCATE`+recarga o `ON CONFLICT`)

## Fase 5 — Tests

- [ ] Suite de tests de contrato: respuesta vieja (Anexo D) vs nueva, por endpoint marcado `=`
- [ ] Tests unitarios `pytest` en `services/svc-privada/` — cobertura > 80%

## Fase 6 — Deploy + gateway config (sin activar)

- [ ] `infra/gateway/openapi.yaml` — repuntar `x-google-backend.address` de `/api/v1/privada/**`;
      quitar `/usuarios/**` y `/catalogos/modulos`; agregar `rollup-territorial`, `departamentos-info`,
      `PATCH /gestiones/{id}`; `options:` para cada path nuevo
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
