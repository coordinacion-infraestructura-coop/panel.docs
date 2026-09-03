# Roadmap de implementación

## Decisiones de contexto

- **GCP Project nuevo**: `gestorcooperativo` (región `southamerica-east1`)
- **svc-privada**: sistema existente en `essential-haiku-482815-u4` — se expone vía API Gateway sin migración
- **Panel Cordón Cuneta**: HTML standalone actual → migrar a módulo de svc-vivienda con PostgreSQL
- **Filosofía**: Spec Driven Development — ningún módulo se implementa sin spec aprobado + reunión de área

---

## Etapa 0 — Ordenar el proyecto (~2hs) ✅

- [x] Actualizar región `us-central1` → `southamerica-east1` en toda la documentación
- [x] Registrar ADR-006: svc-privada en proyecto GCP separado con BigQuery
- [x] Actualizar OpenAPI spec del gateway: agregar ruta svc-privada + issuer Google OAuth
- [x] Crear estructura de directorios: `infra/`, `services/`, `frontend/`
- [x] Documentar arquitectura actual del svc-privada existente
- **Entregable**: Documentación consistente, estructura de repo lista

---

## Etapa 1 — Infraestructura GCP (Sem 1-2)

**Objetivo**: Proyecto GCP listo, API Gateway funcional, CI/CD configurado.

- [ ] Ejecutar `infra/gcp-setup.sh` — proyecto, APIs habilitadas, service accounts
- [ ] Ejecutar `infra/cloudsql-setup.sh` — instancia PostgreSQL 15, 5 bases de datos
- [ ] Ejecutar `infra/pubsub-setup.sh` — tópicos y subscripciones
- [ ] Configurar Firebase proyecto + Identity Platform (Google OAuth)
- [ ] Crear repositorio GitHub + conectar Cloud Build
- [ ] Crear Artifact Registry en southamerica-east1
- [ ] Desplegar API Gateway con spec OpenAPI base (rutas svc-privada + svc-vivienda)
- **Entregable**: Gateway funcionando, se puede llamar `/api/v1/privada/gestiones/`

---

## Etapa 2 — Conectar svc-privada vía API Gateway (Sem 2, paralelo) ✅

**Objetivo**: Sistema de gestiones existente accesible desde el nuevo gateway.

- [x] Agregar dominio del API Gateway a `allow_origins` en svc-privada existente
- [x] Validar compatibilidad JWT
- [x] Prueba end-to-end: login → token → `/api/v1/privada/gestiones/` → respuesta
- [x] Frontend `src/modules/privada/` reimplementado en React (lista, drawer, cambiar-estado, tablero)
- **Entregable**: Gateway → svc-privada funcional y verificado

> **ADR-006 SUPERSEDIDO por ADR-008 (2026-08-31).** La decisión de "no migrar svc-privada" se
> revirtió. La migración al monorepo (BigQuery → PostgreSQL) + 5 mejoras funcionales se planifican
> en **Etapa 2-bis** (abajo). Specs: `spec-migracion-svc-privada.md` (approved v1.0.0) +
> `spec-privada-categorias-programas.md`, `spec-privada-flujo-derivaciones.md`,
> `spec-privada-tablero.md`, `spec-resumen-territorial-ficha-localidad.md`,
> `spec-privada-informe-cooperativas-v2.md` (draft).

---

## Etapa 2-bis — Migración de svc-privada al monorepo + mejoras (2026-08-31)

**Objetivo**: Absorber el sistema de la Secretaría Privada al monorepo sobre PostgreSQL, apagar el
proyecto GCP personal `essential-haiku-482815-u4`, y sumar las mejoras pedidas (categorías/programas/
áreas editables, DAG de flujo, ficha de localidad, Ok Gobernador/Ok Ministro, Tablero nativo).

