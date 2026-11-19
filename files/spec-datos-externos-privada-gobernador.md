# Spec: Planilla "Gobierno" de la Privada del Gobernador — anuncios, FOCOM e histórico ATP

**Estado**: approved
**Versión**: 1.0.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-19
**Servicio**: `svc-datos-externos` (lectura de la planilla); `svc-gralgob` (un endpoint IAM nuevo, §5.2)
**Spec padre**: `spec-datos-externos-tablero-gobierno.md` (paraguas del Tablero de Gobierno)
**Relacionados**: `spec-datos-externos-fase3-fondos.md` (FOCOM "sin fuente"), `spec-atp-interno.md`,
`spec-sync-atp-compromiso-gobernador.md`, ADR-029 (superposición ATP Interno / Compromisos),
ADR-033 (svc-agente, uso de la API de Anthropic)

## Changelog

- **1.0.1** (2026-11-19): Etapas A y B implementadas en `svc-datos-externos` (migración `0011`,
  rama `datos-externos-privada-gobernador`). Se verificaron con la planilla real (copia del
  2026-11-19) contra un Postgres 15 descartable, en 4 s y sin descartes:

  | Fuente | Resultado |
  |---|---|
  | Anuncios | 1.324; 12 con fecha fuera de rango |
  | FOCOM | 433 convenios: puro $37.631 M (361), pagado por ATP $4.050 M, pendiente por ATP $250 M |
  | ATP histórico | 3.807 filas, $70.356 M |
  | Control contra ATP Interno | Coincide al peso |

  Tres ajustes de implementación respecto del texto de la 1.0.0:
  1. **Reemplazo completo en lugar de upsert más `deleted_at`** (§2 y §3.2). Las tres pestañas son
     fuentes del sync de planillas de Gobierno ya existente, que reemplaza cada tabla en un
     SAVEPOINT y frena si la pestaña llega vacía o con menos del 90 % de las filas anteriores. Es
     la misma excepción documentada que las Fases 2 y 3. Los vínculos de la Etapa C se apoyan en
     las claves naturales, que no cambian con el reemplazo.
  2. **Localidad por `codigo_indec` del nomenclador de Gobierno, no por `id_geo`** (§3.4). El
     Tablero filtra por código INDEC. FOCOM lo trae en `ID_LOC`. Anuncios y ATP histórico se
     resuelven por departamento y nombre y, si el nombre es único, solo por nombre; es el mismo
     criterio que la Fase 6.
  3. **Fecha de anuncio** (§1.6): se descarta si el año es anterior a 2023 o si la fecha cae más de
     60 días en el futuro. 2027 cuenta como typo.

  FOCOM y el ATP histórico se sirven como secciones `focom` y `atp-historico` de
  `GET .../tablero/transferencias/{seccion}`, una ruta que ya está en el gateway, así que no hace
  falta config nueva.
  - La pantalla de ATP Interno une el histórico con lo suyo, en la misma forma de respuesta.
  - El **Resumen general** toma el ATP del histórico para los años anteriores a 2026. Antes, esos
    años mostraban ATP en cero, porque ATP Interno solo tiene 2026.
- **1.0.0** (2026-11-19): el usuario lo aprueba con tres decisiones:
  - **Fuente del ATP histórico**: se usa primero lo que ya tenemos accesible. Se lee la pestaña
    `Gobierno!ATP`, que trae exactamente las mismas filas que la planilla semanal. La planilla
    semanal `1ySgcr1t…` queda como fuente opcional si el área la comparte (§3.3).
  - **Vínculos del LLM**: al principio solo sugieren. Después de un período de validación, se
    confirman solos los de confianza alta (§4.3).
  - **Pregunta 6 al área** (compartir la planilla semanal): se hace, pero no bloquea nada.
- **0.1.0** (2026-11-19): primera versión. Se basa en la lectura de la planilla y en dos análisis
  comparativos de solo lectura hechos el 2026-11-19 (§1.4 y §1.5). Los scripts y los CSV de ese
  análisis están en el scratchpad de la sesión y no se versionan.

## 0. Alcance

La Privada del Gobernador lleva la planilla **"Gobierno"**
(`1u1Yfzaih21Pv9QPPYGeJrwjExWyZXPi9qOu5GZqQp-Q`, dueño `tomigaston15@gmail.com`). Desde el
2026-11-19 está compartida como Lector con `svc-datos-externos@gestorcooperativo.iam.gserviceaccount.com`.
Este spec define qué se lee de cada pestaña, cómo se vincula con lo que ya tenemos y qué se muestra.

