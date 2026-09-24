# Spec: Sincronización Google Sheet "ATP - Compromiso Gobernador" → `svc-gralgob` (Fase 0)

**Estado**: approved
**Versión**: 1.6.0
**Servicio**: `svc-gralgob` (módulo de sync + panel preliminar de solo lectura, sin panel de negocio)
**Última actualización**: 2026-09-23

---

## Changelog

- **1.6.0** (2026-09-23): agrega §14 — notificación a svc-vivienda cuando el
  sync detecta un compromiso NUEVO (ADR-023). Ver §14.
- **1.5.0** (2026-09-23): agrega §13 — rollup territorial y federación
  server-side a `resumen_territorial` de svc-vivienda (ADR-022, mismo patrón
  que ADR-016/ADR-021 usaron para Privada/Gasífera). Ver §13.
- **1.4.0** (2026-09-23): agrega §12.9 — vinculación manual confirmada para
  las 17 localidades del §12.8 sin match automático. 16 resueltas contra
  `viv_geo_localidades` real (typos/abreviaturas/formato), 1 documentada
  como pendiente real (falta en el padrón). Ver §12.9.
- **1.3.0** (2026-09-22): agrega §12.8 — cruce de Departamento/Localidad
  contra el padrón geográfico canónico `viv_geo_localidades`, 100%
  client-side sobre un endpoint público ya existente de `svc-vivienda`, sin
  cambios de backend en `svc-gralgob`. Ver §12.8.

- **1.2.0** (2026-09-22): agrega §12.7 — rediseño del panel tras probarlo con
  datos reales (filtro de Localidad no dependía del Departamento elegido,
  layout poco funcional) + endpoint nuevo
  `GET /compromisos/{id}/cronograma` para el panel lateral de detalle por
  localidad. Ver §12.7.
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

## 12.7 Rediseño del panel (2026-09-22)

### Por qué

El usuario probó la v1 del panel (KPI strip + tabla plana con filtros
Departamento/Localidad/Ministerio destino, todos independientes) y reportó
dos problemas concretos:

1. **"Los filtros no funcionan bien, no filtra correctamente las
   localidades"** — causa real: el desplegable de Localidad listaba las 246
   localidades de toda la provincia sin filtrar por el Departamento ya
   elegido. Elegir una combinación Departamento+Localidad que no coexiste en
   los datos reales devolvía 0 resultados, lo que se percibía como "el
   filtro está roto" cuando en realidad la lógica de filtrado (AND de ambas
   condiciones) siempre fue correcta — el problema era la falta de cascada
   en las *opciones* del desplegable.
2. **Estética/funcionalidad**: pidió ver en la tabla principal el monto del
   compromiso y lo entregado hasta el momento, y — copiando el esquema de
   `CordonCunetaPage.tsx` en `svc-vivienda` — que un clic en la localidad
   abra un panel lateral con el detalle de las entregas de dinero y sus
   fechas.

Mismo patrón que el pedido de rediseño que tuvo el Tablero PIT Gas de
`svc-gasifera` el mismo día (`spec-sync-gasifera-pit.md §12.7`): primera
versión mínima, ajuste real una vez que el usuario la prueba con datos de
producción.

### Qué cambió

- **Filtro Localidad en cascada**: sus opciones ahora se recalculan a partir
  del Departamento seleccionado (`compromisos.filter(c => c.departamento ===
  deptoFilter)` antes de derivar el set de localidades); si el Departamento
  cambia y la Localidad elegida deja de pertenecer a él, se limpia
  automáticamente. El filtro Ministerio destino queda independiente (no es
  una jerarquía geográfica).
- **Columnas de la tabla principal**: se agrega `N° Expediente` (ya estaba
  sincronizado y expuesto por la API, pero no se mostraba). Se reemplaza la
  columna cruda `Saldo ATP` (la fórmula del Sheet la fuerza a 0 para
  compromisos derivados, ver §3.2) por dos columnas calculadas en el
  frontend a partir de `total_pagado` (ya expuesto desde v1): `Entregado`
  (`abs(total_pagado)`) y `Pendiente` (`monto - Entregado`), útiles para
  **todas** las filas, no solo las que quedan en "Gobierno". Nuevos KPIs:
  "Entregado a la fecha" y "Pendiente de entrega" (sumas sobre todo el
  dataset, no solo lo filtrado).
