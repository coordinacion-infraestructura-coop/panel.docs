# Spec: Fase 4 — ATP Interno de Gobierno (planilla "4. ATP 2026")

**Estado**: implemented
**Versión**: 1.0.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-21
**Servicio**: `svc-gralgob` (módulo nuevo `atp_interno`, ADR-029); `svc-datos-externos` para el
Resumen general
**Spec padre**: `spec-datos-externos-tablero-gobierno.md` (approved 1.4.1), Fase 4
**Pantallas que habilita**: `spec-tablero-gobierno-panel.md`, Etapa 4

## Changelog

- **1.0.1** (2026-10-21): implementado y en producción. Backend `b691382` (svc-gralgob migración
  `0003` aplicada; svc-datos-externos con ATP en el Resumen), gateway `ministerio-config-v20261021`
  (infra `3fc4017`), frontend `a586c4b`. Cloud Scheduler `sync-gralgob-atp-interno` (`15 * * * *`)
  y `roles/run.invoker` de `svc-datos-externos@` sobre `svc-gralgob`. Verificado con la planilla
  real contra Postgres 15 descartable y en producción: 2.220 filas, $76.209 M, 407 localidades,
  15 fechas rotas, 0 descartes; ninguna respuesta trae el referente. La planilla ya estaba
  compartida con `svc-gralgob@`. Nota de despliegue: el push no disparó los triggers de Cloud
  Build (webhook de GitHub); se corrieron a mano `deploy-svc-gralgob` y
  `deploy-svc-datos-externos` con `gcloud builds triggers run`.
- **1.0.0** (2026-10-21): aprobado por el usuario sin cambios, con autorización para implementar
  y desplegar. El área ya pidió compartir la planilla con `svc-gralgob@`.
- **0.9.0** (2026-10-21): primera versión, a partir de la lectura de estructura y agregados de la
  planilla (solo lectura, con la cuenta `svc-datos-externos@`, autorizada por el usuario; el
  referente no se imprimió). Incluye dos decisiones del usuario del mismo día: el referente se
  guarda pero no se expone (§5) y ATP Interno entra en el Resumen general igual que en el Looker,
  con nota para definirlo con el área (§4.3).

## 0. Alcance

Sincronizar en solo lectura la planilla de carga del ATP Interno de la Secretaría y exponerla al
Tablero de Gobierno:

- Página **Fondos › ATP Interno** 🔒 (solo `Admin` y `Autoridad`): KPIs, pagos por mes, por
  departamento, por localidad y la lista de pagos.
- **Resumen general**: columna ATP, semáforo ATP / (COPA + FOFINDES) y ATP dentro del TOTAL, como
  en el Looker.

Sigue vigente lo que dice ADR-029: son dos planillas distintas que **no se fusionan**. La
planilla de Compromisos queda como está (`atp_compromisos`).

**Fuera de alcance**:
- **Histórico 2023–2025**: estaba en la pestaña `ATP` de "0. Resumen General", que el área ya no
  comparte (paraguas §2). El Looker muestra 6.009 pagos y $146.490 M porque lee ese histórico;
  nosotros, por ahora, solo 2026 (§8.1).
- **"Saldo disponible ATP"** ($16.021 M en el Looker): no sale de esta planilla y no sabemos de
  dónde (§8.2). Se muestra "sin fuente".
- **Federación a `resumen_territorial`** (ficha de localidad): ATP Interno es sensible y ese
  módulo lo ven roles de todas las áreas. No se federa.
- **Panel editable** que reemplace la planilla (ADR-029 lo señala como candidato prioritario):
  necesita su propio spec.
- **Regla de deduplicación con Compromisos**: la sigue esperando el área (paraguas §3.3).

## 1. Estructura relevada (2026-10-21)

Planilla `1h3FzAL-X0_tgIGNgnZ_DgKd5rDo8oLg0wNxY0gisBAU`, título "4. ATP 2026". Dos pestañas:
`ATP ACUMULADO` (la base) y `Validadores` (copia del padrón: `ID_LOC`, `Localidades`, `ID_DPTO`,
`DEPARTAMENTO`), que no se lee.

