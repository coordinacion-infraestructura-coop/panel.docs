# Spec: Resumen Territorial v2 — tablero de tres niveles + servicio de datos externos

**Estado**: approved
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-29
**Servicios**: `svc-datos-externos` (nuevo, nombre confirmado) + `svc-vivienda` (módulo `resumen_territorial` existente, se amplía)

Spec hijo de `spec-resumen-territorial.md` (v0.5.0, approved) — no lo reemplaza. La decisión de
ADR-007 (`resumen_territorial` vive en `svc-vivienda`, sin servicio propio) **sigue vigente y no
se toca**: este spec no crea un servicio para `resumen_territorial`, crea un servicio para datos
que hoy no existen en ningún lado del sistema (censo, INDEC, transferencias fiscales) y que
`resumen_territorial` va a consumir como quinta fuente federada.

> Este documento tiene dos partes independientes en cuanto a implementación (Parte A puede
> aprobarse y construirse sin esperar el diseño final de Parte B), pero comparten un mismo
> spec porque Parte B es la razón de ser de Parte A — no tiene sentido aprobar una sin la otra
> a la vista.

---

## 0. Origen

El usuario pidió rediseñar `/resumen-territorial` (hoy una tabla filtrable por localidad) como
un tablero de navegación en tres niveles (Provincia → Departamento → Localidad) con mapa
coroplético, KPIs demográficos y fiscales, e indicador de focalización de ATP. El diagnóstico
de esta sesión (ver `docs/files/auditoria-codigo.md` § sesión 2026-09-29 una vez registrada, y
el historial de esta conversación) encontró que:

- El stack ya tiene todo lo necesario del lado de visualización: `CoropletiqueDepartamentos.tsx`
  y `MapaDualPuntos.tsx` (Leaflet + Chart.js) ya existen y ya se usan en 3 páginas.
- **Ningún dato demográfico/fiscal real existe hoy en el sistema**: sin código INDEC, sin
  población censal, sin transferencias a municipios/comunas.
- Dos de esos datos ya están, sin cargar, en `docs/data/`: el Censo 2022 a nivel gobierno local
  (`c2022_cordoba_gobierno_local_c1 (5).xlsx`, 427 filas con INDEC + categoría + población) y el
  padrón geográfico (`geo_localidades.json`, usado para sembrar `viv_geo_localidades`).
- Las transferencias automáticas a municipios/comunas (`transparencia.cba.gov.ar`) son la única
  fuente genuinamente externa y recurrente — requieren un scraper mensual de PDFs.

## 1. Decisión de arquitectura: por qué un servicio nuevo y no un módulo de `svc-vivienda`

**Decisión**: se crea `svc-datos-externos` (nombre confirmado 2026-09-29 — no puede ser
`svc-territorial`, reservado en `arquitectura.md` para la futura Secretaría de Planificación y
Articulación Territorial), microservicio nuevo con el mismo molde que `svc-gasifera`/
`svc-gralgob`: FastAPI + Alembic + Cloud Run + una base nueva (`db_datos_externos`) en la
instancia compartida `ministerio-postgres` — **no una instancia de Cloud SQL propia**.

**Alternativas evaluadas y descartadas (sesión 2026-09-29)**:
- **Sumar todo a `svc-vivienda`**: es el módulo más grande del sistema (CC/CH/ML/checklist/
  informes/resumen_territorial/portal/internal/geo); sumarle un scraper de un sitio externo que
  "bloquea clientes automatizados" comparte blast radius de deploy con el panel que usa DGV
  todos los días. Descartado por aislamiento operativo, no por costo.
- **BigQuery como capa analítica transversal**: descartado — el equipo ya se alejó de BigQuery
  dos veces (ADR-008: OLTP de Privada migrado a Postgres; ADR-014: tablero de Privada dejó de
  ser un iframe de Looker sobre BigQuery) y la baja del proyecto BigQuery viejo sigue pendiente,
  gateada por ADR-014. La escala de datos (cientos de localidades, miles de filas/año) no
  justifica reintroducirlo.
