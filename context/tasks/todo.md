# Registro de tareas — GestorCooperativo

**Última actualización**: 2026-07-02 (sesión Córdoba Hogar + Tablero)

---

## ✅ COMPLETADAS

### Sesión 2026-06-29 — Etapa 0: Ordenar el proyecto

- [x] **Revisar toda la documentación existente**
  - Explorados: `docs/files/` (10 archivos), `docs/context/` (organigrama, idea_gral, áreas), sistema existente svc-privada
  - Hallazgo clave: proyecto era 100% documentación, 0% código — sistema existente (gestiones ministro) usa BigQuery, no PostgreSQL

- [x] **Definir decisiones de arquitectura** (con el usuario)
  - GCP Project nuevo: `gestorcooperativo`; Región: `southamerica-east1` (São Paulo)
  - svc-privada: exponer vía API Gateway sin migrar

- [x] **Actualizar `docs/files/spec-infraestructura-gcp.md`**
  - Región cambiada a `southamerica-east1`, registry actualizado, URLs Cloud Run ajustadas
  - `securityDefinitions` actualizado a Google OAuth estándar, rutas svc-privada agregadas al OpenAPI

- [x] **Actualizar `docs/files/CLAUDE.md`** — región y registry actualizados

- [x] **Actualizar `docs/files/arquitectura.md`** — ADR-006 agregado (svc-privada en proyecto separado)

- [x] **Reescribir `docs/files/roadmap.md`** — nuevo roadmap con Etapas 0-5

- [x] **Crear estructura de directorios del repositorio**
  - `infra/gateway/`, `services/svc-vivienda/`, `services/svc-privada-adapter/`, `frontend/src/shared/`, `frontend/src/modules/`

- [x] **Documentar svc-privada existente**
  - `docs/context/areas/Privada Ministro/arquitectura_actual.md`
  - `services/svc-privada-adapter/README.md`

---

### Sesión 2026-06-29 — Etapa 1: Scripts de infraestructura GCP (generados)

- [x] **`infra/gcp-setup.sh`** — Crear proyecto, habilitar 11 APIs, crear 5+1 service accounts, permisos Cloud Build
- [x] **`infra/cloudsql-setup.sh`** — Instancia PostgreSQL 15, 5 bases de datos, usuarios, secrets en Secret Manager
- [x] **`infra/pubsub-setup.sh`** — 5 tópicos + dead letter + subscripciones BigQuery y notificaciones
- [x] **`infra/gateway/openapi.yaml`** — Spec OpenAPI 2.0 completo con svc-privada y svc-vivienda
- [x] **`infra/gateway/deploy-gateway.sh`** — Script de deploy/actualización del API Gateway
- [x] **`cloudbuild.yaml`** — Pipeline CI/CD parametrizado (build → push → deploy Cloud Run)

---

### Sesión 2026-06-29 — Etapa 3: svc-vivienda (código inicial)

- [x] **Scaffold del servicio** — `pyproject.toml`, `Dockerfile` multi-stage, `alembic.ini`, `.env.example`, `docker-compose.dev.yml`
- [x] **Módulos base** — `config.py`, `database.py`, `auth.py` (JWT validation), `audit.py`, `pubsub.py`, `main.py`
- [x] **Módulo `programas`** — catálogo (Córdoba Hogar, Mi Lugar, Cordón Cuneta, Loteos), seed incluido
- [x] **Módulo `beneficiarios`** — CRUD + búsqueda por DNI + soft delete
- [x] **Módulo `expedientes`** — máquina de 8 estados + historial + transiciones validadas
- [x] **Módulo `asignaciones`** — requiere expediente en ASIGNADO
- [x] **Módulo `cordon_cuneta`** — 46 municipios migrados del panel HTML a PostgreSQL + KPIs
- [x] **Migración Alembic 0001** — schema inicial (7 tablas)
- [x] **Tests iniciales** — `conftest.py`, `test_health.py`, `test_programas.py`, `test_expedientes_state_machine.py`

---

### Sesión 2026-07-01 — Etapa 1: Infraestructura GCP ejecutada en producción

- [x] **Cloud SQL PostgreSQL 15** operativo — instancia `ministerio-postgres`, región `southamerica-east1`
- [x] **Base de datos** `db_vivienda` + usuario `user_vivienda` + secret en Secret Manager
- [x] **Firebase** configurado — Authentication (Google Sign-In habilitado), Hosting activo
- [x] **Artifact Registry** creado en `southamerica-east1`
- [x] **API Gateway** desplegado — `ministerio-gateway-3j5k00ma.uc.gateway.dev` (región `us-central1`)
- [x] **Service Account** `api-gateway-sa@gestorcooperativo.iam.gserviceaccount.com` con `roles/run.invoker`

---

### Sesión 2026-07-01 — Etapa 3: svc-vivienda en producción

