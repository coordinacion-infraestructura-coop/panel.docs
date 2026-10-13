# Spec: Datos externos — Fase 1: nomenclador INDEC de Gobierno + ficha de localidad (demografía y datos institucionales)

**Estado**: approved
**Versión**: 1.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-13
**Servicios**: `svc-datos-externos` (sync + rollup) · `svc-vivienda` (`resumen_territorial`) · `frontend` (`resumen-territorial`)

Spec hijo de `spec-datos-externos-tablero-gobierno.md` (approved, paraguas) — es su Fase 1.
Extiende `spec-resumen-territorial-tablero-v2.md` (approved): cierra dos de los huecos de datos
de su §5 (superficie/densidad y población 2010/crecimiento intercensal).

## Changelog

- **1.1.0** (2026-10-13): corrección del usuario — los datos institucionales los ven los roles
  **`Admin` y `Autoridad`** (antes: solo `Autoridad`). Confirmó además que el dato viaje por el
  snapshot de `db_vivienda`, con la condición de que los usuarios de Vivienda no puedan verlo (§7).
- **1.0.2** (2026-10-13): código escrito en la rama `datos-externos-fase1-nomenclador` de
  `services/` y `frontend/` (sin commitear ni desplegar; el spec sigue en `approved`). Precisiones
  que salieron de la implementación, en §5.1.
- **1.0.1** (2026-10-13): respuestas del área de Gobierno (§11). Sin cambio de alcance. La
  población de zona rural es una estimación del área y se rotula así (§3.2, §8).
- **1.0.0** (2026-10-13): aprobado por el usuario con correcciones. Entran a la Fase 1 los datos
  institucionales y sensibles de `DATOS LOC` (electores, empleados, masa salarial, intendente,
  partido, reelecto, continuidad, legislador y contacto del intendente), visibles **solo para el
  rol `Autoridad`** (§7). Se suma la población de zonas rurales (§6.2). Respuestas a las
  preguntas abiertas en §11.
- **0.1.0** (2026-10-13): primera versión, en review.

---

## 1. Propósito

1. Incorporar el **nomenclador de localidades de Gobierno** (código INDEC de gobierno local) como
   **llave alternativa** hacia nuestro padrón oficial. Todas las planillas del Tablero de
   Gobierno identifican la localidad por ese código; con este puente, las fases siguientes
   vinculan por código y no por nombre.
2. Incorporar **superficie** y **población 2010** por localidad, y mostrar **densidad** y
   **crecimiento intercensal** en el Resumen Territorial, hoy en "N/D".
3. Cargar en el sistema la **ficha institucional** de cada localidad que mantiene Gobierno, con
   acceso restringido a los roles `Admin` y `Autoridad`.

Decisión confirmada por el usuario (2026-10-13): el padrón oficial sigue siendo
`viv_geo_localidades` (ADR-024/026). El código INDEC **no lo reemplaza**.

## 2. Alcance

### Incluido

- Sync de solo lectura de dos pestañas de Google Sheets hacia `db_datos_externos`.
- Tabla puente código INDEC ↔ `id_geo`, con código de coparticipación (`ID_COPA`) y tipo.
- Catálogo de "entes" que usan las planillas de Gobierno y **no son localidades** (comunidades
  regionales y zonas rurales), con la población de las zonas rurales.
- Cierre de las filas de `ext_geo_censo` que quedaron sin `id_geo`.
- Datos públicos nuevos: superficie, población 2010, densidad y crecimiento intercensal.
- Datos restringidos a `Admin` y `Autoridad`: electores, empleados, masa salarial, intendente, partido,
  reelecto, continuidad, legislador y su partido, contacto del intendente.

### Fuera de alcance

- **Escritura desde el sistema hacia las planillas**: ninguna. El sistema solo lee.
- Edición de estos datos dentro del sistema (Fase 8 del spec paraguas).
- Cambiar de dónde sale el intendente/partido que hoy muestra la ficha desde Privada: sigue
  igual hasta que se defina el mecanismo de ADR-027 (ver §7.3).