> **Estado 2026-09-03**: migración-paridad + cutover + E1/E2/E5 + Tablero nativo **en producción**.
> Detalle vivo en `TODO-migracion-svc-privada.md`. Falta: paridad numérica Tablero vs Looker →
> decommission; E3/E4 dependen de la reunión de relevamiento.

### Migración-paridad (contrato `/api/v1/privada/**` intacto en todo momento)

- [x] **Fase 0** — ADR-008..016 + `spec-migracion-svc-privada.md` `approved` v1.0.0; Anexos A–H.
      *(Reunión de relevamiento sigue pendiente — no bloqueó las Fases 0–6.)*
- [x] **Fase 1** — Scaffold `services/svc-privada/`; `db_privada`; secreto; Alembic `0001`; Cloud Run.
- [x] **Fase 2** — Endpoints a paridad (gestiones CRUD + PATCH + eventos + cambiar-estado, catálogos,
      `localidades-info`, `departamentos-info`, informe ×4, `rollup-territorial`). Byte-compatible.
- [x] **Fase 3** — Auth a `portal_usuarios` vía `GET /internal/portal/usuarios/{email}` (IAM-only).
- [x] **Fase 4** — ETL `migrar_desde_bigquery.py` — corrida `--truncate` en prod (2123/1987/110/0).
- [x] **Fase 5** — 73 tests svc-privada (incl. contrato vs `anexos/D/`).
- [x] **Fase 6** — Deploy Cloud Run + `ministerio-config-v20260901`.
- [x] **Cutover** (2026-09-01) — `gateway update` + alta de 17 usuarios en `portal_usuarios` + smoke
      OK. Frontend `privada`: sin cambios de contrato + modal "Nueva gestión" (hueco de paridad) +
      paridad total de la vista Gestiones (export Excel/PDF, columnas, sort, copiar, Derivado a /
      Acciones / editar localidad en cambiar-estado).

### Mejoras (specs hijos)

- [x] **E1** — 3 catálogos editables (`priv_categorias`/`priv_programas`/`priv_areas`, migración
      `0002`), CRUD + panel de administración (`GestionarCatalogosModal`), desplegables con "＋ Cargar
      nueva opción…", `ok_gobernador`/`ok_ministro`, columna + filtro "Campo de Trabajo",
      `backfill_categorias.py` corrido (1946/2047). Spec `approved` v1.0.0.
- [x] **E2** — `acciones_implementadas` persistida en `priv_gestiones`; `derivado_a`/`acciones`
      cableados en `CambiarEstadoModal`.
- [ ] **E3** — `spec-privada-flujo-derivaciones.md`: **bloqueado** — necesita la taxonomía de áreas
      del relevamiento; `metadata_json.derivado_a` es null en los 166 eventos (sin datos de backfill).
- [ ] **E4** — `spec-privada-informe-cooperativas-v2.md`: `categoria_id` ya poblado; falta el mapa
      `categoria_id → tema_informe` + **sign-off del área** sobre la reclasificación (RE-1). Spec `draft`.
- [x] **E5** — **E5b** ficha de localidad (drawer del Resumen Territorial, on-demand) + **E5a**
      federación server-side de Privada (ADR-016, endpoint IAM-only `/internal/privada/rollup-territorial`).
      Diferido: columnas de ficha en Excel/impresión del RT (necesita el bulk endpoint — hecho:
      `/localidades-info/all`) → pendiente cablearlo.
- [x] **Tablero nativo** — `TableroPage.tsx` sin iframe Looker (KPIs + donut por tema + barras por
      depto + evolución + mapa, sobre `informe/cooperativas/**`). **Falta paridad numérica vs el
      Looker** para un rango de control antes de apagar BigQuery.

### Decommission (post T+30d estable)

- [ ] Backup frío de `infra_gestion` a GCS; apagar Cloud Run viejo; retirar GitHub Pages + informe
      Looker; revocar OAuth client + los grants temporales de BigQuery a `infraestructura.coop@gmail.com`;
      actualizar `CLAUDE.md` raíz + `docs/files/CLAUDE.md` + `arquitectura_actual.md`; suspender/borrar
      `essential-haiku-482815-u4`.

