# Spec: Vivienda — Tablero analítico + correcciones de listados

**Estado**: approved
**Versión**: 1.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-16
**Servicios**: `svc-vivienda` (módulo nuevo `tablero_analitico`, `resumen_territorial`, `checklist_tecnico`, paneles CC/CH/ML) · `frontend` (`vivienda`) · `infra` (gateway)

Origen: pedido del área de Vivienda (PDF `docs/context/areas/secretaria_Vivienda/Observaciones y
pedidos para la App GestorCoop 2026 10 02.pdf`) y el tablero que el área armó en base44
(`analisisterritorialhoracio.base44.app`), relevado en vivo el 2026-10-15. Spec hermano de
`spec-checklist-tecnico-dgv.md` v1.6.0, que cubrió avance de obra, hitos y "No Aplica" del mismo PDF.

## Changelog

- **1.1.0** (2026-10-16, aprobado por el usuario): pedidos del área técnica después de la primera revisión.
  - **KPI "Localidades con ítems faltantes"** (§4.3, §5.2): reemplaza a "Con ítems faltantes", que
    contaba **expedientes** con el rótulo de localidades. Ejemplo que lo destapó: Mi Lugar mostraba
    49 (sus 49 proyectos) frente a 33 localidades del programa. Ahora cuenta localidades y, por
    programa, dibuja una barra al 100 % con el total de localidades del programa en gris y el dato
    nominal "N de M".
  - **Filtro "Est. técnico"** (§5.1): las opciones salen del catálogo de estados del expediente del
    Checklist Técnico, no de los valores presentes en los datos. Así aparece "RECHAZADO por M/C"
    aunque hoy ningún expediente activo lo tenga.
  - **Checklist de la tabla** (§5.4): el texto "X de Y faltan" / "Completo" / "No iniciado" es un
    enlace al Checklist Técnico de ese expediente.
  - Backfill: modo `--completar-claves` para sumar la clave nueva a la historia (§4.3).
  - Al probar el enlace apareció un bug del Checklist Técnico: solo mostraba un expediente por
    programa y localidad. Se corrigió en `spec-checklist-tecnico-dgv.md` v1.6.1.
- **1.0.2** (2026-10-15): desplegado en producción. Migración `0040` + backfill de `kpis` corridos en
  Cloud Shell; backend `a4c415d` (revisión `svc-vivienda-00201-d69`), frontend `140ea78`, gateway
  `ministerio-config-v20261015b`. Falta la verificación visual del área contra base44 (§10).
- **1.0.1** (2026-10-15): precisiones de la implementación.
  - **Denominador de localidades:** el "de N" del tablero son los gobiernos locales (427), igual que en
    el Resumen Territorial. Ese recorte se calcula con datos de `svc-datos-externos`; para no volver a
    federar en cada pedido, se toma `total_localidades_por_departamento` del último snapshot. Si no hay
    snapshot, se usa el padrón activo (§4.2).
  - **Documentación en los paneles:** viaja en el mismo resumen del checklist que el avance de obra
    (`estado_tecnico.avance_por_entidad`). Así también la traen las respuestas de alta y edición, sin
    tocar cada llamada.
  - **Hora de los KPIs:** los KPIs por snapshot y el "día" de Tendencias usan la hora de Argentina (UTC-3).
- **1.0.0** (2026-10-15): aprobado por el usuario, con las decisiones de §9 y la regla de secretaría de §4.2.
- **0.1.0** (2026-10-15): primera versión, en review. Plan aprobado por el usuario el mismo día.

---

## 1. Propósito

1. Tener dentro del sistema el **tablero analítico** de Vivienda que hoy el área mantiene en base44.
   base44 se alimenta del Excel que exporta el Resumen Territorial (`resumen_territorial_AAAA-MM-DD.xlsx`).
   El área lo descarga y lo vuelve a subir a mano cada vez, por lo que el tablero siempre está
   desactualizado.
2. Resolver los pedidos del PDF que el usuario eligió sumar:
   - **Listados:** orden alfabético, filtro por estado de la documentación y área responsable de cada estado.
   - **Distancias:** listar por distancia a una localidad, desde una localidad o desde la ubicación del usuario.
   - **Mapa:** mapa para armar el recorrido de un viaje de inspección por obras de distintos programas.

## 2. Alcance