| Pestaña | Qué es | Decisión |
|---|---|---|
| `Gobierno` | Registro de prensa de los anuncios del Gobernador. Una fila por anuncio, 1.324 filas, 2024–2026 | **Se sincroniza** y se vincula con FOCOM, ATP y Compromisos (§4) |
| `FOCOM` | Convenios FOCOM por localidad, con `ID_LOC` | **Se sincroniza**. Es la fuente que faltaba en la Fase 3 |
| `ATP` | `IMPORTRANGE` de la planilla semanal "ATP TRANSFERIDOS…" (§1.3) | **Se sincroniza solo el histórico 2023–2025**. Es la fuente principal porque ya tenemos acceso. La planilla semanal es la alternativa (§3.3) |
| `Tabla dinámica 1` | Conteo por estado | No se lee. Se recalcula |

**Fuera de alcance (v1):**
- Escribir en la planilla.
- Federar estos datos a `resumen_territorial`. Queda para una versión posterior, con el mismo
  patrón que ADR-021/022.
- Reemplazar ATP Interno para 2026 (§3.3).
- Resolver la regla de ADR-029 (ATP Interno contra pagos de Compromisos). Este spec no la cambia.
- Columnas de personas, que no se leen nunca (§6).

## 1. Estructura relevada (2026-11-19)

### 1.1 Pestaña `Gobierno` (anuncios)

Rango `A1:T`. Las columnas `U:AG` están vacías.

| Col | Encabezado | Uso |
|---|---|---|
| A | `ascasu` | **Clave**: único y nunca vacío en las 1.324 filas |
| B | Fecha de carga del anuncio | Casi siempre vacía. Se guarda |
| C | Fuente del anuncio | URL o la palabra "Link" (1.254 URLs reales). Se guarda la URL si es válida |
| D | Fecha de anuncio | `d/m/aaaa` con y sin cero inicial. 12 filas con año 2016 o 2027 (typos, §1.6) |
| E, F | Departamento, Localidad | Texto libre. Se resuelve contra el padrón (§3.4) |
| G | Ministerio | **Vacío en 910 filas**. "Gobierno" en 398 (todas de 2024) |
| H | Área | Vacía. No se lee |
| I, J | Anuncio, Detalle del anuncio | Texto. El monto aparece solo dentro del texto (1.131 filas lo mencionan) |
| K | Estado | Pagado 982 · Pendiente 239 · Espera documentación 50 · Firmado 32 · Pago parcial 1 · vacío 20 |
| L, M, N | Línea base, Valor actual, Grado de cumplimiento | Cargados en 84–126 filas |
| O, P | Responsable, Fecha de actualización | Vacías. No se leen |
| Q | Comentarios | Texto libre del área (92 filas). Se guarda con acceso restringido (§6) |
| R | Indicador de seguimiento Gabriel | Puntero a otra fila (§4.1) |
| S, T | Estado Gabriel, Coinciden | Estado de la fila apuntada y control del área. Se guardan como referencia |

### 1.2 Pestaña `FOCOM`

Rango `A1:V`. Datos reales en los primeros cientos de filas. El resto del rango está vacío.

Columnas que se leen: `ID_FOCOM` (**clave**, "N-FOCOM"), `EXPTE N°`, `N° CONVENIO`,
`FECHA CONVENIO`, `N° RESOLUCIÓN`, `DEPARTAMENTO`, `TIPO`, `ID_LOC` (código INDEC, el mismo de la
Fase 1), `LOCALIDAD`, `MONTO` (`$30.000.000,00`), `CONCEPTO`, `CATEGORÍA`, `ESTADO`, `FECHA PAGO`,
`COMPUTO PLAZO VISACIÓN`, `AÑO` y `ESTADO OBRA`.

**No se leen**: `ADMINISTRATIVO ASIGNADO`, `COORDINADOR ASIGNADO`, `REFERENTE POLITICO`,
`OBSERVACIONES` y `OBSERVACIONES - ENCARGADO` (§6). Como esas columnas están en el medio, la
lectura pide dos rangos (`A:O` y `U:V`), igual que en la Fase 3.

### 1.3 Pestaña `ATP` y la planilla semanal de origen

