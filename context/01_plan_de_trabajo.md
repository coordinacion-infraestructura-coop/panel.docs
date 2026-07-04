# Plan de Trabajo — GestorCooperativo

**Ministerio de Cooperativas y Mutuales de Córdoba**
**Última actualización**: 2026-07-02

---

## Decisiones de arquitectura tomadas

| Decisión | Valor |
|----------|-------|
| GCP Project nuevo | `gestorcooperativo` |
| Región | `southamerica-east1` (São Paulo — menor latencia desde Córdoba) |
| svc-privada (gestiones ministro) | Se expone vía API Gateway **sin migrar** — queda en `essential-haiku-482815-u4` con BigQuery |
| Nuevos servicios | Cloud SQL PostgreSQL 15 + FastAPI 0.115 + Python 3.12 |
| Auth | Firebase Authentication — JWT validado por backend; rol y secretarías leídos de Cloud SQL (`portal_usuarios`) |
| CI/CD | Cloud Build + Artifact Registry `southamerica-east1-docker.pkg.dev` |
| Frontend | React 18 + Vite + TypeScript + TanStack Query v5 + Firebase Hosting |
| Roles del portal | Admin / Supervisor / Operador / Consulta (sin custom claims JWT — DB-first, ADR-003 revisado) |

---

## Estado actual — resumen ejecutivo (2026-07-02)

| Etapa | Descripción | Estado |
|-------|-------------|--------|
| 0 | Ordenar proyecto y documentación | ✅ COMPLETA |
| 1 | Infraestructura GCP (Cloud SQL, Gateway, Firebase, CI/CD) | ✅ COMPLETA |
| 2 | Conectar svc-privada vía API Gateway | ✅ COMPLETA Y EN PRODUCCIÓN (2026-07-02) |
| 3 | svc-vivienda en Cloud Run + migraciones | ✅ COMPLETA Y EN PRODUCCIÓN |
| Portal | Módulo portal: gestión de usuarios (roles/secretarías) | ✅ COMPLETO Y EN PRODUCCIÓN |
| 4 | Frontend React en Firebase Hosting | ✅ COMPLETA Y EN PRODUCCIÓN |
| 5+ | Áreas pendientes (gasífera, infraestructura, territorial, desarrollo) | ⏳ Pendiente reuniones |

---

## Etapas de desarrollo

### Etapa 0 — Ordenar el proyecto ✅ COMPLETADA (2026-06-29)

Objetivo: repositorio consistente y documentación alineada con las decisiones tomadas.

- [x] Actualizar región `us-central1` → `southamerica-east1` en toda la documentación
- [x] Registrar ADR-006: svc-privada en proyecto GCP separado (BigQuery)
- [x] Actualizar OpenAPI spec: issuer Google OAuth, agregar ruta `/api/v1/privada/*`
- [x] Reescribir `roadmap.md` con el plan aprobado
- [x] Crear estructura de directorios: `infra/`, `services/`, `frontend/`
- [x] Documentar arquitectura actual del svc-privada en `docs/context/areas/Privada Ministro/arquitectura_actual.md`
- [x] Crear `services/svc-privada-adapter/README.md` explicando la estrategia de integración

---

### Etapa 1 — Infraestructura GCP ✅ COMPLETADA (2026-07-01)

Objetivo: proyecto GCP operativo con API Gateway, Cloud SQL, Firebase y CI/CD.

- [x] Ejecutar `infra/gcp-setup.sh` — proyecto, APIs, service accounts
- [x] Ejecutar `infra/cloudsql-setup.sh` — instancia PostgreSQL 15, 5 BDs, usuarios, secrets
- [x] Ejecutar `infra/pubsub-setup.sh` — tópicos y subscripciones
- [x] Configurar Firebase + Google Identity Platform (Google OAuth habilitado)
- [x] Artifact Registry creado en `southamerica-east1`
- [x] Cloud Build conectado al repositorio GitHub (`panel.backend`)
- [x] API Gateway desplegado con spec OpenAPI (rutas svc-vivienda + svc-privada)
- [x] Config activa: `ministerio-config-v20260702b` (incluye rutas portal)

