# Spec: Datos externos — fuentes oficiales públicas por gobierno local (Estadística Córdoba, IDECOR, RENABAP)

**Estado**: approved
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-19
**Servicios**: `svc-datos-externos` (sync + tablas + rollup) · `svc-vivienda` (`resumen_territorial`, consumidor) · `frontend` (`resumen-territorial`, solo textos) · `infra` (Cloud Scheduler)

Spec hijo de `spec-datos-externos-tablero-gobierno.md` (paraguas), §3.10: superficie y población
2010 se toman de fuentes oficiales públicas, con spec propio. Cierra los huecos de
`spec-datos-externos-fase1-nomenclador-demografia.md` §11.1 y de
`spec-resumen-territorial-tablero-v2.md` §5 (población 2010, superficie/densidad e indicadores de
necesidad). Base: relevamiento de fuentes del 2026-10-19 (conversación con el usuario; resumen en §2).

## Changelog

- **1.0.0** (2026-10-19): aprobado por el usuario. Decisiones: (1) la superficie que se usa para
  la densidad es la de **Estadística** (la del Censo), no la del radio legal de IDECOR; (2) la
  población rural por departamento pasa a ser el **dato oficial** del Censo (departamento menos sus
  gobiernos locales), en lugar de la estimación de Gobierno; (3) "sumar en el mismo sync toda la
  información relevante de fuentes externas" relevada, en tablas de `db_datos_externos`, federando
  a `resumen_territorial` el subconjunto definido en §6.

---

## 1. Propósito

1. Llenar `superficie_km2` y `poblacion_2010` de `ext_gob_localidades` desde el Censo (Estadística
   Córdoba). Con eso el Resumen Territorial muestra densidad y crecimiento intercensal sin cambios
   de lógica (ya se calculan en el cliente) y el Tablero de Gobierno los lee de la misma columna.
2. Reemplazar la población rural estimada por Gobierno por el dato oficial del Censo.
3. Incorporar a `db_datos_externos` los indicadores municipales relevados como útiles:
   condiciones de vida (NBI, servicios, tenencia, calidad de la vivienda), educación, salud,
   empleo, Índice de Progreso Social, barrios populares (RENABAP), establecimientos de salud y
   educación, y nacimientos/defunciones por año.
4. Federar a `resumen_territorial` un subconjunto acotado (§6) para el panel de necesidad
   (tablero v2, Etapa 6) y la ficha de localidad.

## 2. Fuentes

Todas públicas, sin autenticación, consultadas y validadas el 2026-10-19.

### 2.1 Estadística Córdoba — ArcGIS REST (fuente principal)

El portal CKAN `datosestadistica.cba.gov.ar` devuelve 404 en todo (incluida la home) desde
octubre de 2026; la Dirección General de Estadística y Censos migró a un ArcGIS Enterprise. Base:
`https://smart-geo-data.cba.gov.ar/server/rest/services/`. Consulta:
`<servicio>/FeatureServer/<capa>/query?where=1=1&outFields=*&returnGeometry=false&f=json`
(hasta 2.000 filas por página: todo entra en una llamada).

Clave: **código INDEC de gobierno local de 6 dígitos** (`140077`), el mismo de
`ext_geo_censo.codigo_indec` y `ext_gob_localidades.codigo_indec`. Validado: 427/427 coinciden.

| Capa | Filas | Uso |
|---|---|---|
| `Estadistica_Censos/Conoce_cordoba_Municipio_FichaGral_VP` | 427 + fila Provincia (código 14) | Población 2010 y 2022, variación intercensal, superficie, densidad, hogares, estructura etaria |
| `Estadistica_Censos/MUN_CondVida_VP` | 427 + Provincia (14000) | NBI total y por tipo, agua de red, cloacas, gas de red, internet, tenencia, calidad de materiales (Censo 2022) |
| `Sitio_Web_Estadistica/PobTotalViv_VP` | 427 + Provincia | Las mismas condiciones en **cantidades** de hogares |
| `Estadistica_Censos/MUN_Educacion_VP` | 427 | Asistencia 10–18, secundario y universitario completo |
| `Estadistica_Censos/MUN_Salud_VP` | 427 | Cobertura de salud |
| `Estadistica_Censos/MUN_MercadoLaboral_VP` | 427 | Tasa de desocupación, actividad principal |
| `IPS/IPS_23102025` | 427 | Índice de Progreso Social (valor, nivel, 3 dimensiones, variación interanual); clave `id_2` |
| `Estadistica_Censos/Dpto_2022_VP` | 26 | Población 2010/2022 y superficie por departamento; clave `link` = código INDEC de departamento (`14140`) |
| `Estadistica_Censos/MyC_2022_VP` | 427 | Solo geometría de los gobiernos locales (`outSR=4326`, simplificada), para ubicar los puntos de §2.2 y §2.3 |