- **Panel lateral por localidad** (nuevo, mismo esquema visual que el
  `DetailPanel` de `CordonCunetaPage.tsx` — header navy, botón cerrar,
  timeline con punto+línea): al hacer clic en la Localidad de una fila se
  abre con el resumen del compromiso (expediente, fecha, monto, ministerio
  destino, entregado, pendiente, destino, alerta si está derivado) y un
  timeline de "Entregas de dinero" (mes/año + monto), cargado bajo demanda
  vía el endpoint nuevo del §12.4 — **no** se trae el cronograma completo de
  los ~1100 compromisos en el listado principal, solo el del compromiso que
  se abre.

### Endpoint nuevo

```
GET /api/v1/gralgob/compromisos/{compromiso_id}/cronograma
```
Ver §12.4 (agregado ahí). 404 si el compromiso no existe; lista vacía
(`200 []`) si existe pero no tiene cronograma cargado. Mismo auth que el
resto del panel (`require_gralgob(*ROLES_LECTURA)`).

### Deploy

Redeploy de `svc-gralgob` (revisión con el router actualizado), gateway
actualizado a `ministerio-config-v20260922` (agrega
`/api/v1/gralgob/compromisos/{compromiso_id}/cronograma`, GET + OPTIONS) y
frontend redeployado a `gestorcooperativo.web.app`. **Detalle técnico no
obvio**: la primera creación de esta config falló con
`INVALID_ARGUMENT: ... undefined field 'compromiso_id' on message
google.protobuf.Empty` — el bloque `options:` de un path con parámetro
(`{compromiso_id}`) también necesita declarar ese `parameters: - in: path`,
no solo el `get:` (mismo patrón ya usado en
`/api/v1/privada/gestiones/{gestion_id}`); si falta, la traducción a
gRPC/HTTP transcoding no encuentra dónde mapear el parámetro del path para
esa operación. Corregido y verificado (config activa, `GET .../cronograma`
sin token → 401, no 404).

**Nota de proceso** (igual criterio que el resto de esta sesión): tanto la
config de gateway como el build de frontend se generaron excluyendo
temporalmente el trabajo en curso de `svc-gasifera` (otra sesión en
paralelo) — copias temporales sin ese código, verificadas con `grep`/diff
antes de cada deploy, y el trabajo de Gasífera restaurado sin commitear
inmediatamente después en los 4 repos afectados.

## 12.8 Cruce contra el padrón geográfico canónico (2026-09-22)

### Por qué

Pedido explícito del usuario en el planning original: "Departamento y
localidad (luego cruzarlo con nuestra base geo_localidades)". El Sheet trae
texto libre tipeado a mano por el área — mismo riesgo de calidad de dato que
`SPIP`/enums casi-duplicados en Gasífera — así que conviene detectar cuándo
una localidad no coincide con el catálogo canónico antes de usarla para
federar a Resumen Territorial (que busca por localidad).

### Qué se cruza y contra qué

`viv_geo_localidades` (propiedad de `svc-vivienda`, ADR-001) — no se duplica
el catálogo en `db_gralgob` ni se agrega un endpoint interno IAM-only nuevo
para esto. En cambio, el frontend llama directamente al endpoint público que
ya existe para el propio uso de Cordón Cuneta/Córdoba Hogar:

```
GET /api/v1/vivienda/cordon-cuneta/geo
```