`ATP ACUMULADO`, encabezado en la fila 1, 6 columnas:

| Columna | Tipo | Observado |
|---|---|---|
| `ID_LOC` | entero | Código INDEC de gobierno local, siempre lleno, 407 distintos, todos `14xxxx`. Un solo nombre por código |
| `LOCALIDAD` | texto | 407 distintos |
| `DEPARTAMENTO` | texto | 26 distintos |
| `FECHA` | serial de fecha | 2026-01-07 a 2026-10-09; sin fechas futuras. **15 filas con un serial imposible** (`6693568`) |
| `MONTO` | entero | De $1 M a $1.000 M. Todos múltiplos de $100.000 salvo uno |
| `REFERENTE` | texto | Siempre lleno. 13 grafías, 10 después de normalizar espacios, mayúsculas y tildes. **No se reproduce en ningún documento** |

Totales: 2.220 filas, $76.209 M, 407 localidades. Con fecha válida: 2.205 pagos, $76.058 M (las
15 filas con fecha rota suman $151 M). No hay filas repetidas (ni por localidad + fecha + monto).

Por mes (2026, fecha válida): ene 208 / $6.461 M · feb 150 / $3.343 M · mar 202 / $7.502 M ·
abr 237 / $7.299 M · may 202 / $6.226 M · jun 223 / $7.910 M · jul 302 / $11.342 M ·
ago 268 / $9.278 M · sep 324 / $13.044 M · oct (hasta el 9) 89 / $3.652 M.

Coincide con el cruce del 2026-10-13 (paraguas §3.3.1: 2.205 pagos, $76.058 M).

## 2. Modelo de datos (`db_gralgob`, migración `0003`)

**`atp_interno_pagos`** — una fila por fila de la planilla.

| Campo | Tipo | Nota |
|---|---|---|
| `id` | String(36) PK | uuid |
| `fila` | Integer | Número de fila en la planilla (para las anomalías) |
| `codigo_indec` | String(6) | `ID_LOC`, solo dígitos. Es la llave con el resto del tablero |
| `localidad`, `departamento` | String | Como vienen en la planilla, para mostrar |
| `fecha` | Date NULL | NULL si el serial es imposible |
| `anio_planilla` | Integer | Año de la planilla de origen (setting, hoy 2026). El reemplazo de cada corrida es por este valor (§3) |
| `anio` | Integer | Año de `fecha`; si es NULL, `anio_planilla` |
| `monto` | Numeric(18,2) | |
| `referente` | String(120) NULL | Texto con espacios normalizados. **Ningún endpoint lo devuelve** (§5) |
| `last_synced_at` | timestamptz | |

Índices: `(anio, codigo_indec)`.

**`atp_interno_sync_log`** — una fila por corrida (append-only, como `atp_sync_log`): inicio,
fin, estado (`ok` / `error` / `frenado`), filas leídas, cargadas y descartadas, anomalías (JSON),
`triggered_by`.

No se resuelve `id_geo`: `codigo_indec` alcanza para filtrar (el filtro del tablero ya entrega los
códigos) y evita la llamada a svc-vivienda.

## 3. Sincronización

- Endpoint IAM-only `POST /internal/sync/atp-interno` (+ `GET .../estado`), mismo patrón que
  `/internal/sync/atp-compromiso-gobernador`.
- Lee `'ATP ACUMULADO'!A1:F` con `UNFORMATTED_VALUE` y fechas como serial. Ubica las columnas por
  nombre de encabezado (sin tildes ni mayúsculas), no por posición.
- **Reemplazo completo** en una transacción: borra las filas del `anio_planilla` configurado y
  carga las leídas. La planilla
  no tiene un id por fila y las filas se reordenan; un upsert por número de fila asociaría datos
  a la fila equivocada.
- **Freno**: si la lectura trae menos del 50 % de las filas de la carga anterior, o ninguna, no se
  reemplaza nada y la corrida queda `frenado` (protege contra una planilla vaciada por error).