- La columna `FOTO` (enlace a Drive): no se sincroniza.
- Endpoints por gateway o panel propio en `svc-datos-externos` (ADR-028 los habilita; esta fase
  no los necesita).

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

### 3.2 Ficha de localidad — "0. Resumen General", pestaña `DATOS LOC`

Sheet `1Xl4uIIH39_L3HiosrrbXNrYaAbRb8p2pmrwggd7A0vM`. Encabezados en fila 1. 452 filas (427
gobiernos locales + 25 zonas rurales).

| Columna de la planilla | Campo | Visibilidad | Observado |
|---|---|---|---|
| `ID_LOC` | clave | — | — |
| `SUPERFICIE` | `superficie_km2` | Pública | 427/427 numérico. Es la superficie del polígono del **radio municipal** en km², fuente IDECOR (confirmado por el área; suma provincial ≈ 9.727 km²), no la del departamento. 49 localidades tienen menos de 0,5 km² |
| `POBLACION 2010` | `poblacion_2010` | Pública | 427/427 numérico. Censo 2010, INDEC (confirmado por el área) |
| `POBLACION 2022` | `poblacion_2022_gob` | No se expone (control) | Difiere del Censo cargado en 3 localidades, por ajustes manuales del área (§11) |
| `ELECTORES` | `electores` | Admin y Autoridad | 427/427 |
| `EMPLEADOS` | `empleados` | Admin y Autoridad | 427/427 |
| `MASA SALARIAL` | `masa_salarial` | Admin y Autoridad | 425/427 con dato |
| `INTENDENTE` | `intendente` | Admin y Autoridad | 427/427 |
| `REELECTO` | `reelecto` | Admin y Autoridad | SI/NO |
| `PARTIDO` | `partido` | Admin y Autoridad | 427/427 |
| `CONTINUIDAD` | `continuidad` | Admin y Autoridad | SI/NO |
| `LEGISLADOR` | `legislador` | Admin y Autoridad | dato departamental repetido por localidad |
| `PARTIDO_LEG` | `partido_legislador` | Admin y Autoridad | ídem |
| `celular` | `contacto_intendente` | Admin y Autoridad | dato personal |

`EMPLEADOS` no estaba en la lista del usuario; se incluye junto con `MASA SALARIAL` porque son el
mismo relevamiento y se trata como restringido.

Las 25 filas de zona rural traen población 2022 y 2010 (≈70.000 y ≈98.000 habitantes) y
superficie 0. Explican la diferencia entre los dos totales provinciales del Looker (3.840.905 con
zona rural, 3.770.861 solo gobiernos locales). **Es una estimación**: el Censo 2022 no desagrega
la población sin gobierno local por departamento; el área la reparte según la participación
general de cada departamento.

### 3.3 Regla de lectura por lista blanca

El sync **toma columnas por nombre de encabezado**, solo las de la tabla de §3.2. Una columna
nueva en la planilla no entra al sistema hasta que este spec la sume; una columna que cambia de
posición se sigue leyendo bien; una columna esperada que falta hace fallar la corrida entera sin
tocar datos.

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
  -- públicos
  superficie_km2      NUMERIC(12,4) NULL      -- radio municipal
  poblacion_2010      INTEGER NULL
  poblacion_2022_gob  INTEGER NULL            -- solo control; la población que se muestra es la del Censo
  -- restringidos (solo Admin y Autoridad)
  electores           INTEGER NULL
  empleados           INTEGER NULL
  masa_salarial       NUMERIC(18,2) NULL
  intendente          VARCHAR(150) NULL
  reelecto            BOOLEAN NULL
  partido             VARCHAR(150) NULL
  continuidad         BOOLEAN NULL
  legislador          VARCHAR(150) NULL
  partido_legislador  VARCHAR(150) NULL
  contacto_intendente VARCHAR(40) NULL        -- dato personal
  created_at, updated_at  TIMESTAMPTZ
  deleted_at          TIMESTAMPTZ NULL        -- la fila dejó de venir en la planilla