- **Costo de GCP como factor de decisión**: descartado explícitamente. Se verificó contra el
  proyecto real (`gcloud sql instances list`, `gcloud run services list`) que los 4
  microservicios actuales comparten una sola instancia `db-f1-micro` (no una por servicio) y
  escalan a cero. El costo marginal de un quinto servicio (una base más + Cloud Run scale-to-zero
  + 1 Cloud Scheduler job) se estima en **~$0.10–2/mes** — irrelevante frente al resto de la
  factura (~$25–45/mes estimados para toda la plataforma).

**Consecuencia**: `svc-datos-externos` es dueño de datos de referencia (censo, INDEC,
transferencias), nunca de lógica de negocio de ninguna secretaría. No reemplaza ni compite con
`resumen_territorial` — lo alimenta, mismo rol que Privada/Gasífera/Gralgob hoy.

## 2. Parte A — `svc-datos-externos`

### 2.1 Alcance (v1)

- Carga única (no recurrente) del Censo 2022 INDEC a nivel gobierno local.
- ETL mensual (+ disparo manual) de transferencias automáticas a municipios y comunas.
- Un endpoint interno de rollup para que `svc-vivienda` federe (§3).
- **Fuera de alcance v1**: panel de negocio propio, polígonos/radios de localidad, Censo 2010,
  índices de coparticipación (Decreto 403/2024 — el Excel ya está identificado en la fuente,
  pero no se carga salvo que se pida explícitamente), indicadores de necesidad para el panel
  secundario (§4, Etapa 6 del plan por etapas — condicionado a conseguir esa fuente).

### 2.2 Modelo de datos (prefijo `ext_`, Alembic `0001` del servicio nuevo)

```
ext_geo_censo
  id_geo              VARCHAR    -- texto, se resuelve contra viv_geo_localidades (§2.4), sin FK real cross-DB
  codigo_indec        VARCHAR    -- "código de gobierno local" del Censo 2022 (ej. "142049")
  categoria           VARCHAR(2) -- "MU" | "CO"
  departamento_censo  VARCHAR
  localidad_censo     VARCHAR
  poblacion_2022      INTEGER
  viviendas_2022      INTEGER
  deleted_at          TIMESTAMP NULL   -- soft delete universal (convención del repo)

ext_transferencias
  id                  UUID PK
  periodo             DATE        -- primer día del mes (ej. 2026-07-01)
  tipo                VARCHAR(10) -- "municipio" | "comuna"
  id_geo              VARCHAR NULL     -- resuelto vía §2.4; NULL si sin_match
  codigo_indec        VARCHAR NULL
  nombre_pdf          VARCHAR     -- texto crudo tal como aparece en el PDF, nunca se pierde
  departamento_pdf    VARCHAR
  concepto            VARCHAR     -- coparticipacion_ley_8663 | fasamu | fofindes | fondo_compensacion | bono_consenso_fiscal | total
  monto               NUMERIC(18,2)
  created_at          TIMESTAMP
  UNIQUE (periodo, tipo, nombre_pdf, concepto)   -- permite re-correr el mismo mes (el sitio publica versiones -v1/-v2)

ext_transferencias_sync_log
  id                  UUID PK
  periodo             DATE
  tipo                VARCHAR(10)
  filas_procesadas    INTEGER
  filas_sin_match     INTEGER
  anomalias           JSON        -- discrepancias de validación de TOTAL fuera de tolerancia, TOTAL duplicados, etc.
  corrida_en          TIMESTAMPTZ
  disparado_por       VARCHAR     -- 'cloud-scheduler' | email del actor (carga manual)
```

`deleted_at` en `ext_geo_censo` por convención universal del repo, aunque en la práctica esta
tabla no tiene alta desde UI (carga única) — no se agrega a `ext_transferencias`/
`ext_transferencias_sync_log`, que son snapshots append-only (mismo criterio que
`viv_cc_sync_log`/`viv_informe_snapshot`: no se borra, se audita agregando filas).

### 2.3 ETL de transferencias — ya prototipado y validado en esta sesión

El algoritmo de extracción del PDF (`pdfplumber`) fue prototipado y validado contra los PDF
reales de julio 2026 (Municipios y Comunas) antes de escribir este spec. Se documentan acá los
hallazgos para que la implementación no los redescubra:

1. **Descubrimiento de links**: el sitio (`transparencia.cba.gov.ar/transferencias-a-municipios-y-comunas/`)
   bloquea clientes automatizados sin user-agent de navegador (`curl` con
   `User-Agent: Mozilla/5.0 ...` funcionó; un fetch sin ese header devuelve 403). Los nombres de
   archivo no son regulares (`-1`, `-v1`, `-v2` de sufijo) — los links se extraen de la página,
   nunca se construyen.
2. **Layout del PDF**: las 6 columnas de monto están alineadas a la derecha; un monto más ancho
   que el típico de su columna (sobre todo las filas `TOTAL {Departamento}`) hace que
   `pdfplumber` lo separe en dos tokens de texto. La reconstrucción robusta es por **posición
   (bins de columna con límite en el punto medio del hueco real entre clusters, no por umbral de
   texto)**, no por heurística de espacios.
3. **Tres anomalías reales ya encontradas y resueltas en el prototipo**, que la implementación
   final debe manejar igual:
   - Una localidad con dígito en el nombre ("Kilómetro 658", comuna de Río Primero) — el split
     nombre/monto no puede ser "primer token numérico", tiene que ser por posición (bin) también
     para el nombre, no solo para los montos.
   - Colisión de prefijo: departamentos reales llamados "General Roca" y "General San Martín"
     hacen que sus filas `TOTAL General Roca`/`TOTAL General San Martín` empiecen con el mismo
     texto que la fila `TOTAL GENERAL` del final del documento — el chequeo debe ser exacto
     (`nombre == "TOTAL GENERAL"`), no un prefijo sobre la línea completa.
   - **Anomalía real de la fuente** (no del parser): en el PDF de Comunas de julio 2026, un
     bloque de 10 comunas encabezado "RÍO SECO" tiene su fila de subtotal mal rotulada
     "TOTAL Río Primero" (error de copiado del propio PDF gubernamental). El ETL debe **detectar
     y loguear** un departamento con más de una fila `TOTAL` (`ext_transferencias_sync_log.anomalias`),
     nunca fallar silenciosamente ni promediar/descartar datos.
4. **Validación**: sumar las filas cargadas por `(departamento, concepto)` y compararlas contra
   la fila `TOTAL {Departamento}` de cada uno y contra `TOTAL GENERAL`, con una tolerancia de
   redondeo (~50 pesos observados — atribuible al redondeo de los coeficientes de coparticipación
   por municipio, no a un error de extracción). Discrepancias fuera de tolerancia → anomalía
   logueada, no bloquea la carga del resto.

**Pasos del job** (mensual, día 5 — da margen a que el PDF del mes anterior ya esté publicado, +
endpoint de disparo manual):
1. Traer la página, extraer los links de `Recaudacion-Municipios-*.pdf` / `Recaudacion-Comunas-*.pdf`
   del período buscado.
2. Descargar y parsear cada PDF con el algoritmo de §2.3.2-3.
3. Validar sumas (§2.3.4).
4. Resolver `(departamento_pdf, nombre_pdf)` → `id_geo` en batch llamando **una vez por corrida**
   a `POST /internal/geo/resolver-localidades` de `svc-vivienda` (§2.4) — no se mantiene tabla de
   alias propia.
5. `UPSERT` en `ext_transferencias` (`ON CONFLICT (periodo, tipo, nombre_pdf, concepto) DO UPDATE`).
6. Escribir `ext_transferencias_sync_log` con el resumen de la corrida.
7. **Modo manual de respaldo**: `POST /internal/datos-externos/transferencias/cargar-manual`
   (IAM-only, `multipart/form-data` con el PDF) para cuando el scraping automático falle —
   reutiliza el mismo parser desde el paso 2. Previsto explícitamente por el pedido original
   ("si falla, dejar un modo de carga manual de PDF").

### 2.4 Resolución territorial — reusa ADR-024, no reinventa