**GCP Project**: `gestorcooperativo` | **Gateway host**: `ministerio-gateway-3j5k00ma.uc.gateway.dev`

---

### Etapa 2 — Conectar svc-privada vía API Gateway ✅ COMPLETA Y EN PRODUCCIÓN (2026-07-02)

Objetivo: el sistema de gestiones del ministro accesible desde el nuevo gateway.

- [x] Rutas `/api/v1/privada/*` en openapi.yaml — backend: Cloud Run `essential-haiku-482815-u4`
- [x] Módulo `privada` en frontend React — `GestionesListPage`, `TableroPage`
- [x] IAM cross-project binding ejecutado — `api-gateway-sa@gestorcooperativo` tiene `roles/run.invoker` en `essential-haiku-482815-u4`
- [x] Prueba end-to-end verificada — 1830 gestiones visibles desde el portal unificado

**Resultado**: las 1830 gestiones del sistema de infraestructura son accesibles desde `gestorcooperativo.web.app` vía API Gateway sin migrar el backend original.

---

### Etapa 3 — svc-vivienda ✅ COMPLETADA Y EN PRODUCCIÓN (2026-07-01)

Objetivo: microservicio de Vivienda desplegado en Cloud Run.

**URL**: `https://svc-vivienda-iwni7vc2qq-rj.a.run.app`

Módulos implementados y deployados:
- [x] `programas` — catálogo (Córdoba Hogar, Mi Lugar, Cordón Cuneta, Loteos)
- [x] `beneficiarios` — CRUD + búsqueda por DNI + soft delete
- [x] `expedientes` — máquina de 8 estados + historial + transiciones validadas
- [x] `asignaciones` — vinculación vivienda/lote a expediente ASIGNADO
- [x] `cordon_cuneta` — 46 municipios migrados del panel HTML a PostgreSQL + KPIs
- [x] `portal` — gestión de usuarios del portal (roles + secretarías asignadas)
- [x] Cloud Run deploy exitoso (rolling update sin downtime)
- [x] Alembic migrations 0001→0004 ejecutadas en producción
- [x] Seed: 4 programas + 46 municipios Cordón Cuneta + 2 admins portal

---

### Módulo Portal — Gestión de usuarios ✅ COMPLETADO Y EN PRODUCCIÓN (2026-07-02)

Objetivo: panel de administración para gestionar acceso de funcionarios al portal.

