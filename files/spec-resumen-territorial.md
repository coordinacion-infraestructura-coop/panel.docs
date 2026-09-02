# Spec: Resumen Territorial — panel consolidado por localidad y departamento

**Estado**: approved
**Versión**: 0.2.0
**Aprobado**: 2026-08-28 (Pedro Bonafe) — decisiones de arquitectura, alcance con Privada,
enmascarado de comunicaciones, coordinación de gateway y número de migración confirmados.
**Servicio**: `svc-vivienda` (módulo nuevo `app/resumen_territorial/`, sin servicio nuevo)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-08-28

> **Cambio 0.2.0 (2026-08-28)**: la v1 incluye **también las gestiones de la Secretaría
> Privada del Ministro**, no sólo los 3 programas de Vivienda. El panel debe mostrar *todos*
> los programas/gestiones que tiene una localidad. Privada deja de ser Fase 2. Ver §2, §3.4,
> §5.2, §6.3, §7, §12.

---

## 0. Origen

Pedido del usuario: un panel-resumen para funcionarios de alto rango cuya unidad de análisis
sea la localidad (y opcionalmente el departamento), que muestre por localidad qué programas
tiene, en qué `estado_general` está cada uno, qué ítems del checklist técnico faltan y cuál fue
la última comunicación registrada. El mismo panel debe servir a todas las áreas con control de
visibilidad: un usuario de Vivienda ve sólo lo de Vivienda por localidad, uno de Gasífera lo
suyo, etc.; `Admin` y un rol nuevo `Autoridad` ven todo. Debe ser exportable a Excel e
imprimible.

