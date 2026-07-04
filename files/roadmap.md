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

## Etapa 2 — Conectar svc-privada vía API Gateway (Sem 2, paralelo)

**Objetivo**: Sistema de gestiones existente accesible desde el nuevo gateway.

- [ ] Agregar dominio del API Gateway a `allow_origins` en svc-privada existente
- [ ] Validar compatibilidad JWT: Google OAuth (existente) ↔ Google Identity Platform (nuevo)
- [ ] Prueba end-to-end: login → token → `/api/v1/privada/gestiones/` → respuesta
- [ ] Crear `docs/context/areas/Privada Ministro/contrato_api.md` con mapeo de roles
- [ ] **Reunión pendiente** con área: mapear roles existentes (Admin/Supervisor/Operador) a roles ministeriales
- **Entregable**: Gateway → svc-privada funcional y verificado

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