La pestaña es
`FILTER(IMPORTRANGE("1ySgcr1tXy-aFvueWADZNdXhCe3hq9DGZpnHZ1eWsvaY", "ATP ACUMULADO!A4:H"), …)`.
La planilla de origen se llama **"ATP TRANSFERIDOS SEMANA DEL …"**, se carga a mano y se actualiza
todas las semanas. Comparada con la pestaña, coinciden las 6.233 filas, campo por campo.

Columnas: `ID ATP` ("N-ATP"), `DEPARTAMENTO`, `LOCALIDAD`, `FECHA (TRANSFERENCIA)`,
`MONTO TRANSFERENCIA` y `REFERENTE`. La columna `REFERENTE` no se lee.

### 1.4 Comparación de la pestaña `ATP` con ATP Interno (verificado)

| | Planilla semanal | ATP Interno (`'ATP ACUMULADO'`, hoy en `svc-gralgob`) |
|---|---|---|
| Filas | 6.233 | 2.426 |
| Fechas | 28-12-2023 → 16-11-2026 | 07-01-2026 → 16-11-2026 |
| Monto | $158.389 M | $88.033 M |
| 2026 | 2.426 filas, $88.033 M | Igual, en el mismo orden |

- Cruzadas por (localidad normalizada, fecha, monto): **las 2.426 filas de 2026 coinciden 1 a 1**.
  No hay diferencias de monto, fecha ni localidad.
- Hay 5 filas con el departamento mal en la planilla semanal y bien en ATP Interno (Costa Sacate,
  Cruz Alta, Noetinger, San José). ATP Interno tiene `ID_LOC` y el padrón aplicado.
- **Qué aporta**: el **histórico 2023–2025, 3.807 filas y $70.356 M**. Por año: 2023, 7 filas y
  $34 M; 2024, 1.517 filas y $12.434 M; 2025, 2.283 filas y $57.888 M. Es la fuente del "histórico
  ATP" que el spec del Tablero marca como pendiente.
- La "Tabla de coincidencias" (columnas K–L) es una tabla fija de equivalencias de estado, de 10
  filas. No asigna un estado a cada pago y no se lee.

### 1.5 Comparación de la pestaña `Gobierno` con "ATP - Compromiso Gobernador" (BD)

**No es la misma información**. Las dos se superponen solo en 2026, y en forma parcial:

| | BD Compromisos | Pestaña `Gobierno` |
|---|---|---|
| Qué es | Apertura administrativa de cada anuncio: destino, monto, ministerio | Registro de prensa: un anuncio por fila |
| Filas | 1.104 | 1.324 |
| Período | 2026. **204 filas sin fecha** | 2024 (496), 2025 (136), 2026 (678) |
| Monto | Columna numérica | Solo dentro del texto |
| Ministerio | Lleno: Gobierno 770, Cooperativas 204, Vinculación 67, Salud 46… | Vacío en el 69 % de las filas |
| Localidades | 246 | 399 (231 en común) |

- La pestaña **no trae información de otros ministerios** que no tengamos: ese dato está mejor en
  BD.
- Lo que **sí aporta**:
  - historia 2024–2025 (unas 637 filas sin equivalente en BD);
  - estado del anuncio;
  - 1.254 links de prensa;
  - grado de cumplimiento;
  - comentarios del área;
  - el vínculo con FOCOM y ATP (§4.1);
  - unos 94 anuncios posteriores al último registro de BD.
- **La relación no es 1 a 1.** Un acto con varios destinos es un anuncio en `Gobierno` y varias
  filas en BD: 28 anuncios tienen más de una fila de BD con match de alta confianza.

> El análisis usó una copia de BD descargada el 2026-11-19 a las 11:53. El conector de Drive de la
> sesión no tenía acceso a la planilla real (`1x3E73…`). Los números pueden moverse un poco.

### 1.6 Calidad de datos que el sync tolera

- Fechas con año fuera de 2023–2027: se guardan como `NULL`, con el texto original en
  `fecha_anuncio_texto`, y se cuentan en el log como advertencia.
- Montos con `$`, puntos de miles y coma decimal: se usa el mismo parser que la Fase 3.
- IDs ATP del histórico: hay 1 duplicado (`3130-ATP`), 1 mal escrito (`1782,-ATP`) y 2 fuera de
  secuencia. La clave de la tabla es `(id_atp_normalizado, ocurrencia)`; el sync no los descarta.
- Hay 4 pares de filas ATP con igual localidad, fecha y monto. Son pagos distintos: no se
  deduplican.