- Filas descartadas (no se cargan, se informan): sin `ID_LOC` numérico o sin `MONTO` > 0.
- Anomalías informativas (la fila se carga): `fecha_invalida` (fecha NULL, año de la planilla),
  `codigo_con_mas_de_un_nombre`.
- Si el encabezado no trae alguna de las columnas `ID_LOC`, `FECHA` o `MONTO`, la corrida falla
  (`error`) sin tocar los datos, y el endpoint responde 502 como Compromisos.
- Cloud Scheduler: job nuevo `sync-gralgob-atp-interno`, `15 * * * *` (cada hora, desfasado del de
  Compromisos), OIDC con `svc-gralgob@`. Mismo patrón y permisos que `sync-atp-compromiso-gobernador`.
- Setting nuevo `GOOGLE_SHEET_ATP_INTERNO_ID` en `services/cloudbuild.yaml` para `svc-gralgob`.
  Setting `ATP_INTERNO_ANIO_PLANILLA` = 2026. Cuando el área abra "4. ATP 2027", se cambia el id y
  el año, y 2026 queda en la base porque el reemplazo borra solo el `anio_planilla` configurado.

**Requisito de despliegue**: la planilla tiene que estar compartida como lector con
`svc-gralgob@gestorcooperativo.iam.gserviceaccount.com`. Hoy solo se verificó el acceso de
`svc-datos-externos@`. Se prueba con la primera corrida manual en producción; si responde 502
por permisos, se le pide al área que la comparta.

## 4. Qué expone al tablero

### 4.1 `GET /api/v1/gralgob/tablero/atp-interno` (gateway, config nueva)

Roles `Admin` y `Autoridad` (`ROLES_TABLERO`), sin chequeo de secretaría, igual que
`atp-compromisos`. Parámetros: `codigo_indec` (repetible; vacío = provincia).

Respuesta:
- `anios`: años con datos.
- `gestion` y `por_anio[]` con `kpis`: localidades con ATP, pagos, monto, pagos sin fecha válida.
- `mensual` por año (12 valores).
- `por_departamento[]`: pagos, monto, localidades.
- `por_localidad[]`: código, nombre, departamento, pagos, monto, último pago.
- `pagos[]`: fecha, año, código, localidad, departamento, monto. **Sin referente.**
- `saldo_disponible`: siempre `null` por ahora (§8.2).
- `ultima_sincronizacion`, `estado_sincronizacion`.

El filtro de año es propio de la página (como FOMMEP y los fondos de la Etapa 3), con "Toda la
gestión" por defecto. Hoy solo hay 2026.

### 4.2 `GET /internal/atp-interno/por-localidad?anio=` (IAM-only)

Para el Resumen general de `svc-datos-externos`: `[{codigo_indec, monto, pagos}]` del año pedido.
Lo llama `svc-datos-externos@`, que necesita `roles/run.invoker` sobre `svc-gralgob`, y un
setting `SVC_GRALGOB_INTERNAL_URL` más `ATP_INTERNO_FETCH_ENABLED` (por defecto `False` en
`config.py`, `True` en `cloudbuild.yaml`, igual que las federaciones de ADR-016/021/022).

### 4.3 Resumen general (`svc-datos-externos`, `planillas.get_resumen`)

Decisión del usuario (2026-10-21): **igual que el Looker, documentado para definirlo con el
área.**

- Columna **ATP** = ATP Interno pagado en el año, por localidad y por departamento.
- **Semáforo** ATP / (COPA + FOFINDES), con los cortes de `spec-tablero-gobierno-panel.md §2.5 #1`
  (verde ≥ 1,5 %, amarillo de 0,8 % a 1,5 %, rojo < 0,8 %). La fila de total no lleva color.
- **TOTAL** = COPA + FOFINDES + Fondo Ambiental + ATP Interno.
- **Compromisos nunca entra en el Resumen.** El área informó que los compromisos del Gobernador
  se registran también en la base que alimenta ATP Interno (paraguas §3.3), así que sumar solo
  ATP Interno no cuenta dos veces lo que está en las dos planillas.
