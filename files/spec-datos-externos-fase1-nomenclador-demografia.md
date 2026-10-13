# Spec: Datos externos — Fase 1: nomenclador INDEC de Gobierno + demografía por localidad

**Estado**: review
**Versión**: 0.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-13
**Servicios**: `svc-datos-externos` (sync + rollup) · `svc-vivienda` (`resumen_territorial`, passthrough) · `frontend` (`resumen-territorial`)

Spec hijo de `spec-datos-externos-tablero-gobierno.md` (draft, paraguas) — es su Fase 1.
Extiende `spec-resumen-territorial-tablero-v2.md` (approved): cierra dos de los huecos de datos
de su §5 (superficie/densidad y población 2010/crecimiento intercensal).

---

## 1. Propósito

1. Incorporar el **nomenclador de localidades de Gobierno** (código INDEC de gobierno local) como
   **llave alternativa** hacia nuestro padrón oficial. Todas las planillas del Tablero de
   Gobierno identifican la localidad por ese código; con este puente, las fases siguientes
   vinculan por código y no por nombre.
2. Incorporar **superficie** y **población 2010** por localidad, y mostrar **densidad** y
   **crecimiento intercensal** en el Resumen Territorial, hoy en "N/D".

Decisión confirmada por el usuario (2026-10-13): el padrón oficial sigue siendo
`viv_geo_localidades` (ADR-024/026). El código INDEC **no lo reemplaza**.

## 2. Alcance

### Incluido

- Sync de solo lectura de dos pestañas de Google Sheets hacia `db_datos_externos`.
- Tabla puente código INDEC ↔ `id_geo`, con código de coparticipación (`ID_COPA`) y tipo.
- Catálogo de "entes" que usan las planillas de Gobierno y **no son localidades** (comunidades
  regionales y zonas rurales).
- Cierre de las filas de `ext_geo_censo` que quedaron sin `id_geo`.
- Dos campos nuevos en el rollup (`superficie_km2`, `poblacion_2010`) y su paso por
  `resumen_territorial` hasta el frontend.
- KPIs de densidad y crecimiento intercensal en el tablero y en la ficha de localidad.

### Fuera de alcance

- Intendente, partido, reelecto, continuidad, legislador (Fase 6, ADR-027).
- **Contacto de intendentes: no se lee nunca en esta fase** (ver §3.3).
- Empleados y masa salarial (Fase 5, junto con la planilla "13. EMPLEADOS").
- Electores: se guarda, pero **no se expone** — hoy el dato visible sale de Privada (ADR-012) y
  cuál manda se define con ADR-027.
- Endpoints por gateway o panel propio (ADR-028 los habilita; esta fase no los necesita).
- Cualquier escritura hacia las planillas.

## 3. Fuentes

Compartidas como Lector con `svc-datos-externos@gestorcooperativo.iam.gserviceaccount.com`
(verificado 2026-10-13). Lectura con `valueRenderOption=UNFORMATTED_VALUE`.

### 3.1 Nomenclador — "0. LOCALIDADES - NOMBRES"

Sheet `1RlV9U86RjRZafet8TOyCchwDjlP7Gcr95s7FVuNgCXg`, pestaña `LOCALIDADES`. Fila 1 es un título,
**encabezados en fila 2**, datos desde fila 3. 478 filas:

| Grupo | Filas | `ID_LOC` | `ID_DPTO` | `ID_COPA` | `TIPO` |
|---|---|---|---|---|---|
| Gobiernos locales | 427 | entero de 6 dígitos (código INDEC) | código INDEC de departamento (`14007`) | entero, único | `MU` (271) / `CO` (156) |
| Comunidades regionales | 26 | entero `428`–`453` | ordinal `1`–`26` | vacío | vacío |
| Zonas rurales | 25 | texto `nnZR` | ordinal | vacío | vacío |

Los 427 códigos coinciden **exactamente** con los 427 de `ext_geo_censo.codigo_indec`.

### 3.2 Demografía — "0. Resumen General", pestaña `DATOS LOC`

Sheet `1Xl4uIIH39_L3HiosrrbXNrYaAbRb8p2pmrwggd7A0vM`. Encabezados en fila 1. 452 filas (427
gobiernos locales + 25 zonas rurales). Columnas que se leen:

| Columna | Uso | Observado |
|---|---|---|
| `ID_LOC` | clave | — |
| `SUPERFICIE` | `superficie_km2` | 427/427 numérico. Es la superficie del **radio municipal** en km² (suma provincial ≈ 9.727 km²), no la del departamento. 49 localidades tienen menos de 0,5 km² |
| `POBLACION 2010` | `poblacion_2010` | 427/427 numérico |
| `POBLACION 2022` | control contra el Censo | Difiere del Censo cargado en 3 localidades |
| `ELECTORES` | se guarda, no se expone | 427/427 numérico |