Este endpoint gatea por **rol** (`ROLES_LECTURA` = Admin/Supervisor/Operador/
Consulta), **no por secretaría** — así que un usuario con la secretaría
`gralgob` asignada (sin `vivienda`) también puede leerlo. Verificado
revisando `app/cordon_cuneta/router.py`/`app/auth.py` de `svc-vivienda`
(usa `require_roles(*ROLES_LECTURA)`, no un chequeo de secretaría). Se
decidió no agregar un endpoint interno IAM-only nuevo (mismo patrón que
`priv_localidades_info`/ADR-012) porque el catálogo ya es de solo lectura,
público, y estático (cambia poquísimo) — el costo de una llamada cross-
secretaría extra a un endpoint que ya existe es menor que el de coordinar un
cambio en `svc-vivienda` para esto.

### Cómo matchea

`src/modules/gralgob/pages/AtpPage.tsx`: matching por nombre normalizado
(`shared/utils/normalizeName.ts`, mismo criterio que `app/geo/matching.py`
del backend — sin acentos, minúsculas). Para cada compromiso:
- **`ok`**: la localidad existe en el padrón y coincide también el
  departamento.
- **`depto-distinto`**: la localidad existe en el padrón pero bajo otro
  departamento al cargado en `BD`.
- **`sin-match`**: la localidad no aparece en absoluto en el padrón.
- **`sin-dato`**: el compromiso no tiene localidad cargada (no es un error
  de cruce, no se marca).

**No** resuelve alias entre paréntesis ni nombres separados por guion como sí
hace `candidatos_localidad()` del lado del backend (`app/geo/matching.py`) —
es un primer cruce best-effort para exponer problemas de calidad de dato, no
una normalización exhaustiva. Si el volumen de falsos positivos resulta alto
en la práctica, extenderlo es la siguiente iteración natural.

### UI

Ícono "!" con tooltip junto al nombre de la localidad (solo para
`depto-distinto`/`sin-match`, no para `ok`/`sin-dato`), mismo aviso repetido
en el panel lateral de detalle, KPI nuevo "Sin coincidencia en el padrón geo"
y un filtro (checkbox) para aislar esas filas y facilitar que el área
corrija el Sheet.

### Alcance

100% frontend, sin cambios de backend en `svc-gralgob` ni en `svc-vivienda`,
sin nuevo deploy de servicios — solo el redeploy de frontend ya cubierto en
"Deploy" más arriba. No resuelve el dato en la base (`atp_compromisos` sigue
guardando el texto crudo del Sheet) — el cruce es puramente de presentación,
para ahora. Si más adelante hace falta persistir el `id_geo` resuelto (por
ejemplo, para la federación a Resumen Territorial), es un cambio de sync en
`svc-gralgob` que requeriría que `svc-gralgob` sí lea `viv_geo_localidades`
desde el backend (vía un endpoint interno IAM-only nuevo de `svc-vivienda`,
mismo patrón que ADR-015/ADR-016) — no implementado, evaluar cuando se
aborde esa federación.

## 12.9 Vinculación manual de las localidades sin match (2026-09-23)

### Investigación

Con los 57 falsos-positivo de matching resueltos (§12.8: abreviaturas de
departamento, alias entre paréntesis/guion), quedaban 17 discrepancias
reales. Se investigó cada una contra `viv_geo_localidades` real (consulta
directa a `db_gralgob`/`db_vivienda` vía `cloud-sql-proxy`, solo lectura,
sin tocar producción) para determinar si eran typos/variantes de nombre o
localidades genuinamente ausentes del padrón.

**Resultado — 16 de 17 tenían coincidencia real**, todas confirmadas con el
usuario:

| ATP (Sheet) | Depto | Padrón real (`viv_geo_localidades`) | Motivo |
|---|---|---|---|
| Paso del Durazno | Juárez Celman (Sheet) | Río Cuarto (`id_geo=443`) | límite departamental — el usuario confirmó usar el depto del padrón oficial |
| Nicolás Bruzzone | Gral Roca | `Nicolas Bruzone` (`id_geo=55`) | una sola "z" |
| Huanchilla | Juárez Celman | `Huanchillas` (`id_geo=92`) | plural |
| Capitán General Bernardo O'Higgins | Marcos Juárez | `Cap. Gral. B.Ohiggins` (`id_geo=103`) | abreviado |
| Colonia Barge | Marcos Juárez | `Castro Urdiales - Colonia 25 de Mayo` (`id_geo=105`) | nombre distinto (confirmado por el usuario, conocimiento local) |
| General Levalle | Pte Roque Saenz Peña | `General Le Valle` (`id_geo=128`) | con espacio |
| Villa Río Icho Cruz | Punilla | `Icho Cruz` (`id_geo=158`) | nombre más corto |
| La Carolina El Potosí | Río Cuarto | `La Carolina (El Potosí)` (`id_geo=170`) | mismo dato, el padrón usa paréntesis y el Sheet no (el alias genérico de §12.8 no lo detecta porque el paréntesis está del lado del padrón sin separador) |
| Las Peñas Sud | Río Cuarto | `Las Peñas Sur` (`id_geo=175`) | Sud/Sur |
| Santa Catalina Holmberg | Río Cuarto | `Santa Catalina (Est. Holmberg)` (`id_geo=182`) | mismo motivo que La Carolina |
| Montecristo | Río Primero | `Monte Cristo` (`id_geo=392`) | con espacio |
| Villa de María | Río Seco | `Villa de Maria de Rio Seco` (`id_geo=221`) | nombre completo |
| San Javier y Yacanto | San Javier | `San Javier` (`id_geo=261`) | el padrón no lista "Yacanto" aparte — el usuario confirmó que es la forma abreviada de la misma localidad |
| Miramar de Ansenuza | San Justo | `Miramar` (`id_geo=289`) | nombre más corto |
| Saturnino María Laspiur | San Justo | `Saturnino M. Laspiur` (`id_geo=296`) | abreviado |
| Dalmacio Vélez | Tercero Arriba | `Dalmacio Velez Sarsfield` (`id_geo=324`) | nombre completo |
| James Craik | Tercero Arriba | `James Craick` (`id_geo=327`) | con "c" |

**1 de 17 queda sin vincular, documentada como pendiente real**: **Santiago
Temple** (Río Segundo) — localidad real conocida que **falta directamente**
en `viv_geo_localidades` (se revisó el departamento completo, no aparece
bajo ningún nombre similar). No es un problema del Sheet de ATP ni de este
panel — el usuario confirmó que debería estar en el padrón y que se
documenta para que el área de Vivienda la agregue más adelante. **No se
inserta en `viv_geo_localidades` desde acá** (esa tabla es propiedad de
`svc-vivienda`, fuera del alcance de `svc-gralgob`).

### Implementación

`VINCULACION_MANUAL` en `AtpPage.tsx`: `Record<string_normalizado, id_geo>`.
`matchGeo()` la consulta primero (antes del matching algorítmico genérico) —
si la localidad del compromiso tiene una entrada ahí, se resuelve directo
por `id_geo` sin depender de que el nombre coincida en absoluto. Santiago
Temple no está en el mapa a propósito, para que el panel lo siga marcando
como "sin match" hasta que exista de verdad en el padrón.

### Alcance

100% frontend, mismo criterio que §12.8 — sin cambios de backend, sin
persistir el `id_geo` resuelto en `atp_compromisos` (sigue siendo una
vinculación de presentación, no de datos). Si más adelante se persiste
(por ejemplo para la federación a Resumen Territorial), este mapa es el
punto de partida natural para poblar esa migración.

## 13. Rollup territorial — federación a `resumen_territorial` (agregado 2026-09-23, v1.5.0, ADR-022)

### 13.1 Endpoint

```
GET /internal/atp/rollup-territorial
```
`app/internal/router.py`, IAM-only (sin `Depends(get_current_user)`, no declarado en `infra/gateway/openapi.yaml`) — mismo criterio que el resto de este router. Consumido exclusivamente por `svc-vivienda` (`app/resumen_territorial/service.py::fetch_atp_lineas`), nunca por el frontend.

### 13.2 Qué agrega