Hallazgos que el sync debe respetar:

- La población 2010 de FichaGral está **recalculada sobre los límites de 2022** (Córdoba Capital:
  1.327.716 contra 1.329.604 publicados en 2010). Es la correcta para el crecimiento intercensal
  (misma geografía). No se usa la capa original de 2010 (`Censo_2010_Visualizador_VP`): tiene
  códigos de 4 dígitos de 2010 y 20 nombres que no matchean.
- `variación_intercensal` viene como **fracción** (0,61 = 61%).
- Hay filas de Provincia (código `14` / `14000`) y una fila repetida (Paso del Durazno): se
  descartan y se registran como anomalía.
- La superficie es la del polígono del gobierno local según el Censo. Difiere en más de 10% del
  radio legal de IDECOR en 116 de 416 localidades (ej. Tanti 47 km² vs 95 km² por la Ley 10870/2023;
  comunas "SIN PLANO" con áreas mínimas). Decisión del usuario: densidad con la superficie del Censo.
- Los nombres de campo tienen tildes (`código_gobierno_local`): se leen por nombre exacto; si falta
  uno esperado, la fuente falla entera.
- Los nombres de servicio pueden traer fecha (`IPS_23102025`): el nombre va en configuración, no en
  código, para cambiarlo sin redeploy de lógica.

### 2.2 IDECOR — WFS (`idecor-ws.mapascordoba.gob.ar/geoserver/idecor/wfs`)

| Capa | Filas | Uso |
|---|---|---|
| `Centros_Salud` | 889 puntos | Establecimientos de salud (CUIE, tipología, dependencia) |
| `establecimientos_educativos` | 5.489 puntos | Establecimientos educativos (CUE anexo, sector, ámbito, nivel) |
| `radios_urbanos` | 427 polígonos | Solo para vincular las capas de registro civil a un gobierno local |
| `registro_c_nacimientos` / `registro_c_defunciones` | 2.562 c/u (427 × 2020–2025) | Nacimientos y defunciones por localidad y año |

Se piden con `srsName=EPSG:4326` y se leen como UTF-8. Ninguna trae código INDEC: los puntos se
ubican en el polígono de gobierno local de §2.1 (punto en polígono, local); el registro civil se
vincula por el centroide del radio de IDECOR del mismo nombre. Validado: salud 874/889, educación
4.965/5.489 (el resto son escuelas rurales fuera de un gobierno local), registro civil 2.472/2.562
filas.

### 2.3 RENABAP — vía Poblaciones (CONICET/UCA)

Registro Nacional de Barrios Populares, versión 2023. El organismo no publica API; se descarga el
CSV desde el servidor de datos de Poblaciones (endpoint no documentado, sin autenticación):
`https://datos-poblaciones.conicet.gov.ar/services/download/GetDatasetFile?t=c&d=383501&w=191801`.
Córdoba: 318 barrios, 37.039 familias; trae id, nombre, familias, año y década de creación,
tipología, situación dominial, acceso a servicios, superficie y centroide. Se ubican en el
polígono de gobierno local (316/318). El registro está cerrado a barrios creados hasta el
31/12/2021 (Decreto 769/2024): alcanza con refrescarlo de vez en cuando; si la descarga falla, se
conservan los datos anteriores.

### 2.4 Fuera de alcance (relevado y descartado)

- Permisos de edificación de Estadística: solo 18 ciudades.
- Nacimientos/defunciones de Estadística: solo por departamento y solo 2022 (IDECOR es mejor).
- Indicadores censales de Poblaciones: vienen por radio o departamento y repiten lo de §2.1.
- Mapa de valores de la tierra: ya se usa en Mi Lugar (`spec-mi-lugar-valor-fiscal-idecor.md`).
- Electores, empleados, masa salarial, intendente, legislador, contacto: no están en ninguna
  fuente pública; siguen esperando al área (paraguas §3.10).

## 3. Modelo (`db_datos_externos`, migración `0005`)

Las tablas son un espejo de la fuente: cada corrida buena reemplaza el contenido de su fuente
(dentro de un SAVEPOINT); una corrida que falla no toca lo que había.

### 3.1 `ext_est_gobierno_local` — una fila por gobierno local

PK `codigo_indec` (`VARCHAR(10)`). Columnas tipadas (las que se federan o tienen uso previsto),
todas NULL-ables salvo la clave:

| Grupo | Columnas |
|---|---|
| Identificación | `nombre`, `codigo_dpto_indec`, `tipo_2022` (`MUN`/`COM`) |
| Demografía (FichaGral) | `poblacion_2010`, `poblacion_2022`, `variacion_intercensal` (fracción), `superficie_km2`, `densidad`, `hogares`, `personas_por_hogar`, `pct_0_17`, `pct_65_mas`, `coef_vejez` |
| Condiciones de vida, % (CondVida) | `nbi_pct`, `nbi_vivienda_pct`, `nbi_sanitaria_pct`, `nbi_hacinamiento_pct`, `nbi_escolar_pct`, `nbi_subsistencia_pct`, `agua_red_pct`, `cloacas_pct`, `gas_red_pct`, `internet_pct`, `propietarios_pct`, `inquilinos_pct`, `otra_tenencia_pct`, `calidad_i_pct`, `calidad_ii_pct`, `calidad_iii_pct`, `calidad_iv_pct` |
| Condiciones de vida, hogares (PobTotalViv) | `hogares_total`, `hogares_hacinamiento`, `hogares_sanitaria`, `hogares_vivienda_inconveniente`, `hogares_calidad_iii`, `hogares_calidad_iv`, `hogares_lena_carbon`, `hogares_alquilan` |
| Educación | `asistencia_10_18_pct`, `secundario_completo_pct`, `universitario_completo_pct` |
| Salud | `cobertura_salud_pct` |
| Empleo | `desocupacion_pct`, `actividad_principal` |
| IPS | `ips_valor`, `ips_nivel`, `ips_necesidades_basicas`, `ips_fundamentos_bienestar`, `ips_oportunidades`, `ips_variacion_interanual` |
| Trazabilidad | `crudo` (`TEXT`, JSON: todos los atributos originales por capa, para usos futuros sin migración), `synced_at` |

### 3.2 `ext_est_departamento` — una fila por departamento

PK `codigo_dpto_indec` (`14140`). `nombre`, `poblacion_2010`, `poblacion_2022`, `superficie_km2`,
`poblacion_fuera_gl_2010`, `poblacion_fuera_gl_2022` (departamento menos la suma de sus gobiernos
locales de §3.1; ≥ 0), `synced_at`.

### 3.3 `ext_renabap_barrios`

PK `id_renabap`. `nombre`, `familias`, `anio_creacion`, `decada_creacion`, `tipologia`,
`situacion_dominial`, `energia`, `cloacas`, `agua`, `gas`, `superficie_km2`, `lat`, `lng`,
`codigo_radio`, `codigo_indec` (NULL si cae fuera de todo gobierno local), `version`, `synced_at`.

### 3.4 `ext_equipamientos`

PK `id` (uuid). `tipo` (`salud` | `educacion`), `codigo_fuente` (CUIE o CUE anexo), `nombre`,
`categoria` (tipología de salud / nivel educativo), `sector` (dependencia / sector), `ambito`,
`localidad_fuente`, `departamento_fuente`, `lat`, `lng`, `codigo_indec` (NULL si fuera de gobierno
local), `synced_at`. Índice por `(tipo, codigo_indec)`.

### 3.5 `ext_estadisticas_vitales`

PK (`radio_idecor`, `anio`, `tipo`). `tipo` (`nacimientos` | `defunciones`), `total`,
`codigo_indec` (NULL si el radio no se pudo ubicar), `synced_at`.

### 3.6 `ext_fuentes_oficiales_sync_log` — append-only

Una fila por fuente y corrida: `fuente` (`estadistica` | `renabap` | `idecor_equipamientos` |
`idecor_vitales`), `iniciado_at`, `finalizado_at`, `estado` (`ok` | `parcial` | `error`),
`filas_leidas`, `filas_guardadas`, `anomalias` (JSON), `mensaje`, `disparado_por`. Lo puede leer el
Tablero de Gobierno para la "frescura por fuente".

### 3.7 `ext_est_poligonos`

PK `codigo_indec`. `geojson` (`TEXT`: anillos en WGS84, simplificados a ~20 m), `synced_at`. Solo
para ubicar puntos; no se federa.

### 3.8 Escritura en `ext_gob_localidades`

Al terminar bien la fuente `estadistica`, se copian `superficie_km2` y `poblacion_2010` a
`ext_gob_localidades` por `codigo_indec` (paraguas §3.10). El sync de Gobierno no toca esas
columnas (verificado). Es la columna que leen el rollup actual y el Tablero de Gobierno.

## 4. Sync

- `POST /internal/datos-externos/fuentes-oficiales/sync?fuentes=...&triggered_by=...` (IAM-only,
  mismo patrón que los demás syncs). Sin `fuentes`, corre las cuatro.
- Cada fuente es independiente: corre en su SAVEPOINT, escribe su fila de log y una falla no frena
  a las otras. La respuesta lista el resultado de cada una; 502 solo si fallaron todas.
