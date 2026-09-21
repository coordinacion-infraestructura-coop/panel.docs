# Spec: Sincronización Google Sheet "ATP - Compromiso Gobernador" → `svc-gralgob` (Fase 0)

**Estado**: approved
**Versión**: 1.1.0
**Servicio**: `svc-gralgob` (módulo de sync + panel preliminar de solo lectura, sin panel de negocio)
**Última actualización**: 2026-09-21

---

## Changelog

- **1.1.0** (2026-09-21): agrega §12 — panel de visualización de solo lectura
  (2 endpoints públicos + página frontend) sobre los datos ya sincronizados.
  **Excepción explícita y documentada** a la regla de `CLAUDE.md` ("no agregar
  `/api/v1/gralgob/**` sin `spec-svc-gralgob.md` `approved`") — mismo carve-out
  ya usado para `svc-gasifera` (`spec-sync-gasifera-pit.md §12`), extendido acá
  con el mismo sign-off explícito del usuario. Ver §12 para el detalle.
- **1.0.0** (2026-09-21): aprobado con el alcance descrito (espejo de solo
  lectura de la hoja `BD` únicamente, sin reglas de negocio nuevas, sin
  pantallas — ver §0/§2). Aprobación puntual de este spec angosto, **no**
  sustituye la reunión real con el área ni aprueba
  `docs/files/spec-svc-gralgob.md` (dominio completo, sigue en `draft`) — las
  preguntas abiertas del §10 siguen sin resolver. Mismo criterio que
  `spec-sync-gasifera-pit.md` para `svc-gasifera`.
- **0.1.0** (2026-09-21): borrador inicial.

## 0. Por qué existe este spec (y por qué es angosto)

Este spec cubre **solo** el espejo de solo lectura de la hoja `BD` del Sheet
"ATP - Compromiso Gobernador" — mismo alcance y mismo criterio que tuvo
`spec-sync-gasifera-pit.md` para Gasifera antes de que exista un panel de
negocio. **No sustituye la reunión real con el área** (pendiente, ver
`docs/context/areas/README.md`) ni el spec de dominio completo
(`docs/files/spec-svc-gralgob.md`, también draft) — no interpreta reglas de
negocio nuevas, no agrega pantallas, solo refleja 1:1 lo que ya existe en el
Sheet para poder empezar a construir sobre datos reales.

## 1. Propósito

La Secretaría General de Gobierno gestiona el programa **ATP (Aporte del
Tesoro Provincial)** — compromisos de fondos anunciados por el Gobernador a
localidades de la provincia — en un Google Sheet ("ATP - Compromiso
Gobernador", copia local analizada:
`docs/context/areas/secretaria_Gral_Gobierno/ATP - Compromiso Gobernador.xlsx`,
Sheet real id `1x3E73hlGwDPBiLc9fbQuWxWUxjaibs3glModHsxCMZM`), uno de al menos
16 pestañas que mantiene el área (ver `contexto_detallado.md §2`). El objetivo
de esta feature es traer al sistema, mediante sincronización de solo lectura,
los compromisos ATP y su cronograma de pago, como primer paso hacia
reemplazar la carga en el Sheet por un ABM propio — igual criterio que se
usó con Checklist Técnico DGV y con el sync de Gasifera.

Este es además **el primer módulo del nuevo servicio `svc-gralgob`**
(Secretaría General de Gobierno), que no existía en el plan original —
ver la actualización de alcance en el `CLAUDE.md` raíz y en
`docs/files/roadmap.md`.

## 2. Alcance

### Incluido
- Lectura vía Google Sheets API v4 de **una única pestaña**: `BD` (headers en
  fila 5; filas 1-4 son un resumen `TOTAL ANUNCIADO/PAGADO/SALDO`, no datos).
- Upsert idempotente hacia 3 tablas nuevas en `db_gralgob` (base de datos
  propia, ADR-001).
- Un único endpoint interno IAM-protegido que dispara la sincronización.
- Log de cada corrida (filas leídas/insertadas/actualizadas/con error).

### Fuera de alcance (v1.0.0)
- Escritura hacia el Sheet (unidireccional: Sheet → Postgres).
- Cualquier pantalla o endpoint de negocio (`/api/v1/gralgob/**`) — eso es el
  spec de dominio completo (`spec-svc-gralgob.md`), que requiere la reunión
  real con el área.
- `app/auth.py` / integración con `portal_usuarios` — no hace falta en esta
  fase porque no hay ningún endpoint alcanzable por un usuario final.
- La hoja `Estado de exp` (seguimiento de expedientes — entidad distinta, ver
  `contexto_detallado.md §2.2`).
- Los catálogos de soporte del propio Sheet (`Validadores`, `Datos
  Localidad`, `DPTO`) — `Datos Localidad` en particular duplicaría el padrón
  canónico ya propiedad de `svc-privada` (`priv_localidades_info`/
  `priv_departamentos_info`, ADR-012); si `svc-gralgob` necesita esos datos a
  futuro, se leen read-only sobre el gateway, no se vuelven a sincronizar acá.
- Los pivots/copias derivadas de `BD` (`BD ORDENADA - Gobierno/Por
  ministerio/POR MES`, `Copia de BD`, `Hoja 31`) — vistas, no fuente.
- Los otros programas de fondos que conviven en el mismo libro (`ATP
  ACUMULADO`, `COPA+FOFINDES`, `SALDO`, `BD NATALIO`) — no son
  "ATP - Compromiso Gobernador", pertenecen conceptualmente a specs propios
  si el área confirma que son programas vigentes a sistematizar.
- Federación de estas líneas hacia `resumen_territorial` — requiere un ADR y
  spec propios (mismo patrón que ADR-016 hizo para Privada), no es parte de
  esta entrega aunque sea el objetivo final declarado por el usuario.
- Deploy real (Cloud SQL, Cloud Run, Cloud Scheduler, IAM) — se hace en una
  sesión aparte, explícitamente, con la skill `/deploy-servicio`.

## 3. Fuente de datos

Analizada con `openpyxl` sobre la copia `.xlsx` provista. Ver el análisis
completo referenciado desde
`docs/context/areas/secretaria_Gral_Gobierno/contexto_detallado.md`.

### 3.1 Hoja `BD` (headers fila 5, datos filas 6-1115 al momento del análisis, ~1110 compromisos)

| Columna Sheet | Campo destino | Tipo / notas |
|---|---|---|
| `#` (col. A, header en blanco) | — | índice manual del área, **no se sincroniza** (no es estable como clave, ver §4) |
| DEPARTAMENTO | `departamento` | texto; 24 valores distintos |
| LOCALIDAD | `localidad` | texto; 246 valores distintos |
| Ministerio | `ministerio_destino` | texto; área a la que se derivó la ejecución del compromiso. `"Gobierno"` (69% de las filas) = queda dentro de la propia Secretaría. Se normaliza solo espacio en blanco (colapsar dobles espacios) — **no se corrigen typos de nombre de área** (`"Biagroindustria"`, `"Habitat y desarrollo emprendendor"`) sin confirmar con el área la lista cerrada real (ver `contexto_detallado.md §7.2`) |
| Fecha de anuncio | `fecha_anuncio` | DATE; ~19% de las filas sin fecha |
| Nro. Expediente | `nro_expediente` | texto; mayormente vacío |
| Derivado | `derivado` | booleano (checkbox nativo del Sheet) |
| Monto | `monto` | NUMERIC(18,2); ~17% de las filas sin monto cargado |
| Destino | `destino` | texto libre (descripción de la obra/uso) |
| SALDO ATP | `saldo_atp` | NUMERIC(18,2); **valor calculado por una fórmula `FILTER` del Sheet, se sincroniza el resultado ya calculado, no se reimplementa la fórmula** (ver 3.2) |
| `<NombreMes> <AA>` (34 columnas al momento del análisis: Marzo 25 → Diciembre 27, creciendo) | → `atp_cronograma_pagos` (tabla hija) | NUMERIC; negativo = pagado ese mes, mismo signo que trae el Sheet. El parseo de esta columna es **genérico por patrón de encabezado** (`"^<letras> <2 dígitos>$"`), no una lista fija de 34 nombres — si el área agrega una columna "Enero 28" a la derecha, se sincroniza sola sin requerir migración ni cambio de código (asume que el área solo agrega columnas nuevas a la derecha con ese mismo formato de encabezado, nunca inserta en el medio — ver pregunta abierta en `contexto_detallado.md §7.4`) |
| IGNORAR / CUOTA SIN ASIGNAR | — | columnas de ajuste/basura del Sheet, **no se sincronizan** (no matchean el patrón de columna mensual, se descartan automáticamente) |

### 3.2 Calidad de datos observada (el sync debe tolerar esto sin frenarse)

- **No hay clave de negocio única** en `BD`: el índice manual de la columna
  `#` no es estable ante reordenamientos, `Nro. Expediente` está vacío en la
  gran mayoría de las filas, y `(departamento, localidad, destino)` puede
  repetirse (una misma localidad puede tener varios compromisos con destinos
  parecidos). La clave natural del upsert es **`sheet_row_number`** — mismo
  criterio y mismo riesgo aceptado que `gas_pit_acciones_territorio`
  (sensible a que el área reordene filas manualmente).
- `SALDO ATP` es una fórmula Google Sheets (`FILTER`) que, al exportar a
  `.xlsx`, aparece como `__xludf.DUMMYFUNCTION(...)` con el último valor
  calculado cacheado — esto es un artefacto **solo del export a `.xlsx`**;
  leyendo vía Sheets API v4 (como hace este sync) se obtiene directamente el
  valor ya calculado, sin necesidad de manejar ese artefacto. La fórmula en
  sí fuerza `SALDO ATP = 0` cuando `Ministerio != "Gobierno"` — se mirror-ea
  tal cual, no se reinterpreta.
- `Ministerio` trae al menos 2 valores que parecen typos de nombres de área
  reales (`"Biagroindustria"`, `"Habitat y desarrollo emprendendor"`, y
  `"Secretaría de  Vivienda"` con doble espacio) — se normaliza únicamente
  espacio en blanco repetido; los nombres en sí **no se corrigen** sin
  confirmar la lista cerrada con el área.
- `Monto`/`SALDO ATP`/columnas mensuales pueden venir formateados con `$` y
  separador de miles `.`/decimal `,` (formato AR) si la API los devuelve como
  `FORMATTED_VALUE` — el parser numérico tolera ambos casos (con o sin
  formato), mismo criterio que `_parse_number` de `gas_pit/sync.py`.
- `Fecha de anuncio` puede venir como fecha nativa, string `DD/MM/AAAA`, o
  (raro) serial numérico de Excel — el parser tolera los tres casos, mismo
  criterio que `gas_pit/sync.py`.

## 4. Decisiones de arquitectura (confirmadas con el usuario)

1. **Nuevo servicio `svc-gralgob`** (Secretaría General de Gobierno, no
   existía en el plan original), no un módulo dentro de un servicio
   existente — mismo criterio que separó `svc-gasifera` de `svc-vivienda`.
2. **Cronograma de pago mensual normalizado en tabla hija**
   (`atp_cronograma_pagos`, 1 fila por compromiso×mes), no como 34+ columnas
   fijas en `atp_compromisos` — decisión explícita del usuario, prioriza
   consultabilidad para reportes/Resumen Territorial y evita que agregar un
   mes nuevo requiera migración.
3. **Clave natural del upsert**: `sheet_row_number` (no hay combinación de
   columnas de negocio confiable, ver §3.2).
4. **Autenticación al Sheet**: ADC (`google.auth.default()`), scope
   `spreadsheets.readonly` — mismo cliente que `svc-vivienda`/`svc-gasifera`
   (no se comparte código entre servicios, database-per-service implica
   deploy independiente).
5. **Disparo**: en producción, Cloud Scheduler vía HTTP + token OIDC (a
   configurar en el deploy real). En desarrollo, `curl` manual.
6. **Endpoint interno** en el mismo servicio HTTP —
   `POST /internal/sync/atp-compromiso-gobernador` **no se agrega a**
   `infra/gateway/openapi.yaml`. Único control de acceso en producción: IAM
   de Cloud Run (`--no-allow-unauthenticated` + `roles/run.invoker`).
   `svc-gralgob` en esta fase **no tiene `app/auth.py`** — no hay ningún
   endpoint alcanzable por un usuario final.
7. **Escritura idempotente**: `SELECT` por `sheet_row_number` → mutar o
   crear → `flush`, con `SAVEPOINT` por fila (`async with db.begin_nested()`)
   desde el primer commit — mismo criterio que `gas_pit/sync.py` (lección de
   `spec-sync-cc-checklist-tecnico.md §13.5`).
8. **Logging**: cada corrida registra contadores y detalle de errores en
   `atp_sync_log`, sin abortar el batch por errores de fila individuales.

## 5. Modelo de datos

Tablas nuevas, prefijo `atp_`, migración Alembic `0001` (primera migración
del servicio nuevo). Ver `services/svc-gralgob/app/atp/models.py` para el
detalle completo de columnas.

### 5.1 `atp_compromisos`
Snapshot actual (no histórico) de cada compromiso ATP. Columnas: `id`,
`sheet_row_number` (UNIQUE, clave natural), `departamento`, `localidad`,
`ministerio_destino`, `fecha_anuncio`, `nro_expediente`, `derivado`, `monto`,
`destino`, `saldo_atp` (mirror del valor calculado por el Sheet), `last_synced_at`.

### 5.2 `atp_cronograma_pagos`
Una fila por compromiso×mes con monto pagado/planificado. Columnas:
`compromiso_id` (FK a `atp_compromisos`, `ON DELETE CASCADE`), `periodo`
(DATE, primer día del mes), `monto`. `UNIQUE(compromiso_id, periodo)`.
Estrategia de upsert: `DELETE` + reinsert de las columnas mensuales vigentes
en cada corrida (mismo patrón que `gas_pit_obras_localidades`).

### 5.3 `atp_sync_log`
Una fila por corrida: `started_at`, `finished_at`, `filas_leidas`,
`filas_insertadas`, `filas_actualizadas`, `filas_error`, `errores` (JSON,
`[{"fila", "motivo"}]`), `triggered_by`.

## 6. Endpoint interno

```
POST /internal/sync/atp-compromiso-gobernador
GET  /internal/sync/atp-compromiso-gobernador/estado
```
Router `app/internal/router.py`, montado en `main.py` sin prefijo `/api/v1`
y sin `Depends(get_current_user)`. No se declara en
`infra/gateway/openapi.yaml`.

## 7. Cliente de Google Sheets

`app/integrations/google_sheets.py` — mismo cliente mínimo que
`svc-vivienda`/`svc-gasifera` (no se comparte código entre servicios).
`spreadsheet_id` y el rango van en `config.py` (`Settings`), no
hardcodeados. Rango configurado con margen amplio de columnas
(`BD!A5:EZ5000`) para tolerar que el cronograma mensual siga creciendo sin
requerir un cambio de configuración cada pocos meses.

## 8. Tests

- `tests/test_atp_sync.py`: inserta un compromiso completo, parsea el
  cronograma mensual desde encabezados genéricos (incluye un mes fuera del
  rango "conocido" para probar que el parseo por patrón funciona sin
  hardcodear), tolera fila en blanco, idempotencia (UPSERT no duplica),
  aislamiento por `SAVEPOINT` (regresión forzando un `IntegrityError` real en
  la fila del medio de un batch), falla total de lectura → `SheetReadError` +
  log.
- `tests/test_internal_router.py`: el endpoint responde 200 con el resumen
  de la corrida y no requiere JWT; 502 si falla la lectura del Sheet.

## 9. Criterios de aceptación (esta entrega)

- [x] Servicio `svc-gralgob` nuevo, scaffold completo (`pyproject.toml`,
      `app/`, `alembic/`, `tests/`, `docker-compose.dev.yml`, `Dockerfile`,
      `README.md`).
- [x] Migración `0001` crea las 3 tablas nuevas.
- [x] `pytest` en verde sobre SQLite in-memory, sin requerir Postgres (14/14).
- [x] `POST /internal/sync/atp-compromiso-gobernador` no usa
      `Depends(get_current_user)`; no se declara en `openapi.yaml`.
- [x] Corridas repetidas no duplican filas (UPSERT verificado en tests **y**
      en producción — ver corrida real más abajo).
- [x] Una fila con error real de DB no frena el resto del batch ni impide el
      log final (test de regresión con `IntegrityError` forzado).
- [x] Migración `0001` verificada contra Postgres real: local (Docker,
      `docker-compose.dev.yml`, `upgrade head` / `downgrade base` / `upgrade
      head` limpios, constraints y `ON DELETE CASCADE` verificados con `\d`)
      **y contra `db_gralgob` en Cloud SQL real** (vía `cloud-sql-proxy`
      local con `--gcloud-auth`, sin tocar el ADC compartido de la máquina
      que otra sesión usa para `svc-gasifera`).
- [x] Deploy real a Cloud Run (2026-09-21, `gcloud run deploy --source .`,
      revisión `svc-gralgob-00001-b7r`, 100% tráfico, servicio en
      `https://svc-gralgob-276787280674.southamerica-east1.run.app`,
      `--no-allow-unauthenticated` — verificado: sin token da 403, con
      identity token da 200). Infra creada en esta sesión: SA
      `svc-gralgob@gestorcooperativo.iam.gserviceaccount.com` (roles
      `cloudsql.client`, `secretmanager.secretAccessor`, `pubsub.publisher`
      — mismo set que `svc-gasifera`), `db_gralgob` + `user_gralgob` en
      `ministerio-postgres`, secret `svc-gralgob-db-url`.
- [x] **Corrida real contra el Sheet en vivo** (2026-09-21, vía el endpoint
      recién desplegado): 1104 filas leídas, 1104 insertadas, 0 errores. El
      Sheet ya estaba accesible para la identidad de runtime de Cloud Run sin
      pasos adicionales de compartición. Segunda corrida inmediata para
      verificar idempotencia: 0 insertadas, 1104 actualizadas, 0 errores —
      sin duplicados.
- [x] **Cloud Scheduler configurado** (2026-09-21): job
      `sync-atp-compromiso-gobernador` (`southamerica-east1`, `0 * * * *`,
      cada hora) invoca el endpoint vía OIDC usando la propia identidad del
      servicio (`svc-gralgob@...` con `roles/run.invoker` sobre sí mismo —
      mismo patrón que `sync-cc-checklist-tecnico` en `svc-vivienda`).
      **Detalle no obvio**: el binding `run.invoker` en el Cloud Run no
      alcanza por sí solo — el agente de servicio de Cloud Scheduler
      (`service-{project_number}@gcp-sa-cloudscheduler.iam.gserviceaccount.com`)
      necesita además `roles/iam.serviceAccountTokenCreator` **sobre la
      propia SA `svc-gralgob@...`** para poder emitir el token OIDC en su
      nombre; sin ese segundo binding la corrida falla con 403 aunque el
      `run.invoker` esté bien puesto (diagnosticado y corregido en esta
      sesión). Corrida automática real verificada:
      `triggered_by: "cloud-scheduler"`, 1104 filas, 0 errores.
- [ ] `infra/gateway/openapi.yaml` — no aplica en esta fase (endpoint
      IAM-only, sin exposición pública).

## 10. Pendiente / preguntas abiertas para la reunión con el área

Ver `docs/context/areas/secretaria_Gral_Gobierno/contexto_detallado.md §7`
para el detalle completo. Las más relevantes para este sync en particular:

1. ¿Cuál es la lista cerrada real de ministerios/secretarías destino (para
   poder tratar los valores con typo como alias de un valor conocido)?
2. ¿`Ministerio != "Gobierno"` y `Derivado = true` son siempre equivalentes?
3. ¿El área reordena manualmente las filas de `BD`? (afecta la estabilidad
   de la clave `sheet_row_number`, mismo riesgo que `gas_pit_acciones_territorio`).
4. ¿El único patrón de crecimiento del cronograma es agregar columnas nuevas
   a la derecha con encabezado `"<Mes> <AA>"`, o el área a veces inserta
   columnas en otro lugar?
5. Alcance real del archivo: ¿"ATP - Compromiso Gobernador" es solo `BD`, o
   el sistema debería terminar cubriendo también `Estado de exp` y/o los
   otros programas de fondos del mismo libro?

## 12. Panel de visualización de solo lectura (agregado 2026-09-21, v1.1.0)

### 12.1 Por qué esto es una excepción documentada, no una violación silenciosa

`CLAUDE.md` (raíz) tiene la regla: *"Don't add `/api/v1/gralgob/**` ... endpoints
without that domain spec [`spec-svc-gralgob.md`] going `approved` first."* — y,
tras el carve-out de Gasífera, una nota explícita: *"Don't extend that carve-out
to writes, other resources, or `svc-gralgob` without the same explicit
sign-off."* Antes de implementar, se le señaló el conflicto al usuario
(siguiendo el hard rule de Spec Driven Development), explicando en qué consiste
el mismo camino ya recorrido para `svc-gasifera` (`spec-sync-gasifera-pit.md
§12`). El usuario, con conocimiento del conflicto, **pidió explícitamente
seguir ese mismo camino** ("sigamos ese mismo camino 'preliminar de solo
lectura'"). Mismo criterio de acotamiento que ya usa este spec para el sync en
sí (§0): endpoints de **solo lectura**, sobre datos que **ya están
sincronizados** (`atp_*`, ya cubiertos por este spec `approved`), **sin
interpretar ninguna regla de negocio nueva**. No autoriza ningún endpoint de
escritura, catálogo administrable, ni nada que dependa de las preguntas
todavía abiertas del §10.

### 12.2 Alcance

**Incluido**: 2 endpoints GET públicos (vía API Gateway, con auth JWT estándar)
que leen `atp_compromisos` tal cual está, más una página frontend que los
muestra. La única cifra que no viene 1:1 del Sheet es `total_pagado` — un
`SUM(atp_cronograma_pagos.monto)` agrupado por compromiso, no una regla de
negocio nueva (mismo criterio que el `SUM` de KPIs del Tablero PIT Gas). Sin
filtros de negocio nuevos más allá de los de UI (departamento/localidad/
ministerio destino, client-side).

**Fuera de alcance**: cualquier escritura, el panel de negocio completo
(`spec-svc-gralgob.md`, sigue `draft`), el detalle del cronograma de pago mes
a mes (`atp_cronograma_pagos` se usa solo para el agregado `total_pagado`, no
se expone fila por fila en esta versión — se puede agregar después si se pide,
mismo criterio de iterar sobre lo mínimo que ya se usó con Gasífera),
resolución de localidad/departamento contra un catálogo canónico (se muestra
el texto crudo del Sheet), gráficos/mapas (primera versión solo KPI strip +
tabla filtrable, igual que terminó el Tablero PIT Gas).

### 12.3 Auth (nuevo — Fase 0 no tenía ningún endpoint público)

`app/auth.py` nuevo, calcado de `services/svc-gasifera/app/auth.py` (ADR-015)
— `svc-gralgob` **no se conecta a `db_vivienda`**, resuelve rol + secretarías
llamando a `GET {SVC_VIVIENDA_INTERNAL_URL}/internal/portal/usuarios/{email}`
con un ID token (audience = esa URL), degradando a rol `invitado` ante
cualquier falla (nunca 500). Roles: `ROLES_LECTURA = ("Admin","Supervisor",
"Operador","Consulta")` + pertenencia a la secretaría `"gralgob"` — agregada a
`SECRETARIAS_VALIDAS` en `services/svc-vivienda/app/portal/schemas.py` (no
estaba, a diferencia de `"gasifera"` que ya estaba presente) y a la lista de
checkboxes de `AdminUsuariosPage.tsx`. Sin roles acotados nuevos tipo
`TecnicoDGV`/`Autoridad`.

### 12.4 Endpoints

```
GET /api/v1/gralgob/compromisos?limit=1200&offset=0
GET /api/v1/gralgob/sync-estado
```
Router nuevo `app/atp/router.py`, montado con prefijo `/api/v1/gralgob` en
`main.py`. Los 2 requieren `Depends(get_current_user)` + rol de lectura +
secretaría `gralgob`. `sync-estado` envuelve `atp_sync.get_last_sync_status`
(mismo dato que el endpoint interno de sync, expuesto de forma pública y de
solo lectura para mostrar "Sincronizado hace X" en el panel).

### 12.5 Gateway e IAM (deploy real, pasos separados — ver criterios de aceptación)

Se agregan a `infra/gateway/openapi.yaml` (patrón idéntico a
`/api/v1/gasifera/**`) + nueva config de gateway. Dos grants IAM nuevos:
`svc-gralgob@` necesita `roles/run.invoker` sobre `svc-vivienda` (para el
lookup de auth); `api-gateway-sa@` necesita `roles/run.invoker` sobre
`svc-gralgob` (para poder enrutarle tráfico).

### 12.6 Criterios de aceptación

- [x] `app/auth.py` + tests (mock de `get_current_user` vía
      `dependency_overrides`, casos Consulta/sin-secretaría/invitado/sin-token
      — 22/22 tests en verde incluyendo los 8 nuevos de `test_atp_router.py`).
- [x] `app/atp/router.py` con los 2 endpoints + `response_model`.
- [x] `"gralgob"` agregado a `SECRETARIAS_VALIDAS`
      (`services/svc-vivienda/app/portal/schemas.py`) — suite completa de
      `svc-vivienda` verificada en verde (289/289) tras el cambio.
- [x] Frontend: `src/modules/gralgob/` (API client + `AtpPage.tsx`),
      activación en `DashboardPage.tsx`/`Layout.tsx`/`App.tsx`/
      `AdminUsuariosPage.tsx`. `npm run build` verde.
- [x] Redeploy de `svc-gralgob` con el código nuevo (2026-09-21, revisión
      `svc-gralgob-00002-2f4`, 100% tráfico, `SVC_VIVIENDA_INTERNAL_URL`
      agregado a las env vars).
- [x] IAM: los 2 `run.invoker` otorgados — `svc-gralgob@` sobre `svc-vivienda`
      (lookup de auth) y `api-gateway-sa@` sobre `svc-gralgob` (ruteo del
      gateway).
- [x] Gateway actualizado con los 2 paths nuevos, config
      `ministerio-config-v20260921b` activa. **Nota de proceso**: la config se
      generó a propósito **sin** los paths de `svc-gasifera` (también
      pendientes de deploy, trabajados en otra sesión en paralelo) — se
      excluyeron de una copia temporal del `openapi.yaml` usada solo para
      generar esta config puntual, sin tocar el archivo real del repo ni
      avanzar el deploy de Gasífera sin permiso. Verificado:
      `GET /api/v1/gralgob/compromisos` sin token → `401 Jwt is missing` (no
      404 — confirma que el path está bien enrutado), vía
      `https://ministerio-gateway-3j5k00ma.uc.gateway.dev`.
- [x] Frontend deployado a producción (Firebase Hosting,
      `https://gestorcooperativo.web.app`). **Nota de proceso**: el build de
      producción se generó con los cambios de Gasífera apartados
      temporalmente (`git stash`) para no desplegar ese trabajo en curso sin
      permiso — verificado con `grep` sobre el bundle que no incluía
      `GasiferaPitPage`/`gasifera/pit` antes del deploy, y los cambios de
      Gasífera se restauraron (sin commitear) inmediatamente después.
- [ ] Verificación end-to-end en navegador con un usuario real: falta que un
      Admin le asigne la secretaría `gralgob` a un usuario de prueba desde
      `AdminUsuariosPage` y confirme visualmente el panel — no se hizo desde
      esta sesión para no usar la cuenta de test compartida de
      `agentes_test/` contra producción (esa cuenta es solo para QA contra
      backend local, ver `CLAUDE.md` raíz).