- [x] **Cloud Run deploy** exitoso — `https://svc-vivienda-iwni7vc2qq-rj.a.run.app`
- [x] **Migraciones Alembic 0001 → 0003** ejecutadas vía Cloud SQL Proxy en Cloud Shell
- [x] **Seed** de programas y municipios Cordón Cuneta cargado en producción

---

### Sesión 2026-07-01 — Etapa 4: Frontend React

- [x] **Setup** — React 18 + Vite + TypeScript + TanStack Query v5 + Tailwind CSS v4
- [x] **Firebase Authentication** — Google Sign-In, `AuthContext`, `ProtectedRoute`, `useAuth()`
- [x] **axios client** con interceptores JWT (adjunta Bearer token en cada request)
- [x] **Layout.tsx** — nav contextual por secretaría activa, paleta gobierno, responsive
- [x] **DashboardPage.tsx** — cards de secretarías, enlace a módulos
- [x] **Módulo vivienda** — Programas, Beneficiarios, Expedientes (transiciones UI), Cordón Cuneta (tabla editable)
- [x] **Módulo privada** — `GestionesListPage.tsx` (7 filtros + drawer + modal cambio estado), `TableroPage.tsx` (iframe Looker Studio)
- [x] **Firebase Hosting deploy** — `https://gestorcooperativo.web.app` operativo

---

### Sesión 2026-07-02 — Módulo portal: gestión de usuarios

- [x] **Decisión ADR-003 revisada** — DB-first (tabla `portal_usuarios`) en lugar de JWT custom claims
- [x] **Migración 0004** — tablas `portal_usuarios` + `portal_usuario_secretarias` con seed admins
- [x] **`app/portal/models.py`** — PortalUsuario (email PK, nombre, rol, activo, audit) + PortalUsuarioSecretaria
- [x] **`app/portal/schemas.py`** — ROLES_VALIDOS = (Admin, Supervisor, Operador, Consulta), SECRETARIAS_VALIDAS, schemas Pydantic
- [x] **`app/portal/repository.py`** — get_portal_user, get_portal_user_any_status, list_usuarios, create_usuario, update_usuario
- [x] **`app/portal/router.py`** — GET /portal/me + CRUD admin (protegido con `_require_admin`)
- [x] **`app/auth.py`** refactorizado — DB lookup de rol+secretarías, try/except resiliente (fallback invitado), nuevas constantes ROLES_*
- [x] **`app/main.py`** — router portal incluido con prefix `/api/v1`
- [x] **`alembic/env.py`** — importa modelos PortalUsuario, PortalUsuarioSecretaria
- [x] **Gateway `openapi.yaml`** — paths `/api/v1/portal/me`, `/api/v1/portal/admin/usuarios`, `/api/v1/portal/admin/usuarios/{email}` con OPTIONS
- [x] **Deploy gateway** `ministerio-config-v20260702b` activo (resolvió CORS en rutas portal)
- [x] **Migración 0004 ejecutada** en producción vía Cloud SQL Proxy
- [x] **`shared/hooks/usePortalUser.ts`** — hook react-query → `GET /api/v1/portal/me` (staleTime 5min)
- [x] **`Layout.tsx` refactorizado** — nav contextual por secretaría, chip rol usuario, link Administración (solo Admin)
- [x] **`DashboardPage.tsx` refactorizado** — filtra secretarías por `portalUser.secretarias`, Admin ve todo
- [x] **`modules/admin/AdminUsuariosPage.tsx`** — tabla usuarios + modal crear/editar con checkboxes secretarías
- [x] **`App.tsx`** — ruta `/admin/usuarios` agregada
- [x] **Deploy frontend** con cambios portal
- [x] **Seed admins** — `bonafepedro@gmail.com` + `infraestructura.coop@gmail.com` (Admin, vivienda+privada)
- [x] **Bug CORS** `/api/v1/portal/me` resuelto (try/except en auth.py + migración 0004 ejecutada)
- [x] **Documentación actualizada** — arquitectura.md (ADR-003 revisado, ADR-007 nuevo), CLAUDE.md (roles, portal), deploy.md, plan de trabajo, todo.md

---

### Sesión 2026-07-02 — Módulo Córdoba Hogar + Tablero de Programas

