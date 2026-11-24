# Spec: Inscriptos a la capacitación en comunicación → Planificación (Privada)

**Estado**: review
**Versión**: 0.1.0
**Servicios**: `svc-privada` (sync, tabla y endpoint IAM). Sin cambios en frontend
ni en el gateway.
**ADR**: ninguno nuevo. Aplica ADR-036 (la planificación por localidad es dato de
Privada) y el patrón de sync de Sheets por ADC (`spec-sync-gasifera-pit.md`,
`spec-sync-atp-compromiso-gobernador.md`).
**Depende de**: `spec-privada-planificacion-programas.md` (implemented),
`spec-privada-padron-oficial.md` (implemented)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-24

---

## 0. Por qué existe este spec

El sábado se hace una jornada de capacitación en comunicación. La inscripción se
hace por un **formulario de Google que sigue abierto**. Sus respuestas caen en el
Sheet `1bvY7X7_jbzMf8XXajJmwsjY1s0dRP4TBdJqsQ2H1t5w`.

Privada necesita una **tabla de control**: todas las localidades del padrón, cuáles
ya tienen inscriptos y cuáles no, para salir a buscar a las que faltan. Las
respuestas siguen entrando, así que la tabla se tiene que **actualizar sola**. Una
carga única de un Excel no sirve.

No hacen falta datos de los programas de Vivienda: se controla solo este programa.

## 1. Solución

La pantalla de Planificación (`/resumen-territorial/planificacion`) ya lista todas
las localidades por programa, con estado, columnas, filtros, KPIs, exportación a
Excel e impresión. Se usa tal como está:

1. Se crea en Planificación el programa **"Capacitación en comunicación"**: área
   `privada`, **sin `fuente`**. Por eso no aparecen "¿Está en el panel?" ni
   "¿Convenio?" (§5.1 del spec de planificación).
2. Un **sync** de svc-privada lee el Sheet y escribe en ese programa la cantidad
   de inscriptos, los nombres y un estado "Con inscriptos" (§4).
3. **Cloud Scheduler cada 15 minutos** dispara el sync.

## 2. Acceso a la planilla

Mismo patrón que el resto de los syncs de Sheets del sistema:

- El dueño de la planilla la comparte como **Lector** con
  `svc-privada@gestorcooperativo.iam.gserviceaccount.com`.
- El backend la lee con ADC (`google.auth.default()`), con scope
  `spreadsheets.readonly`. No usa claves ni Secret Manager.
- No depende de que una persona tenga la planilla compartida ni la sesión abierta.

svc-privada no tiene todavía un cliente de Sheets. Se agrega uno mínimo:
`app/integrations/google_sheets.py`, con lectura `values.get` por REST con `httpx`,
que ya es dependencia. No se suma `googleapiclient`. Lo usa solo este sync.

## 3. Lo que se lee del formulario

> **Pendiente (§9.1)**: la estructura real del formulario todavía no se pudo ver,
> porque la planilla no está compartida con la cuenta del conector ni con la
> cuenta de servicio. Esta sección se completa antes de pasar a `approved`.

Supuestos de trabajo:

| Dato | Uso |
|---|---|
| Marca temporal | Identifica la respuesta junto con el número de fila. Se muestra la fecha de la última inscripción |
| Nombre y apellido | Columna "Nombres" |
| Localidad (y departamento, si el formulario lo pide) | Se resuelve a `id_geo` (§5) |
| Cargo / institución, contacto | **No se copian a la tabla de control**. Ver §8 |

## 4. Qué escribe en Planificación

### 4.1 Configuración del programa (se crea una vez, idempotente)

| Elemento | Valor |
|---|---|
| Programa | "Capacitación en comunicación", área `privada`, `fuente` NULL |
| Estado | **"Con inscriptos"** (verde `#C8E6C9` / `#1B5E20`), además del "Resuelto por ATP" que trae por defecto (se puede borrar) |
| Columna `inscriptos` | "Inscriptos (automático)", tipo `numero` |
| Columna `inscriptos_nombres` | "Inscriptos – nombres (automático)", tipo `texto_largo`, nombres separados por "; " |
| Columna `inscriptos_ultima` | "Última inscripción (automático)", tipo `fecha` |

Las cuatro columnas por defecto (anuncio, montos, observación) se **ocultan**
(`visible=false`). La observación queda disponible para el seguimiento manual si
alguien la vuelve a mostrar.

El sync encuentra el programa, el estado y las columnas **por `clave` o nombre**,
nunca por un id fijo. Si no existen, los crea.

### 4.2 Por localidad, en cada corrida

- **Columnas automáticas**: el sync es su dueño. Si alguien las edita a mano, la
  próxima corrida las pisa con lo que dice la planilla.
- **Estado**: el sync **no pisa nunca un estado puesto por una persona**.
  - Si la localidad tiene inscriptos y su estado está vacío, pasa a "Con inscriptos".
  - Si se quedó sin inscriptos (alguien borró respuestas) y su estado es "Con
    inscriptos", el estado se vacía.
  - Cualquier otro estado (por ejemplo "Contactada", si Privada lo crea) se deja
    como está.
- Solo se escribe lo que **cambió**. Una corrida sin cambios no toca filas ni
  historial.
- Cada cambio pasa por la misma lógica que la edición desde la pantalla
  (`planificacion/service.actualizar_localidad`, o una variante interna sin lock
  optimista). Se escribe en `priv_plan_historial` y en `priv_audit_log` con el
  usuario `sync-capacitacion`.
