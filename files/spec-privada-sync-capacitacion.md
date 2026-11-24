# Spec: Inscriptos a la capacitación en comunicación → Planificación (Privada)

**Estado**: approved
**Versión**: 1.0.0
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

1. Se crea en Planificación el programa **"Congreso de Formación Digital 2026"**: área
   `privada`, **sin `fuente`**. Por eso no aparecen "¿Está en el panel?" ni
   "¿Convenio?" (§5.1 del spec de planificación).
2. Un **sync** de svc-privada lee el Sheet y escribe en ese programa la cantidad
   de inscriptos, los nombres y un estado automático (§4).
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

Formulario "Congreso de Formación Digital 2026": jornada para líderes territoriales y
sus equipos de comunicación, sábado 28/11 a las 9 h en Villa Carlos Paz. Cada líder
puede ir con hasta 2 integrantes de su equipo. **La inscripción cierra el miércoles
25/11 a las 23 h.** Estructura relevada el 2026-11-24 sobre una descarga con 366
respuestas.

Pestaña **`Respuestas de formulario 1`** (la que escribe el formulario):

| Columna | Tipo en el formulario | Uso |
|---|---|---|
| Marca temporal | automática | Fecha de la última inscripción; desempata duplicados |
| Dirección de correo electrónico | texto, obligatoria | **Solo para deduplicar**. No se guarda (§8) |
| APELLIDO, NOMBRE | texto, obligatorias | Columna de nombres |
| LOCALIDAD | **desplegable** con 427 localidades + "OTRA LOCALIDAD" | Se resuelve a `id_geo` (§5) |
| CARGO | desplegable: Intendente · Jefe comunal · Equipo de comunicación · Legislador · Ministro | Desglose por cargo (§4.1) |

Pestaña **`Lista`** (DEPARTAMENTO, LOCALIDAD): las opciones del desplegable, con su
departamento. Los nombres no se repiten; los homónimos vienen desambiguados, por
ejemplo `LOS CHAÑARITOS (C. D. E.)`. El sync la lee para conocer el departamento de
cada opción.

Las otras pestañas (`Resumen`, `Para Acreditaciones`, `Intendentes y jefes
comunales`) son fórmulas del área y no se leen.

Lo que se vio en la descarga:
- **Duplicados**: 39 correos repetidos (la persona se inscribió dos veces). Se
  cuenta **una persona por correo** (normalizado en minúsculas), con su última
  respuesta.
- "OTRA LOCALIDAD": 1 respuesta. Queda sin resolver y se informa (§5).
- Legisladores (14) y ministros (5) eligen una localidad (por ejemplo, 7 legisladores
  en CORDOBA). Cómo cuentan: §4.1.

## 4. Qué escribe en Planificación

### 4.1 Configuración del programa (se crea una vez, idempotente)

| Elemento | Valor |
|---|---|
| Programa | "Congreso de Formación Digital 2026", área `privada`, `fuente` NULL |
| Estado automático | **"Con inscriptos"** (verde `#C8E6C9` / `#1B5E20`) |
| Estado automático | **"Solo intendente / jefe comunal"** (amarillo `#FFF9C4` / `#8D6E00`) |
| Columna `insc_lider` | "Intendente / jefe comunal inscripto (automático)", tipo `si_no` |
| Columna `insc_equipo` | "Equipo de comunicación (automático)", tipo `numero` |
| Columna `insc_total` | "Inscriptos en total (automático)", tipo `numero` |
| Columna `insc_otros` | "Legisladores / ministros (automático)", tipo `numero` |
| Columna `insc_nombres` | "Inscriptos – nombres (automático)", tipo `texto_largo`: `APELLIDO, Nombre (cargo)` separados por "; " |
| Columna `insc_ultima` | "Última inscripción (automático)", tipo `fecha` |

**Qué cuenta como "presentó inscriptos"** (usuario, 2026-11-24): **cualquier
inscripción cuenta**, sin importar el cargo, también la de un legislador o un
ministro. Además, se marca aparte cuando **solo se inscribió el intendente o jefe
comunal** y nadie más de la localidad:

| Inscripciones de la localidad | Estado automático |
|---|---|
| Ninguna | (sin estado) |
| Solo intendente(s) / jefe(s) comunal(es) | "Solo intendente / jefe comunal" |
| Cualquier otra combinación | "Con inscriptos" |

El programa trae además el "Resuelto por ATP" de todo programa nuevo, que no se usa
y se puede borrar.