### Incluido
- Página nueva **Tablero analítico** en Vivienda, con pestañas Tablero, Mapa y Tendencias (§5–§7).
- Endpoint en vivo `GET /api/v1/vivienda/tablero-analitico` y su serie histórica `.../tendencias` (§4).
- KPIs precalculados por snapshot del Resumen Territorial, que son la base de Tendencias (§4.3).
- En los paneles CC/CH/ML (§8):
  - orden alfabético por defecto;
  - filtro "Documentación";
  - área responsable de cada estado.

### Fuera de alcance (pendientes del PDF y de base44, quedan para evaluar)
- **Chat con IA sobre los datos**, la pestaña "Chat" de base44. Necesita spec propio que defina:
  - el proveedor de LLM;
  - el costo;
  - qué datos salen del sistema.
- **Cantidades por programa** (ml de cordón, m² de adoquín, m² de badenes, cantidad de viviendas)
  editables desde el checklist, y **última observación** del expediente y de la obra en los listados.
- **Fotos por hito** de avance. Requiere un bucket de GCS y endpoints de subida.
- **Nombres oficiales de localidades.** Hallazgo: el padrón tiene `id_geo 105` = "CASTRO URDIALES -
  Colonia 25 de Mayo", y Vivienda dice que el nombre oficial es "Colonia Barge".
  - Propuesta: cruzar el padrón contra el nomenclador de Gobierno
    (`spec-datos-externos-fase1-nomenclador-demografia.md`) y que el área confirme los renombres.
  - Por ADR-026, renombrar en el padrón alcanza para toda la plataforma.
- **Ruteo real por calles.** El recorrido se arma con distancia en línea recta y se delega el ruteo
  a Google Maps mediante un enlace (§6.3).
- **Geocodificar lo que no está vinculado al padrón.** Esas filas se cuentan como "sin coordenadas" (§6.2).

## 3. Decisión de fuente de datos (ADR-030)

El tablero **no lee el snapshot del Resumen Territorial**: lee en vivo de `db_vivienda` por un
endpoint propio.

- **El Resumen Territorial no es otro servicio.** `resumen_territorial/service.compute_resumen_territorial`
  ya arma las líneas de Vivienda leyendo la misma `db_vivienda`:
  - CC/CH/ML y sus catálogos de estado;
  - checklist e ítems;
  - pedidos y padrón geo.

  Solo Privada, Gasífera, ATP y Datos Externos se federan por red.
- **El snapshot no le suma nada al tablero y le resta:**
  - llega con hasta 30 minutos de atraso;
  - trae áreas que el tablero descarta;
  - cada campo que el tablero necesite lo engordaría para todo el Resumen Territorial;
  - acopla la forma de dos pantallas.
- **Una sola lógica:** el armado de las líneas de Vivienda se factoriza en una función compartida,
  `resumen_territorial.lineas.lineas_vivienda(db)`. La usan el cómputo del snapshot, cuya salida no
  cambia, y el endpoint del tablero.
- **Tendencias** es lo único que necesita historia. Usa los snapshots que ya se guardan cada
  30 minutos (append-only, nunca purgados, con historia desde 2026-08-27), con KPIs precalculados
  en una columna nueva.

## 4. Backend (`svc-vivienda`)

### 4.1 Factorización
`app/resumen_territorial/lineas.py` (nuevo) contiene la función `async lineas_vivienda(db) -> list[dict]`,
movida desde `service.py`:
- el cuerpo de `compute_resumen_territorial` desde la lectura de CC/CH/ML hasta el armado de las
  líneas de Vivienda;
- los helpers `_catalogo`, `_estado_fields`, `_pedidos_por_entidad` y `_linea_vivienda`.

Cada línea conserva la forma actual (`departamento`, `nombre_localidad`, `id_geo`, `programa`) y suma
dentro de `programa` campos **solo de uso interno**, que el snapshot descarta: `programa_cod` y `localidad_id`.
`compute_resumen_territorial` la llama, y un test de regresión verifica que el payload no cambia.

### 4.2 `GET /api/v1/vivienda/tablero-analitico`
Módulo nuevo `app/tablero_analitico/` (`router.py`, `schemas.py`, `service.py`), registrado en `main.py`.

- **Roles:** tupla local `ROLES_LECTURA_TABLERO_ANALITICO = ROLES_LECTURA + ("Autoridad", "TecnicoDGV")`.
  Sigue la convención de `TecnicoDGV`: no se toca `app/auth.py`.