`app/atp/rollup.py::rollup_territorial(db)` — `GROUP BY UPPER(TRIM(departamento)), UPPER(TRIM(localidad))` sobre `atp_compromisos` + un `outerjoin` a la suma agrupada de `atp_cronograma_pagos` (calcado de `gas_pit_rollup.rollup_territorial`, ADR-021). Por fila: `departamento`, `localidad`, `total_compromisos`, `derivados` (`derivado = true`), `monto_total_sum`, `entregado_sum` (valor absoluto de la suma del cronograma — el signo se mirror-ea del Sheet, negativo = pagado, mismo criterio que `total_pagado` en `atp/sync.listar_compromisos`), `fecha_max` (`MAX(fecha_anuncio)`).

### 13.3 Por qué texto normalizado y no el `id_geo` de la vinculación manual (§12.9)

`VINCULACION_MANUAL` (§12.9) es 100% frontend, sólo para el panel de solo lectura — no persiste `id_geo` en `atp_compromisos`, así que el rollup del lado backend no tiene ese dato disponible. Mismo criterio que ADR-021 confirmó para Gasífera: ni Vivienda ni Privada usan hoy `id_geo` como llave de join para el matching territorial en `resumen_territorial` — agrupar por texto normalizado `(departamento, localidad)` es consistente con el resto del pipeline, no una excepción. Las ~17 localidades que necesitaron vinculación manual en el panel de ATP (§12.9) van a aparecer en `resumen_territorial` bajo el texto tal cual está en el Sheet, no bajo el nombre oficial del padrón — mismo riesgo de colisión ya documentado en ADR-016/`spec-resumen-territorial-ficha-localidad.md` para las demás áreas.

### 13.4 IAM

`roles/run.invoker` de `svc-vivienda@gestorcooperativo.iam.gserviceaccount.com` sobre el servicio Cloud Run `svc-gralgob` — dirección **inversa** al grant de ADR-015 (`svc-gralgob@` invoker sobre `svc-vivienda`, para que Gralgob resuelva `portal_usuarios`). Otorgado 2026-09-23. Los dos grants coexisten sin conflicto (son sobre servicios distintos).

### 13.5 Criterios de aceptación

- [x] `app/atp/rollup.py` + endpoint interno + tests (`tests/test_atp_rollup.py`, 6 tests: agregación, entregado en valor absoluto, sin cronograma, vacío, fecha máxima, endpoint sin JWT) — suite completa de `svc-gralgob` (31 tests) en verde.
- [x] `svc-vivienda`: `fetch_atp_lineas`/`_map_atp_payload` (calcados de los de Gasífera) + `resumen_atp_estado`/`detalle_atp` en `aggregations.py` + tests (`tests/test_resumen_territorial.py`, 5 tests nuevos) — suite completa de `svc-vivienda` (299 tests) sigue en verde.
- [x] IAM real otorgado (`svc-vivienda@` invoker sobre `svc-gralgob`).
- [x] `services/cloudbuild.yaml`: sustituciones `_ATP_FETCH_ENABLED`/`_SVC_GRALGOB_INTERNAL_URL`.
- [x] Redeploy de `svc-gralgob` (revisión `svc-gralgob-00004-82z`, endpoint nuevo) y `svc-vivienda`
  (revisión `svc-vivienda-00166-tw9`, fetch nuevo + `ATP_FETCH_ENABLED=true`/`SVC_GRALGOB_INTERNAL_URL`
  agregados con `--update-env-vars`, sin tocar el resto del set de env vars).
- [x] Verificación end-to-end: `POST /internal/resumen-territorial/actualizar` en producción
  devolvió `"generado_para_areas":["vivienda","privada","gasifera","gralgob"]` — `"gralgob"` sólo
  se agrega si `fetch_atp_lineas()` trajo al menos una línea real, así que confirma el fetch +
  mapeo end-to-end contra datos reales. No se verificó visualmente una localidad puntual en el
  payload completo (requiere JWT de portal real, fuera de lo que esta sesión puede hacer sin
  credenciales) — pendiente de confirmación visual por un Admin, mismo criterio que el resto del
  panel ATP (spec §12.6).