- [x] **`app/cordoba_hogar/`** — módulo ABM provisorio: modelos (LocalidadCordobaHogar, EstadoCordobaHogar, ConfigCordobaHogar, PedidoCordobaHogar), schemas, repository, router
- [x] **Migración 0005** — update_cc_data (sincronización datos Cordón Cuneta con panel HTML actualizado)
- [x] **Migración 0006** — tablas Córdoba Hogar: viv_ch_estados, viv_cordoba_hogar, viv_ch_config, viv_ch_pedidos
- [x] **`seed_cordoba_hogar.py`** — seed de 43 localidades + estados + config inicial; fix date.fromisoformat() para asyncpg
- [x] **Migraciones 0005 + 0006 ejecutadas** en producción vía Cloud SQL Proxy
- [x] **Seed Córdoba Hogar ejecutado** en producción (43 localidades)
- [x] **Gateway actualizado** — `ministerio-config-v20260703a` con rutas `/api/v1/vivienda/cordoba-hogar*` + OPTIONS para CORS
- [x] **CORS corregido** — `/api/v1/vivienda/cordoba-hogar` dejó de fallar con "No Access-Control-Allow-Origin"
- [x] **Backend deploy** — svc-vivienda redesplegado con módulo cordoba_hogar incluido
- [x] **`ProgramasPage.tsx` reescrita** — Tablero de Programas con KPIs en vivo (Córdoba Hogar + Cordón Cuneta)
- [x] **`DashboardPage.tsx`** — "Córdoba Hogar" va a `/vivienda/cordoba-hogar`; Beneficiarios y Expedientes ocultos (`hidden: true`)
- [x] **`Layout.tsx`** — nav vivienda muestra Tablero, Córdoba Hogar, Cordón Cuneta; Beneficiarios/Expedientes comentados
- [x] **Firebase Hosting deploy** desde Windows local (`npm run build` → `firebase deploy --only hosting --project gestorcooperativo`) — ✅ operativo

---

## ⏳ PENDIENTES

### Etapa 2 — Contrato svc-privada (✅ IAM binding ejecutado)

- [ ] Crear `docs/context/areas/Privada Ministro/contrato_api.md` con mapeo de roles
- [ ] Reunión con área: confirmar si roles del sistema existente (Admin/Supervisor/Operador del sistema viejo) mapean a roles del nuevo portal

### Etapas 5+ — Por área (requieren reuniones)

- [ ] Reunión Secretaría Infraestructura Gasífera → spec `approved` → implementar `svc-gasifera`
- [ ] Reunión Secretaría Infraestructura Eléctrica/Agua (Luis Molinari) → spec `approved` → implementar `svc-infraestructura`
- [ ] Reunión Secretaría Territorial (Gabriel Fizza) → spec `approved` → implementar `svc-territorial`
- [ ] Reunión Secretaría Desarrollo (Domingo Benso) + contrato API UTN → spec `approved` → implementar `svc-desarrollo`

### Mejoras futuras (cuando haya demanda real)

- [ ] Paginación server-side en GestionesListPage (actualmente carga todo en cliente)
- [ ] Módulo de notificaciones (Pub/Sub → Cloud Scheduler → email/push)
- [ ] Dashboard ejecutivo BigQuery (Looker Studio embebido con datos de todas las secretarías)
- [ ] Dominio ministerial custom para API Gateway y Firebase Hosting

---

## Lecciones aprendidas

- El sistema existente (svc-privada) usa BigQuery como BD principal, no PostgreSQL. Documentarlo desde el inicio evita decisiones erróneas de migración.
- Para organismos gubernamentales en Córdoba, Argentina: `southamerica-east1` es la región correcta (latencia ~20ms vs ~180ms de `us-central1`).
- El Panel Cordón Cuneta era un HTML standalone con datos hardcodeados — migrar el seed a `seed_data.py` garantizó que no se perdiera la información al hacer el deploy inicial.
- Las configs de API Gateway son inmutables — si `api-configs create` falla con `ALREADY_EXISTS`, usar un sufijo nuevo (`ministerio-config-v{FECHA}b`, `c`, etc.).
- Todo path nuevo en el gateway necesita su operación `options:` con `security: []` o el preflight CORS falla con un error engañoso.
- Al correr Alembic en Cloud Shell, usar TCP URL (`127.0.0.1:5432` via Cloud SQL Proxy), no el socket URL. Los caracteres `+` y `=` en el password deben URL-encodarse como `%2B` y `%3D`.
- Correr migraciones SIEMPRE desde el directorio del servicio (`cd svc-vivienda/`). Alembic usa `alembic.ini` que tiene rutas relativas.
- DB-first para roles es mejor que JWT custom claims a la escala de este sistema: cambios de rol son inmediatos y el overhead es despreciable.
- Antes de implementar, leer los ADRs en `docs/files/arquitectura.md`. Si una orden contradice un ADR, señalarlo antes de actuar.
- El usuario del Cloud SQL para `db_vivienda` es `user_vivienda`, no `svc_vivienda`. Los secrets en Secret Manager confirman el nombre correcto.
- El `npm run build` del frontend DEBE correrse desde la máquina Windows local donde se editaron los archivos. Cloud Shell tiene una copia independiente que puede estar desactualizada — deployar desde ahí sube código viejo sin avisar.
- `tsc -b` falla silenciosamente ante errores TypeScript estrictos (`noUnusedLocals`, etc.): la carpeta `dist/` mantiene el build anterior y Firebase lo despliega. Siempre verificar que el output diga `✓ built in X.XXs`.
- Para asyncpg + SQLAlchemy, las columnas `DATE` requieren `datetime.date`, no strings. En seed scripts: `date.fromisoformat(str_date)` antes de insertar.