- **Secretaría:** `Admin` y `Autoridad` acceden siempre. El resto necesita `vivienda` en sus
  secretarías; si no la tiene, recibe 403. Es la misma regla que aplica el Resumen Territorial
  (`filtrar_por_visibilidad`), así el tablero no abre a otras áreas más de lo que ya abre ese panel.
- **Respuesta:**
  ```
  {
    generado_en: datetime,
    total_localidades_padron: int,                       # gobiernos locales (último snapshot; si no, padrón activo)
    total_localidades_por_departamento: {depto: int},
    expedientes: [ExpedienteTablero]
  }
  ```
- **Cada `ExpedienteTablero`:**
  - `programa` (`cordon_cuneta`|`cordoba_hogar`|`mi_lugar`), `programa_label`, `entidad_id`, `detalle`;
  - `localidad`, `departamento`, `id_geo`, `lat`, `lon` (centroide del padrón, `null` sin vínculo);
  - estado general (`id`, `label`, `bg`, `text_color`) y `subestados` (`juridico`, `tecnico`, `financiero`);
  - `checklist_total`, `checklist_faltan`, `checklist_iniciado`, `checklist_faltantes`;
  - `ultima_comunicacion` (`fecha`, `texto`, `area`, `autor`);
  - `monto`, `expediente`;
  - `avance_obra_pct`, `avance_obra_fecha`, `fecha_fin_obra`, `hitos_obra`. Salen de
    `estado_tecnico.avance_por_entidades`, con la misma forma que exponen los paneles desde v1.6.0.
- **Ubicación:** el nombre de localidad y departamento sale del padrón cuando hay `id_geo`
  (ADR-026); si no hay, del texto de la entidad.
- **Visibilidad de comunicaciones:** el texto de las comunicaciones de infraestructura y supervisión
  se enmascara con la misma regla que `resumen_territorial.service.filtrar_por_visibilidad` y
  `cordon_cuneta/service.listar_pedidos`. Se reutiliza esa función o su predicado; no se reescribe.
- **Costo:** unas 10 consultas fijas, sin N+1, para unos 330 expedientes.

### 4.3 KPIs precalculados y `GET /api/v1/vivienda/tablero-analitico/tendencias`
- **Migración `0040`:** columna `kpis JSON NULL` en `viv_resumen_territorial_snapshot`.
- **Función pura:** `resumen_territorial.aggregations.kpis_por_area(payload) -> dict[area, dict]`.
  Por área calcula:
  - `localidades`, `departamentos`;
  - `programas_activos` y `programas_por_programa: {programa: n}`;
  - `items_faltantes`, que es la cantidad de programas con al menos un ítem faltante;
  - `localidades_items_faltantes` (v1.1.0): localidades distintas con al menos un expediente con
    ítems faltantes. Un checklist no iniciado cuenta como faltante, porque le falta todo;
  - `comunicaciones_30d`, programas con una comunicación en los últimos 30 días contados desde el
    cómputo del snapshot;
  - `monto_total`;
  - `avance_obra_promedio`, sobre las obras con al menos una medición, o `null`.

  `compute + save` la guarda en cada snapshot nuevo.
- **Backfill:** script one-off `scripts/backfill_kpis_snapshot.py`. Toma el último snapshot de cada
  día que todavía no tenga `kpis` y lo completa. Es idempotente y se corre en Cloud Shell con
  `cloud-sql-proxy`. Los snapshots anteriores al deploy no traen avance, así que su
  `avance_obra_promedio` queda `null`.
- **`--completar-claves`** (v1.1.0): para el snapshot que lee Tendencias en cada día (el último con
  `kpis`), recalcula `kpis_por_area` y agrega **solo las claves que faltan**, sin pisar las
  existentes (así `avance_obra_promedio` ya guardado no se pierde). Idempotente; también acepta
  `--dry-run`.
- **`GET .../tendencias?dias=90`** (1–365, default 90), con los mismos roles que §4.2. Devuelve
  `[{fecha, kpis}]` con los `kpis["vivienda"]` del último snapshot con `kpis` de cada día, en orden
  ascendente. Los días sin dato no aparecen.

### 4.4 Área responsable del estado
- **Migración `0040`:** `area_responsable VARCHAR(20) NULL` en `viv_cc_estados`, `viv_ch_estados`
  y `viv_ml_estados`.
- **Valores:** `juridica` | `tecnica` | `financiera` | `externa` | `null`. Se valida en los schemas
  y la base agrega un `CHECK`.