- Anuncios casi duplicados en `Gobierno` (por ejemplo, Laboulaye G1606/G1607): se guardan los dos.
  El vinculador los trata como candidatos empatados (§4).

## 2. Modelo de datos (`svc-datos-externos`, módulo nuevo `gob_privada`)

Las tablas siguen el prefijo `ext_gob_*`. Hay una migración nueva y todas las tablas tienen
`created_at`, `updated_at` y `deleted_at`.

| Tabla | Clave natural | Contenido |
|---|---|---|
| `ext_gob_anuncio` | `ascasu` | §1.1, más `id_geo`, `fecha_anuncio_texto`, `fuente_url`, `montos_mencionados` (array numérico extraído del texto), `hash_contenido` |
| `ext_gob_focom` | `id_focom` | §1.2, más `id_geo` (por `ID_LOC`) |
| `ext_gob_atp_historico` | `(id_atp, ocurrencia)` | Transferencias **anteriores al 2026-01-01**, más `id_geo` |
| `ext_gob_anuncio_vinculo` | `(ascasu, destino_tipo, destino_clave)` | Vínculos de §4 |
| `ext_gob_privada_sync_log` | — | Un registro por corrida, sin actualizar filas anteriores, como el resto de los syncs |

`ext_gob_anuncio_vinculo`:

| Campo | Tipo | Nota |
|---|---|---|
| `ascasu` | str | Anuncio |
| `destino_tipo` | enum `focom` · `atp` · `compromiso` | |
| `destino_clave` | str | `id_focom`, `id_atp` o la huella del compromiso (§5.2) |
| `destino_sheet_row` | int null | Solo para `compromiso`: la fila al momento de vincular |
| `metodo` | enum `puntero` · `puntero_corrido` · `difuso` · `llm` · `manual` | Capa que lo propuso (§4) |
| `confianza` | numeric(3,2) | 0–1 |
| `estado` | enum `confirmado` · `sugerido` · `rechazado` | Solo `confirmado` se muestra como vínculo |
| `motivo` | text | Puntaje del difuso o explicación breve del LLM |
| `revisado_por`, `revisado_at` | | Para `manual` y para la revisión de sugeridos |

Un anuncio puede tener muchos vínculos y un compromiso puede estar vinculado a más de un anuncio.

## 3. Sincronización

### 3.1 Disparo

- `POST /internal/datos-externos/gob-privada/sync`: solo IAM. Lo llama Cloud Scheduler todos los
  días.
- Se suma a `POST /api/v1/datos-externos/tablero/actualizar` como una fuente más
  (`spec-tablero-gobierno-actualizar.md`).
- Cada fila se escribe en su propio `SAVEPOINT`. El sync distingue "falló una fila" de "falló toda
  la lectura" (502), con el mismo patrón que `checklist_sync.py`.

### 3.2 Anuncios y FOCOM

Se hace un upsert por clave natural. Si una clave desaparece de la planilla, la fila se marca con
`deleted_at`: no se borra.

### 3.3 Histórico ATP

- **Fuente**: la pestaña `Gobierno!ATP`, rango `A1:E`. La columna `F` (`REFERENTE`) no se pide.
  - `IMPORTRANGE` se calcula del lado de Google, así que la API de Sheets devuelve los valores
    importados sin que la cuenta de servicio tenga acceso a la planilla de origen. **Esto hay que
    comprobarlo en la primera corrida real.**
  - Si el `IMPORTRANGE` falla (`#REF!` o la pestaña vacía), la corrida no borra el histórico ya
    cargado. Se registra como "falló toda la lectura" y se conserva el último dato bueno.
- **Fuente alternativa**: la planilla semanal `1ySgcr1t…` (`'ATP ACUMULADO'!A4:E`). Se configura
  con `settings.google_sheet_atp_semanal_id`, vacío por defecto. Si el área la comparte con
  `svc-datos-externos@`, se pasa a leer de ahí sin cambiar el código. Las filas son las mismas, así
  que el cambio de fuente no debería mover ningún total.
- **Corte**: solo se guardan filas con fecha **anterior al 2026-01-01**. 2026 sigue viniendo de ATP
  Interno, que tiene `ID_LOC` y los departamentos corregidos. Así no se cuenta nada dos veces.
- **Control**: en cada corrida el sync compara el 2026 de la planilla semanal con ATP Interno
  (cantidad de filas y suma) y registra la diferencia en el log. Si deja de coincidir, el área
  cambió algo y hay que revisarlo. No es un error del sync.

