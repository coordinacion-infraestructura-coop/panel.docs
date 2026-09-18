# Spec: Informe "Localidades por Departamento" (Cordón Cuneta / Córdoba Hogar / Habitantes)

**Estado**: ✅ approved — implementado, ver §10 para el cierre del caso de completitud
de `Cant. Habitantes`
**Versión**: 1.1.0
**Servicios**: `svc-vivienda` (módulo nuevo `app/informe_localidades/`), `svc-privada`
(endpoint interno nuevo en `app/internal/router.py`, sin servicio nuevo en ninguno de los
dos)
**Última actualización**: 2026-09-17

---

## 1. Propósito

La DGV necesita, de forma recurrente, un informe que cruce el catálogo geográfico completo
de localidades de la provincia con el estado de dos programas de vivienda (Cordón Cuneta y
Córdoba Hogar) y con la cantidad de habitantes de cada localidad. Como el pedido es
recurrente ("es probable que esto vuelvan a pedirlo en otras ocasiones"), se deja un botón
de descarga permanente en el panel `/vivienda/programas` en vez de armarlo a mano cada vez.

## 2. Alcance

### Incluido

- Un endpoint nuevo en `svc-vivienda` que devuelve, para **todas** las localidades activas
  de la provincia (universo = `viv_geo_localidades`), una fila por localidad con las
  columnas descriptas en §4.
- Un botón "↓ Descargar informe por localidad" en `ProgramasPage.tsx`
  (`/vivienda/programas`) que llama a ese endpoint y genera el `.xlsx` client-side con la
  librería `xlsx` ya usada en el resto del proyecto (`frontend/src/shared/utils/exportTable.ts`)
  — sin generación de Excel en el backend, no hay precedente de eso en el proyecto.
- Cálculo **on-the-fly** en cada request (sin tabla de snapshot) — el universo es chico
  (~450 localidades) y no se necesita historial de corridas, a diferencia de
  `docs/files/spec-informes-programa.md`.
- Un endpoint IAM-only nuevo en `svc-privada` (`GET /internal/privada/localidades-habitantes`)
  para que `svc-vivienda` pueda traer `habitantes` sin cruzar bases directamente (ADR-012:
  `priv_localidades_info` es propiedad de `svc-privada`, solo se lee vía gateway/IAM).

### Fuera de alcance

- Filtros o parámetros en el endpoint (por departamento, por programa, etc.) — el informe
  siempre trae el universo completo; el filtrado, si se pide, se hace en el Excel mismo.
- Persistencia/histórico de corridas del informe.
- Cualquier cambio a `priv_localidades_info` (es de solo lectura para este informe).
- Reemplazar o tocar `resumen_territorial` ni `informes/` (CC/CH) — este es un módulo
  nuevo e independiente, aunque reutiliza matching y el patrón de federación con
  `svc-privada` ya validado por ADR-016.

## 3. Decisiones de diseño y correcciones sobre el pedido original

- **No existe un campo "m² de Cordón Cuneta"** en `viv_cordon_cuneta`. La tabla tiene
  `adoquinado_m2` (m² de adoquinado, otro dato) y `cordon_cuneta_ml` (metros **lineales**
  de cordón cuneta). Se usa `cordon_cuneta_ml`, y la columna del Excel se llama **"Metros
  Lineales Cordón Cuneta"** — no "M2 Cordón Cuneta" — para no rotular mal la unidad.
  (Decisión tomada con el usuario, 2026-09-16.)
- **Habitantes no existe en ninguna tabla de `svc-vivienda`.** Es un dato propiedad de
  `svc-privada` (`priv_localidades_info`, carga Excel one-off, ver ADR-012 —
  "staleness risk" documentado: nadie refresca esa carga). Cuando no hay match de
  `(departamento, localidad)` en `priv_localidades_info`, o cuando `svc-privada` no
  responde, la columna queda `null` — nunca se inventa un valor.
- **"Tiene X (SI/NO)"** se define como: existe al menos una fila `deleted_at IS NULL` en
  la tabla del programa cuyo `(departamento, municipio/localidad)` normalizado
  (`app/geo/matching.py::normalize_name`) coincide con el de la fila de
  `viv_geo_localidades`, **sin importar su `estado_general`** (una obra en cualquier
  estado ya cuenta como "tiene CC/vivienda" a los fines de este informe territorial).

## 4. Columnas del informe

| # | Columna Excel | Tipo | Fuente | Regla |
|---|---|---|---|---|
| 1 | Departamento | string | `viv_geo_localidades.departamento` | |
| 2 | Localidad | string | `viv_geo_localidades.localidad` | solo `activo=true` |
| 3 | Cant. Habitantes | int \| null | `priv_localidades_info.habitantes` (svc-privada, vía endpoint interno) | `null` si no hay match o el servicio no responde |
| 4 | Tiene Cordón Cuneta (SI/NO) | "SI"/"NO" | existencia en `viv_cordon_cuneta` (ver §3) | |
| 5 | Metros Lineales Cordón Cuneta | numeric \| null | `viv_cordon_cuneta.cordon_cuneta_ml` de la fila matcheada | `null` si no tiene CC |
| 6 | Tiene Viviendas (SI/NO) | "SI"/"NO" | existencia en `viv_cordoba_hogar` (ver §3) | |
| 7 | Cantidad de Viviendas | int \| null | `viv_cordoba_hogar.cantidad_casas` de la fila matcheada | `null` si no tiene viviendas |