- [ ] Frontend `ResumenTerritorialPage.tsx`: badge de área `gralgob` (label + color propio) — no evaluado en esta entrega, el badge se renderiza igual de genérico que las otras áreas hasta que se pida un color distintivo.

## 14. Notificación a svc-vivienda cuando hay un compromiso nuevo (agregado 2026-09-23, v1.6.0, ADR-023)

### 14.1 Qué dispara la alerta

Cada vez que `sync_from_sheet` detecta un compromiso **nuevo** (`is_new = True` en `_upsert_compromiso` — no una fila que ya existía y se actualizó), se llama, después de loguear la corrida en `AtpSyncLog` y fuera del `SAVEPOINT` por fila, a `POST /internal/notificaciones` de `svc-vivienda` (ADR-019) con:

```
"La localidad {localidad}, {departamento} fue visitada por el gobernador el día {fecha_anuncio}."
```

`destino_tipo="secretaria"`, `destino_valor="gralgob"`, `nivel="info"`, `origen="atp_sync"`. Si falta `localidad`/`departamento`/`fecha_anuncio`, se sustituye por `"(sin localidad)"`/`"(sin departamento)"`/`"(sin fecha)"` — nunca se omite el envío por datos parciales.

### 14.2 Por qué después del loop, no dentro del SAVEPOINT

Atar la llamada HTTP a `svc-vivienda` a la transacción de cada fila arriesgaría: (a) sumarle latencia de red al batch completo, fila por fila; (b) que un fallo de red dispare el `except Exception` del loop y la fila se registre como error aunque el upsert haya sido válido. Se acumulan las filas nuevas en una lista (`localidad`, `departamento`, `fecha_anuncio`) durante el loop y se notifican todas al final, ya con el log de sync persistido.

### 14.3 Implementación

`app/integrations/notificaciones_vivienda.py::notificar_visita_gobernador(...)` — best-effort, nunca lanza (mismo criterio que el resto de las integraciones salientes del proyecto: `resumen_territorial.fetch_privada_lineas`, `privada_sync.sync_gestion_privada`). Reusa `settings.svc_vivienda_internal_url` (ya sembrada para el lookup de `portal_usuarios`, ADR-015) — no hace falta una URL nueva. Gate propio: `settings.notificar_fila_nueva_enabled` (default `False` en `config.py`, `True` en prod vía `services/cloudbuild.yaml` — sustitución `_NOTIFICAR_FILA_NUEVA_ENABLED`, compartida con Gasífera porque ambos servicios leen la misma env var). ID token de la SA de runtime, audiencia = URL base de `svc-vivienda` — mismo patrón que `app/auth.py::_fetch_portal_user`, pero en sentido inverso: acá `svc-gralgob` es quien llama.

### 14.4 IAM

Mismo grant que ya usa ADR-015 (`svc-gralgob@gestorcooperativo.iam.gserviceaccount.com` con `roles/run.invoker` sobre `svc-vivienda`) — no es un permiso nuevo, el mismo grant habilita tanto `GET /internal/portal/usuarios/{email}` como este `POST /internal/notificaciones`, porque ambos apuntan al mismo servicio destino.

### 14.5 Criterios de aceptación

- [x] `app/integrations/notificaciones_vivienda.py` + wiring en `app/atp/sync.py::sync_from_sheet`.
- [x] Tests (`tests/test_atp_notificaciones.py`, 3): compromiso nuevo notifica con localidad/fecha correctas; un compromiso ya existente no vuelve a notificar en una corrida posterior; con el flag apagado no se instancia el cliente HTTP. Suite completa de `svc-gralgob` (34 tests) en verde.
- [x] `services/cloudbuild.yaml`: sustitución `_NOTIFICAR_FILA_NUEVA_ENABLED` (compartida con Gasífera).
- [ ] Verificación end-to-end en producción (corrida real de sync con un compromiso nuevo + confirmación visual de la alerta en el panel de notificaciones) — pendiente, análoga a la ya hecha para la vinculación Privada (`spec-vinculacion-vivienda-privada.md`).