**Decisión**: `svc-datos-externos` **no mantiene su propia tabla de alias/correspondencias**.
Llama a `POST /internal/geo/resolver-localidades` de `svc-vivienda` (`spec-normalizacion-localidades.md §4.3`,
ADR-024) con todas las filas del batch de una corrida, igual que ya hacen `svc-gasifera`/
`svc-gralgob` en su propio sync. Ese endpoint ya centraliza: match exacto → alias manual
(`viv_geo_alias_manual`) → candidatos por paréntesis/guion → `sin_match`.

En el prototipo de esta sesión (matching client-side contra `docs/data/geo_localidades.json`,
sin el endpoint real) se validó que el enfoque funciona: 414 de 428 localidades (97%) matchean
con confianza, 14 quedan para revisión manual — incluyendo los tres casos difíciles del pedido
original ("Villa C.Par. Los Reartes" → Villa Ciudad Parque los Reartes, "Bouchardo", "Jovita" →
Santa Magdalena (Est. Jovita)), que resolvieron bien con matching por prefijo/fuzzy scopeado al
departamento. La implementación real debe usar el endpoint de `svc-vivienda`, no este prototipo
client-side, para no duplicar la lógica de alias que ADR-024 acaba de centralizar.

Las filas `sin_match` quedan con `id_geo = NULL` en `ext_transferencias`, se listan en
`ext_transferencias_sync_log.anomalias`, y no bloquean la carga del resto (mismo criterio
tolerante que CC/CH/ML/Gasífera/Gralgob).

### 2.5 Endpoints

```
GET  /internal/datos-externos/rollup-territorial              (IAM-only, sin JWT)
POST /internal/datos-externos/transferencias/sync              (IAM-only — Cloud Scheduler)
POST /internal/datos-externos/transferencias/cargar-manual      (IAM-only — multipart, respaldo)
```

`rollup-territorial` devuelve, por `id_geo`: `poblacion_2022`, `viviendas_2022`, `categoria`, y
`transferencias` del período pedido (total + desglose por concepto) — mismo espíritu de rollup
acotado que ya usan Privada/Gasífera/Gralgob (no expone todo el modelo interno, solo lo que
`resumen_territorial` necesita). Ningún endpoint de este servicio pasa por el Gateway ni se
declara en `infra/gateway/openapi.yaml` — es exclusivamente IAM-only, mismo patrón que
`app/internal/router.py` del resto de los servicios.

### 2.6 Infraestructura

- Base nueva `db_datos_externos` en `ministerio-postgres` (ya existe la instancia; sin Cloud SQL
  propio).
- Cloud Run: 512Mi/1CPU, `min-instances=0`, `max-instances=10` (mismos valores que los 4
  servicios existentes — confirmado por `gcloud run services describe`).
- Cloud Scheduler: 1 job mensual (`sync-transferencias-municipios-comunas`, día 5 de cada mes).
- IAM: `svc-vivienda@` necesita `roles/run.invoker` sobre `svc-datos-externos` (mismo sentido que
  ADR-015/021/022); `svc-datos-externos@` necesita `roles/run.invoker` sobre `svc-vivienda` para
  llamar a `resolver-localidades` (mismo sentido que ADR-015).
- `services/cloudbuild.yaml`: nuevo trigger/substitución para el servicio, más
  `_DATOS_EXTERNOS_FETCH_ENABLED`/`_SVC_DATOS_EXTERNOS_INTERNAL_URL` para el lado de
  `svc-vivienda` (mismo criterio que las substituciones de Privada/Gasífera/ATP).

## 3. Parte B — Federación en `resumen_territorial` (backend)

- `svc-vivienda/app/resumen_territorial/service.py` gana `fetch_datos_externos_lineas()`, calcada
  de `fetch_gasifera_lineas` (mismo try/except fail-open a `[]`). Settings nuevas en
  `config.py`: `datos_externos_fetch_enabled` (default `False`) / `svc_datos_externos_internal_url`;
  `cloudbuild.yaml` las enciende por default en cada deploy de CI (mismo criterio que las demás).
- `aggregations.py` gana los cálculos derivados que hoy no existen en ningún lado: densidad
  (requiere superficie — **no disponible en v1**, densidad queda pendiente hasta conseguir esa
  fuente), transferencias per cápita (mediana para comunas chicas, por el componente fijo de la
  fórmula que dispara valores per cápita altos en comunas — pedido explícito del usuario),
  inversión ATP per cápita, índice de focalización ATP (`% inversión ATP del depto / % población
  del depto`).