### 3.4 Localidades

- FOCOM: `ID_LOC` → `id_geo`, con el mismo camino que la Fase 1.
- Anuncios y ATP histórico: usan `POST /internal/geo/resolver-localidades` de `svc-vivienda`. El
  servicio no tiene alias propios (ADR-025).
  - Lo que no resuelve queda con `id_geo NULL` y se cuenta en el log.
  - "Comunidad regional" y "PROVINCIA" no son localidades y quedan sin vínculo a propósito.

## 4. Vinculación de anuncios (capas)

Cada corrida vuelve a calcular solo los anuncios **nuevos o modificados** (`hash_contenido`
distinto) y los vínculos de compromisos cuya huella cambió (§5.2). **Nunca se pisa un vínculo
`confirmado` por `manual` ni uno `rechazado`.**

| Capa | Regla | Resultado | Volumen estimado |
|---|---|---|---|
| 1. Puntero FOCOM/ATP | `R` = `N-FOCOM` / `N-ATP` y el ID existe | `confirmado`, confianza 1 si coincide la localidad; `sugerido` si no | 223 FOCOM (214 con localidad OK), 85 ATP (53 OK) |
| 2. Puntero numérico a BD | `R` = número: se busca la fila de BD con desfasaje +5 y, si la localidad no coincide, ±1–3 filas | `confirmado` si la localidad coincide y hay fecha o monto que confirme; si no, pasa a la capa 3 | ~250 de 322 |
| 3. Difuso | Bloqueo por localidad. Puntaje: fecha ±3 días (+3), monto de BD presente en `montos_mencionados` (+3), similitud de texto ≥70 (+2) / ≥50 (+1) | Puntaje ≥6 con ganador único: `confirmado`. Puntaje 3–5 o empate: pasa a la capa 4 | ~100–200 adicionales |
| 4. LLM | Para cada caso ambiguo, el modelo recibe el anuncio y hasta 8 candidatos de la misma localidad (±60 días, o sin fecha) y elige 0, 1 o varios | Siempre `sugerido`, con `confianza` y `motivo` | ~400–450 filas de BD la primera vez; después, solo lo nuevo |
| 5. Revisión humana | Pantalla de revisión (§7.3) | `confirmado` o `rechazado` por `manual` | ~100–150 residuales, más los sugeridos de la capa 4 |

La capa 2 se validó contra la capa 3: en los matches de alta confianza que tienen puntero, coinciden
178 y no coinciden 7 (~96 %).

### 4.1 Por qué no alcanza el cruce determinístico

- No hay un ID compartido entre `Gobierno` y BD.
- El puntero numérico se fue corriendo al insertar filas.
- En los actos con varios destinos, el texto de prensa no dice el monto de cada destino. Por
  ejemplo: "17 instituciones" en BD contra "5 M por institución" en el anuncio.

Emparejar esos casos requiere interpretar el sentido del texto, y ahí se usa el LLM (capa 4).

### 4.2 Reglas del LLM

- Se usa la API de Anthropic con la misma clave de Secret Manager que `svc-agente` (ADR-033). El
  modelo se configura con `settings.vinculador_modelo`.
- La respuesta es JSON con esquema validado por Pydantic: `{candidatos_elegidos: [...], confianza, motivo}`.
  Si la respuesta no es válida, el caso queda sin vínculo y se cuenta en el log. **El sync no
  falla por esto.**
- **Qué se le envía**: localidad, departamento, fecha, el texto del anuncio y el detalle, y de cada
  candidato BD su destino, monto, ministerio y fecha. **Nunca** se envían referentes, comentarios,
  ni datos de personas.
- **Tope por corrida**: `settings.vinculador_max_llamadas` (por defecto 100). Lo que no entra queda
  para la corrida siguiente. Todo el procesamiento se hace en el backend, nunca en el navegador.
- `settings.vinculador_llm_habilitado` vale `false` por defecto. Las capas 1–3 funcionan sin el
  LLM.
- **Al principio el LLM nunca confirma solo** (§4.3).

### 4.3 Del modo sugerencia a la confirmación automática (decisión del usuario, 2026-11-19)

**Fase de validación** (inicial):
- Todo vínculo de la capa 4 queda `sugerido`.
- En cada revisión humana se guarda en el mismo vínculo la `confianza` original del LLM y el
  resultado (`confirmado` o `rechazado`).
