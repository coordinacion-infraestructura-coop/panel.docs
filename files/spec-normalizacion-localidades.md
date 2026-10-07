# Spec: Normalización de nombres de localidad (transversal)

**Estado**: approved
**Versión**: 0.7.0
**Servicios**: `svc-vivienda` (padrón + endpoint interno de resolución + fixes de
matching), `svc-gasifera` (sync), `svc-gralgob` (sync), `svc-privada` (rollup
territorial), `frontend` (retira el matching hardcodeado de `AtpPage.tsx`)
**Última actualización**: 2026-10-07

---

## Changelog

- **0.7.0** (2026-10-07): **cambia §2.6.** Hasta ahora el texto de localidad
  de cada registro de Cordón Cuneta / Córdoba Hogar / Mi Lugar no se pisaba
  (se resolvía `localidad_id` al lado). Por decisión del usuario (ADR-026),
  cuando hay vínculo lo que se guarda es **el nombre y el departamento del
  padrón oficial**; el texto que llegó sólo se conserva cuando la localidad no
  está en el padrón. Sigue sin bloquearse el alta de una localidad que no
  matchea. Aplica al alta y la edición (`cordon_cuneta` / `cordoba_hogar` /
  `mi_lugar` `service.py`) y a lo ya cargado (migración `svc-vivienda` 0037:
  36 nombres y 3 departamentos, con rastro en `viv_audit_log`).
- **0.6.0** (2026-10-07): criterio general fijado por el usuario y registrado
  como **ADR-026** — Vivienda, Privada, Gral. de Gobierno, Gasífera y Resumen
  Territorial usan la misma base oficial de localidades. Privada deja de tener
  padrón propio (`priv_geo_localidades` pasa a ser espejo de solo lectura; ver
  `spec-privada-padron-oficial.md`). Dos salvedades que quedan escritas acá
  porque hasta ahora sólo existían como comportamiento: (1) **Gasífera y Gral.
  de Gobierno** cargan desde planillas externas que escriben algunos nombres
  distinto — se resuelven con el mapeo previo a la carga (`viv_geo_alias_manual`
  + resolución en sync-time), y el texto de la planilla se conserva; (2) **Mi
  Lugar** tiene proyectos que son loteos nombrados por barrio de Córdoba
  Capital ("Barrio Chingolo", "Santa Teresa") — no matchean contra el padrón y
  es correcto: se registran como "confirmado sin vínculo" (alias con
  `id_geo = NULL`, migración `svc-vivienda` 0036, igual que "Santiago Temple")
  para que no generen el aviso de §4.11. Pendientes anotados en
  `spec-privada-padron-oficial.md §7`: el panel de asignación manual de §9 (el
  usuario lo quiere para lo nuevo que no resuelva) y pasar al nombre oficial
  el texto guardado en CC/CH/ML (hoy §2.6 dice que no se pisa).