- Las localidades sin inscriptos **no generan fila**. Se muestran igual, con las
  columnas vacías y "sin estado" (spec de planificación §4.5).

Cómo se usa en pantalla, sin tocar el frontend:

- **Filtro Estado**: "Con inscriptos" muestra las que presentaron, "sin estado"
  muestra las que faltan.
- **Color de la fila**: verde en las que tienen inscriptos, porque toma el color
  del estado.
- **KPI por estado**: muestra cuántas localidades presentaron.
- El filtro de departamento, la exportación a Excel y el botón Imprimir funcionan
  como en los demás programas.

## 5. Vínculo con el padrón

- Se resuelve contra `priv_geo_localidades`, el espejo del padrón oficial
  (ADR-026), con las mismas reglas de normalización de `app/geo/matching.py` de
  svc-vivienda, replicadas o reusadas en svc-privada.
- **No se llama** a `POST /internal/geo/resolver-localidades` de svc-vivienda.
  Cada nombre sin match registraría un pendiente y una notificación en Vivienda
  cada 15 minutos.
- Si el formulario no pide departamento y hay nombres repetidos en la provincia
  (por ejemplo LOS CHAÑARITOS), la respuesta queda **sin resolver**. No se asigna
  al azar.
- Las respuestas sin resolver **no se pierden**: quedan en la tabla del §6 con
  `id_geo` NULL y el resultado del sync informa cuántas son y cuáles.
- Vinculación manual: un diccionario versionado en el código del sync (texto
  normalizado → `id_geo`), igual que se hizo con ATP. Si en el futuro hace falta
  cargarla desde la pantalla, se especifica aparte.

## 6. Tabla propia (`db_privada`, Alembic `0006`)

`priv_capacitacion_inscripcion`: espejo de las respuestas. Sirve para recalcular
los conteos y para auditar qué se leyó.

| Columna | Tipo | Notas |
|---|---|---|
| `id` | serial PK | |
| `capacitacion` | varchar(60) | `comunicacion-2026-11` (una planilla = una capacitación) |
| `fila_origen` | int | Número de fila en el Sheet |
| `marca_temporal` | timestamptz NULL | |
| `nombre` | varchar(200) NULL | |
| `localidad_texto` | varchar(200) NULL | Tal como vino |
| `departamento_texto` | varchar(120) NULL | |
| `id_geo` | varchar(30) NULL | Resultado del vínculo |
| `match_tipo` | varchar(20) | `exacto` · `manual` · `ambiguo` · `sin_match` |
| `synced_at` | timestamptz | |
| `deleted_at` | timestamptz NULL | Se marca si la fila desaparece del Sheet |

UNIQUE `(capacitacion, fila_origen, marca_temporal)`. Cada fila se escribe en su
propio SAVEPOINT (patrón de `checklist_sync.py`). Una fila mala no corta el
resto; un error al leer la planilla entera devuelve 502.

## 7. Endpoint y disparo

- `POST /internal/privada/capacitacion/sync`: IAM-only, sin `/api/v1` y sin
  gateway. Devuelve un resumen con leídas, resueltas, sin resolver (con el
  texto), localidades con inscriptos y filas de Planificación cambiadas.
- Configuración (`config.py`): `capacitacion_sheet_id` y
  `capacitacion_sheet_rango`. Con el id vacío, el endpoint responde 200 con
  `deshabilitado: true` y no hace nada. Se setean con `--update-env-vars`; no
  hace falta que vayan en `cloudbuild.yaml`.
- Cloud Scheduler `sync-privada-capacitacion`, cada 15 minutos, OIDC con la
  misma SA que ya dispara `sync-privada-padron-politico`. Después de la jornada,
  se **pausa** el job y los datos quedan.
- No hay botón "Sincronizar ahora" en la pantalla: con 15 minutos alcanza. Si se
  necesita antes, se dispara el job a mano desde Cloud Scheduler.

## 8. Datos personales

La planilla puede traer DNI, teléfono o mail de los inscriptos. La tabla de
control **solo guarda y muestra nombre y localidad**: es lo que hace falta para
saber quién presentó. Cualquier otro dato se queda en la planilla. Si Privada
quiere ver el contacto en el panel, se decide aparte, porque Planificación la ve
todo el personal con secretaría `privada`.

## 9. Pendientes antes de `approved`

1. **Estructura del formulario** (§3): nombres de columnas, si la localidad es
   texto libre o lista, si pide departamento, y si una respuesta puede traer
   varias personas.
2. Confirmar que el dueño compartió la planilla con la cuenta de servicio (§2).
3. Confirmar el id de la capacitación y si va a haber otras jornadas con otro
   formulario. Si las hay, se parametriza por planilla y programa en vez de
   duplicar el sync.

## 10. Pruebas

`tests/test_capacitacion_sync.py`, con SQLite en memoria y el cliente de Sheets
mockeado:

- Alta inicial: crea el programa, las columnas y el estado.
- Conteo y nombres por localidad.
- Una segunda corrida sin cambios no escribe historial.
- No pisa un estado manual.
- Vacía "Con inscriptos" cuando la localidad se queda sin inscriptos.
- Nombre ambiguo o sin match: queda en la tabla con `id_geo` NULL.
- Una fila mala no corta el resto.
- Planilla ilegible → 502.

## 11. Changelog

- 0.1.0 (2026-11-24): borrador. Reusa Planificación (ADR-036) y el patrón de
  sync de Sheets. Pendiente la estructura del formulario.
