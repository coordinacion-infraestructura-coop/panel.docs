# Spec: Informe por programa (Cordón Cuneta / Córdoba Hogar)

**Estado**: ✅ implementado (Fase 1) — pendiente de deploy
**Versión**: 1.0.0
**Servicio**: `svc-vivienda` (módulo nuevo `app/informes/`, sin servicio nuevo)
**Última actualización**: 2026-08-03

---

## 1. Propósito

Dar a cualquier funcionario (no solo a quien carga datos) una pestaña de análisis visual
rápido por programa: cobertura territorial, avance por estado y distribución geográfica,
sin necesidad de leer la tabla operativa fila por fila. Inspirado en el patrón de
`proyecto_sistema_gestiones/informe` (paleta gov, KPI strip, mapa con toggle
puntos/coroplético, donuts, evolución temporal, tabla exportable) y en las skills
`/mapa_coropleth_filtros` y `/mapa-dual-puntos`.

## 2. Alcance

### Incluido (Fase 1)
- Un informe por programa — **Cordón Cuneta** y **Córdoba Hogar** — accesible como
  subruta de cada panel (`/vivienda/cordon-cuneta/informe`, `/vivienda/cordoba-hogar/informe`).
- Motor de cálculo genérico compartido (`app/informes/aggregations.py`), dado que ambos
  programas comparten ~95% de estructura de datos (municipio/localidad + departamento +
  catálogo de estados con `estado_general`).
- Cálculo cacheado en DB (`viv_informe_snapshot`), no en vivo — botón "🔄 Actualizar
  informe" dispara el recálculo; la carga normal de la página lee el último snapshot.
- Mapas nativos en React/Leaflet (no HTML estático embebido): coroplético por
  departamento y puntos duales (cluster/jitter) por municipio/localidad.

### Fuera de alcance (Fase 1) — pendiente
- **Informe de Gestiones Privadas**: requiere que `svc-privada` implemente el mismo
  patrón de snapshot cacheado, pero su código fuente no estaba disponible en la sesión
  donde se implementó esto. Queda como Fase 2, a retomar cuando se tenga acceso al
  repositorio de `svc-privada` (o se decida computar client-side sin cache, ver decisión
  del usuario del 2026-08-03).
- Filtros dentro del informe (por rango de fechas, por dimensión jurídico/técnico/
  financiero en vez de solo `estado_general`) — se puede sumar después sin cambiar el
  modelo de datos.
- Descarga del informe completo en PDF — hoy solo se exporta a Excel la tabla de
  cobertura por departamento (reusa `exportToXlsx`, ya existente).

## 3. Modelo de datos

### `viv_informe_snapshot` (migración `0019`)

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | UUID | PK |
| `programa` | VARCHAR(30) | `cordon_cuneta` \| `cordoba_hogar` |
| `payload` | JSON | Informe completo calculado (ver forma abajo) |
| `computed_at` | TIMESTAMPTZ | Cuándo se calculó |
| `computed_by` | VARCHAR(255) | Email del actor que apretó "Actualizar" |
| `duracion_ms` | INTEGER | Tiempo que tardó el cálculo |

Índice `(programa, computed_at DESC)`. **No se sobreescribe** — cada actualización agrega
una fila nueva (mismo criterio que `viv_cc_sync_log`), da trazabilidad gratis de quién
actualizó y cuándo. El GET siempre lee la última fila por programa.

### Forma del payload (`InformePayload`, igual para cualquier programa)

```
programa, total_entidades, monto_total,
metricas_extra: dict[str, number]   # específico de cada programa, ver abajo
departamentos_cubiertos,
por_estado_general: [{estado_id, label, bg, text_color, cantidad}],
por_departamento: [{departamento, cantidad, localidades_cubiertas, localidades_totales, pct_cobertura}],
evolucion_temporal: [{mes, cantidad_cambios}],
puntos: [{id, nombre, departamento, lat, lon, estado_general_id, estado_label, estado_bg, estado_text_color, expediente, monto}]
```

`metricas_extra` por programa:
- Cordón Cuneta: `cordon_cuneta_ml_total`, `adoquinado_m2_total`.
- Córdoba Hogar: `cantidad_casas_total`, `monto_por_casa` (de `viv_ch_config`).

## 4. Cálculo (`app/informes/aggregations.py` + `service.py`)

Funciones puras, sin DB, testeadas por separado (`tests/test_informes.py`):

- **`kpis_por_estado`**: conteo por `estado_general`, orden del catálogo (`orden`),
  agrupa aparte las entidades sin estado.
- **`cobertura_por_departamento`**: cuenta entidades activas por depto vs. localidades
  activas totales en `viv_geo_localidades` para ese depto. No hace falta matchear
  entidad↔localidad una por una: la unicidad de (municipio/localidad, departamento) ya
  está garantizada por constraint de DB entre registros activos (migraciones 0014/0015),
  así que cada entidad ES una localidad distinta cubierta. El nombre de display del
  departamento prioriza la grafía del padrón geo (para alinear con el GeoJSON del mapa).
