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

### Migración-paridad (contrato `/api/v1/privada/**` intacto en todo momento)

- [ ] **Fase 0** — ADR-008..016 + `spec-migracion-svc-privada.md` `approved`; reunión de
      relevamiento; Anexos A–G. *(ADR y spec ya escritos 2026-08-31.)*
- [ ] **Fase 1** — Scaffold `services/svc-privada/`; `db_privada` en `ministerio-postgres`; secreto
      `svc-privada-db-url`; Alembic `0001` (schema `priv_*`); CI/CD; Cloud Run `/health`.
- [ ] **Fase 2** — Endpoints a paridad: gestiones (CRUD + `PATCH` + eventos + `cambiar-estado` con
      lock `updated_at`), catálogos, `localidades-info`, `departamentos-info` (nuevo), informe (4),
      `rollup-territorial` (nuevo). Byte-compatible.
- [ ] **Fase 3** — Auth: `svc-privada/app/auth.py` estándar + `GET /internal/portal/usuarios/{email}`
      IAM-only en svc-vivienda; `"privada"` en `ROLES_VALIDOS`/`SECRETARIAS`/`AdminUsuariosPage`.
- [ ] **Fase 4** — ETL `migrar_desde_bigquery.py` (12 tablas + verificación conteos/sumas + backfill
      `fecha_finalizacion`). Idempotente.
- [ ] **Fase 5** — Tests de contrato (viejo vs nuevo) + unitarios > 80%.
- [ ] **Fase 6** — Deploy Cloud Run + `ministerio-config-v{YYYYMMDD}` (creada, no activada);
      validación por URL directa; rollback ensayado.
- [ ] **Cutover** — ventana: viejo sólo-lectura → ETL delta → `gateway update` → alta usuarios en
      `portal_usuarios` → smoke desde el portal. Frontend `privada` v1: `updated_at`/`409`, quitar
      `/usuarios/**` y `modulos`, permisos vía `usePortalUser`.

### Mejoras (specs hijos, post-cutover, aditivas)

- [ ] **E1** — `spec-privada-categorias-programas.md`: 3 catálogos editables + panel admin +
      desplegables con "+ nueva opción" + `ok_gobernador`/`ok_ministro` + backfill.
- [ ] **E2** — Campos de gestión (fusionable con E1): `derivado_a`/`acciones_implementadas` cableados
      en el modal de cambiar-estado.
- [ ] **E3** — `spec-privada-flujo-derivaciones.md`: `priv_gestion_derivaciones` + backfill + `/flujo`
      + vista DAG en el drawer.
- [ ] **E4** — `spec-privada-informe-cooperativas-v2.md`: reclasificación `tema_informe` sobre
      `categoria_id` + doble corrida + sign-off.
- [ ] **E5** — `spec-resumen-territorial-ficha-localidad.md`: federación server-side de Privada
      (ADR-016) + ficha de localidad en Resumen Territorial.
- [ ] **Tablero nativo** — `spec-privada-tablero.md`: reemplazo del iframe Looker. **Gate del
      decommission** (apagar BigQuery).

### Decommission (post T+30d estable)

- [ ] Backup frío de `infra_gestion` a GCS; apagar Cloud Run viejo; retirar GitHub Pages + informe
      Looker; revocar OAuth client; actualizar `CLAUDE.md`/`arquitectura_actual.md`; suspender/borrar
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