- **Nota al pie en la pantalla**: dice que ATP en el total sigue el criterio del Looker y que
  está pendiente de definir con el área. Es un recordatorio visible, no un aviso de error.
- Si la llamada a `svc-gralgob` falla, la columna ATP viene `null`, el TOTAL se calcula sin ATP y
  la respuesta lo marca (`atp_disponible: false`), para que la pantalla lo diga. Nunca se
  muestra un total que parezca completo sin serlo.

## 5. Privacidad

- La página y el endpoint son solo para `Admin` y `Autoridad` (paraguas §5, nivel alto, político).
- **Referente** (decisión del usuario, 2026-10-21): se guarda en la base para un uso futuro,
  pero **ningún endpoint ni esquema de respuesta lo incluye**, ni el de la página ni el interno.
  Un test lo verifica. Exponerlo después requiere otro spec.
- El sync no loguea valores de filas, solo números de fila y motivos.

## 6. Pantallas (Etapa 4 del panel)

**Fondos › ATP Interno** 🔒 (`secciones.ts` ya la tiene como `restringida`):

- Selector de año propio.
- KPIs: Localidades con ATP · Pagos · Monto pagado · Saldo disponible ("sin fuente", con la
  pregunta abierta).
- Pagos por mes (barras) y por departamento (barras horizontales).
- Tabla por localidad (pagos, monto, último pago) y tabla de pagos (fecha ▼, localidad,
  departamento, monto), con exportación a Excel como el resto del tablero.
- Aviso fijo: "No se suma con ATP Compromisos: hay pagos cargados en las dos planillas".
- Aviso cuando hay pagos con fecha rota: cuántos y cuánto suman, contados en el año de la
  planilla.
- Pie de fuente: planilla "4. ATP 2026", sincronizada cada hora.

**Resumen general**: columna ATP, semáforo y total según §4.3, con la nota al pie.

## 7. Tests y verificación

- `svc-gralgob`: parseo (fechas rotas, códigos con separadores, encabezados por nombre), freno,
  reemplazo por año de planilla, agregados del tablero, filtro por código, que ninguna respuesta
  incluya `referente`, y el endpoint interno.
- `svc-datos-externos`: Resumen con ATP (total, semáforo, departamento), con la federación
  apagada y con `svc-gralgob` caído (fail-open marcado).
- Verificación con la planilla real contra Postgres descartable antes del deploy: 2.220 filas,
  $76.209 M, 15 fechas rotas, 407 localidades.
- Frontend: `npm run build` y verificación visual del usuario.

## 8. Preguntas para el área (no bloquean la implementación)

1. **Histórico**: ¿hay planillas anuales de ATP Interno 2023–2025, o una copia de la pestaña
   `ATP` de "0. Resumen General"? Sin eso, el tablero solo tiene 2026.
2. **Saldo disponible ATP**: ¿de dónde sale el valor del Looker ($16.021 M)? ¿Es un crédito
   presupuestario cargado en otra planilla?
3. **Fechas rotas**: 15 filas tienen una fecha imposible. ¿Las corrigen en la planilla?
4. **ATP en el total del Resumen** (§4.3): ¿confirman que ATP Interno ya incluye los pagos de
   compromisos y que va en el total? ¿Y la regla para identificar los duplicados (paraguas
   §3.3)?

## 9. Criterios de aceptación

1. El sync carga las 2.220 filas de la planilla real, informa las 15 fechas rotas y no descarta
   ninguna fila válida.
2. La página muestra 407 localidades, 2.220 pagos y $76.209 M en "Toda la gestión", y responde a
   los filtros de departamento y localidad.
3. Ninguna respuesta HTTP contiene el referente.
4. El Resumen general suma ATP Interno en el TOTAL, muestra el semáforo y la nota al pie; si
   `svc-gralgob` no responde, lo dice.
5. Un usuario sin rol `Admin` ni `Autoridad` recibe 403.