- **0.5.0** (2026-10-07): grafía única del padrón. De 544 filas de
  `viv_geo_localidades`, 30 tenían el nombre entero en minúsculas ("Villa de
  Pocho", casi todas las altas posteriores, id 533–560) y 24 el alias entre
  paréntesis en minúsculas, varias con una "O"/"I" en lugar de la vocal
  acentuada ("CHARRAS (Villa ColOn)"). Reportado por el usuario al filtrar el
  Resumen Territorial por departamento Pocho. Decisión del usuario: el nombre
  que se muestra tiene que ser el del padrón oficial, así que se corrige el
  padrón y no la presentación — migración `svc-vivienda` 0035 pasa `localidad`
  a mayúsculas (en Python, no con `UPPER()` de SQL, por la collation). No
  cambia ningún `id_geo` ni ningún matching, y no toca el texto libre de
  CC/CH/ML (§2.6 sigue vigente). De paso, en frontend: los desplegables de
  localidad de los modales de CC/CH/ML resolvían la opción elegida por
  igualdad exacta de texto contra el padrón y pasan a comparar sin
  tildes/mayúsculas, y la Ficha de Localidad deja de filtrar Mi Lugar por
  nombre exacto en el servidor. **Pendiente**: correr la migración 0035 en
  producción y recalcular el snapshot de `resumen_territorial`.
- **0.4.0** (2026-09-29): deploy real completado (las 3 migraciones —
  `svc-vivienda` 0030, `svc-gasifera` 0003, `svc-gralgob` 0002 — corridas
  contra Cloud SQL tras una interrupción por facturación/suspensión de la
  instancia, resuelta por el usuario; los 3 servicios redeployados; gateway
  actualizado con `GET /api/v1/vivienda/geo/duplicados`). Verificación en
  producción de Resumen Territorial encontró KPIs inflados (708 localidades
  en vez de ~426, 30 departamentos en vez de 26) por variantes de
  departamento no cubiertas por `normalize_name` (ej. "GENERAL" vs "Gral",
  "PRESIDENTE" vs "PTE") — se agrega `normalize_departamento()` y se usa
  también para desambiguar homónimos en `_elegir()` (migración 0031, además
  de 2 alias puntuales: General Baldissera, Plaza Luxardo). Investigación
  caso por caso de los ~30 residuales restantes (pedida explícitamente por
  el usuario, corrida con el resolver real de producción, no una heurística
  offline): 6 resultaron typos/variantes con alias agregado (migración
  0032 — Eufrasio Loza, Brinckmann, Capilla de Sitón, Estación General Paz,
  Villa Quilino, Chuña/carácter espurio), 7 resultaron ser **duplicados
  dentro del propio padrón** (`viv_geo_localidades` con dos `id_geo` para la
  misma localidad real, confirmados por el usuario y desactivados con
  `activo=false` en migración 0033 — nunca borrados, para no romper
  referencias históricas) y 2 quedan genuinamente sin resolver (localidades
  reales ausentes del padrón, mismo criterio que "Santiago Temple"). KPI
  final tras las 4 migraciones: 413 localidades / 26 departamentos. Además,
  a pedido explícito del usuario ("cerremos el gap de Mi Lugar ahora, y
  armemos notificaciones"): (a) se extiende la resolución in-process de
  §4.9 a Mi Lugar, que hasta ahora aceptaba `localidad_id` sin validar desde
  el cliente (cierra el último módulo sin resolución automática — ver
  §4.10); (b) se agrega un mecanismo de notificación + log para
  localidades que no resuelven (`match_tipo = "sin_match"`), centralizado en
  el propio resolver para cubrir las 5 fuentes con un solo cambio — ver
  §4.11. Explícitamente **no** se construye un frontend de asignación
  manual en esta entrega (ver §9).
- **0.3.0** (2026-09-28, ADR-024): el usuario señaló que "localidad +
  departamento" es la unidad de análisis central de toda la plataforma (la
  razón de ser del sistema es centralizar información de distintas áreas a
  ese nivel), y pidió evaluar una solución robusta a futuro sin caer en
  sobre-ingeniería. Se decide **extender la resolución de `id_geo` a Cordón
  Cuneta/Córdoba Hogar también** (no solo Gasífera/Gralgob) — es casi gratis
  porque corren en el mismo servicio que el padrón, sin llamada de red — y
  usar `id_geo` como **llave de join primaria en `resumen_territorial`**
  (con fallback a texto normalizado para lo que quede sin resolver). No se
  crea ningún servicio nuevo — sigue viviendo en `db_vivienda`. Ver §2-§6 y
  ADR-024 en `arquitectura.md` (supersede parcialmente ADR-021/ADR-022 en el
  punto de "`id_geo` no se usa como llave de join").
- **0.2.0** (2026-09-28): resuelve las dos preguntas abiertas sobre el
  reporte de duplicados (§4.4/§10) — confirmado con el usuario: solo JSON
  por ahora (sin pantalla), y la fusión de duplicados confirmados se hace a
  mano vía la UI existente de CC/CH/ML (sin endpoint de fusión asistida).
  Pasa a `review`, pendiente de aprobación explícita antes de implementar.
- **0.1.0** (2026-09-28): borrador inicial, a partir de un pedido del usuario
  de corregir datos repetidos a nivel localidad en Cordón Cuneta, Córdoba
  Hogar, Mi Lugar, Gasífera, Gralgob y Resumen Territorial, y de agregar un
  paso de normalización en la ingesta de los Sheets de Gasífera/Gralgob.

## 0. Por qué existe este spec

Seis áreas del sistema (`cordon_cuneta`, `cordoba_hogar`, `mi_lugar`,
`svc-gasifera`, `svc-gralgob`, `resumen_territorial`) guardan o agregan datos
por localidad, y hoy lo hacen con **al menos 4 criterios de normalización de
texto distintos** conviviendo sin coordinación (ver §1). El síntoma que
reportó el usuario — la misma localidad real apareciendo dos veces con
grafías distintas — es consecuencia directa de esa fragmentación, no de un
bug puntual en un módulo. Por eso el fix correcto es transversal: un único
mecanismo de resolución contra el padrón oficial, reutilizado por los seis
puntos en vez de que cada uno seguure normalizando por su cuenta.

Este spec es angosto a propósito (mismo criterio que usaron
`spec-sync-gasifera-pit.md`/`spec-sync-atp-compromiso-gobernador.md` en su
momento): resuelve el problema de matching/normalización de nombres. **No**
decide la evolución de largo plazo del padrón geográfico hacia un registro
enriquecido con censo/catastro/economía que el usuario mencionó como
objetivo — eso se señala como fuera de alcance en §9 y necesita su propio
ADR.

## 1. Problema (evidencia relevada en el código actual)

- **Cuatro criterios de normalización distintos conviven**:
  `app/geo/matching.py::normalize_name` (saca tildes + minúsculas, Python,
  svc-vivienda) y su réplica en `frontend/src/shared/utils/normalizeName.ts`;
  `func.lower()` sin sacar tildes en el chequeo de duplicados al crear un
  municipio de Cordón Cuneta (`cordon_cuneta/service.py:216-217,257-258`);
  `func.upper(func.trim())` sin sacar tildes en los rollups territoriales de
  Gasífera y Gralgob (`gas_pit/rollup.py:30-31`, `atp/rollup.py` análogo).
- **Cordón Cuneta y Córdoba Hogar guardan la localidad como texto libre**, sin
  FK a ningún padrón (`cordon_cuneta/models.py`, `cordoba_hogar/models.py`).
  Mi Lugar tiene FK (`mi_lugar/models.py:31`, `localidad_id → viv_geo_localidades.id_geo`,
  nullable) pero además duplica el nombre en texto
  (`localidad_nombre`), y nada obliga a que ese texto coincida con el `id_geo`.
- **Gasífera y Gralgob no cruzan contra ningún padrón en el backend.** El
  único cruce que existe hoy es 100% frontend y hardcodeado:
  `frontend/src/modules/gralgob/pages/AtpPage.tsx`, con una constante
  `VINCULACION_MANUAL` (17 entradas resueltas a mano, documentadas en
  `spec-sync-atp-compromiso-gobernador.md §12.9`, más 1 caso adicional
  investigado y confirmado ausente del padrón ("Santiago Temple", no incluido
  en el mapa a propósito — 18 casos investigados en total). Cubre solo ATP; no
  existe nada equivalente para Gasífera, y no persiste el resultado en la
  base (`atp_compromisos` sigue el texto crudo del Sheet).
- **El agregador de `resumen_territorial`
  (`aggregations.py::agrupar_por_localidad`) usa solo `normalize_name`**, no
  `candidatos_localidad()` (que resuelve alias entre paréntesis o separados
  por guion) — a diferencia de `informes/aggregations.py` y
  `informe_localidades/service.py`, que sí lo usan. Una localidad con alias en
  una fuente (ej. Gasífera concatenando `"TANTI - EL DURAZNO"`) puede no
  calzar con el padrón aunque el helper para resolverlo ya exista en el
  mismo servicio.
- **Hay dos padrones geográficos independientes y desincronizados**:
  `viv_geo_localidades` (svc-vivienda, `id_geo` String(20)) y
  `priv_geo_localidades` (svc-privada, `id_geo` String(30), columnas
  renombradas `lat`/`lon`; ADR-012 documenta que la capa demográfica/electoral
  enriquecida —`priv_localidades_info`/`priv_departamentos_info`— es
  propiedad de svc-privada). No hay proceso de sync entre ambos.

## 2. Alcance

### Incluido

1. Un endpoint interno IAM-only nuevo en `svc-vivienda`,
   `POST /internal/geo/resolver-localidades`, que resuelve en batch
   `(departamento, localidad)` crudos → `id_geo` del padrón oficial, con el
   mismo criterio de matching que ya usa `geo/matching.py`
   (`normalize_name` + `candidatos_localidad`) más una tabla nueva de
   vinculación manual (§4.2) que reemplaza y centraliza lo que hoy vive
   hardcodeado en `AtpPage.tsx`.
2. `svc-gasifera` y `svc-gralgob` llaman a ese endpoint durante el sync
   (después de leer el Sheet, antes de persistir) y guardan el `id_geo`
   resuelto junto al texto crudo — el texto crudo del Sheet **no se pierde ni
   se sobrescribe**, se agrega el `id_geo` como dato adicional.
3. Migración de las 17 vinculaciones manuales ya investigadas en
   `spec-sync-atp-compromiso-gobernador.md §12.9` a la tabla nueva de
   vinculación manual (seed data), más el registro de "Santiago Temple" como
   conocido-pero-sin-`id_geo`.
4. `frontend/AtpPage.tsx` deja de recalcular el matching client-side
   (`VINCULACION_MANUAL`, `candidatosLocalidad()`, `normalizeDepartamento()`)
   y en cambio consume el `id_geo`/`match_tipo` que ya vienen resueltos desde
   el backend.
5. Un endpoint de **detección de duplicados existentes** en Cordón Cuneta,
   Córdoba Hogar y Mi Lugar — agrupa registros por nombre normalizado y
   devuelve los grupos con más de una grafía distinta, para revisión manual.
   **No hace merge automático** (decisión confirmada con el usuario, §3).
6. **Cordón Cuneta y Córdoba Hogar resuelven y persisten `id_geo` en el
   momento de crear/editar un registro** (además de detectar duplicados
   existentes, ítem 5) — como FK real (misma base que `viv_geo_localidades`,
   sin llamada de red: se resuelve en proceso, no vía HTTP). Mismo mecanismo
   de matching que §4.3, sin exponer el endpoint interno para esto (es
   in-process). El campo de texto libre existente **no se toca ni se
   bloquea** *(revisado en v0.7.0: no se bloquea, pero con vínculo se guarda
   el nombre oficial — ver changelog)* — `id_geo` es un dato adicional resuelto automáticamente, no un
   requisito para guardar. Se agrega también un backfill best-effort de
   `id_geo` para las filas activas ya existentes en CC/CH/ML (ver §4.9).
7. `resumen_territorial/aggregations.py::agrupar_por_localidad` pasa a
   agrupar **primero por `id_geo`** cuando ambos lados de la comparación lo
   tienen resuelto (Vivienda vía ítem 6, Gasífera/ATP vía ítem 2), con
   **fallback a `candidatos_localidad()`** (en vez de solo `normalize_name`)
   para las líneas que queden sin `id_geo` resuelto — ver §4.6. Privada sigue
   matcheando solo por texto (padrón independiente, ADR-012) hasta que se
   resuelva §9.
8. Fix del chequeo de duplicado al crear un municipio de Cordón Cuneta
   (`cordon_cuneta/service.py:216-217,257-258`) para usar
   `geo.matching.normalize_name` en vez de `func.lower()` crudo (deja de
   pasar por alto duplicados que solo difieren en tildes).

### Excluido (de esta entrega)

- Unificar `viv_geo_localidades` con `priv_geo_localidades`/
  `priv_localidades_info` o construir un registro enriquecido con
  censo/catastro/economía — es la ambición de largo plazo que mencionó el
  usuario, pero es una decisión de arquitectura mayor (toca ADR-012) que
  merece su propio ADR y spec. Ver §9.
- **Bloquear** la escritura de texto libre en Cordón Cuneta/Córdoba Hogar
  (obligar a elegir de un dropdown, rechazar si no matchea el padrón). El
  ítem 6 de arriba agrega `id_geo` resuelto en paralelo, pero nunca impide
  guardar un registro cuya localidad no está en el padrón (mismo criterio
  tolerante que Gasífera/Gralgob) — decisión sin cambios respecto a la
  ronda anterior.
- Migrar `gas_pit_obras`/`gas_pit_obras_localidades` (nivel obra) al mismo
  mecanismo de resolución — el rollup territorial de Gasífera usa
  `gas_pit_acciones_territorio`, no `gas_pit_obras` (`rollup.py:26-29`); si
  hace falta resolver `id_geo` también a nivel obra queda para una iteración
  posterior si se confirma que hace falta.
- Pantalla frontend de administración de la tabla de vinculación manual — en
  esta entrega se administra por migración/consulta directa. Ver pregunta
  abierta en §10.

## 3. Decisiones de arquitectura confirmadas con el usuario

1. **Padrón canónico para esta spec**: se reutiliza `viv_geo_localidades`
   (svc-vivienda) tal cual existe hoy — no se crea una tabla nueva de cero.
   Es ya la fuente que usan el FK de Mi Lugar, los dropdowns de Cordón
   Cuneta/Córdoba Hogar y el agregador de `resumen_territorial`. La ambición
   de convertirlo en un registro que vaya sumando información oficial de
   fuentes externas (censo, catastro, economía) queda para una decisión de
   arquitectura aparte (§9) — no se resuelve en esta spec.
2. **Duplicados ya existentes en producción** (Cordón Cuneta/Córdoba
   Hogar/Mi Lugar): se detectan y se listan para revisión manual. No hay
   merge automático — el riesgo de fusionar dos localidades reales distintas
   que casualmente normalizan igual no se asume sin que un humano lo
   confirme fila por fila.
3. **Gasífera y Gralgob resuelven `id_geo` en sync-time** llamando a un
   endpoint interno IAM-only nuevo en `svc-vivienda` — mismo patrón
   arquitectónico ya usado para todo lo `internal/**` (ADR-015/016/021/022):
   un solo lugar de verdad para el matching + los overrides manuales, en vez
   de que cada servicio mantenga su propia copia divergente del algoritmo
   (que es exactamente el problema que hoy tiene `AtpPage.tsx` vs.
   `geo/matching.py`).
4. **Reporte de duplicados de CC/CH/ML (§4.4): solo respuesta JSON en esta
   entrega**, sin pantalla frontend — si el uso resulta recurrente, se agrega
   UI en una iteración posterior.
5. **Aplicación de la corrección cuando el reporte confirma un duplicado
   real**: edición manual vía la UI ya existente de CC/CH/ML (cada registro
   se corrige a mano para usar la misma grafía). No se construye un
   endpoint/script de fusión asistida en esta entrega.
6. **`id_geo` pasa a ser la llave de join primaria en `resumen_territorial`**
   (con fallback a texto normalizado) — extiende el alcance original de la
   spec (que solo tenía a Gasífera/Gralgob resolviendo `id_geo`) a Cordón
   Cuneta/Córdoba Hogar también, a pedido del usuario, que señaló que
   "localidad + departamento" es la unidad de análisis central de toda la
   plataforma. Registrado como ADR-024 (supersede parcialmente ADR-021/022
   en el punto puntual de que `id_geo` no se usaba como llave de join).
   Ver §0.3.0 del changelog.
7. **No se crea ningún servicio nuevo para esto** — la resolución en CC/CH es
   in-process (mismo servicio que el padrón), sin llamada de red; se
   descartó explícitamente construir un `svc-geo` u otro servicio dedicado
   por ahora (sobreingeniería a esta escala — ver §9 para el criterio de
   cuándo sí evaluarlo).

## 4. Diseño

### 4.1 Padrón canónico reutilizado

Sin cambios de schema en `viv_geo_localidades` (`app/geo/models.py`). Sigue
siendo `id_geo` (PK), `departamento`, `localidad`, `lat_centro`/`lon_centro`,
`activo`.

### 4.2 Tabla nueva: vinculación manual (`viv_geo_alias_manual`)

Reemplaza y centraliza lo que hoy es `VINCULACION_MANUAL` en `AtpPage.tsx`
(hardcodeado, solo frontend, solo ATP). Vive en `db_vivienda`, migración
Alembic nueva en `svc-vivienda`.

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `String(36)` PK | uuid |
| `texto_normalizado` | `String(200)`, UNIQUE | `normalize_name()` del texto crudo tal cual aparece en la fuente (ej. Sheet) |
| `texto_original` | `String(200)` | el texto crudo tal cual se vio, para trazabilidad/auditoría |
| `id_geo` | `String(20)`, nullable, FK a `viv_geo_localidades.id_geo` | `NULL` = "conocido pero ausente del padrón" (caso Santiago Temple) — nunca se inventa un `id_geo` |
| `motivo` | `Text` | por qué se vincula así (typo, abreviatura, límite departamental mal cargado, etc.) — mismo nivel de detalle que la tabla de `spec-sync-atp-compromiso-gobernador.md §12.9` |
| `origen` | `String(50)` | qué sync/módulo aportó la entrada (`atp`, `gas_pit`, `cordon_cuneta`, …) — informativo, no filtra el matching |
| `created_at`, `created_by` | | `created_by` = email o `"migracion-spec-normalizacion-localidades"` para las 18 filas seed |

Seed: las 17 filas ya confirmadas en `spec-sync-atp-compromiso-gobernador.md
§12.9` (Paso del Durazno → Río Cuarto id 443, Nicolás Bruzzone → id 55, …)
más una fila para "Santiago Temple" con `id_geo = NULL` y `motivo` explicando
que está confirmada como ausente del padrón (18 filas en total — verificado
contra `frontend/AtpPage.tsx`'s `VINCULACION_MANUAL`, que tiene exactamente
17 entradas).

### 4.3 Endpoint interno de resolución batch

```
POST /internal/geo/resolver-localidades
```

Agregado a `app/internal/router.py` (mismo archivo que ya centraliza todos
los `internal/**` de svc-vivienda — no se crea un router nuevo). IAM-only,
sin `Depends(get_current_user)`, no declarado en `infra/gateway/openapi.yaml`
(mismo criterio que el resto de ese router).

Body:
```json
{"items": [{"departamento": "Juárez Celman", "localidad": "Paso del Durazno"}, ...]}
```

Respuesta (mismo orden que el input):
```json
{"resultados": [
  {"departamento_in": "Juárez Celman", "localidad_in": "Paso del Durazno",
   "id_geo": "443", "departamento_oficial": "Río Cuarto",
   "localidad_oficial": "Paso del Durazno", "match_tipo": "manual"}
]}
```

`match_tipo`: `"manual"` (encontrado en `viv_geo_alias_manual` por
`texto_normalizado` de la localidad) → `"exacto"` (`normalize_name` calza
directo contra el padrón) → `"alias"` (calza vía `candidatos_localidad`,
paréntesis/guion) → `"sin_match"` (`id_geo = null`). Se prueba en ese orden;
el primero que matchea gana. Pensado para llamarse una vez por corrida de
sync con todas las filas del batch (cientos), no una llamada HTTP por fila.

### 4.4 Endpoint de detección de duplicados existentes

```
GET /api/v1/vivienda/geo/duplicados?modulo=cordon_cuneta|cordoba_hogar|mi_lugar
```

Nuevo `app/geo/router.py` + lógica en `app/geo/service.py` (el paquete
`app/geo/` hoy solo tiene `models.py`/`matching.py`, sin router — se agrega
siguiendo la convención `router.py`/`service.py` del resto del proyecto).
Gateway: nuevo path, requiere `options:` + config nueva (§6). Gatea a
`Admin`/`Supervisor` (reutiliza las tuplas compartidas de `app/auth.py`, no
hace falta un rol acotado nuevo).

Agrupa las filas activas (`deleted_at IS NULL`) del módulo pedido por
`normalize_name(localidad/municipio)`, devuelve solo los grupos con más de
una grafía cruda distinta:

```json
{"grupos": [
  {"normalizado": "villa dolores", "variantes": [
    {"texto": "Villa Dolores", "cantidad_filas": 12},
    {"texto": "VILLA DOLORES ", "cantidad_filas": 1}
  ]}
]}
```

Solo lectura — no fusiona ni borra nada. La resolución (cuál grafía es la
correcta, si corresponde fusionar) queda en manos de quien revisa (ver
pregunta abierta §10 sobre si hace falta pantalla o alcanza con esto).

### 4.5 Sync de Gasífera y Gralgob: nueva columna + llamada al endpoint

**`svc-gasifera`** (`app/gas_pit/models.py`): agrega `id_geo: str | None` y
`match_tipo: str | None` a `GasPitAccionTerritorio` (departamento/localidad a
nivel de esa tabla) y a `GasPitObraLocalidad` (localidad a nivel de obra,
donde el Sheet trae nombres concatenados con guion — ya se separan en
`_split_localidades`, `sync.py:137-140`, cada localidad separada se resuelve
individualmente). Sin FK real cross-DB (mismo criterio que el resto del
proyecto con referencias cross-servicio) — `id_geo` es texto, no
`ForeignKey`.

**`svc-gralgob`** (`app/atp/models.py`): agrega `id_geo: str | None` y
`match_tipo: str | None` a `AtpCompromiso`.

En ambos `sync.py`: después de leer y parsear el Sheet, junta todas las
filas nuevas/actualizadas en un batch y llama
`POST {SVC_VIVIENDA_INTERNAL_URL}/internal/geo/resolver-localidades` (mismo
cliente HTTP + audiencia + manejo de fallos *best-effort* que ya usa
`notificaciones_vivienda.py` — si la llamada falla, el sync **no se aborta**:
persiste las filas con `id_geo = NULL`/`match_tipo = NULL` y loguea el fallo,
para no bloquear la sincronización del Sheet por una falla de red
transitoria hacia svc-vivienda). Mismo IAM ya otorgado por ADR-015/ADR-023
(`svc-gasifera@`/`svc-gralgob@` ya tienen `run.invoker` sobre `svc-vivienda`
— no hace falta un grant nuevo).

### 4.6 Fix `resumen_territorial/aggregations.py::agrupar_por_localidad` — `id_geo` como llave primaria (ADR-024)

Cambia la clave de agrupación en dos niveles, en este orden de prioridad:

1. **`id_geo`**, si la línea lo trae resuelto (Vivienda vía §4.9, Gasífera/ATP
   vía §4.5). Dos líneas con el mismo `id_geo` se agrupan aunque su texto
   crudo sea distinto — ya no depende de que el matching de texto sea
   exhaustivo, porque la resolución ya ocurrió río arriba (en la escritura o
   en el sync).
2. **Fallback a texto normalizado** para las líneas sin `id_geo` (legado no
   resuelto, o Privada — que sigue con su propio padrón independiente,
   ADR-012, y no manda `id_geo` en este join). Acá se reemplaza el uso de
   `normalize_name` solo por `candidatos_localidad()` — mismo criterio que ya
   usan `informes/aggregations.py` e `informe_localidades/service.py` —
   probando cada candidato contra el padrón (`geo_full`) antes de caer al
   `normalize_name(nombre_localidad)` crudo tal cual hoy.

El nombre de display sigue priorizando la grafía del padrón `viv_geo_localidades`
(sin cambios en ese punto).

### 4.7 Fix dedup de Cordón Cuneta

`cordon_cuneta/service.py:216-217,257-258`: reemplaza
`func.lower(MunicipioCordonCuneta.municipio) == data.municipio.strip().lower()`
por una comparación usando `geo.matching.normalize_name` (aplicada en Python
sobre los candidatos, ya que `strip_accents` no tiene equivalente directo en
SQL sin una extensión como `unaccent`) — evalúa si conviene traer los
candidatos por `departamento` normalizado con SQL y filtrar el resto en
Python, o instalar `unaccent` en Postgres; se decide en implementación según
qué tan grande es el universo de municipios a comparar (54 municipios reales
— probablemente alcanza con traer todos los activos del mismo departamento y
comparar en Python).

### 4.8 Frontend: retirar el matching hardcodeado de `AtpPage.tsx`

Una vez que `atp_compromisos` persiste `id_geo`/`match_tipo` resueltos por el
backend (§4.5), `AtpPage.tsx` deja de recalcular `VINCULACION_MANUAL` +
`candidatosLocalidad()` + `normalizeDepartamento()` client-side y consume
directamente los campos ya resueltos que devuelve
`GET /api/v1/gralgob/compromisos`. Elimina la duplicación de lógica de
matching entre frontend y backend que hoy existe (`normalizeName.ts` sigue
existiendo para otros usos — matching contra el GeoJSON de departamentos en
mapas — pero deja de reimplementar `candidatos_localidad`).

### 4.9 Cordón Cuneta y Córdoba Hogar: resolución in-process + backfill (ADR-024)

**Nuevas columnas** en `MunicipioCordonCuneta` (`viv_cordon_cuneta`) y
`LocalidadCordobaHogar` (`viv_cordoba_hogar`): `localidad_id: String(20) |
None` (FK real a `viv_geo_localidades.id_geo` — misma base, a diferencia de
Gasífera/Gralgob que son cross-DB y guardan texto) + `localidad_match_tipo:
String(20) | None`. Mismo naming que ya usa Mi Lugar
(`localidad_id`/`localidad_nombre`) para consistencia entre los tres
módulos.

**En la escritura** (`crear_municipio`/`actualizar_municipio` de
`cordon_cuneta/service.py`, análogos en `cordoba_hogar/service.py`): después
de validar el payload, se llama en proceso a una función nueva
`geo.matching.resolver(db, departamento, localidad, alias_repo)` — misma
lógica que expone `POST /internal/geo/resolver-localidades` (§4.3), factorizada
para no duplicar el algoritmo entre el endpoint HTTP y esta llamada
in-process — y se guarda `localidad_id`/`localidad_match_tipo` junto al resto
de la fila, en la misma transacción. No bloquea el guardado si no hay match
(`localidad_id = NULL`, mismo criterio tolerante que Gasífera/Gralgob).

**Backfill de filas existentes**: la migración que agrega las columnas
incluye un paso de datos (o un script aparte corrido una vez, a definir en
implementación según el volumen) que resuelve `localidad_id` para todas las
filas activas de CC/CH/ML que todavía no lo tienen, reutilizando la misma
función de resolución. Es una operación aditiva y segura — nunca cambia el
texto visible, sólo completa un puntero derivado — a diferencia de la fusión
de duplicados (§4.4), que sigue siendo manual.

### 4.10 Mi Lugar: cierre del gap de resolución in-process (2026-09-29)

Mi Lugar ya tenía las columnas `localidad_id`/`localidad_match_tipo` desde la
migración 0030 (backfill incluido), pero a diferencia de Cordón Cuneta/
Córdoba Hogar, `crear_proyecto_ml`/`actualizar_proyecto_ml` nunca llamaban al
resolver — `localidad_id` se persistía tal cual lo mandaba el cliente
(`data.localidad_id`), sin validar contra el padrón. Se cierra con el mismo
patrón exacto que §4.9: en creación se resuelve siempre server-side
(`geo_service.resolver_uno(db, data.departamento, data.localidad_nombre,
origen="mi_lugar")`) y el `localidad_id` que mande el cliente se ignora — en
edición, sólo se re-resuelve cuando cambia `localidad_nombre` o
`departamento`. No requiere migración nueva (las columnas ya existían); sólo
cambio de código en `app/mi_lugar/service.py` + `localidad_match_tipo` sumado
a `ProyectoMLOut`. Con esto, los 3 módulos de `svc-vivienda` (CC/CH/ML)
quedan con el mismo comportamiento de resolución-al-escribir.

### 4.11 Notificación + log de localidades sin resolver (2026-09-29)

Pedido explícito del usuario: "por el momento dejemos una notificación con
log, no es necesario un frontend de asignación manual aún, con que quede el
log que permita luego corregirlo desde esta sesión es suficiente". Se
descartó construir un panel de asignación manual ahora (§9) a favor de la
opción más simple que deja trazabilidad suficiente para una sesión futura.

**Dónde vive**: centralizado en `app/geo/service.py::resolver_lote` — es el
único punto por el que pasan las 5 fuentes (CC/CH/ML in-process, y
Gasífera/Gralgob/Privada vía `POST /internal/geo/resolver-localidades`), así
que un solo cambio cubre todo sin que cada servicio externo tenga que
construir su propio cliente de notificaciones.

**Diseño**:
- `resolver_lote`/`resolver_uno` ganan un parámetro `origen: str | None`
  (identifica quién llama — `"cordon_cuneta"`, `"cordoba_hogar"`,
  `"mi_lugar"`, `"gas_pit"`, `"atp"`, `"privada"` — sólo para logs/
  notificación, nunca afecta el matching). Cada caller in-process pasa su
  propio nombre de módulo; `ResolverRequest` (el body del endpoint interno)
  suma un campo `origen` opcional que los 3 clientes externos (`app/
  integrations/geo_resolver.py` en Gasífera/Gralgob/Privada) ya completan.
- Tras resolver el lote, si hay una o más filas con `match_tipo ==
  "sin_match"` (y localidad de entrada no vacía), se emite un
  `logger.warning` con el detalle completo y **una sola notificación
  batcheada** (no una por fila) vía el módulo `app/notificaciones/`
  existente (ADR-019, feed de campanita ya en el frontend — no se construye
  UI nueva): `nivel="advertencia"`, `origen="geo_resolver"`,
  `destino_tipo="rol"`, `destino_valor="Admin"`, título con la cantidad y el
  `origen`, mensaje con hasta 10 ejemplos `localidad (departamento)`. Usa un
  actor sintético `_RESOLVER_ACTOR` (mismo patrón que `_SCHEDULER_ACTOR` en
  `app/internal/router.py`, no hay JWT real en este flujo).
- La creación de la notificación está en un `try/except` best-effort: una
  falla ahí nunca debe abortar la creación/edición de la entidad que disparó
  el resolver.

**Limitación aceptada explícitamente** (no dedup): no hay tabla de "ya
notificado" — una localidad persistentemente sin resolver vuelve a generar
notificación en cada sync/alta hasta que alguien le agregue un alias en
`viv_geo_alias_manual`. Es la simplificación que el usuario pidió ("por el
momento... es suficiente") a cambio de no construir un mecanismo de
deduplicación/estado. Si el volumen de notificaciones repetidas resulta
molesto en la práctica, es la primera mejora candidata cuando se retome este
tema (junto con el panel de asignación manual, §9).

## 5. Modelo de datos — resumen de migraciones nuevas

- **`svc-vivienda`**: migración nueva crea `viv_geo_alias_manual` + seed de
  18 filas (§4.2).
- **`svc-gasifera`**: migración nueva agrega `id_geo`/`match_tipo` a
  `gas_pit_acciones_territorio` y `gas_pit_obras_localidades`.
- **`svc-gralgob`**: migración nueva agrega `id_geo`/`match_tipo` a
  `atp_compromisos`.
- **`svc-vivienda`** (adicional, ADR-024): migración nueva agrega
  `localidad_id`/`localidad_match_tipo` a `viv_cordon_cuneta` y
  `viv_cordoba_hogar` (§4.9), más el paso de backfill best-effort sobre
  filas activas existentes en CC/CH/ML.
- **`svc-vivienda`** (0031, 2026-09-28): 2 alias puntuales (General
  Baldissera, Plaza Luxardo) + re-backfill con `normalize_departamento()`
  corregido.
- **`svc-vivienda`** (0032, 2026-09-29): 6 alias de la investigación de
  residuales + re-backfill (§0.4.0 del changelog).
- **`svc-vivienda`** (0033, 2026-09-29): desactiva (`activo=false`, sin
  borrar) 7 filas de `viv_geo_localidades` confirmadas como duplicados
  internos del propio padrón.
- **`svc-vivienda`** (código, sin migración — 2026-09-29): Mi Lugar pasa a
  resolver in-process (§4.10); `resolver_lote`/`resolver_uno` ganan
  `origen` + notificación de `sin_match` (§4.11).

## 6. Endpoints — resumen

```
POST /internal/geo/resolver-localidades          # svc-vivienda, IAM-only, batch (+ origen opcional)
GET  /api/v1/vivienda/geo/duplicados              # svc-vivienda, Admin/Supervisor, vía gateway
```

El segundo requiere path + `options:` nuevos en `infra/gateway/openapi.yaml`
y una config de gateway nueva (`ministerio-config-v{fecha}`) — deploy en
sesión aparte, siguiendo `/deploy-servicio` (mismo criterio que el resto del
proyecto).

## 7. Tests

- `svc-vivienda`: resolución batch — match manual, match exacto, match por
  alias (paréntesis/guion), sin match; endpoint de duplicados — grupo con
  variantes, sin duplicados, filtra por `deleted_at`; fix del dedup de CC
  (accents-insensitive); `crear_municipio`/`crear_localidad_ch` persisten
  `localidad_id` resuelto y no bloquean el alta cuando no hay match;
  backfill resuelve `localidad_id` sobre filas preexistentes sin tocar el
  texto; `agrupar_por_localidad` agrupa por `id_geo` cuando ambas líneas lo
  tienen, y cae a `candidatos_localidad()` cuando no.
- `svc-gasifera`/`svc-gralgob`: sync persiste `id_geo` resuelto; fallo de red
  hacia svc-vivienda no aborta el sync (best-effort); `match_tipo` correcto
  para una fila que solo matchea por la tabla de vinculación manual
  migrada.
- Frontend: `npm run build` en verde tras retirar `VINCULACION_MANUAL`.
- `svc-vivienda` (2026-09-29, `test_mi_lugar_geo.py` nuevo): alta/edición de
  Mi Lugar resuelve y persiste `localidad_id`/`localidad_match_tipo`, ignora
  un `localidad_id` mandado por el cliente, no bloquea el alta sin match, y
  no re-resuelve si la edición no toca `localidad_nombre`/`departamento`.
- `svc-vivienda` (2026-09-29, `test_geo.py`/`test_internal_router.py`): una
  corrida de `resolver_lote` con `sin_match` crea exactamente una
  notificación batcheada (no una por fila) con los datos correctos
  (`nivel`, `destino_tipo`, `destino_valor`, `origen`); una corrida
  totalmente resuelta no crea ninguna; el `origen` se propaga desde
  `resolver_uno` y desde el body del endpoint interno hasta el título de la
  notificación.

## 8. Criterios de aceptación

- [x] `viv_geo_alias_manual` creada + seed de 18 filas verificado contra
      Postgres real local (`docker-compose.dev.yml`, upgrade + downgrade
      limpios) — prod pendiente del deploy real.
- [x] `POST /internal/geo/resolver-localidades` responde los 4 `match_tipo`
      correctamente sobre casos reales de las 18 filas migradas (tests +
      verificación manual contra Postgres real).
- [x] `GET /api/v1/vivienda/geo/duplicados` implementado + tests (grupo con
      variantes, sin duplicados, filtra `deleted_at`, 403 fuera de
      Admin/Supervisor) — verificación contra duplicados reales de producción
      pendiente del deploy.
- [x] Gasífera y Gralgob: `sync_from_sheet` persiste `id_geo`/`match_tipo` en
      batch al final de la corrida; tests cubren match real, fallo de red
      real (no mockeado a nivel de función) sin abortar el sync, y flag
      apagado sin llamada HTTP — corrida real contra Sheets en producción
      pendiente del deploy.
- [x] `resumen_territorial.agrupar_por_localidad` agrupa por `id_geo` cuando
      está disponible, con fallback a `candidatos_localidad()` — tests
      cubren ambos casos (`id_geo` compartido entre Vivienda/Gasífera con
      texto crudo distinto, y alias sin `id_geo` tipo "X - Y") — verificación
      en producción pendiente del deploy.
- [x] CC/CH: alta/edición real persiste `localidad_id`, backfill corrido
      contra Postgres real local (`alembic upgrade head` con datos de prueba
      + el padrón real de 544 localidades cargado — verificó `exacto`,
      `manual` y `sin_match` correctamente) — prod pendiente del deploy real.
- [x] `AtpPage.tsx` sin `VINCULACION_MANUAL`/`candidatosLocalidad`/
      `normalizeDepartamento` hardcodeados — consume `id_geo`/`match_tipo`
      del backend, `npm run build` verde. Verificación visual en navegador
      contra datos reales pendiente del deploy.
- [x] Reporte de duplicados de CC/CH/ML corrido contra producción y revisado
      con el usuario: los 30 residuales se investigaron uno por uno (2026-09-29)
      — 6 resueltos con alias nuevos (migración 0032), 7 confirmados como
      duplicados internos del propio padrón y desactivados (migración 0033,
      revisados y aprobados por el usuario par por par), 2 genuinamente sin
      resolver y documentados (mismo criterio que "Santiago Temple").
- [x] Deploy real de las 3 migraciones nuevas (`svc-vivienda` 0030,
      `svc-gasifera` 0003, `svc-gralgob` 0002) vía Cloud Shell/cloud-sql-proxy,
      redeploy de los 3 servicios + frontend, `services/cloudbuild.yaml` con
      `_RESOLVER_LOCALIDADES_ENABLED=true`, y gateway nuevo
      (`ministerio-config-v20260928`) para `GET /api/v1/vivienda/geo/duplicados`
      — completado 2026-09-28/29 (incluyó recuperarse de una suspensión de
      Cloud SQL por facturación, resuelta por el usuario). KPI verificado en
      vivo en Resumen Territorial tras las 4 migraciones (0030-0033): 413
      localidades / 26 departamentos.
- [ ] **Hallazgo no relacionado, detectado al verificar la migración 0030
      contra Postgres real desde cero**: la migración `0008_data_update.py`
      de `svc-vivienda` falla (`ForeignKeyViolationError` en
      `viv_cordon_cuneta.ejuridico`) al correr la cadena completa
      `alembic upgrade head` contra una base nueva vacía — parece un problema
      de orden de seed preexistente, no introducido por esta spec (la base de
      producción ya está migrada y no lo sufre). No se investigó a fondo ni
      se corrigió — fuera de alcance de esta entrega. Reportar/registrar en
      `docs/files/auditoria-codigo.md` en una sesión de auditoría.
- [x] Mi Lugar resuelve `localidad_id`/`localidad_match_tipo` in-process en
      alta/edición, mismo criterio que CC/CH (§4.10) — sin migración nueva,
      tests en `test_mi_lugar_geo.py` (2026-09-29). Pendiente: deploy +
      verificación contra producción.
- [x] Notificación + log de `sin_match` centralizado en `resolver_lote`,
      cubre las 5 fuentes con un solo cambio (§4.11) — tests en `test_geo.py`/
      `test_internal_router.py` (2026-09-29). Pendiente: deploy + verificar
      que la notificación aparece en el feed del frontend contra datos
      reales, y que el volumen de notificaciones repetidas (no hay dedup, ver
      §4.11) resulta manejable en la práctica.
- [ ] Deploy de los cambios de Mi Lugar + notificaciones (código nuevo en
      `svc-vivienda` + los 3 clientes `geo_resolver.py` en Gasífera/Gralgob/
      Privada pasando `origen`) — no hecho todavía, sesión aparte.

## 9. Fuera de alcance / decisiones futuras

- **Panel de asignación manual de localidades sin resolver** (2026-09-29,
  pedido explícito del usuario para más adelante): hoy §4.11 deja
  trazabilidad vía notificación + log, pero corregir un `sin_match`
  persistente todavía requiere una sesión de Claude Code escribiendo un
  alias en `viv_geo_alias_manual` a mano (como se hizo con los ~30
  residuales de esta entrega). Un futuro panel en el frontend permitiría a
  un Admin, desde el feed de notificaciones o una pantalla dedicada, ver las
  localidades sin resolver y vincularlas a una fila del padrón (o marcarlas
  explícitamente como "confirmada ausente", como "Santiago Temple")
  directamente, sin pasar por una migración de datos. Requiere: un endpoint
  de alta en `viv_geo_alias_manual` gateado a Admin (hoy la tabla sólo se
  escribe desde migraciones), y una pantalla — spec propia cuando se
  retome, fuera de alcance de esta entrega.

- **Unificar `viv_geo_localidades` y `priv_geo_localidades`/
  `priv_localidades_info` en un solo padrón enriquecido** (censo, catastro,
  economía, y lo que se sume después) es la ambición de largo plazo que
  planteó el usuario. Hoy `priv_localidades_info`/`priv_departamentos_info`
  ya cubre parte de eso (habitantes, electores, intendente, partido
  político — ADR-012) pero es propiedad de svc-privada; construir un segundo
  registro enriquecido en paralelo desde svc-vivienda recrearía el mismo
  problema de padrones divergentes que esta spec busca resolver. Antes de
  avanzar necesita una decisión explícita: ¿se enriquece
  `viv_geo_localidades`, se adopta `priv_localidades_info` como fuente única
  y el resto lo consume read-only, o se migra todo a un servicio nuevo
  dedicado? Cualquiera de las tres implica su propio ADR — no se decide acá.

  **Nota para esa ADR futura, a favor de no rehacer el trabajo de esta
  spec**: todo lo construido acá (tabla de vinculación manual, endpoint de
  resolución, `id_geo` persistido en Vivienda/Gasífera/Gralgob, join por
  `id_geo` en `resumen_territorial`) es reutilizable sin cambios *siempre que
  el padrón que termine siendo canónico preserve los valores de `id_geo` de
  `viv_geo_localidades`* (como PK propia, o al menos con una tabla de
  equivalencia 1:1). Si en cambio el padrón futuro usa una identidad propia
  distinta (caso de adoptar `priv_geo_localidades`, cuyo `id_geo` ya tiene
  otro formato/String(30), o un servicio nuevo con IDs propios), el único
  costo de esta entrega es un remapeo puntual de `id_geo` en
  `gas_pit_acciones_territorio`/`gas_pit_obras_localidades`,
  `atp_compromisos`, `viv_cordon_cuneta`/`viv_cordoba_hogar` y
  `viv_geo_alias_manual` (un `UPDATE` con join por nombre normalizado) —
  acotado porque el matching queda centralizado en un solo lugar, no un
  rediseño. El resto del trabajo (limpieza de datos existentes, fix de alias
  en `resumen_territorial`, retirar el hardcode del frontend) no se toca en
  ningún escenario.

  **Criterio para evaluar extraer el padrón a un servicio propio (`svc-geo`
  u otro nombre) en vez de seguir enriqueciendo `viv_geo_localidades` in
  place**: no es una decisión a tomar ahora (sería sobreingeniería sin un
  trigger concreto), pero vale dejar por escrito qué señal la justificaría —
  (a) el padrón necesita un ABM propio con ciclo de vida/aprobaciones
  distinto al de Vivienda (ej. carga de catastro con su propio flujo de
  revisión), (b) aparecen jobs propios (Cloud Scheduler) que no tienen nada
  que ver con vivienda (ej. ingesta periódica de una API de censo), o (c) el
  número de servicios que dependen de él crece más allá de los actuales
  (Gasífera, Gralgob, y a futuro Infraestructura/Territorial/Desarrollo) al
  punto de que acoplar su ciclo de deploy al de `svc-vivienda` empieza a
  doler. Ninguna de las tres se da hoy.
- Enforcement de escritura contra el padrón en CC/CH (dejar de admitir texto
  libre, bloquear el alta si no matchea) — evaluar una vez que se vea qué tan
  seguido aparecen duplicados nuevos después de esta limpieza. Distinto de
  resolver y persistir `id_geo` en paralelo, que sí queda incluido en esta
  entrega (§2/§4.9).
- Persistir `id_geo` a nivel `gas_pit_obras`/`gas_pit_obras_localidades` más
  allá de `gas_pit_acciones_territorio` si hiciera falta para otro reporte.

## 10. Preguntas abiertas

1. ¿Confirmar la pregunta de §9 (destino de largo plazo del padrón
   enriquecido: evolucionar `viv_geo_localidades`, adoptar
   `priv_localidades_info` como fuente única, o un servicio nuevo dedicado)
   ahora, o se deja pendiente hasta después de que esta entrega esté en
   producción y se pueda evaluar con más información? No bloquea esta spec.