- Orden: `estadistica` primero (trae los polígonos que usan las otras). Si `estadistica` falla,
  las otras ubican con los polígonos guardados en la corrida anterior; si no hay, guardan
  `codigo_indec` NULL.
- Flag `fuentes_oficiales_sync_enabled` (default apagado; `cloudbuild.yaml` lo prende).
- Cloud Scheduler `sync-fuentes-oficiales`: mensual (día 2, 05:00). Las fuentes cambian a lo sumo
  una vez por año; mensual alcanza.
- Los polígonos de gobierno local (§2.1) se guardan en `ext_est_poligonos` (§3.7), para que las
  otras fuentes no los descarguen de nuevo y puedan ubicar puntos aunque `estadistica` falle.

## 5. Calidad y tolerancia

- Filas sin código, con código fuera del nomenclador o repetidas: se descartan y van a `anomalias`.
- Valores no numéricos: NULL + anomalía.
- Faltante de un campo esperado o respuesta ilegible: falla la fuente entera (`estado = error`).

## 6. Federación a `resumen_territorial`

`GET /internal/datos-externos/rollup-territorial` agrega, de forma **aditiva**, un objeto
`indicadores` por `id_geo` (vínculo `codigo_indec` → `id_geo` por `ext_gob_localidades`):

| Campo | Fuente |
|---|---|
| `anio_censo` (2022) | — |
| `nbi_pct`, `gas_red_pct`, `agua_red_pct`, `cloacas_pct`, `internet_pct`, `inquilinos_pct`, `nbi_hacinamiento_pct`, `calidad_iii_iv_pct` (suma de III y IV) | §3.1 |
| `hogares_hacinamiento`, `hogares_calidad_iv` | §3.1 |
| `desocupacion_pct`, `cobertura_salud_pct`, `secundario_completo_pct` | §3.1 |
| `ips_valor`, `ips_nivel` | §3.1 |
| `renabap_barrios`, `renabap_familias` | §3.3 agregado por `codigo_indec` |
| `establecimientos_salud`, `establecimientos_educativos` | §3.4 contados por `codigo_indec` |
| `vitales_anio`, `nacimientos`, `defunciones` | §3.5, último año con datos |

`superficie_km2` y `poblacion_2010` siguen saliendo de `ext_gob_localidades` (ahora con dato).

`GET /internal/datos-externos/zonas-rurales` devuelve, si hay datos de §3.2, la población fuera de
gobierno local de cada departamento con `fuente = "censo"`; si no, cae a la estimación de Gobierno
con `fuente = "gobierno_estimacion"`. El nombre de departamento sale del nomenclador de Gobierno
(misma grafía de hoy), vinculado por `codigo_dpto_indec`.

`svc-vivienda`: `ResumenLocalidad` suma `indicadores: IndicadoresSociales | None` (todos los
campos opcionales) y `ZonaRural` suma `fuente: str | None`. Se pasan tal cual, sin cálculo. Son
datos públicos: los ve cualquier rol que vea la localidad.

## 7. Frontend

Solo textos, sin pantallas nuevas:

- Ayuda del KPI de densidad: "Población (Censo 2022) ÷ superficie del gobierno local según el
  Censo (Dirección General de Estadística y Censos de Córdoba)".
- Ayuda de la población rural: según `fuente` — "Censo 2022: población del departamento fuera de
  los gobiernos locales" o el texto actual de estimación de Gobierno.

Mostrar los `indicadores` (panel de necesidad de la Etapa 6, bloque en la ficha de localidad) es
**fuera de alcance**: requiere su sección aprobada en `spec-resumen-territorial-tablero-v2.md` o en
la spec de la ficha. Los datos quedan en el snapshot desde esta versión.

## 8. Pruebas

- Parsers con respuestas simuladas por capa (incluye fila Provincia, fila repetida, campo
  faltante → error, variación como fracción).
- Punto en polígono (anillos con agujero; punto fuera → NULL).
- Población fuera de gobierno local (nunca negativa).
- Independencia de fuentes: una falla y las demás guardan.
- Rollup: `indicadores` presentes y agregados de RENABAP/equipamientos/vitales; zonas rurales con
  `fuente`.
- `svc-vivienda`: `indicadores` y `fuente` pasan al snapshot.

## 9. Criterios de aceptación

1. Migración `0005` aplicada en producción.
2. Primera corrida real: las cuatro fuentes en `ok` o `parcial`, con las cifras de §2 (±).
3. `ext_gob_localidades`: 427 filas con `superficie_km2` y `poblacion_2010`.
4. Resumen Territorial recalculado: densidad y crecimiento intercensal dejan de mostrar "N/D";
   zonas rurales con `fuente = "censo"`.
5. Cloud Scheduler mensual creado.
6. Verificación visual del usuario.