Orden de filas: `(Departamento, Localidad)` ascendente.

## 5. Backend — `svc-vivienda`

Nuevo módulo `app/informe_localidades/` (un módulo por recurso, convención del proyecto):

- **`schemas.py`**: `LocalidadInformeRow` — `departamento: str`, `localidad: str`,
  `cant_habitantes: int | None`, `tiene_cordon_cuneta: bool`,
  `ml_cordon_cuneta: Decimal | None`, `tiene_viviendas: bool`,
  `cantidad_viviendas: int | None`.
- **`service.py`**: `compute_informe(db)`:
  1. Trae `viv_geo_localidades` activas.
  2. Trae `viv_cordon_cuneta` / `viv_cordoba_hogar` no borradas, arma diccionarios de
     lookup por `(normalize_name(departamento), normalize_name(municipio|localidad))`
     (mismo mecanismo que `app/informes/aggregations.py::puntos_mapa`).
  3. Llama `fetch_habitantes_privada(db_settings)` (nueva función, calco de
     `app/resumen_territorial/service.py::fetch_privada_lineas`: mismo uso de
     `settings.privada_fetch_enabled` / `settings.svc_privada_internal_url`, mismo
     ID-token de audiencia, mismo *fail-open* — si falla o está deshabilitado, devuelve
     `{}` y todas las filas quedan con `cant_habitantes=None`, el informe se sigue
     generando igual).
  4. Arma y devuelve la lista de `LocalidadInformeRow` ordenada.
- **`router.py`**: `GET /api/v1/vivienda/informe-localidades` → `list[LocalidadInformeRow]`,
  gateado con `ROLES_LECTURA_TABLERO` (la misma tupla que ya usa
  `app/programas/router.py` para `/programas-tablero` — así el botón funciona para
  cualquier rol que ya llega a `/vivienda/programas`, incluido `TecnicoDGV`). Agregar
  `options:` con `security: []` en `infra/gateway/openapi.yaml` para este path nuevo.
- Registrar el router en `main.py` junto a los demás (`# noqa: F401` del modelo si
  aplica — este módulo no tiene modelo propio, solo lee de `geo`/`cordon_cuneta`/
  `cordoba_hogar`, así que no hace falta).
- **Sin migraciones nuevas** — no hay tablas nuevas.

## 6. Backend — `svc-privada`

- Nuevo endpoint en `app/internal/router.py` (montado sin prefijo `/api/v1`, sin
  `Depends(get_current_user)`, protegido solo por Cloud Run IAM — mismo patrón que
  `GET /internal/privada/rollup-territorial`):
  ```
  GET /internal/privada/localidades-habitantes
  ```
  Implementación: expone tal cual la función de servicio ya existente
  `app/gestiones/service.py::listar_localidades_info(db)` (su propio docstring ya dice
  que está pensada "para el export Excel / impresión" — cero lógica de query nueva).
- Sin cambios de modelo, sin migraciones.

## 7. Frontend

- `frontend/src/modules/vivienda/api/vivienda.api.ts`: agregar
  `informeLocalidadesApi.get()` → `GET /api/v1/vivienda/informe-localidades`.
- `frontend/src/modules/vivienda/pages/ProgramasPage.tsx`: botón "↓ Descargar informe por
  localidad" en el header de la página (mismo estilo visual que el botón "Exportar" de
  `CordonCunetaPage.tsx`). Al click: fetch al endpoint nuevo, mapeo de cada fila a las 7
  columnas exactas de §4 (con "SI"/"NO" como string), y `exportToXlsx(rows, 'Localidades',
  'informe_localidades_AAAAMMDD.xlsx')`.
- Sin dependencias nuevas — reusa `xlsx` vía `exportTable.ts`.

## 8. Infraestructura / deploy (pendiente de confirmación aparte, no se ejecuta al aprobar esta spec)

- Nuevo path de gateway (`/api/v1/vivienda/informe-localidades`) → requiere nueva config
  `ministerio-config-v{FECHA}` y `gateway update`, igual que cualquier path nuevo.
- El endpoint interno de `svc-privada` no pasa por el gateway (IAM-only), pero si
  `SVC_PRIVADA_INTERNAL_URL`/`privada_fetch_enabled` no están seteados en el entorno de
  `svc-vivienda` (ya deberían estarlo desde `resumen_territorial`, ver `services/cloudbuild.yaml`),
  este informe funciona igual — solo con `Cant. Habitantes` siempre en `null`.

## 9. Verificación planeada