- **Schemas:** `EstadoResponse`/`EstadoCreate`/`EstadoUpdate` (CC y CH) y `EstadoMLOut`/`EstadoMLCreate`/`EstadoMLUpdate`
  lo aceptan y lo devuelven. Los endpoints `POST`/`PATCH .../estados` ya registran audit log.
- **Carga de datos:** la hace el área desde la UI (§8.3). Ninguna migración lo precarga, porque
  la asignación es decisión del área.

### 4.5 Documentación en los paneles
- **`estado_tecnico.documentacion_por_entidad(db, programa, ids)`** devuelve
  `{entidad_id: (completos, total, iniciado)}`.
  - `total` excluye los ítems "No Aplica" (`excluye_completitud`), con la misma regla que
    `resumen_territorial` desde v1.6.0.
  - `iniciado` vale `False` si la entidad no tiene checklist.
- **Paneles:** los `get_full` de CC/CH/ML suman `doc_completos`, `doc_total` y `doc_iniciado`.

## 5. Frontend — pestaña Tablero

**Ruta:** `/vivienda/tablero-analitico`.

**Navegación:** ítem "Tablero analítico" en `Layout.tsx` `navItemsFor` y en `DashboardPage.tsx`
`SecretariaCard`. El filtro de `TecnicoDGV` pasa a incluirlo, junto con Tablero y Checklist Técnico.

**Código:** carpeta `src/modules/vivienda/tablero-analitico/`.

### 5.1 Filtros, compartidos por las tres pestañas
- **Ubicación:**
  - texto, que busca en localidad y departamento;
  - departamento;
  - localidad, que depende del departamento.
- **Programa.**
- **Estados:** general, jurídico, técnico y financiero. Las opciones de general, jurídico y
  financiero se arman con los valores que aparecen en los datos. Las de **técnico** salen del
  catálogo activo de estados del expediente del Checklist Técnico (`GET .../checklist-tecnico/catalogos`,
  ya accesible para los mismos roles), aunque ningún expediente tenga ese estado (v1.1.0). Todas van
  en orden alfabético (§9).
  - Nota: "APROBADO por TRIB.C", "En CURSO en TRIB.C", "OBRA en EJECUCIÓN", etc. son estados de ese
    catálogo porque el estado técnico de los paneles es el estado del expediente del checklist
    (`spec-estado-tecnico-desde-checklist.md`). Sacar uno es una baja en el catálogo que decide el
    área desde la administración del Checklist, no un cambio del tablero.
- **Documentación:** Completa (`faltan = 0` e iniciado), Incompleta (`faltan > 0`) y No iniciada.
  base44 lista cada combinación "X de Y faltan"; se agrupan porque esa lista no escala.
- **Obra:** situación de obra (`situacionObra` de `avanceObra.tsx`).
- **Distancia:** §6.1.
- **"Limpiar filtros"** y contador **"N expedientes"**.

### 5.2 KPIs, recalculados con los filtros
- **Localidades:** "X de `total_localidades_padron`", con barra apilada por programa y el resto en gris.
- **Departamentos:** "X de 26", con barra apilada por programa.
- **Localidades con ítems faltantes** (v1.1.0): "X de Y", con Y = localidades del conjunto
  filtrado. Debajo, una barra horizontal al 100 % por programa: el tramo de color son las
  localidades del programa con ítems faltantes y el resto, en gris, las demás localidades del
  programa; al costado, el dato nominal "N de M". El delta usa `localidades_items_faltantes` y no se
  muestra si la serie todavía no tiene la clave.
- **Comunicaciones de los últimos 30 días:** dona "X de N".
- **Programas activos:** barra apilada por programa.
- **Monto total.**
- **Avance de obra promedio**, de las obras con medición.

Cada KPI sin filtros muestra el **delta contra hace 7 días**, tomado de `/tendencias`. Con filtros
el delta se oculta, porque la serie histórica no está filtrada.

### 5.3 Mapa de calor y ranking
- **Mapa de calor:** `CoropletiqueDepartamentos` con la cantidad de programas por departamento. Un
  clic filtra por ese departamento.
- **Ranking de departamentos:** con más programas activos y con más comunicaciones en 30 días (top 5).