- **`evolucion_temporal`**: agrupa por mes el historial de cambios de cualquier
  dimensión (`viv_cc_estado_historial` / `viv_ch_estado_historial`).
- **`puntos_mapa`**: matching best-effort de cada entidad contra `viv_geo_localidades`
  por (departamento, nombre) normalizado — reusa `app/geo/matching.py` (extraído de
  `checklist_sync.py`, antes vivía duplicado ahí). Sin match, `lat`/`lon` quedan `null`
  y el punto simplemente no se dibuja — no rompe nada.

## 5. Endpoints

En los routers existentes (no uno nuevo genérico), mismo patrón que
`cordon-cuneta-checklist-tecnico/{estado,sync}`:

```
GET  /api/v1/vivienda/cordon-cuneta/informe            (ROLES_LECTURA)  → último snapshot o null
POST /api/v1/vivienda/cordon-cuneta/informe/actualizar (ROLES_ESCRITURA) → recalcula y guarda
GET  /api/v1/vivienda/cordoba-hogar/informe            (ROLES_LECTURA)
POST /api/v1/vivienda/cordoba-hogar/informe/actualizar (ROLES_ESCRITURA)
```

## 6. Frontend

- `frontend/src/modules/vivienda/pages/informe/ProgramaInformePage.tsx` — un único
  componente parametrizado por `programa: 'cordon_cuneta' | 'cordoba_hogar'`, montado en
  dos rutas (`vivienda/cordon-cuneta/informe`, `vivienda/cordoba-hogar/informe`).
- Componentes compartidos nuevos en `frontend/src/shared/components/informe/`:
  `KpiStrip`, `CoropletiqueDepartamentos` (puerto nativo de `/mapa_coropleth_filtros`),
  `MapaDualPuntos` (puerto nativo de `/mapa-dual-puntos`, mismo algoritmo de clustering
  en píxeles + jitter), `DonutChart`/`BarChart`/`LineChart` (wrappers finos sobre
  `chart.js`, sin `react-chartjs-2`).
- Dependencias nuevas: `leaflet`, `@types/leaflet`, `chart.js` — sin wrappers extra,
  montados con `useRef`/`useEffect` (coherente con que el proyecto no tenía ninguna
  librería de mapas/gráficos hasta ahora).
- Acceso: botón "📊 Ver informe" en la barra de acciones de `CordonCunetaPage.tsx` /
  `CordobaHogarPage.tsx`.
- Fetching: `useQuery({ staleTime: Infinity })` (no refetch automático) + `useMutation`
  sobre `/informe/actualizar` que actualiza la query al terminar. Indicador "Actualizado
  hace X min" — mismo estilo que ya existe para el sync del checklist técnico.

## 7. Asset — GeoJSON de departamentos

No existía en este proyecto un polígono de los 26 departamentos de Córdoba (solo
`docs/data/geo_localidades.json`, que son puntos). Se reusó el que ya tiene
`proyecto_sistema_gestiones/informe/bq_views/departamentos.json` (EPSG:22174),
reproyectado una sola vez a WGS84 con `pyproj` + `shapely.simplify(0.001)` vía
`frontend/scripts/prepare_departamentos_geojson.py` (script one-off, no se re-ejecuta en
cada deploy) → `frontend/public/geo/departamentos_cba.json` (223 KB, servido como asset
estático).

## 8. Verificación realizada

- 12 tests nuevos en `tests/test_informes.py` (unitarios de `aggregations.py` +
  integración de los 4 endpoints, incluye permisos) — 12/12 verdes. Suite completa:
  183/187 verdes (los 4 que fallan son preexistentes y no relacionados — prueban
  `_recompute_all_estado_general`, función eliminada el 2026-07-31 según memoria del
  proyecto; el ledger de auditoría quedó desactualizado en ese punto).
- Migración `0019` verificada contra Postgres real (no solo SQLite de los tests): se
  levantó `docker-compose.dev.yml` local, se corrió `upgrade`/`downgrade` y se confirmó
  el tipo de columna JSON y el índice compuesto.
- `npm run build` (`tsc -b && vite build`) verde, sin errores de tipos.
- **Pendiente de esta sesión**: prueba visual en navegador real — requiere login Google
  contra el proyecto Firebase real, no automatizable en este entorno. Además, para que
  la prueba tenga sentido primero hay que desplegar `svc-vivienda` (los endpoints nuevos
  no existen todavía en producción) y una nueva config de gateway.

## 9. Pendiente de infraestructura (no ejecutado en esta sesión)

Igual que el precedente ya documentado en `docs/files/auditoria-codigo.md` (Corregido
#2): el YAML del gateway ya tiene los 4 paths nuevos (`GET`/`POST` × CC/CH, con sus
`options:` de CORS), pero deployar `svc-vivienda` y una nueva versión de config del
gateway (`ministerio-config-v{FECHA}`) son acciones de infraestructura que requieren
confirmación aparte antes de ejecutarse.