Las cuatro columnas por defecto (anuncio, montos, observación) se **ocultan**
(`visible=false`). La observación queda disponible para el seguimiento manual si
alguien la vuelve a mostrar.

El sync encuentra el programa, el estado y las columnas **por `clave` o nombre**,
nunca por un id fijo. Si no existen, los crea.

### 4.2 Por localidad, en cada corrida

- **Columnas automáticas**: el sync es su dueño. Si alguien las edita a mano, la
  próxima corrida las pisa con lo que dice la planilla.
- **Estado**: el sync **no pisa nunca un estado puesto por una persona**.
  - Si el estado está vacío o es uno de los dos automáticos, el sync lo lleva al
    que corresponde según la tabla de §4.1. Eso incluye pasar de "Solo intendente"
    a "Con inscriptos" cuando se suma el equipo.
  - Si la localidad se quedó sin inscriptos (alguien borró respuestas), el estado
    automático se vacía.
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

- **Filtro Estado**: "sin estado" muestra las que faltan. "Solo intendente / jefe
  comunal" muestra las que tienen que sumar a su equipo. "Con inscriptos" muestra
  las completas.
- **Color de la fila**: verde, amarillo o sin color, porque la fila toma el color
  de su estado.
- **KPI por estado**: muestra cuántas localidades presentaron.
- El filtro de departamento, la exportación a Excel y el botón Imprimir funcionan
  como en los demás programas.

## 5. Vínculo con el padrón

- Cada opción de la `Lista` (departamento + localidad) se resuelve contra
  `priv_geo_localidades`, el espejo del padrón oficial (ADR-026). Se usan las reglas
  de normalización de `app/geo/matching.py` de svc-vivienda, copiadas a svc-privada.
  También se normaliza la puntuación del departamento (`PTE. ROQUE SAENZ PEÑA`).
  Las respuestas se vinculan por el texto exacto de la opción.
- Resultado sobre la Lista actual: **395 de 427 por nombre**. Las otras 32 son
  variantes de escritura y van en un diccionario versionado en el código
  (`opción → id_geo`). Se revisó a mano el 2026-11-24 con los mismos criterios
  que la recarga del semáforo de ese día. Ejemplos: `BOUWER`→303 Bower,
  `HUANCHILLA`→92, `VILLA RIO ICHO CRUZ`→158, `COLONIA BARGE`→105 Castro Urdiales,
  `PASO DEL DURAZNO`→443 (la Lista la pone en Juárez Celman y el padrón en Río
  Cuarto). Ningún `id_geo` queda asignado a dos opciones.
- **No se llama** a `POST /internal/geo/resolver-localidades` de svc-vivienda:
  cada nombre sin match registraría un pendiente y una notificación en Vivienda
  cada 15 minutos.
- Si se agrega al desplegable una opción que no cruza, o alguien elige "OTRA
  LOCALIDAD", la respuesta **no se pierde**: queda en la tabla del §6 con `id_geo`
  NULL, y el resultado del sync la informa.

## 6. Tabla propia (`db_privada`, Alembic `0007`)

`priv_capacitacion_inscripcion`: espejo de las respuestas. Sirve para recalcular
los conteos y para auditar qué se leyó.

| Columna | Tipo | Notas |
|---|---|---|
| `id` | serial PK | |
| `capacitacion` | varchar(60) | `congreso-formacion-digital-2026` (una planilla = una capacitación) |
| `fila_origen` | int | Número de fila en el Sheet |
| `marca_temporal` | timestamptz NULL | |
| `apellido`, `nombre` | varchar(120) NULL | |
| `cargo` | varchar(60) NULL | |
| `localidad_texto` | varchar(200) NULL | Opción elegida, tal como vino |
| `departamento_texto` | varchar(120) NULL | De la pestaña `Lista` |
| `email_hash` | char(64) | SHA-256 del correo normalizado. Solo sirve para deduplicar; el correo no se guarda |
| `id_geo` | varchar(30) NULL | Resultado del vínculo |
| `match_tipo` | varchar(20) | `exacto` · `manual` · `ambiguo` · `sin_match` |
| `synced_at` | timestamptz | |
| `deleted_at` | timestamptz NULL | Se marca si la fila desaparece del Sheet |