### 5.4 Tabla
- **Una fila por localidad**, agrupando sus expedientes: localidad, departamento y "N expedientes".
- **Columnas por expediente:**
  - programa;
  - chips de estado general, J, T y F;
  - checklist "X de Y faltan" / "Completo" / "No iniciado", como **enlace** al Checklist Técnico de
    ese expediente (v1.1.0): `/vivienda/checklist-tecnico?programa={cc|ch|ml}&entidad={id}`, en una
    pestaña nueva para no perder los filtros del tablero. `ChecklistTecnicoPage` lee esos parámetros
    y abre la localidad con ese programa ya elegido; un id inexistente se ignora;
  - avance de obra (`AvanceCelda`);
  - última comunicación (fecha).
- **Orden:** alfabético por localidad (`localeCompare('es')`), o por distancia cuando el filtro de
  distancia está activo.
- **Fila expandible** con los ítems faltantes del checklist y el texto de la última comunicación.
- **Exportar Excel** de lo filtrado (`exportToXlsx`): una fila por expediente, más `obraColumnasExcel`
  y la distancia si corresponde.

## 6. Frontend — distancias y pestaña Mapa

### 6.1 Filtro de distancia
- **Origen**, con dos opciones:
  - "Desde una localidad": desplegable con las localidades que tienen coordenadas, "LOCALIDAD (DEPTO)", en orden alfabético;
  - "Desde mi ubicación": `navigator.geolocation`, con un mensaje claro si el permiso se niega.
- **Radio:** campo numérico en km.
- **Cálculo:** `haversineKm` en `src/shared/utils/geo.ts`, en línea recta.
- **Con origen y km cargados:**
  - se filtran las filas con coordenadas a ≤ km;
  - la tabla suma la columna "Distancia";
  - la tabla se ordena por distancia;
  - se muestra el texto "dentro de N km desde X".

### 6.2 Mapa
- **Leaflet/OSM**, con el patrón de `MapaDualPuntos`. Hay un punto por localidad:
  - color por programa: CC naranja, CH azul y ML verde; con más de un programa, borde gris;
  - tamaño según la inversión, con leyenda menor/medio/mayor.
- **Popup:** localidad, departamento, N expedientes, inversión, cantidad por programa y el botón
  "Agregar al recorrido".
- **Filtros:** responde a los mismos de §5.1. Con el filtro de distancia activo dibuja el círculo
  del radio y el origen.
- **Contador:** "N localidades en mapa · M expedientes sin coordenadas". No se geocodifica.

### 6.3 Armar recorrido
- **Panel lateral** con las localidades agregadas, desde el popup o desde la tabla.
- **Origen:** el del filtro de distancia, o la primera localidad agregada.
- **"Ordenar":** ordena por vecino más cercano desde el origen y se puede reordenar a mano.
- **Ruta en el mapa:** polilínea y km totales en línea recta.
- **"Abrir en Google Maps":** `https://www.google.com/maps/dir/?api=1&origin=…&destination=…&waypoints=…`
  con las coordenadas.
  - Si hay más paradas que el límite de waypoints de la URL, avisa y abre solo las primeras.
  - Solo se envían coordenadas de localidades, nunca datos de expedientes ni personales.
- **Persistencia:** el recorrido no se guarda en el servidor. Es estado local de la página.

## 7. Frontend — pestaña Tendencias
- **Series:** líneas de Chart.js (vía `chartSetup.ts`) con datos de `/tendencias`:
  - programas activos;
  - localidades;
  - departamentos;
  - localidades con ítems faltantes (v1.1.0; los días sin la clave quedan sin punto);
  - comunicaciones en 30 días;
  - monto total;
  - avance de obra promedio.
- **Rango:** 30, 90 o 180 días.
- **Alcance:** sin filtros; son series de toda la provincia. La pestaña lo aclara.

## 8. Correcciones de listados (CC, CH y ML)

Los cambios van en los tres archivos hermanos.

### 8.1 Orden alfabético
- **Tabla:** `sortCol` inicial = localidad/municipio ascendente con `localeCompare('es')`. El usuario
  puede reordenar por columna como hoy.
- **Desplegable de departamentos:** `localeCompare('es')`.

### 8.2 Filtro "Documentación"
Opciones Completa, Incompleta y No iniciada, sobre `doc_*` (§4.5). Entra en `hasFilters` y en "limpiar".

### 8.3 Área responsable de cada estado
- **Edición:** `GestionarEstadosModal` suma un selector "Área" con Jurídica, Técnica, Financiera,
  Externa y "—".