- `services/svc-vivienda/tests/test_informe_localidades.py`: localidad con CC y sin CH,
  con CH y sin CC, sin ninguno, matching insensible a mayúsculas/acentos, y
  `cant_habitantes=None` cuando `privada_fetch_enabled=False`.
- Test análogo en `services/svc-privada/tests/` para el endpoint interno nuevo.
- `pytest` verde en ambos servicios.
- Prueba manual end-to-end: `docker-compose.dev.yml` de ambos servicios levantado
  (`:8001`/`:8002`), `npm run dev`, click en el botón desde `/vivienda/programas`,
  verificar el `.xlsx` descargado contra los datos cargados en las tres bases.
- `npm run build` sin errores de tipos.

## 10. Cierre del caso — completitud de `Cant. Habitantes` (2026-09-16/17)

En el primer uso real del informe, `priv_localidades_info` tenía huecos de `habitantes`
para varias localidades — se hizo un trabajo de backfill y limpieza de datos que terminó
tocando, además de `priv_localidades_info`, el propio padrón geográfico
(`geo_localidades.json`/`viv_geo_localidades`/`priv_geo_localidades`) y el matching del
informe. Se documenta acá el resultado final para no repetir la investigación.

### 10.1 Backfill con el Censo Nacional 2022

`services/svc-privada/scripts/cargar_habitantes_censo2022.py` (RE-1, re-ejecutable) cruza
`docs/data/c2022_cordoba_gobierno_local_c1 (5).xlsx` (Cuadro 1.6 INDEC — población por
"gobierno local", no trae departamento) contra `priv_geo_localidades` por nombre
normalizado con alias de paréntesis/guion. De los 427 gobiernos locales del censo, el
matching automático resolvía 390 solo; los 37 restantes (10 ambiguos por nombre duplicado
en 2 departamentos + 27 "sin match") se resolvieron a mano — ver el docstring del script
para el detalle caso por caso. Conclusión relevante: **ninguna de las 13 candidatas
originales a "localidad nueva" resultó serlo** — las 13 ya existían en el padrón bajo un
typo viejo, abreviatura, nombre histórico o alias, detectado recién con
`services/svc-privada/scripts/buscar_habitantes_faltantes.py` (diagnóstico de similitud de
texto). De paso se corrigieron 2 errores reales preexistentes del padrón (departamento
incorrecto de Monte Cristo — COLÓN → RÍO PRIMERO, confirmado por fuente externa y por las
coordenadas ya cargadas) y se borraron **20 filas duplicadas** de `geo_localidades.json`
que representaban el mismo lugar dos veces con grafías distintas (ej. "PLAZA COLAZO" /
"COLAZO", "LUXARDO" / "PLAZA LUXARDO", "SAN FRANCISCO" / "PLAZA SAN FRANCISCO").

### 10.2 Matching con alias también en el informe (no sólo en la carga)

El match de habitantes en `app/informe_localidades/service.py` usaba comparación exacta
por nombre normalizado — insuficiente para las localidades con alias entre paréntesis/guion
del propio `viv_geo_localidades`. Se agregó `candidatos_localidad()` a
`app/geo/matching.py` (misma lógica de alias que el script de carga) y se la usa en ambos
lados del match (geo ↔ `priv_localidades_info`). Esto solo mejora el match de
**habitantes** — CC/CH siguen con comparación exacta, sin cambios.

### 10.3 Resultado final y límite real de la fuente

Tras el backfill + la limpieza de duplicados + el matching con alias: de **544 localidades
activas**, **~110 quedan sin dato de población**, y esto **no es un problema de datos
faltantes por completar** sino un límite estructural de cómo Argentina mide población:

- El Censo 2022 (Cuadro 1.6, INDEC) solo publica población a nivel **gobierno local**
  (municipio/comuna), no por cada localidad/paraje nombrado dentro de su jurisdicción.
- Se investigó exhaustivamente buscando una fuente más granular: no existe una tabla INDEC
  "por localidad" (todas las tablas temáticas del Censo 2022 llegan como máximo a
  departamento, salvo el propio Cuadro 1.6); un archivo histórico propio con **radios
  censales** (`P14Rad.xlsx`, la unidad geográfica más chica que mide el INDEC) solo tiene
  población hasta el **Censo 2010**, y aun así ninguno de estos ~110 nombres aparece como
  categoría propia — su población queda disuelta dentro del radio de la localidad vecina
  más grande. El catálogo de "Parajes" del INDEC/Mapa Educativo (carpeta `parajes/` del
  archivo histórico) confirma independientemente que estos lugares están clasificados como
  "Paraje", no como unidad con población propia.
- Conclusión: **no hay ninguna fuente oficial, actual ni histórica, que permita completar
  estos ~110 sin inventar el dato.** Quedan correctamente en `null` — no es una tarea
  pendiente, es el piso real de la fuente. Si en el futuro se necesita este dato, la única
  vía sería una fuente no-oficial por localidad (ej. relevamiento propio o de cada
  municipio/comuna), no un backfill automatizable.