Las zonas rurales traen población (≈70.000 habitantes en 2022) y superficie 0. Explican la
diferencia entre los dos totales provinciales del Looker (3.840.905 con zona rural, 3.770.861
solo gobiernos locales).

### 3.3 Regla de lectura por lista blanca

El sync pide el rango `A1:T` de `DATOS LOC` y **toma columnas por nombre de encabezado**, solo
las de la tabla de §3.2. La columna de contacto (`celular`, columna U) queda fuera del rango
pedido y además fuera de la lista blanca: si la planilla se reordena, el dato no entra igual.

## 4. Modelo de datos (Alembic `0003`, prefijo `ext_`)

```
ext_gob_localidades                 -- puente INDEC ↔ padrón oficial (solo los 427 gobiernos locales)
  codigo_indec        VARCHAR(20) PK
  nombre_gob          VARCHAR(150) NOT NULL   -- texto de la planilla, nunca se pierde
  departamento_gob    VARCHAR(100) NOT NULL
  codigo_dpto_indec   VARCHAR(10)
  id_copa             INTEGER NULL UNIQUE     -- código de la localidad en las planillas de coparticipación
  tipo                VARCHAR(2)              -- "MU" | "CO"
  id_geo              VARCHAR(20) NULL        -- resuelto vía ADR-024; sin FK (cross-DB)
  match_tipo          VARCHAR(20) NULL
  superficie_km2      NUMERIC(12,4) NULL      -- radio municipal
  poblacion_2010      INTEGER NULL
  poblacion_2022_gob  INTEGER NULL            -- solo control; la población que se muestra es la del Censo
  electores           INTEGER NULL            -- no se expone en esta fase
  created_at, updated_at  TIMESTAMPTZ
  deleted_at          TIMESTAMPTZ NULL        -- la fila dejó de venir en la planilla

ext_gob_entes                       -- códigos de Gobierno que NO son localidades
  codigo              VARCHAR(20) PK          -- "428" | "1ZR"
  nombre              VARCHAR(150) NOT NULL
  tipo_ente           VARCHAR(20) NOT NULL    -- "comunidad_regional" | "zona_rural"
  departamento        VARCHAR(100)
  created_at, updated_at, deleted_at

ext_gob_sync_log                    -- append-only, una fila por corrida y fuente
  id                  UUID PK
  fuente              VARCHAR(40) NOT NULL    -- "nomenclador" | "datos_loc" (fases siguientes suman valores)
  filas_leidas        INTEGER
  filas_upsert        INTEGER
  filas_error         INTEGER
  filas_sin_match     INTEGER
  anomalias           JSON
  estado              VARCHAR(20)             -- "ok" | "parcial" | "fallido"
  corrida_en          TIMESTAMPTZ
  disparado_por       VARCHAR(80)
```

Una sola tabla para nomenclador y demografía: son 427 filas con la misma clave y se consultan
siempre juntas. `ext_gob_sync_log` nace genérico (columna `fuente`) para que las fases
siguientes no creen un log por planilla.

## 5. Sync

`POST /internal/datos-externos/gobierno/localidades/sync` — IAM-only, mismo patrón que el resto
de `app/internal/router.py`. Cloud Scheduler **diario**.

1. Lee el nomenclador. Valida encabezados esperados en fila 2; si faltan, falla entero (502) sin
   tocar datos.
2. **Freno de seguridad**: si vienen menos de 400 gobiernos locales, falla entero. Una planilla
   vaciada o mal filtrada no debe dar de baja el padrón.
3. Clasifica cada fila: gobierno local (`TIPO` en `MU`/`CO` y código de 6 dígitos) o ente.
   Normaliza `ID_LOC` quedándose solo con dígitos cuando es numérico (hay planillas que lo
   muestran como `140.105`).
4. Resuelve `id_geo` con el resolver existente (`app/integrations/geo_resolver.py`, ADR-024)
   pasando `(departamento_gob, nombre_gob)`, origen `datos_externos`. Las que no resuelven
   aparecen en la pantalla de localidades sin resolver de `svc-vivienda`, como cualquier otra
   fuente. No se crea una tabla de alias propia.
5. Upsert por `codigo_indec`, cada fila en su propio `SAVEPOINT`. Las filas presentes en la base
   y ausentes en la planilla se marcan `deleted_at` (no se borran).
6. Lee `DATOS LOC` (lista blanca, §3.3) y actualiza las columnas demográficas por
   `codigo_indec`. Valores no numéricos o negativos se guardan como `NULL` y se anotan en
   `anomalias`. Si esta lectura falla, el nomenclador ya sincronizado se conserva y la corrida
   queda `parcial`.
7. **Cierre del Censo**: para las filas de `ext_geo_censo` con `id_geo IS NULL`, copia
   `id_geo`/`departamento_gob` desde `ext_gob_localidades` por `codigo_indec` (`match_tipo =
   "nomenclador_gob"`). Nunca pisa un `id_geo` ya resuelto.