- `GET .../tablero/anuncios/vinculos/precision` (solo `Admin`) muestra, por tramo de confianza
  (`≥0,9`, `0,7–0,9`, `<0,7`): sugeridos, revisados, confirmados y precisión.

**Paso a confirmación automática:**
- Se configura con `settings.vinculador_umbral_auto`, vacío por defecto (sin confirmación
  automática).
- Lo enciende un `Admin` cuando el tramo `≥0,9` tenga **al menos 50 vínculos revisados con
  precisión ≥95 %**.
- Desde ese momento:
  - los vínculos con confianza mayor o igual al umbral nacen `confirmado`, con `metodo = llm`;
  - el resto sigue `sugerido`;
  - el endpoint de precisión sigue midiendo, sobre una muestra que el `Admin` revisa.
- Es reversible: si se vacía el umbral, se vuelve al modo sugerencia. Los vínculos ya confirmados
  se pueden rechazar uno por uno.
- El cambio queda registrado en el changelog de este spec.

Para no inflar el número, un vínculo que el LLM confirmó solo nunca cuenta en la precisión como
"revisado".

## 5. Datos de Compromisos (cruce entre servicios)

### 5.1 Por qué hace falta un endpoint

BD vive en `db_gralgob` (`svc-gralgob`). Por ADR-001 no se hacen joins entre bases. El vinculador
lee una proyección liviana por IAM, con el mismo patrón que los `rollup-territorial`.

### 5.2 Endpoint nuevo en `svc-gralgob`

`GET /internal/atp/compromisos-vinculacion`: solo IAM. `svc-datos-externos@` recibe `run.invoker`
sobre `svc-gralgob`. Devuelve, por cada compromiso:

- `sheet_row_number`, `departamento`, `localidad`, `id_geo`, `fecha_anuncio`, `destino`,
  `ministerio_destino` y `monto_anunciado`;
- `huella`: hash de (localidad normalizada, fecha, monto, destino normalizado).

Nunca devuelve referentes.

**Problema conocido**: la clave de `atp_compromisos` es `sheet_row_number`. Si el área inserta una
fila, la clave cambia (lo dice el comentario en `svc-gralgob/app/atp/models.py`). Por eso el vínculo
guarda la **huella** como `destino_clave` y `sheet_row_number` solo como dato informativo:

- Si la huella sigue existiendo en otra fila: se actualiza `destino_sheet_row`.
- Si la huella desaparece (el área editó monto, fecha o destino): el vínculo vuelve a `sugerido`
  con el motivo "compromiso modificado".

## 6. Privacidad

- **No se leen**: `REFERENTE POLITICO`, `ADMINISTRATIVO ASIGNADO`, `COORDINADOR ASIGNADO`,
  `OBSERVACIONES` y `OBSERVACIONES - ENCARGADO` (FOCOM), y `REFERENTE` (ATP). Los rangos que se
  piden a Sheets saltean esas columnas. Es el mismo criterio que la Fase 3 y que ATP Interno.