ext_gob_entes                       -- códigos de Gobierno que NO son localidades
  codigo              VARCHAR(20) PK          -- "428" | "1ZR"
  nombre              VARCHAR(150) NOT NULL
  tipo_ente           VARCHAR(20) NOT NULL    -- "comunidad_regional" | "zona_rural"
  departamento        VARCHAR(100)
  poblacion_2022      INTEGER NULL            -- solo zona_rural
  poblacion_2010      INTEGER NULL            -- solo zona_rural
  created_at, updated_at, deleted_at

ext_gob_sync_log                    -- append-only, una fila por corrida y fuente
  id                  UUID PK
  fuente              VARCHAR(40) NOT NULL    -- "nomenclador" | "datos_loc" (fases siguientes suman valores)
  filas_leidas        INTEGER
  filas_upsert        INTEGER
  filas_error         INTEGER
  filas_sin_match     INTEGER
  anomalias           JSON                    -- nunca incluye valores de campos restringidos
  estado              VARCHAR(20)             -- "ok" | "parcial" | "fallido"
  corrida_en          TIMESTAMPTZ
  disparado_por       VARCHAR(80)
```

`ext_gob_sync_log` nace genérico (columna `fuente`) para que las fases siguientes no creen un log
por planilla. Los logs de aplicación tampoco deben volcar valores de campos restringidos.

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
6. Lee `DATOS LOC` (lista blanca, §3.3) y actualiza por `codigo_indec`: los gobiernos locales en
   `ext_gob_localidades`, las zonas rurales (población) en `ext_gob_entes`. Números no numéricos
   o negativos se guardan como `NULL` y se anotan en `anomalias` (solo el nombre del campo y el
   código, nunca el valor). `SI`/`NO` pasan a booleano; cualquier otro valor, `NULL`. Si esta
   lectura falla, el nomenclador ya sincronizado se conserva y la corrida queda `parcial`.
7. **Cierre del Censo**: para las filas de `ext_geo_censo` con `id_geo IS NULL`, copia
   `id_geo`/`departamento_gob` desde `ext_gob_localidades` por `codigo_indec` (`match_tipo =
   "nomenclador_gob"`). Nunca pisa un `id_geo` ya resuelto.
8. Anota en `anomalias`: códigos del Censo que no están en el nomenclador (y al revés), y
   localidades donde `poblacion_2022_gob` difiere del Censo.

Cliente de Sheets: se copia `app/integrations/google_sheets.py` de `svc-gralgob` (ADC, sin
claves), agregando el parámetro de `valueRenderOption`.

`GET /internal/datos-externos/gobierno/localidades/sync-estado` devuelve la última corrida por
fuente.

### 5.1 Precisiones de implementación

- Con `gob_localidades_sync_enabled` apagado, el endpoint de sync responde 503.
- Si el resolver de localidades no responde (devuelve todo sin resolver), el sync **no pisa** los
  `id_geo` resueltos en corridas anteriores.
- Cierre del Censo: si el `id_geo` que aportaría el nomenclador ya lo usa otra fila del Censo, no
  se asigna y queda como anomalía `censo_id_geo_en_conflicto` (señal de un match por nombre
  equivocado en la carga original).
- Un contacto de más de 40 caracteres se guarda como `NULL` y se anota el campo como inválido.
- `svc-vivienda` guarda el departamento de cada zona rural con la grafía del padrón oficial (la
  planilla trae, por ejemplo, `COLON`).
- Una corrida que falla al arrancar deja su fila `fallido` en `ext_gob_sync_log` (commit propio).

Configuración nueva (`config.py`, defaults vacíos/apagados; valores reales en
`services/cloudbuild.yaml`): `google_sheet_gob_nomenclador_id`,
`google_sheet_gob_resumen_general_id`, `gob_localidades_sync_enabled`.

## 6. Rollup y federación

Todo el cambio es aditivo: no se renombra ni se quita nada del contrato actual.

### 6.1 Por localidad

`GET /internal/datos-externos/rollup-territorial` suma, por `id_geo`:

- Públicos: `superficie_km2`, `poblacion_2010`.
- Un objeto `ficha_gobierno` con los campos restringidos de §3.2 (`electores`, `empleados`,
  `masa_salarial`, `intendente`, `reelecto`, `partido`, `continuidad`, `legislador`,
  `partido_legislador`, `contacto_intendente`), o `null` si no hay dato.

La población que se expone sigue siendo `poblacion_2022` del Censo.

### 6.2 Zonas rurales

El rollup es una lista por `id_geo` y las zonas rurales no tienen `id_geo`. Van por un endpoint
aparte, también IAM-only: `GET /internal/datos-externos/zonas-rurales` →
`[{departamento, poblacion_2022, poblacion_2010}]`. `svc-vivienda` lo trae junto con el rollup
(misma tolerancia a fallos: si falla, lista vacía) y lo guarda en el snapshot como
`zonas_rurales`, normalizando el departamento con `normalize_departamento()`.

### 6.3 `svc-vivienda`

`aggregations.py` pasa los campos públicos y `ficha_gobierno` tal cual; `schemas.py` los declara
opcionales. No se calcula nada en backend. No cambia `infra/gateway/openapi.yaml`: no hay paths
nuevos por gateway.

## 7. Visibilidad

### 7.1 Regla

`ficha_gobierno` la ven **únicamente los roles `Admin` y `Autoridad`**. Para cualquier otro rol
—`Supervisor`, `Operador`, `Consulta`, `TecnicoDGV`, `invitado`, de cualquier secretaría,
Vivienda incluida— `filtrar_por_visibilidad` la reemplaza por `null` antes de responder.

Implementación a prueba de fallos: la función parte de "no se muestra" y solo conserva el objeto
si el rol está en una lista propia (`_ROLES_FICHA_GOBIERNO`), separada de la de "ve todas las
áreas", para que sumar un rol a esta última no le abra estos datos por arrastre. No hay lista de
roles excluidos que mantener.

### 7.2 Dónde viaja el dato

Sigue el patrón vigente del módulo: el snapshot (`viv_resumen_territorial_snapshot`) guarda todo
sin filtrar y el filtro se aplica al leer. Consecuencia aceptada por el usuario (2026-10-13): **el
contacto de los intendentes queda almacenado también en `db_vivienda`**, dentro del snapshot, con
la condición de que los usuarios de Vivienda no puedan verlo. El único camino por el que el
snapshot sale a un usuario es `GET/POST /api/v1/resumen-territorial`, que siempre pasa por el
filtro de §7.1; el endpoint interno de recálculo no devuelve el payload.

Alternativa descartada por ahora: servir la ficha restringida desde un endpoint propio de
`svc-datos-externos` por gateway. Evita copiar el dato a otra base, pero requiere montar
autenticación, paths de gateway y una config nueva en ese servicio (el trabajo de ADR-028). Se
reevalúa cuando se construya el primer panel de `svc-datos-externos`.

### 7.3 Convivencia con Privada

La ficha de localidad ya muestra intendente, partido, habitantes y electores tomados de Privada
(ADR-012), visibles según las reglas de Privada. Eso **no cambia en esta fase**. Lo de Gobierno
aparece en un bloque aparte, rotulado con su origen, y solo para `Admin` y `Autoridad`. Unificar ambos es
el trabajo pendiente de ADR-027.

## 8. Frontend (`src/modules/resumen-territorial/`)

Cálculo en cliente, igual que el resto de los indicadores del tablero:

- **Densidad** = población 2022 / superficie. Etiqueta: **"Densidad en el radio municipal
  (hab/km²)"** — no es la densidad del departamento, y la etiqueta lo tiene que decir. Se muestra
  tal cual, también en los radios muy chicos.
- **Crecimiento intercensal** = población 2022 / población 2010 − 1, en porcentaje.
- A nivel departamento y provincia: suma de población sobre suma de superficie, y suma 2022
  sobre suma 2010, tomando solo localidades con ambos datos.
- **Población rural**: a nivel departamento y provincia, una línea aparte "Población en zona
  rural (sin gobierno local) — estimación de Gobierno". No se suma a ninguna localidad ni entra en densidad. El total
  provincial puede mostrarse como "gobiernos locales + zona rural", con las dos partes a la
  vista.
- Si falta alguno de los datos: "N/D".
- **Bloque "Datos institucionales (Gobierno)"** en la ficha de localidad y en su exportación
  PDF/Excel: se dibuja solo si la respuesta trae `ficha_gobierno`. El frontend no decide quién lo
  ve; lo decide el backend.

Lugares a tocar: `VistaProvincia.tsx` (los dos KPIs hoy en "N/D" y su texto explicativo),
`utils/departamentoAgregados.ts`, `types/resumenTerritorial.types.ts`, `FichaLocalidadPage.tsx`
y `fichaMunicipio.ts`.

## 9. Tests

- Clasificación de filas del nomenclador: gobierno local, comunidad regional, zona rural, fila
  con código inválido.
- Normalización de `ID_LOC` (`142007`, `"142.007"`, `"1ZR"`).
- Lista blanca: columnas reordenadas o columna extra no cambian lo que se guarda; columna
  esperada faltante hace fallar la corrida.
- Freno de seguridad (< 400 filas): falla sin tocar datos.
- Una fila mala no frena el lote (SAVEPOINT).
- Baja lógica de una fila que desaparece; reaparición la reactiva.
- Conversión `SI`/`NO` a booleano y números inválidos a `NULL`.
- `anomalias` no contiene valores de campos restringidos.
- Cierre del Censo: completa nulos, no pisa resueltos.
- Rollup: campos públicos, `ficha_gobierno` y endpoint de zonas rurales.
- **Visibilidad** (`svc-vivienda`): `Admin` y `Autoridad` reciben `ficha_gobierno`; `Supervisor`,
  `Operador`, `Consulta`, `TecnicoDGV` e `invitado` reciben `null`, tengan las secretarías que
  tengan. Los campos públicos llegan a todos.

## 10. Criterios de aceptación

1. Migración `0003` aplicada en `db_datos_externos` (commiteada antes de correrla).
2. Corrida real: 427 gobiernos locales y 51 entes cargados, sin errores de fila.
3. Se informa cuántos de los 427 quedaron con `id_geo` y cuáles no; los no resueltos figuran en
   la pantalla de localidades sin resolver.
4. Se informa cuántas filas de `ext_geo_censo` sin `id_geo` se cerraron.
5. Cloud Scheduler diario creado y con una ejecución exitosa.
6. Tras recalcular el Resumen Territorial, el tablero muestra densidad y crecimiento intercensal
   en provincia, departamento y localidad, y la población rural por departamento.
7. Verificado contra producción con dos usuarios: uno `Admin` o `Autoridad` ve el bloque
   institucional en pantalla, PDF y Excel; uno de la secretaría de Vivienda (`Operador` o
   `Consulta`) no lo ve, y la respuesta de la API para ese usuario no contiene los campos.
8. Verificación visual por el usuario en producción.
9. `spec-resumen-territorial-tablero-v2.md §5` actualizado: los dos huecos pasan a resueltos.

## 11. Preguntas abiertas — resueltas (2026-10-13)

1. **Radios muy chicos**: se muestran tal cual, por ser el dato oficial. El área confirmó la
   fuente: polígono del radio según IDECOR.
2. **Diferencias de población con el Censo**: se mantiene el dato del Censo. El área explicó que
   Villa San Isidro (1.527 vs. 607) es un ajuste manual a pedido de la localidad, que se unió con
   La Quintana; Cañada de Machado y Los Pozos probablemente también sean ajustes manuales. El
   sync las sigue informando como anomalía.
3. **Zonas rurales**: se suma su población, siempre rotulada como zona rural y como estimación
   (reparto por departamento hecho por el área).

## 12. Aprobación

Aprobado por el usuario el 2026-10-13, con las correcciones registradas en el changelog.