8. Anota en `anomalias`: códigos del Censo que no están en el nomenclador (y al revés), y
   localidades donde `poblacion_2022_gob` difiere del Censo.

Cliente de Sheets: se copia `app/integrations/google_sheets.py` de `svc-gralgob` (ADC, sin
claves), agregando el parámetro de `valueRenderOption`.

`GET /internal/datos-externos/gobierno/localidades/sync-estado` devuelve la última corrida por
fuente.

Configuración nueva (`config.py`, defaults vacíos/apagados; valores reales en
`services/cloudbuild.yaml`): `google_sheet_gob_nomenclador_id`,
`google_sheet_gob_resumen_general_id`, `gob_localidades_sync_enabled`.

## 6. Rollup y federación

`GET /internal/datos-externos/rollup-territorial` suma dos campos por `id_geo`:
`superficie_km2` y `poblacion_2010` (ambos `null` si no hay dato). El cambio es aditivo: no se
renombra ni se quita nada. La población que se expone sigue siendo `poblacion_2022` del Censo.

En `svc-vivienda`: `aggregations.py` pasa los dos campos tal cual y `schemas.py` los declara
opcionales en la localidad. No se calcula nada en backend.

No cambia `infra/gateway/openapi.yaml`: no hay paths nuevos por gateway.

## 7. Frontend (`src/modules/resumen-territorial/`)

Cálculo en cliente, igual que el resto de los indicadores del tablero:

- **Densidad** = población 2022 / superficie. Etiqueta: **"Densidad en el radio municipal
  (hab/km²)"** — no es la densidad del departamento, y la etiqueta lo tiene que decir.
- **Crecimiento intercensal** = población 2022 / población 2010 − 1, en porcentaje.
- A nivel departamento y provincia: suma de población sobre suma de superficie, y suma 2022
  sobre suma 2010, tomando solo localidades con ambos datos. La etiqueta del agregado de
  densidad aclara "radios municipales".
- Si falta alguno de los datos: "N/D", sin el texto actual de "falta fuente".

Lugares a tocar: `VistaProvincia.tsx` (los dos KPIs hoy en "N/D" y su texto explicativo),
`utils/departamentoAgregados.ts`, `types/resumenTerritorial.types.ts`, la ficha de localidad
(`FichaLocalidadPage.tsx`) y su exportación PDF/Excel (`fichaMunicipio.ts`): filas "Población
(Censo 2010)", "Superficie del radio municipal", densidad y crecimiento.

## 8. Tests

- Clasificación de filas del nomenclador: gobierno local, comunidad regional, zona rural, fila
  con código inválido.
- Normalización de `ID_LOC` (`142007`, `"142.007"`, `"1ZR"`).
- Lista blanca: una planilla con columnas reordenadas o con una columna extra no cambia lo que
  se guarda; el contacto no se persiste en ningún caso.
- Freno de seguridad (< 400 filas) y encabezados faltantes: falla sin tocar datos.
- Una fila mala no frena el lote (SAVEPOINT).
- Baja lógica de una fila que desaparece; reaparición la reactiva.
- Cierre del Censo: completa nulos, no pisa resueltos.
- Rollup: campos nuevos presentes y `null` cuando no hay dato.
- `svc-vivienda`: passthrough de los dos campos.

## 9. Criterios de aceptación

1. Migración `0003` aplicada en `db_datos_externos` (commiteada antes de correrla).
2. Corrida real: 427 gobiernos locales y 51 entes cargados, sin errores de fila.
3. Se informa cuántos de los 427 quedaron con `id_geo` y cuáles no; los no resueltos figuran en
   la pantalla de localidades sin resolver.
4. Se informa cuántas filas de `ext_geo_censo` sin `id_geo` se cerraron.
5. Cloud Scheduler diario creado y con una ejecución exitosa.
6. El rollup devuelve `superficie_km2` y `poblacion_2010`; tras recalcular el Resumen
   Territorial, el tablero muestra densidad y crecimiento intercensal en provincia, departamento
   y localidad, y la ficha (pantalla, PDF y Excel) los incluye.
7. Verificación visual por el usuario en producción.
8. `spec-resumen-territorial-tablero-v2.md §5` actualizado: los dos huecos pasan a resueltos.

## 10. Preguntas abiertas (no bloquean la aprobación)

1. Radios muy chicos (49 localidades con menos de 0,5 km²) dan densidades muy altas. Propuesta:
   mostrarlas tal cual, con la aclaración en la etiqueta. Alternativa: "N/D" por debajo de un
   umbral.
2. Las 3 localidades donde la población de Gobierno difiere del Censo: se informa al área; el
   sistema sigue mostrando el Censo.
3. Población de zonas rurales: hoy no se guarda. Sumarla permitiría mostrar el total provincial
   completo (3,84 M en vez de 3,77 M).

## 11. Aprobación

Pendiente de aprobación del usuario para pasar a `approved` e implementar.