- **Comentarios de los anuncios** (`Q`): se guardan porque sirven para vincular (por ejemplo, "HAY
  UN ATP DE 30 MILLONES"). Solo los ven `Admin` y `Autoridad`.
- Ningún dato personal se envía al LLM (§4.2).

## 7. Qué se expone (gateway y pantallas)

Todas las rutas van bajo `/api/v1/datos-externos/tablero/…`, son de solo lectura y exigen rol
`Admin` o `Autoridad`, igual que las Etapas 4–6. Necesitan una config nueva del gateway
(`ministerio-config-v{fecha}`), con su `options` para CORS.

### 7.1 Anuncios del Gobernador (pantalla nueva)

`GET .../tablero/anuncios`, paginado, con filtros de año, departamento, localidad, estado y
"con/sin vínculo".

- KPIs: anuncios, % pagado, % con vínculo.
- Tabla: fecha, localidad, anuncio, estado, link de prensa y chips de vínculos confirmados
  (FOCOM N, ATP N, Compromiso).
- Descarga a Excel.

### 7.2 FOCOM (completa la pantalla de la Fase 3)

`GET .../tablero/fondos/focom`. Reemplaza el "en preparación" con KPIs (monto, pagado, por
categoría y estado de obra) y una tabla con descarga.

### 7.3 Revisión de vínculos (solo `Admin`)

- `GET .../tablero/anuncios/vinculos?estado=sugerido` muestra el anuncio al lado de su o sus
  candidatos, con el motivo.
- `PATCH .../tablero/anuncios/vinculos/{id}` permite confirmar o rechazar.
- `POST .../tablero/anuncios/{ascasu}/vinculos` permite agregar un vínculo manual.
- Todas las escrituras quedan en el registro de auditoría.
- Es la **única escritura** de este spec, y es sobre datos propios (`ext_gob_anuncio_vinculo`),
  nunca sobre la planilla.

### 7.4 ATP histórico

Las pantallas ATP del Tablero suman los años 2023–2025 desde `ext_gob_atp_historico`, rotulados
"Histórico (planilla semanal)". El filtro de año decide de qué tabla salen los datos: 2026 sigue
saliendo de ATP Interno.

## 8. Tests y verificación

- Parser: fechas con y sin cero inicial, años fuera de rango, montos y extracción de
  `montos_mencionados` de textos reales.
- Las capas 1–3 se prueban con datos sintéticos que cubren: puntero bien, puntero corrido, empate,
  acto con varios destinos y BD sin fecha.
- La capa 4 se prueba con un cliente simulado: respuesta válida, inválida, timeout y tope de
  llamadas.
- Huella: si se inserta una fila en BD, se reubica el vínculo; si se edita el monto, el vínculo
  vuelve a `sugerido`.
- No se pisa un vínculo `manual`.
- Corte del 2026-01-01 del ATP histórico y control contra ATP Interno.
- Verificación contra las planillas reales en un Postgres 15 descartable (no en SQLite, ver
  [[feedback_audit_log_schema_y_tests_sqlite]]). Se registran los volúmenes de cada capa y se
  comparan con §4.

## 9. Preguntas para el área (no bloquean las Etapas A y B)

1. ¿La planilla semanal "ATP TRANSFERIDOS…" es la oficial del histórico? ¿"4. ATP 2026" se sigue
   cargando desde ella? Si es así, ¿por qué difieren los departamentos de 5 filas?
2. "Indicador de seguimiento Gabriel" con número simple: ¿apunta a la fila de BD de Compromisos?
   ¿Lo mantienen cuando insertan filas?
3. ¿Qué significa "Coinciden"? Se cree que compara Estado con Estado Gabriel.
4. ¿Por qué `Ministerio` está vacío en 2025–2026? ¿Se puede tomar el de BD cuando hay vínculo?
5. ~~Confirmación automática de los vínculos del LLM~~: la decidió el usuario (§4.3).
6. ¿Pueden compartir la planilla semanal `1ySgcr1t…` con `svc-datos-externos@`? No bloquea nada,
   porque se lee `Gobierno!ATP` (§3.3).

## 10. Etapas

| Etapa | Contenido | Depende de |
|---|---|---|
| A | Sync de anuncios y FOCOM, más la pantalla FOCOM (§7.2) | Aprobación de este spec |
| B | ATP histórico 2023–2025 (§3.3 y §7.4) | A. Se lee de `Gobierno!ATP`, que ya tiene acceso |
| C | Vinculación capas 1–3, endpoint de `svc-gralgob` (§5.2) y pantalla de anuncios (§7.1) | A |
| D | Capa 4 (LLM) en modo sugerencia, pantalla de revisión (§7.3) y medición de precisión (§4.3) | C, y medir el volumen real de casos ambiguos |
| D2 | Encender `vinculador_umbral_auto` | D, más ≥50 revisados con precisión ≥95 % en el tramo `≥0,9` |
| E | Federación a `resumen_territorial` y ficha de localidad | Spec propio |

## 11. Criterios de aceptación

- El sync lee las 1.324 filas de anuncios y todos los FOCOM, sin descartes salvo los contados en el
  log, y nunca lee las columnas de §6.
- El ATP histórico suma $70.356 M (±0,1 %) en 2023–2025 y no trae filas de 2026.
- Las capas 1–3 confirman al menos 400 vínculos de anuncio con compromiso, con una precisión de
  ≥95 % sobre una muestra de 50 revisada a mano.
- Cuando cambia la huella de un compromiso, el vínculo vuelve a `sugerido`: no queda apuntando a
  una fila equivocada.
- Con `vinculador_llm_habilitado=false` todo funciona, salvo la capa 4.
- Ningún endpoint devuelve referentes ni datos de personas. Los comentarios solo los ven `Admin` y
  `Autoridad`.
