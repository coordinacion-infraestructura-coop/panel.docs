# Spec: Sincronización Google Sheet "ATP - Compromiso Gobernador" → `svc-gralgob` (Fase 0)

**Estado**: approved
**Versión**: 1.0.0
**Servicio**: `svc-gralgob` (nuevo — solo el módulo de sync en esta entrega, sin panel de negocio)
**Última actualización**: 2026-09-21

---

## Changelog

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

- [ ] Servicio `svc-gralgob` nuevo, scaffold completo (`pyproject.toml`,
      `app/`, `alembic/`, `tests/`, `docker-compose.dev.yml`, `Dockerfile`,
      `README.md`).
- [ ] Migración `0001` crea las 3 tablas nuevas.
- [ ] `pytest` en verde sobre SQLite in-memory, sin requerir Postgres.
- [ ] `POST /internal/sync/atp-compromiso-gobernador` no usa
      `Depends(get_current_user)`; no se declara en `openapi.yaml`.
- [ ] Corridas repetidas no duplican filas (UPSERT verificado en tests).
- [ ] Una fila con error real de DB no frena el resto del batch ni impide el
      log final (test de regresión con `IntegrityError` forzado).
- [ ] Deploy real a Cloud Run — fuera de esta entrega, sesión aparte vía
      `/deploy-servicio` (mismo criterio que `svc-gasifera`).

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