Se evaluaron alternativas de arquitectura (sesión 2026-08-28) y se aprobó un prototipo visual
(artifact "Resumen Territorial", https://claude.ai/code/artifact/2d5d0c8b-a77f-4554-9da4-3d900a5a6078).
Este spec traduce esas decisiones a un diseño de implementación.

## 1. Propósito

Dar una vista única, de arriba hacia abajo, del estado de los programas del Ministerio sobre el
territorio, sin obligar a abrir el panel operativo de cada programa. Reemplaza el consolidado
"a mano" que hoy hace el frontend (`ProgramasPage` dispara varias queries en paralelo;
`ChecklistTecnicoPage` fusiona CC+CH+ML en el browser por nombre normalizado) por un snapshot
calculado en el backend, con una única regla de visibilidad por área.

## 2. Alcance

### Incluido (v1)
- **Unidad de análisis**: localidad (vista principal) y departamento (rollup de la misma data,
  client-side).
- **Programas / gestiones cubiertos**:
  - **Vivienda** — Cordón Cuneta, Córdoba Hogar, Mi Lugar. Mi Lugar con rollup
    proyecto→localidad (un proyecto = una línea; una localidad puede tener varias líneas de Mi
    Lugar). Fuente: `db_vivienda` (mismo servicio).
  - **Secretaría Privada del Ministro** — las gestiones/demandas del sistema `svc-privada`
    (dataset BigQuery `infra_gestion`, proyecto `essential-haiku-482815-u4`), consumidas vía
    API Gateway (`/api/v1/privada/*`, ADR-006). Representación: **una línea de roll-up por
    localidad** (`area:"privada"`), con conteo de gestiones por estado y fecha de última
    actividad — svc-privada expone un agregado por departamento/localidad, no un feed por
    caso. Mecanismo de obtención: ver §6.3 y §3.4.
- Por cada (localidad × programa): `estado_general` (label + colores del catálogo del
  programa), sub-estados jurídico/técnico/financiero (sólo en la ficha), cantidad y detalle de
  **ítems de checklist faltantes**, y **última comunicación registrada** (fecha, área, autor y
  — según visibilidad — el texto).
- **Cálculo cacheado** en `viv_resumen_territorial_snapshot` (append-only, se lee el último),
  con botón "Actualizar" (recálculo bajo demanda) y refresco por Cloud Scheduler. Igual patrón
  que `viv_informe_snapshot` (`spec-informes-programa.md`).
- **Contrato de API agnóstico de la fuente** — la forma del payload no asume de dónde salen los
  datos, para poder cambiar la fuente en Fase 2 (ver §13) sin tocar el frontend.
- **Visibilidad por área**: el snapshot guarda TODO; el `GET` filtra las filas según el rol y
  las secretarías del usuario. Rol nuevo `Autoridad` (ve todo, como `Admin`).
- **Export a Excel** (reusa `exportToXlsx`, ya existe) e **impresión** (vista `@media print`,
  A4 vertical).
- Módulo frontend nuevo **transversal**: carpeta `src/modules/resumen-territorial/`, ruta
  `/resumen-territorial`, entrada de navegación top-level (no bajo una secretaría).

### Fuera de alcance (v1)
- **Mapa** (coroplético por departamento o de puntos por localidad). Se suma en v2 reutilizando
  `CoropletiqueDepartamentos` / `MapaDualPuntos` y el GeoJSON que ya existen en `informe/`.
- **Datos de Infraestructura, Gasífera, Territorial, Desarrollo**. Sus microservicios no existen
  todavía. Un usuario cuyas secretarías se limitan a esas áreas ve el estado vacío ("tu área no
  tiene un servicio conectado a este panel"). Integración → Fase 2 (§13). **Privada SÍ entra en
  v1** (ver arriba).
- **Detalle por gestión de Privada** (estado, expediente y última comunicación caso por caso).
  v1 muestra el roll-up por localidad; el detalle se consulta en el panel de Privada. Feed por
  caso → Fase 2, si svc-privada expone un endpoint apto.
- **Historial de `estado_general`**: no existe en base (`*_estado_historial` sólo registra las
  3 sub-dimensiones, nunca `estado_general` — gap conocido, CLAUDE.md). El panel muestra el
  estado vigente, sin "desde cuándo".
- **PDF descargable** — se usa la impresión del navegador (Ctrl+P) sobre la vista `@media print`.
- **Descarga / recálculo desde el propio panel de un rol de sólo lectura de otra área** — el
  botón "Actualizar" queda para `ROLES_ESCRITURA` + `Autoridad`.
- **Emisión de eventos Pub/Sub** — este módulo no publica eventos (igual que el resto de los
  "panel modules": CC/CH/ML/checklist).

## 3. Decisiones de arquitectura

### 3.1 Enfoque: agregación en el backend, por fases (no BigQuery ahora)
Los ADR (001, 004, 006) prevén BigQuery (`ministerio_ejecutivo`) para el reporting ejecutivo
alimentado por eventos. **Ese pipeline no existe**: `bigquery.googleapis.com` no está
habilitado, no hay cliente ni dependencias de BQ en `services/`, `publish_event` es no-op en
dev, y CC/CH/ML/checklist/pedidos no emiten ningún evento. Construirlo entero para servir un
panel que hoy sólo puede mostrar una secretaría es desproporcionado.

**Decisión**: v1 calcula el resumen en `svc-vivienda`. Las líneas de Vivienda salen de su
propia base (`db_vivienda`), reutilizando el patrón `app/informes/` (snapshot en Postgres +
funciones puras). Las líneas de Privada las trae el mismo job de cálculo llamando al API
Gateway (§3.4). El contrato de API (`ResumenTerritorialPayload`, §5.2) es agnóstico de la
fuente para que en Fase 2 se puedan sumar más áreas o mover el cómputo a BigQuery sin tocar
el frontend.

### 3.2 Relación con ADR-001 (database per service)
ADR-001: "los reportes ejecutivos se alimentan via eventos a BigQuery, **no por joins entre
bases**". La v1 **no** hace joins entre bases: las líneas de Vivienda se agregan dentro de
`db_vivienda` (cross-programa, no cross-service) y las de Privada llegan por **llamada HTTP al
API Gateway** al endpoint agregado que svc-privada ya expone — nunca por acceso directo a
BigQuery ni join entre bases. Es consistente con ADR-001 ("los reportes se alimentan vía
eventos… no por joins") y con ADR-006 ("el API Gateway puede rutear cross-project").

### 3.3 Cómo se obtienen las líneas de Privada

> **Resuelto en implementación (2026-08-28): federación en el frontend (plan B).**
> El intento server-a-servidor **no funciona** — comprobado contra el sistema real:
> 1. `GET /api/v1/privada/gestiones/resumen-territorial` **exige `departamento` obligatorio**
>    (`Query(..., min_length=1)`) — es un detalle por depto/localidad, no un rollup global — y
>    pide `require_roles("Admin","Supervisor","Operador","Consulta")`, no es "Todos".
> 2. svc-privada valida el token contra su propio client-id de Google Sign-In y rechaza el
>    ID token de la SA de `svc-vivienda`: `401 {"detail":"Invalid token"}`. Y no se puede usar
>    otro `aud` sin que el gateway (`google_accounts`, `x-google-audiences: gestorcooperativo`)
>    rechace el token.

**Implementación activa — el frontend federa Privada** con el token del usuario (el mismo que
ya usa el módulo `privada` en producción):
- `frontend/src/modules/resumen-territorial/api/privadaGestiones.ts` pagina
  `GET /api/v1/privada/gestiones/` (`limit` tope 200 → varias páginas hasta `total`), agrupa las
  gestiones por `(normalize(departamento), normalize(localidad))` y arma **una línea roll-up
  `area:"privada"` por localidad** con `privada_conteos` (por `estado`/`estado_nombre`), badge de
  estado derivado y `ultima_comunicacion.fecha` = `max(fecha_estado|fecha_ingreso)`.
- `ResumenTerritorialPage` dispara esa query **sólo si** `rol ∈ {Admin, Autoridad}` o
  `"privada" ∈ secretarias`, y mergea el resultado con el snapshot de Vivienda por la misma
  clave normalizada. Si la query falla (el usuario no está en `usuarios_roles` de svc-privada),
  la página muestra sólo Vivienda + un aviso "Privada no disponible".
- La regla de visibilidad por área se mantiene: un usuario sin `"privada"` nunca dispara la
  query; uno sin Vivienda ve sólo la línea de Privada.

**Backend — dejado listo detrás de un flag (apagado)**: `service.fetch_privada_lineas()` +
`_map_privada_payload()` siguen en el código pero `compute_resumen_territorial` sólo los llama
si `settings.privada_fetch_enabled` (default `False`). Encender ese flag (y ajustar
`_map_privada_payload` a la forma real) sólo tiene sentido si el equipo de svc-privada habilita
la SA `svc-vivienda@…` en su `usuarios_roles` y expone un endpoint de rollup global. Config en
`app/config.py`: `privada_fetch_enabled`, `gateway_base_url`, `privada_resumen_path`,
`privada_gateway_audience`.

### 3.4 Ubicación (ADR-007)
El módulo vive en `svc-vivienda` y se monta con prefijo `/api/v1` (no `/api/v1/vivienda`),
igual que `app/portal/` — es funcionalidad transversal servida por el único servicio con Cloud
SQL activo. Si en el futuro pasa a ser un agregador ministerial real, es un movimiento de
módulo dentro del mismo patrón (mismo criterio que ADR-007 deja registrado para `portal`).

## 4. Unidad de análisis y agrupación por localidad

- **Clave de agrupación**: `(normalize_name(departamento), normalize_name(nombre_localidad))`
  usando `app/geo/matching.py:normalize_name` (mismo criterio que
  `app/informes/aggregations.py:puntos_mapa`). `nombre_localidad` es `municipio` en CC,
  `localidad` en CH, `localidad_nombre` (fallback `nombre`) en ML.
- **`departamento`** está denormalizado en cada entidad (texto libre tipeado al cargar). Si
  está vacío → clave `("", normalize_name(nombre))` y se muestra como "Sin departamento".
- **Nombre de display** de la localidad y del departamento: prioriza la grafía del padrón
  `viv_geo_localidades` cuando hay match (igual que `cobertura_por_departamento`); si no,
  la grafía de la entidad.
- **Mi Lugar**: cada `ProyectoML` activo es una línea de programa independiente, con
  `detalle = proyecto.nombre` para desambiguar cuando hay más de uno en la misma localidad.
- El resumen **nunca recomputa `estado_general`** — lo muestra tal como está en la fila
  (`estado_general` es 100% manual desde 2026-07-31, CLAUDE.md).

## 5. Modelo de datos

### 5.1 `viv_resumen_territorial_snapshot` (migración nueva, siguiente tras `0022`)
```
id           UUID PK
payload      JSON NOT NULL          -- resumen completo SIN filtrar por visibilidad
computed_at  TIMESTAMPTZ NOT NULL   -- cuándo se calculó
computed_by  VARCHAR(255) NULL      -- email del actor, o 'cloud-scheduler'
duracion_ms  INTEGER NULL
```
Índice `ix_resumen_computed_at (computed_at DESC)`. **No se sobreescribe** — una fila por
corrida (mismo criterio que `viv_informe_snapshot` / `viv_cc_sync_log`). El `GET` lee la
última. No hay columna `programa` (hay un único resumen).

El rol `Autoridad` es sólo un string en `portal_usuarios.rol` — **no requiere migración**;
sólo se agrega a `ROLES_VALIDOS` en `app/portal/schemas.py`.

### 5.2 Forma del payload (`ResumenTerritorialPayload`)
```
generado_para_areas: [str]            # informativo: áreas efectivamente incluidas en el cálculo
                                      #   (p.ej. ["vivienda", "privada"]; sin "privada" si su fetch falló)
total_localidades: int
total_programas: int                  # cuenta líneas de programa/gestión (incluye las de Privada)
localidades: [
  {
    localidad: str,
    departamento: str | null,
    programas: [
      {
        area: str,                    # "vivienda" | "privada"  (clave del filtro de visibilidad, §7)
        programa: str,                # vivienda: "cordon_cuneta"|"cordoba_hogar"|"mi_lugar"
                                      #  privada:  "gestiones"
        programa_label: str,          # "Cordón Cuneta y Adoquinado" | ... | "Gestiones — Sec. Privada"
        entidad_id: str | null,       # id de la fila CC/CH/ML; null para la línea roll-up de Privada
        detalle: str | null,          # ML: nombre del proyecto. Privada: "N gestiones" o breakdown corto
        estado_general_id: int | null,
        estado_general_label: str | null,   # vivienda: label del catálogo. privada: "En curso"/"Finalizadas"/mixto
        estado_general_bg: str | null,
        estado_general_text_color: str | null,
        subestados: { juridico: str|null, tecnico: str|null, financiero: str|null } | null,  # null en Privada
        checklist_total: int,         # vivienda: CC 19 / CH 14 / ML 20.  Privada: 0
        checklist_faltan: int,        # ítems con valor != 'completo' (o = total si no iniciado). Privada: 0
        checklist_iniciado: bool,     # false = no hay fila en viv_checklist_tecnico todavía. Privada: false
        checklist_faltantes: [str],   # labels de los ítems faltantes (vacío si no iniciado o Privada)
        ultima_comunicacion: {
          fecha: date,                # vivienda: fecha_pedido.  privada: fecha de última actividad de la localidad
          texto: str | null,          # vivienda: descripcion (null si el usuario no puede ver esa área, §7). privada: null
          area: str | null,           # vivienda: secretaria del pedido.  privada: "privada"
          autor: str | null
        } | null,
        monto: float | null,          # privada: null en v1 (costo_estimado sólo está en el detalle)
        expediente: str | null,       # privada: null (el nro_expediente es por gestión, no por roll-up)
        # sólo Privada:
        privada_conteos: { por_estado: {<estado>: int}, total: int } | null
      }
    ]
  }
]
```

`ResumenSnapshotResponse` (respuesta del `GET`): `{ payload, computed_at, computed_by,
duracion_ms }` — misma forma que `InformeSnapshotResponse`.

### 5.3 Línea de Privada (`area:"privada"`) — detalle
- **Una sola línea de roll-up por localidad** (no una por gestión), porque svc-privada expone
  un agregado por departamento/localidad, no un feed por caso.
- `programa = "gestiones"`, `programa_label = "Gestiones — Sec. Privada"`.
- `estado_general_label` + colores: se derivan del breakdown por estado (`privada_conteos`):
  si todas están en un estado terminal (`FINALIZADA`/`ARCHIVADO`) → "Finalizadas" (verde); si
  hay alguna activa → "En curso" (azul) con el conteo en `detalle`. Mapa de colores
  hardcodeado en `aggregations.py` (los 6 estados de `cat_estado`), igual criterio que los
  catálogos `viv_{cc,ch,ml}_estados` que traen `bg`/`text_color`.
- `detalle` = texto corto tipo `"5 gestiones · 3 en curso, 2 finalizadas"`.
- `ultima_comunicacion.fecha` = fecha de la actividad más reciente de la localidad que
  devuelva el endpoint agregado; `texto`/`autor` = null en v1.
- `checklist_*` en cero / `false`; `subestados = null`; `entidad_id = null`.

## 6. Cálculo — `app/resumen_territorial/`

Convención "panel module" (CLAUDE.md): sin `repository.py`, queries inline en `service.py`,
sólo audit log, sin Pub/Sub. Espeja `app/informes/` casi 1:1.

### 6.1 `aggregations.py` — funciones puras, sin DB (testeadas por separado)
- `PROGRAMA_A_CHECKLIST = {"cordon_cuneta": "cc", "cordoba_hogar": "ch", "mi_lugar": "ml"}`.
- `PROGRAMA_LABEL = {"cordon_cuneta": "Cordón Cuneta y Adoquinado", "cordoba_hogar": "Córdoba
  Hogar", "mi_lugar": "Mi Lugar"}`.
- **`agrupar_por_localidad(lineas, geo) -> [ResumenLocalidadDict]`**: agrupa las líneas de
  programa por la clave normalizada de §4; resuelve nombre de display contra `geo`; ordena
  localidades por `(departamento, localidad)`.
- **`items_faltantes(items_rows, programa_cod) -> (total, faltan, labels, iniciado)`**:
  `total = len(catalog.todos_los_item_keys(programa_cod))`. Si `items_rows` está vacío (no hay
  checklist) → `(total, total, [], False)`. Si no → `faltan`/`labels` de los ítems con
  `valor != "completo"`, label vía `catalog.item_label(programa_cod, item_num, sub_item_num)`.
- **`ultima_comunicacion(pedidos_rows) -> dict | null`**: máximo por
  `(fecha_pedido, created_at)`; devuelve `{fecha, texto, area, autor}` sin enmascarar (el
  enmascarado es del `GET`, §7).

### 6.2 `service.py` — orquestación (toca DB), espeja `app/informes/service.py`
- `_geo_rows(db)` — copiado de `informes/service.py`.
- **`compute_resumen_territorial(db) -> ResumenTerritorialPayload`**:
  1. CC: `MunicipioCordonCuneta` con `deleted_at IS NULL` + catálogo `EstadoCordonCuneta`.
     CH: `LocalidadCordobaHogar` + `EstadoCordobaHogar`. ML: `ProyectoML` + `EstadoML`
     (los ids de estado son únicos aunque `viv_ml_estados` esté segmentado por `tipo`).
  2. `viv_checklist_tecnico` (todas las filas) + `viv_checklist_items` (una query cada una),
     agrupadas en Python por `(programa, entidad_id)`. **No se crean filas** (a diferencia de
     `checklist_tecnico/service._get_or_create_checklist`): el compute es de sólo lectura.
  3. `viv_cc_pedidos` + `viv_ch_pedidos` + `viv_ml_pedidos`, agrupados por entidad, se conserva
     el último por `(fecha_pedido, created_at)`.
  4. `_geo_rows(db)`.
  5. Construye las líneas `{area:"vivienda", programa, nombre, departamento, estado_general,
     expediente, monto, entidad_id, detalle}`, resuelve label/colores de estado y sub-estados
     contra el catálogo del programa, cuelga checklist y última comunicación por `entidad_id`.
  6. **Privada** (§6.3): `lineas_privada = await _fetch_privada_lineas()` — lista de líneas
     `{area:"privada", ...}` por localidad; `[]` si el fetch falla.
  7. `aggregations.agrupar_por_localidad(lineas_vivienda + lineas_privada, geo)`;
     `generado_para_areas = ["vivienda"] + (["privada"] si hubo líneas de privada)`.
- `get_last_snapshot(db)` / `actualizar_resumen(db, computed_by: str) -> Snapshot` — igual a
  `informes/service.py`. `actualizar_resumen` recibe un `str` (no `AuthUser`) para poder
  llamarse desde el endpoint interno; el audit log usa ese string como actor.
- **`filtrar_por_visibilidad(payload, *, rol, secretarias) -> ResumenTerritorialPayload`** — §7.

### 6.3 `_fetch_privada_lineas()` — cliente del API Gateway
- Mintea un ID token con `google.auth` (SA de `svc-vivienda`, audience según el
  `securityDefinition` `google_accounts` del gateway) y hace
  `httpx.get(settings.gateway_base_url + settings.privada_resumen_path, headers={Authorization: Bearer <id_token>}, timeout=20)`.
- Mapea la respuesta agregada (forma real a confirmar en implementación — ver §3.3) a una
  lista de líneas `area:"privada"` por localidad, con `privada_conteos` y `estado_general_*`
  derivados (§5.3). La clave de localidad usa `normalize_name(departamento, localidad)`, igual
  que las líneas de Vivienda, para que `agrupar_por_localidad` las una en la misma fila.
- **Cualquier excepción / status ≠ 200 / JSON inesperado** → log `warning` y `return []`. El
  snapshot igual se guarda (sólo con Vivienda); `generado_para_areas` lo refleja.
- Testeable con `httpx.MockTransport` / `respx`; el test de servicio mockea esta función.

## 7. Visibilidad por área y rol `Autoridad`

### 7.1 Rol `Autoridad`
- Se agrega a `ROLES_VALIDOS` en `app/portal/schemas.py`
  (`("Admin","Supervisor","Operador","Consulta","TecnicoDGV","Autoridad")`).
- **No** se agrega a las constantes compartidas de `app/auth.py` (`ROLES_LECTURA`, etc.) —
  mismo criterio que `TecnicoDGV` (spec-checklist-tecnico-dgv.md §8): agregarlo ahí le daría
  acceso automático a todos los paneles operativos. En su lugar, el router del módulo define
  tuplas locales.
- Es un rol **fuera de la jerarquía** `Admin > Supervisor > Operador > Consulta`: no es un
  rango, es acceso de lectura consolidada cross-área. Un usuario `Autoridad` puede además
  tener secretarías asignadas (para los paneles operativos), pero para este panel su alcance
  es "todo", igual que `Admin`.

### 7.2 Regla de filtrado (`filtrar_por_visibilidad`, aplicada en el `GET`)
- Si `rol in ("Admin", "Autoridad")` → payload sin cambios (ve Vivienda **y** Privada).
- Si no → por cada localidad, se filtran las líneas de `programas` a las de `area in
  secretarias`; se descartan las localidades que quedan sin programas; se recalculan
  `total_localidades` / `total_programas` / `generado_para_areas`.
  - Un usuario con `secretarias = ["vivienda"]` ve sólo las líneas de Vivienda; uno con
    `["privada"]` ve sólo la línea de Privada de cada localidad; uno con `["vivienda","privada"]`
    ve ambas. `area:"privada"` se filtra con la misma clave (`"privada" in secretarias`) que
    las de Vivienda — no hay lógica aparte.
- **Enmascarado de `ultima_comunicacion.texto`**: replica la regla de
  `cordon_cuneta/service.listar_pedidos`. Si `rol not in ("Admin","Autoridad")` y
  `"supervision" not in secretarias`:
  - si el `area` de la comunicación es `"infraestructura"` y `"infraestructura" not in
    secretarias` → `texto = null` (se conservan `fecha`, `area`, `autor`).
  - si el `area` es `"supervision"` → `texto = null`.
  - resto → `texto` visible.
- **Limitación conocida v1**: el snapshot guarda sólo la última comunicación de cada entidad;
  si esa está enmascarada, el usuario de área ve fecha+área sin texto y sin fallback a la
  última comunicación que sí podría ver. Mejora → Fase 2 (guardar "última visible por
  alcance").

### 7.3 Frontend
- La ruta `/resumen-territorial` es accesible para todo rol de portal
  (`Admin/Autoridad/Supervisor/Operador/Consulta/TecnicoDGV`); `invitado` se redirige a `/`.
- El contenido lo filtra el backend. Un usuario cuyas secretarías no tienen servicio conectado
  (p. ej. sólo `gasifera`) recibe `localidades: []` y ve el estado vacío correspondiente.

## 8. Endpoints

```
GET  /api/v1/resumen-territorial            (ROLES_LECTURA_RESUMEN)  → ResumenSnapshotResponse | null
POST /api/v1/resumen-territorial/actualizar (ROLES_ESCRITURA_RESUMEN) → recalcula y guarda snapshot nuevo
POST /internal/resumen-territorial/actualizar (IAM-only, sin JWT)     → idem, para Cloud Scheduler
```
- `ROLES_LECTURA_RESUMEN = ROLES_LECTURA + ("Autoridad", "TecnicoDGV")` — tupla local en
  `app/resumen_territorial/router.py`.
- `ROLES_ESCRITURA_RESUMEN = ROLES_ESCRITURA + ("Autoridad",)`.
- `GET` devuelve `null` si nunca se calculó un snapshot (estado válido, no error — igual que
  `cordon-cuneta/informe`). Cuando hay snapshot, se devuelve
  `filtrar_por_visibilidad(payload, rol=actor.role, secretarias=actor.secretarias)`.
- El endpoint `/internal/...` sigue el patrón de `internal/router.py`
  (`sync/cordon-cuneta-checklist`): sin `Depends(get_current_user)`, `502` si el cálculo
  falla (dispara la alerta de Cloud Monitoring / marca la corrida de Scheduler como fallida).
  **No** se declara en `openapi.yaml`.
- Cada path público nuevo necesita su `options:` con `security: []` en
  `infra/gateway/openapi.yaml` (CORS preflight), backend → `svc-vivienda`.

## 9. Frontend — `src/modules/resumen-territorial/`

Carpeta nueva (NO `src/modules/territorial/`, reservada para la futura Secretaría de
Planificación y Articulación Territorial).

- **`api/resumenTerritorial.api.ts`**: `resumenTerritorialApi = { getResumen, actualizarResumen }`
  contra `/api/v1/resumen-territorial` — mismo estilo que `vivienda.api.ts`.
- **`types/resumenTerritorial.types.ts`**: espejo del payload de §5.2.
- **`pages/ResumenTerritorialPage.tsx`**:
  - Header con botón "Actualizar" + sello `computed_at`/`computed_by` — copiado de
    `ProgramaInformePage.tsx` (`useQuery({staleTime: Infinity})` + `useMutation` que hace
    `queryClient.setQueryData`).
  - `<KpiStrip>` (`shared/components/informe/KpiStrip.tsx`): localidades con programa,
    programas activos, con ítems faltantes, comunicaciones ≤30 días, departamentos alcanzados
    — recalculados sobre el resultado filtrado.
  - **Una sola query** a `/api/v1/resumen-territorial` — las líneas de Privada ya vienen en el
    payload (el backend las trajo, §6.3). El frontend no llama a `/api/v1/privada/*`.
  - Toggle **unidad**: "Por localidad" (una fila por localidad, una línea por
    programa/gestión: label · badge de `estado_general` · pill "N de M faltan" — o "—" para
    Privada · fecha+área de última comunicación) y "Por departamento" (rollup client-side del
    mismo payload).
  - Filtros client-side (búsqueda, departamento, área [vivienda/privada], programa, estado,
    estado de checklist) — mismo patrón `useMemo` que `CordonCunetaPage.tsx`.
  - Badges de estado con `style={{background: estado_general_bg, color: estado_general_text_color}}`
    (mismo criterio que `EstadoBadge` en los paneles).
  - Línea de Privada: badge de estado derivado, `detalle` con el breakdown de `privada_conteos`,
    checklist mostrado como "—".
  - **Ficha de localidad** (drawer derecho): estructura de `DetailPanel` de
    `CordonCunetaPage.tsx`. Líneas de Vivienda: sub-estados jurídico/técnico/financiero, lista
    de ítems faltantes del checklist, última comunicación con texto (si visible). Línea de
    Privada: `privada_conteos.por_estado` + link "Ver en el panel de Privada"
    (`/privada/gestiones?departamento=…&localidad=…`).
- **Routing / navegación / dashboard**:
  - `src/App.tsx`: ruta `resumen-territorial` envuelta en
    `<ProtectedRoute roles={['Admin','Autoridad','Supervisor','Operador','Consulta','TecnicoDGV']}>`.
  - `src/shared/components/Layout.tsx`: link top-level (junto a "Inicio", siempre visible para
    usuario con perfil de portal) a `/resumen-territorial`.
  - `src/pages/DashboardPage.tsx`: card destacada full-width arriba de la grilla, visible
    cuando hay `portalUser`.
  - `src/shared/hooks/usePortalUser.ts`: `'Autoridad'` en la unión `PortalUser['rol']`.
  - `src/modules/admin/pages/AdminUsuariosPage.tsx`: `Autoridad` en el `<select>` de roles.

## 10. Impresión y export

- **Excel**: botón que arma una fila por (localidad × programa/gestión) y llama
  `exportToXlsx(rows, 'Resumen territorial', 'resumen_territorial_<fecha>.xlsx')`
  (`src/shared/utils/exportTable.ts`, ya existe). Columnas: Localidad, Departamento, Área
  (Vivienda/Privada), Programa, Detalle, Estado general, Checklist faltantes, Checklist total,
  Última comunicación (fecha), Última comunicación (área), Monto, Expediente. En las filas de
  Privada, Checklist/Monto/Expediente van vacíos y Detalle lleva el breakdown de gestiones.
- **Impresión**: la página renderiza un `<div class="rt-print-doc" hidden print:block>` con
  encabezado (título + alcance + filtros aplicados + fecha de generación) y una tabla plana.
  Botón "Imprimir" = `window.print()`. Se agrega a `src/index.css` un bloque `@media print`:
  oculta el `<header>` del `Layout` y todo salvo `.rt-print-doc`, quita paddings de `<main>`,
  `thead { display: table-header-group }`, `tr { break-inside: avoid }`,
  `@page { size: A4 portrait; margin: 14mm }`. Es el primer `@media print` del proyecto — no
  hay estilos de impresión previos que respetar.

## 11. Infraestructura

- **Gateway** (`infra/gateway/openapi.yaml`): agregar `/api/v1/resumen-territorial` (`get` +
  `options`) y `/api/v1/resumen-territorial/actualizar` (`post` + `options`), `x-google-backend`
  → `https://svc-vivienda-iwni7vc2qq-rj.a.run.app`, `path_translation:
  APPEND_PATH_TO_ADDRESS`, `jwt_audience` copiado de un path vecino de vivienda. Nueva config
  inmutable `ministerio-config-v{YYYYMMDD}` + `gcloud api-gateway gateways update` manual
  (el `deploy-gateway.sh` está stale — seguir los comandos del skill `/deploy-servicio`).
  **Coordinación (aprobado 2026-08-28)**: la config activa sigue en
  `ministerio-config-v20260716b`; la nueva config **sube junto** los paths de
  `checklist-tecnico` y `/informe` que ya están commiteados en `openapi.yaml` pero nunca se
  deployaron.
- **Egress svc-vivienda → API Gateway** (para las líneas de Privada, §3.3): la SA de
  `svc-vivienda` mintea un ID token de Google. No requiere IAM nuevo si el endpoint
  `/api/v1/privada/gestiones/resumen-territorial` acepta cualquier token válido (rol "Todos");
  confirmar el `audience` esperado por el `securityDefinition` `google_accounts` del gateway y
  fijarlo en `settings.gateway_base_url` / la lógica de `_fetch_privada_lineas`.
- **Cloud Scheduler**: job (OIDC) → `POST <cloud-run-url>/internal/resumen-territorial/actualizar`,
  cada 30–60 min. `svc-vivienda@…` ya tiene `roles/run.invoker` sobre sí mismo (lo puso el sync
  de checklist CC). Comandos concretos en `infra/deploy-resumen-territorial.md`.

## 12. Criterios de aceptación

- [ ] Migración aplicada: `viv_resumen_territorial_snapshot` + índice `(computed_at DESC)`.
- [ ] `POST /api/v1/resumen-territorial/actualizar` calcula y guarda una fila nueva; el `GET`
      devuelve la última; `GET` sin snapshot previo devuelve `null` (no error).
- [ ] El payload incluye, por cada localidad con programa activo en CC/CH/ML, una línea por
      programa con `estado_general` (label + colores), `checklist_faltan`/`checklist_total`,
      `checklist_iniciado`, y `ultima_comunicacion` (o `null`).
- [ ] Mi Lugar: una localidad con 2 proyectos activos aparece con 2 líneas, cada una con su
      `detalle` (nombre de proyecto).
- [ ] Localidad cuyo checklist nunca se abrió: `checklist_iniciado = false`,
      `checklist_faltan = checklist_total`, `checklist_faltantes = []`.
- [ ] Usuario `Autoridad`: `GET` devuelve todas las localidades y todas las líneas, con
      `ultima_comunicacion.texto` visible.
- [ ] Cuando `_fetch_privada_lineas()` responde OK: cada localidad con gestiones de Privada
      tiene una línea `area:"privada"` con `privada_conteos`, `estado_general_label`/colores
      derivados y `programa_label = "Gestiones — Sec. Privada"`.
- [ ] Cuando el fetch de Privada falla: el snapshot se guarda igual, `generado_para_areas`
      no incluye `"privada"`, y el `GET` no rompe.
- [ ] Usuario `Autoridad`: ve líneas de Vivienda **y** de Privada en cada localidad.
- [ ] Usuario `Operador` con `secretarias = ["vivienda"]`: ve sólo las líneas de Vivienda (no
      la de Privada); `texto` de comunicaciones de `infraestructura`/`supervision` enmascarado.
- [ ] Usuario `Consulta` con `secretarias = ["privada"]`: ve sólo la línea de Privada por
      localidad; ninguna línea de Vivienda.
- [ ] Usuario `Operador` con `secretarias = ["gasifera"]`: `GET` devuelve `localidades: []`.
- [ ] Usuario `invitado`: `403` en el `GET`; en el frontend la ruta redirige a `/`.
- [ ] `Autoridad` es valor válido en `POST/PUT /api/v1/portal/admin/usuarios` y aparece en el
      `<select>` de `AdminUsuariosPage`.
- [ ] Export a Excel: una fila por (localidad × programa) con las columnas de §10.
- [ ] `Ctrl+P` sobre `/resumen-territorial` produce un documento A4 vertical con encabezado,
      filtros aplicados y la tabla, sin el nav ni el header del `Layout`, con el `thead`
      repetido por página.
- [ ] `pytest` en `services/svc-vivienda/` verde, incluido `tests/test_resumen_territorial.py`
      (unitarios de `aggregations.py` + `filtrar_por_visibilidad` + un test de servicio con
      SQLite in-memory que siembra 1 CC + 1 CH + 1 ML y **mockea `_fetch_privada_lineas`**,
      con un caso OK y un caso que devuelve `[]`).
- [ ] `npm run build` sin errores con la página, la ruta, el nav y la card del dashboard.
- [ ] Regresión: `app/informes/`, `app/checklist_tecnico/` y sus tests siguen en verde.

## 13. Fase 2 (fuera de este spec)

- **Infraestructura / Gasífera / Territorial / Desarrollo**: cuando exista su microservicio, el
  job de cálculo trae sus datos vía su endpoint equivalente, o vía BigQuery
  `ministerio_ejecutivo` si para entonces el pipeline de eventos está construido. El contrato
  de `ResumenTerritorialPayload` y la página no cambian — sólo cambia
  `compute_resumen_territorial` / la fuente.
- **Privada — detalle por gestión**: si svc-privada expone un endpoint por caso apto (estado,
  expediente y última comunicación por gestión), reemplazar la línea roll-up por una línea por
  gestión (como Mi Lugar), con `entidad_id` real y link al panel de Privada.
- **Mapa** por departamento y por localidad (reusa `CoropletiqueDepartamentos` /
  `MapaDualPuntos` + `frontend/public/geo/`).
- **Última comunicación visible por alcance** (resolver la limitación de §7.2).
- **Rollup de `estado_general` a nivel departamento** en el payload (hoy el rollup por
  departamento es 100% client-side).