UNIQUE `(capacitacion, marca_temporal, email_hash)`: la clave natural de una
respuesta. El número de fila se actualiza si alguien reordena la planilla. La
deduplicación por persona se hace al contar, no al guardar. Una respuesta que
desaparece de la planilla se marca con `deleted_at` y se reactiva si vuelve. Cada fila se escribe en su
propio SAVEPOINT (patrón de `checklist_sync.py`). Una fila mala no corta el
resto; un error al leer la planilla entera devuelve 502.

## 7. Endpoint y disparo

- `POST /internal/privada/capacitacion/sync`: IAM-only, sin `/api/v1` y sin
  gateway. Devuelve un resumen con leídas, resueltas, sin resolver (con el
  texto), localidades con inscriptos y filas de Planificación cambiadas.
- Configuración (`config.py`): `capacitacion_sheet_id`,
  `capacitacion_rango_respuestas` y `capacitacion_rango_lista`. El id de la
  planilla va **como default en el código**: no es secreto, y así un redeploy de
  CI (`--set-env-vars` reemplaza todo) no lo borra. Con el id vacío, el endpoint
  responde 200 con `deshabilitado: true` y no hace nada.
- `?dry_run=true`: informa qué cambiaría sin escribir nada.
- Cloud Scheduler `sync-privada-capacitacion`, cada 15 minutos, OIDC con la
  misma SA que ya dispara `sync-privada-padron-politico`. La inscripción cierra el
  25/11 a las 23 h: el 26/11 se corre una última vez y se **pausa** el job. Los
  datos quedan.
- No hay botón "Sincronizar ahora" en la pantalla: con 15 minutos alcanza. Si se
  necesita antes, se dispara el job a mano desde Cloud Scheduler.

## 8. Datos personales

La planilla trae el correo de cada inscripto. La tabla de control **solo guarda y
muestra apellido, nombre, cargo y localidad**. Del correo se guarda solo un hash,
para deduplicar: es lo que hace falta para
saber quién presentó. Cualquier otro dato se queda en la planilla. Si Privada
quiere ver el contacto en el panel, se decide aparte, porque Planificación la ve
todo el personal con secretaría `privada`.

## 9. Pendientes (todos resueltos al aprobar)

1. ~~Estructura del formulario~~: relevada (§3).
2. ~~Planilla compartida con la cuenta de servicio~~: confirmado por el usuario
   el 2026-11-24.
3. ~~Regla de §4.1 sobre legisladores y ministros~~: cuentan, y se marca aparte
   "solo intendente" (usuario, 2026-11-24).
4. Otras jornadas futuras con otro formulario: fuera de alcance. Si aparecen, se
   parametriza por planilla y programa en vez de duplicar el sync.

## 10. Pruebas

`tests/test_capacitacion_sync.py`, con SQLite en memoria y el cliente de Sheets
mockeado:

- Alta inicial: crea el programa, las columnas y el estado.
- Conteo y nombres por localidad; deduplicación por correo; un legislador solo no marca la localidad.
- Una segunda corrida sin cambios no escribe historial.
- No pisa un estado manual.
- Vacía "Con inscriptos" cuando la localidad se queda sin inscriptos.
- Nombre ambiguo o sin match: queda en la tabla con `id_geo` NULL.
- Una fila mala no corta el resto.
- Planilla ilegible → 502.

## 11. Validación con datos reales (2026-11-24)

Corrida local del parser y el vínculo sobre la descarga de las 11:18, contra el
padrón de producción (`priv_geo_localidades`, solo lectura):

- 366 respuestas → **327 personas**, deduplicadas por correo.
- Vínculo: 317 exactos, 9 manuales y 1 sin resolver ("OTRA LOCALIDAD").
- **179 localidades con inscriptos** de 427. En 84 de ellas se inscribió solo el
  intendente o jefe comunal.

## 12. Changelog

- 1.0.0 (2026-11-24): approved. Todo inscripto cuenta; se agrega el estado "Solo
  intendente / jefe comunal" y la columna `insc_otros`. El programa se llama como el
  congreso. La migración es la `0007` (la `0006` ya existía). El id de la planilla va
  como default en `config.py`. Se agregan `dry_run` y la validación con datos reales (§11).

- 0.2.0 (2026-11-24): estructura real del formulario (§3), desglose por cargo y regla
  de "presentó" (§4.1), vínculo por la pestaña `Lista` con 32 opciones manuales (§5),
  hash del correo (§6, §8), cierre el 25/11 (§7).
- 0.1.0 (2026-11-24): borrador. Reusa Planificación (ADR-036) y el patrón de
  sync de Sheets. Pendiente la estructura del formulario.