- **Entregable**: sistema de Privada 100% en el monorepo, proyecto GCP personal apagado.

---

## Etapa 3 — svc-vivienda (Sem 2-5)

**Objetivo**: Microservicio completo de Vivienda con PostgreSQL, deployado en Cloud Run.

Prerequisito: marcar `docs/files/spec-svc-vivienda.md` como `approved`.

- [ ] Scaffold `services/svc-vivienda/` (Dockerfile, pyproject.toml, alembic, estructura)
- [ ] Módulo `programas` (catálogo + seed data)
- [ ] Módulo `beneficiarios` (CRUD + búsqueda por DNI + soft delete)
- [ ] Módulo `expedientes` (máquina de 8 estados + historial + transiciones validadas)
- [ ] Módulo `asignaciones` (requiere expediente ASIGNADO)
- [ ] Módulo `cordon_cuneta` (migrar 46 municipios del HTML a PostgreSQL)
- [ ] Migraciones Alembic + seed data de programas
- [ ] Tests >80% de cobertura
- [ ] Dockerfile + cloudbuild.yaml para CI/CD
- [ ] Deploy a Cloud Run + agregar al API Gateway
- **Entregable**: CRUD vivienda completo, Panel Cordón Cuneta persistente

---

## Etapa 4 — Frontend React (Sem 4-7, paralelo con Etapa 3)

**Objetivo**: Portal React con login y módulo vivienda funcional.

- [ ] Scaffold `frontend/` (Vite + React 18 + TypeScript)
- [ ] `shared/auth`: hook `useAuth()`, `<ProtectedRoute>`, Google Identity Platform
- [ ] `shared/api`: axios client con interceptores JWT
- [ ] Módulo vivienda: páginas de beneficiarios, expedientes, programas, Cordón Cuneta
- [ ] Firebase Hosting: deploy con dominio custom
- **Entregable**: Login → módulo vivienda completo con datos reales

---

## Etapas 5+ — Por área (según reuniones)

Para cada área, el ciclo es:

1. **Reunión de relevamiento** → actualizar `docs/context/areas/{area}/`
2. **Validar spec** en `docs/files/spec-svc-{area}.md` → cambiar a `approved`
3. **Implementar servicio** (mismo patrón que svc-vivienda)
4. **Módulo React** (mismo patrón que módulo vivienda)
5. **Deploy + UAT** con operador del área

### Prioridad sugerida post-vivienda

| Área | Titular | Spec actual | Prioridad |
|------|---------|-------------|-----------|
| Infraestructura Gasífera | — | draft | 2 |
| Infraestructura Eléctrica/Agua | Luis Molinari | draft | 3 |
| Territorial | Gabriel Fizza | draft | 4 |
| Desarrollo (UTN) | Domingo Benso | draft — pendiente contrato UTN | 5 |

---

## Pendientes críticos (bloquean Etapa 1)

- [ ] Confirmar dominio ministerial (`api.ministerio-coop.gob.ar` u otro)
- [ ] Confirmar dominio de correos institucionales (para Google Identity Platform)
- [ ] Billing account GCP asignada al proyecto `gestorcooperativo`
- [ ] Datos iniciales de usuarios y roles por secretaría
- [ ] Definir si svc-privada frontend migra de GitHub Pages a Firebase Hosting o queda separado

---

## Flujo de verificación final (Etapas 0-4 completas)

1. `https://ministerio-coop.gob.ar` → login Google → portal ministerial
2. Cargar beneficiario → crear expediente → estado `INGRESADO`
3. Transición a `EN_EVALUACION` → historial registrado
4. Panel Cordón Cuneta → editar municipio → persistido en BD
5. Sección privada → `GET /api/v1/privada/gestiones/` → lista del sistema existente