- **Visualización:**
  - en los desplegables de estado, la opción muestra un prefijo corto (`[J]`, `[T]`, `[F]`, `[Ext]`);
  - el chip de estado de la tabla lo muestra en el `title`.
- **Aclaración del PDF:** "En revisión técnica" no es del área técnica; "En corrección" sí. La
  asignación la hace el área.

## 9. Decisiones tomadas

- **Delta de KPIs:** contra el snapshot de hace 7 días. base44 lo compara con "la versión anterior
  cargada", que acá no existe.
- **Orden de los estados:**
  - en los paneles, los desplegables mantienen el `orden` del flujo, porque el orden de proceso
    tiene significado;
  - en el tablero analítico los estados se agrupan entre programas y no tienen un orden de flujo
    común, así que van en orden alfabético;
  - las filas y los desplegables de localidad y departamento van en orden alfabético en todos lados.
- **Comunicaciones de 30 días:** cuentan desde la fecha de cálculo, que en el endpoint en vivo es "hoy".

## 10. Tests (`svc-vivienda`)
- **Regresión de la factorización:** `compute_resumen_territorial` produce las mismas líneas de Vivienda que antes.
- **`/tablero-analitico`:**
  - forma de la respuesta y coordenadas del padrón;
  - avance de obra;
  - checklist con "No Aplica";
  - enmascarado de comunicaciones;
  - roles: `TecnicoDGV` y `Autoridad` acceden, `invitado` recibe 403.
- **`kpis_por_area`:** función pura, con casos por área y comunicaciones dentro y fuera de 30 días.
- **`/tendencias`:**
  - último snapshot del día;
  - días sin `kpis` omitidos;
  - rango `dias`.
- **`area_responsable`:** alta y edición válidas; un valor inválido devuelve 422.
- **v1.1.0:** `localidades_items_faltantes` cuenta localidades y no expedientes (dos expedientes con
  faltantes en la misma localidad cuentan uno; un checklist no iniciado cuenta).
- **`documentacion_por_entidad`:** "No Aplica" y entidad sin checklist.
- **Regla de fixtures:** los fixtures `client_*` comparten override, así que nunca se combinan dos roles en un mismo test.

## 11. Gateway
- **Paths nuevos** en `infra/gateway/openapi.yaml`, cada uno con su `options` (`security: []`):
  - `/api/v1/vivienda/tablero-analitico`;
  - `/api/v1/vivienda/tablero-analitico/tendencias`.
- **Config nueva:** `ministerio-config-v{AAAAMMDD}`.

## 12. Despliegue

**v1.1.0:** sin migración ni cambios de gateway. Push a `main` (CI), `firebase deploy` y, en Cloud
Shell, `python -m scripts.backfill_kpis_snapshot --completar-claves` (primero con `--dry-run`).

**v1.0.0:**
1. Migración `0040` y backfill de `kpis` en Cloud Shell, desde una rama, antes del push a `main`.
2. Push a `main` de `panel.backend`, que dispara el CI de `deploy-svc-vivienda`.
3. Gateway con la config nueva.
4. `firebase deploy` del frontend.

## 13. Criterios de aceptación
- **Sin cargar Excel:** el tablero muestra los mismos conceptos que base44 con datos al instante.
  Para comprobarlo, se cambia un estado en un panel y el tablero lo refleja al recargar.
- **KPIs, tabla, mapa y Excel** responden a todos los filtros.
- **Distancia:** "40 km desde Cañada de Luque" devuelve los expedientes del radio. En base44, con
  datos del 2026-10-01, fueron 11. La diferencia se explica por los datos actuales o por los
  expedientes sin coordenadas.
- **Recorrido:** se arma, se ordena y abre Google Maps con las paradas.
- **Tendencias** muestra la historia desde el backfill.
- **Listados CC/CH/ML:**
  - abren en orden alfabético;
  - filtran por documentación;
  - muestran el área del estado una vez que el área la carga.
- **Alcance del refactor:** el Resumen Territorial no cambia de comportamiento.
- **Acceso:** `TecnicoDGV` ve el tablero y no gana acceso a los paneles CC/CH/ML.
- **v1.1.0:**
  - Mi Lugar muestra en "Localidades con ítems faltantes" un número ≤ a sus localidades, nunca 49 de 33;
  - el filtro Est. técnico ofrece "RECHAZADO por M/C";
  - un clic en el checklist de una fila abre el Checklist Técnico de ese expediente.