- `agrupar_por_localidad` ya agrupa primero por `id_geo` (ADR-024) — las líneas de
  `datos_externos` entran por esa misma llave, sin fallback a texto (a diferencia de Privada, que
  sigue en el fallback de ADR-024 hasta que se decida unificar su padrón).
- Tests: `test_resumen_territorial.py` gana un caso con la fuente nueva + su fail-open, mismo
  patrón que los tests existentes de Gasífera/ATP.

## 4. Parte C — Rediseño del frontend (resumen; el detalle está en el plan por etapas de esta sesión)

El diseño completo de la navegación de 3 niveles, el mapa coroplético, los KPIs, el gráfico de
focalización ATP y la ficha de localidad está desarrollado etapa por etapa en el plan de
implementación de esta sesión (`Etapas 3-5`). Puntos que este spec fija como decisión, para no
dejarlos abiertos en la implementación:

- **Reutilización obligatoria**: `CoropletiqueDepartamentos.tsx` y `MapaDualPuntos.tsx` ya
  existen y ya se usan en 3 páginas — se extienden, no se reescriben. La escala divergente
  centrada en 1 (para el filtro ATP) es una extensión nueva de `CoropletiqueDepartamentos.tsx`
  (hoy solo soporta secuencial).
- **Sin KPIs de estado de gestión** (24 valores heterogéneos entre áreas, pendientes de
  normalizar aparte — fuera de alcance, igual que ya fijaba `spec-resumen-territorial.md`).
- **Ficha de localidad**: se amplía `fichaMunicipio.ts`/`exportResumen.ts` con los campos nuevos
  (población, transferencias) sin reescribir el mecanismo client-side actual.
- **Panel secundario (Gini + necesidad-vs-programa)**: condicionado a conseguir, en la revisión
  de este spec, los indicadores de necesidad a nivel gobierno local (gas de red, déficit
  habitacional, entorno urbano) del Censo 2022 — si el Censo público no los trae a ese nivel de
  desagregación, esa parte del panel secundario queda fuera de alcance hasta conseguir otra
  fuente, y se documenta como tal en vez de implementarse con datos aproximados.

## 5. Huecos de datos — estado a la fecha de este spec

| Dato | Estado |
|---|---|
| Código INDEC + categoría MU/CO + población 2022 | **Resuelto** — `docs/data/c2022_cordoba_gobierno_local_c1 (5).xlsx`, 427 filas |
| Transferencias automáticas | **Resuelto** — ETL prototipado y validado (§2.3) |
| Población 2010 (crecimiento intercensal) | **Bloqueado** — sin fuente identificada. No gatea Etapas 1-5 (el KPI de crecimiento intercensal queda oculto/vacío hasta conseguir la fuente); tampoco gatea Etapa 6. |
| Superficie / densidad | **Bloqueado** — sin fuente identificada. No gatea Etapas 1-5 (el KPI de densidad queda oculto hasta conseguir la fuente); tampoco gatea Etapa 6. |
| Indicadores de necesidad (gas, déficit habitacional, entorno urbano) | **Bloqueado** — no confirmado si el Censo 2022 público los trae a nivel gobierno local. **Gatea exclusivamente la Etapa 6** (panel secundario Gini + necesidad-vs-programa); no afecta Etapas 1-5. Se investiga en paralelo al arranque de Etapa 1, sin bloquearla. |
| Polígonos/radios a nivel localidad | **Bloqueado** — solo hay centroides puntuales en `viv_geo_localidades`. No gatea ninguna etapa del plan actual (Etapa 5 usa centroide, no polígono, por diseño); relevante solo si en el futuro se pide un zoom con polígono real a nivel localidad. |

### 5.1 Qué buscar para destrabar densidad y crecimiento intercensal

Pendiente de una sesión de web search dedicada (no hecha todavía). Apuntes para esa búsqueda,
para no arrancar de cero:

- **Población 2010 (crecimiento intercensal)**: se necesita el equivalente del Censo 2010 a lo
  que ya tenemos del Censo 2022 (`docs/data/c2022_cordoba_gobierno_local_c1 (5).xlsx`, "Cuadro
  1.6" — población por gobierno local con código INDEC). El Censo 2010 se publicó con una
  estructura de cuadros distinta (por "localidad" censal, no necesariamente por "gobierno local"
  1:1) — el cuadro equivalente históricamente es el de "Población total por localidad censal" del
  Censo Nacional 2010, provincia de Córdoba, en la serie de cuadros del INDEC (buscar en
  redatam.indec.gob.ar o en las publicaciones de resultados definitivos por provincia del Censo
  2010). Riesgo a verificar: el código INDEC de gobierno local puede no ser estable entre censos
  (creación/fusión de comunas entre 2010 y 2022) — el join contra `ext_geo_censo` puede no ser
  1:1 limpio, revisar caso por caso los `sin_match`.
- **Superficie / densidad por departamento**: se necesita superficie en km² por departamento (26
  departamentos de Córdoba). Fuentes candidatas a chequear: (a) el IGN (Instituto Geográfico
  Nacional) publica superficies oficiales por departamento; (b) INDEC también publica superficie
  por departamento en sus cuadros de "División político-territorial"; (c) si ninguna trae un
  dataset descargable directo, se puede calcular a partir del mismo polígono ya usado para el
  mapa (`frontend/public/geo/departamentos_cba.json`) con `shapely`/`pyproj` (el script
  `frontend/scripts/prepare_departamentos_geojson.py` ya usa esas libs) — probablemente el camino
  más rápido si las fuentes oficiales no dan un CSV limpio.
- Ninguno de los dos bloquea nada del tablero ya construido (Etapas 1-3) — son mejoras aditivas a
  `svc-datos-externos` (una tabla más, ej. `ext_censo_2010` y/o una columna `superficie_km2` en
  algún catálogo de departamentos) el día que se consiga la fuente.

## 6. Criterios de aceptación

- [ ] `svc-datos-externos` desplegado (Cloud Run + `db_datos_externos` en la instancia
      compartida), `pytest` en verde.
- [ ] Censo 2022 cargado: 427 filas en `ext_geo_censo` (260 `MU` + 167 `CO`).
- [ ] ETL de transferencias corrido contra un mes real: filas en `ext_transferencias`, log de
      corrida con `filas_sin_match` y `anomalias` (si las hay) poblados correctamente.
- [ ] Re-correr el mismo período no duplica filas (`ON CONFLICT ... DO UPDATE` funcionando).
- [ ] `GET /internal/datos-externos/rollup-territorial` responde con población + transferencias
      por `id_geo`.
- [ ] `svc-vivienda` federa la fuente nueva: snapshot de `resumen_territorial` incluye población,
      transferencias per cápita e índice de focalización ATP; una caída de `svc-datos-externos`
      no rompe el snapshot (fail-open, igual que las demás fuentes).
- [ ] Frontend: navegación de 3 niveles funcional con breadcrumb; mapa coroplético con las 4
      métricas del pedido original (programas distintos, gestiones/10.000 hab, cobertura,
      focalización ATP); ficha de localidad exporta con los campos nuevos.
- [ ] Regresión: `app/resumen_territorial`, `app/geo`, y los tests de Gasífera/Gralgob/Privada
      siguen en verde.

---

## 7. Aprobación

Aprobado 2026-09-29 (mismo día de redacción) por Pedro Bonafe. Decisiones que se confirman al
aprobar, cerrando los dos puntos que este spec dejaba abiertos en su borrador:

- **Nombre del servicio confirmado**: `svc-datos-externos`.
- **Los 3 huecos de datos pendientes (población 2010, superficie/densidad, indicadores de
  necesidad) no gatean el arranque de Etapa 1** — quedan explícitamente marcados como
  bloqueados en §5, con el alcance exacto de lo que sí bloquean (ver tabla). Se investigan en
  paralelo, no antes.

Corresponde registrar el ADR de esta decisión en `arquitectura.md` (mirror de ADR-016/021/022,
en sentido "provee" en vez de "consume") como parte del cierre de Etapa 0.