Implementación completa:
- [x] Tablas `portal_usuarios` + `portal_usuario_secretarias` (migración 0004)
- [x] `app/portal/models.py` — PortalUsuario, PortalUsuarioSecretaria
- [x] `app/portal/schemas.py` — schemas Pydantic, ROLES_VALIDOS, SECRETARIAS_VALIDAS
- [x] `app/portal/repository.py` — get_portal_user, list_usuarios, create_usuario, update_usuario
- [x] `app/portal/router.py` — GET /portal/me + CRUD admin /portal/admin/usuarios/*
- [x] `app/auth.py` — DB-first lookup (ADR-003 revisado), fallback a "invitado" si tabla no existe
- [x] Roles del portal: Admin / Supervisor / Operador / Consulta (reemplaza roles anteriores)
- [x] Admins iniciales seed: `bonafepedro@gmail.com` + `infraestructura.coop@gmail.com` (rol Admin, secretarías vivienda+privada)
- [x] Rutas portal en `openapi.yaml` con OPTIONS para CORS
- [x] Config gateway actualizada a `ministerio-config-v20260702b`

---

### Etapa 4 — Frontend React ✅ COMPLETADA Y EN PRODUCCIÓN (2026-07-01)

Objetivo: portal React con login y módulos vivienda + privada funcionales.

**URL**: `https://gestorcooperativo.web.app`

Stack implementado:
- [x] React 18 + Vite + TypeScript + TanStack Query v5 + Tailwind CSS v4
- [x] Firebase Authentication (Google Sign-In)
- [x] Paleta de colores gobierno: `gov-navy` (#172c3f), `gov-cyan` (#01aae3), `gov-blue` (#398ebd)

Módulos implementados:
- [x] `shared/auth/` — hook `useAuth()`, `<ProtectedRoute>`, contexto Firebase
- [x] `shared/api/` — axios client con interceptores JWT
- [x] `shared/hooks/usePortalUser.ts` — hook react-query → `GET /api/v1/portal/me`
- [x] `shared/components/Layout.tsx` — nav contextual por secretaría activa, chip de rol, link Admin condicional
- [x] `pages/DashboardPage.tsx` — filtra secretarías según `portalUser.secretarias`
- [x] `modules/vivienda/` — Programas, Beneficiarios, Expedientes (máquina de estados), Cordón Cuneta
- [x] `modules/privada/` — GestionesListPage (7 filtros + drawer + modal cambio estado), TableroPage (iframe Looker Studio)
- [x] `modules/admin/AdminUsuariosPage.tsx` — tabla + modal crear/editar usuarios (solo Admin)
- [x] Deploy a Firebase Hosting

---

### Etapas 5+ — Áreas pendientes de reunión

Para cada secretaría, el ciclo es:
1. Reunión de relevamiento → actualizar `docs/context/areas/{area}/`
2. Validar spec en `docs/files/spec-svc-{area}.md` → cambiar estado `draft` → `approved`
3. Implementar servicio (mismo patrón que svc-vivienda)
4. Módulo React (mismo patrón)
5. Deploy + UAT con operador del área

| Área | Titular | Spec | Prioridad | Estado |
|------|---------|------|-----------|--------|
| Infraestructura Gasífera | — | draft | 2 | Pendiente reunión |
| Infraestructura Eléctrica/Agua | Luis Molinari | draft | 3 | Pendiente reunión |
| Territorial | Gabriel Fizza | draft | 4 | Pendiente reunión |
| Desarrollo (UTN) | Domingo Benso | draft | 5 | Pendiente reunión + contrato UTN |

---

## Pendientes activos

| Tarea | Prioridad | Descripción |
|-------|-----------|-------------|
| Reuniones de relevamiento | 🟡 Media | Áreas gasífera, infraestructura, territorial, desarrollo — sin reunión no hay spec y sin spec no hay código |
| Dominio ministerial custom | 🟡 Media | Dominio institucional para el API Gateway y el frontend |

---

## Flujo de verificación end-to-end (estado actual ✅)

1. Abrir `https://gestorcooperativo.web.app` → muestra login Google
2. Login exitoso → `GET /api/v1/portal/me` → dashboard filtra secretarías asignadas al usuario
3. Admin ve todas las secretarías; Operador solo las asignadas
4. Crear beneficiario → crear expediente → estado `INGRESADO`
5. Transición a `EN_EVALUACION` → historial registrado en BD
6. Panel Cordón Cuneta → editar estado de municipio → persistido en PostgreSQL
7. Acceso a `/admin/usuarios` → CRUD de usuarios del portal (solo Admin)
8. Sección privada: 1830 gestiones del sistema de infraestructura visibles vía gateway ✅

---

## Referencia de archivos clave

| Propósito | Archivo |
|-----------|---------|
| Contexto general para Claude Code | `docs/files/CLAUDE.md` |
| Arquitectura + ADRs (incluyendo ADR-003 revisado) | `docs/files/arquitectura.md` |
| Spec infraestructura GCP | `docs/files/spec-infraestructura-gcp.md` |
| Spec svc-vivienda | `docs/files/spec-svc-vivienda.md` |
| OpenAPI Gateway | `infra/gateway/openapi.yaml` |
| Auth backend (DB-first) | `services/svc-vivienda/app/auth.py` |
| Módulo portal backend | `services/svc-vivienda/app/portal/` |
| Panel admin frontend | `frontend/src/modules/admin/pages/AdminUsuariosPage.tsx` |
| Layout contextual | `frontend/src/shared/components/Layout.tsx` |
| Sistema existente (Privada Ministro) | `docs/context/areas/Privada Ministro/arquitectura_actual.md` |
| Guía de deploy | `docs/deploy.md` |
