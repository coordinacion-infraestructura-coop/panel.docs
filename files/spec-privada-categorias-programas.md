# Spec: svc-privada — Categorías, Programas y Áreas editables + panel de administración

**Estado**: **approved** (backend + migración implementados 2026-09-02; frontend en curso)
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-06
**Servicio**: `svc-privada` (módulo `app/catalogos_editables/`)
**Depende de**: `spec-migracion-svc-privada.md` `approved` + Fase 6 (cutover) completada.
**ADRs**: ADR-010 (3 catálogos editables), ADR-011 (`priv_programas` propio con `POST`).

> **Implementado 2026-09-02 (E1 + E2 backend)** — `panel.backend`:
> - Migración `0002`: `priv_categorias` / `priv_programas` / `priv_areas` (patrón `viv_cc_estados`:
>   `id` BigInteger client-gen, `label`, `orden`, `activo`; `bg`/`text_color` en categorías,
>   `codigo` único en programas, `es_centinela` en áreas). `priv_gestiones` += `categoria_id` /
>   `programa_id` / `area_id` (FK nullable) + `ok_gobernador` / `ok_ministro` (`VARCHAR(20)` CHECK
>   `IN ('SI','NO','PENDIENTE')`, default `'PENDIENTE'`) + `acciones_implementadas` (Text).
> - Seed: las **9 categorías** con `orden` 10..90 y colores por defecto (editables en runtime);
>   arranque de 3 programas + 3 áreas (incluye el centinela "Área desconocida" para E3).
> - `app/catalogos_editables/` (models/schemas/service/router): CRUD `GET/POST/PATCH/DELETE`
>   `/api/v1/privada/{categorias,programas,areas}` — GET con `ROLES_LECTURA`, escritura con
>   `ROLES_TRANSICION` (Admin/Supervisor; Operador NO administra catálogos, RE-5). `DELETE` con
>   guard 409 (`CATEGORIA_EN_USO` / `PROGRAMA_EN_USO` / `AREA_EN_USO`) contando refs en
>   `priv_gestiones` no borradas. `POST /programas` valida `codigo` único → 409 `CODIGO_DUPLICADO`.
> - `GestionCreate` / `GestionUpdate` / `CambioEstado` ganan `categoria_id`/`programa_id`/`area_id`/
>   `ok_gobernador`/`ok_ministro`/`acciones_implementadas` (aditivo). **E2**: `cambiar_estado`
>   persiste `acciones_implementadas` **en la gestión**, no sólo en el evento. `_list_item`
>   (+5 campos) y `_detail` (+6) los exponen. `GET /gestiones` filtra por `ok_gobernador`/`ok_ministro`.
> - `scripts/backfill_categorias.py` (RE-1, re-ejecutable, `--dry-run`/`--force`/`--diff-informe`):
>   deriva `categoria_id` de `tema_informe(...)` → categoría, con fallback por `categoria_general_id`
>   (los 14 `CAT_*` legacy); lo que no matchea queda NULL (el área lo completa en el panel).
>   `acciones_implementadas` se backfillea del último evento con ese campo en `metadata_json`.
> - Gateway: `infra/gateway/openapi.yaml` con los 6 paths nuevos (+ `options`). Requiere nueva
>   config `ministerio-config-v{FECHA}` + `gateways update`.
> - Tests: 73 en verde (`test_catalogos_editables.py` + contrato ajustado a superset).
>
> **Nomenclatura en la UI (2026-09-02)**: el catálogo `priv_categorias` (tabla/endpoint/`categoria_id`
> sin cambios) se rotula **"Campo de Trabajo"** en pantalla; el viejo `categoria_general_id` se
> rotula **"Categoría General"**. El desplegable de cada catálogo editable tiene como última opción
> "＋ Cargar nueva opción…" (input inline) en vez de un botón lateral.
>
> **Implementado 2026-09-02 (frontend)** — `panel.front`: `CatalogoEditableSelect` (3 desplegables
> con "＋ Cargar nueva opción…" como última opción), campos en `AgregarGestionModal` /
> `CambiarEstadoModal` / `GestionDetalleDrawer`, filtros Ok Gob/Min en la lista, y
> `GestionarCatalogosModal` (botón "⚙ Catálogos", Admin/Supervisor — editar label/orden/colores/
> código/activo, borrar con guard 409, agregar). `backfill_categorias.py` corrido en prod
> (1946/2047 con `categoria_id`).
>
> **Pendiente**: sólo deploy del frontend (columna + filtro "Campo de Trabajo" ya hechos, panel.front d4ce471).
> **A revisar con Secretaría Privada**: `orden`/colores concretos de los catálogos y el mapa de
> backfill `categoria_general_id → categoria_id` (RE-1: doble corrida + `--diff-informe` + sign-off
> antes de que E4 retire el regex).
>
> **Reasignación masiva de "Otras Obras" (2026-10-06)** — operación de datos en prod, sin cambios
> de código ni de schema: 450 de las 484 gestiones que estaban en "Otras Obras" pasaron a su Campo
> de Trabajo específico, con mapeo validado fila por fila por el responsable. Se creó el campo
> **PLAYON DEPORTIVO**. Detalle, criterio y reversión en el **Anexo B**. ⚠ Desde esta fecha
> `scripts/backfill_categorias.py --force` **no debe correrse**: pisaría la reasignación.

---

## 0. Origen

Pedido del usuario (2026-08-31), mejoras 4, 5 y 6:
- Categorías editables en runtime como los estados de Cordón Cuneta / Córdoba Hogar. Lista inicial:
  Vivienda, Loteos, Cordón Cuneta y adoquinado, Pedidos por ATP, Pedidos NorOeste y Sur Sur,
  Obras de Recursos Hídricos, Pedidos Administrativos (ERSEP, EJIDO, FONDO FEDERAL, etc.),
  Otras Obras (plazas), Ayudas a instituciones.
- Lógica de carga **Categoría → Programa asociado → Área** con **desplegables** para Programa y
  Área, ampliables desde un **panel de administración** (D-1). No hay mapa autoritativo cargado; el
  usuario elige libremente de cada desplegable.
- Ejemplos de uso (no son un mapeo obligatorio): Vivienda → Córdoba Hogar → DGV; Loteos → Mi Lugar
  → DGV; Cordón Cuneta → CORDON CUNETA → DGV; Pedidos por ATP → Programa A (creable al vuelo) → Área.
- Campos nuevos en toda gestión: `Ok Gobernador` y/o `Ok Ministro`, Nro de expediente (opcional,
  ya existe), Derivado a (ya existe como texto libre).

## 1. Propósito

Reemplazar la clasificación de gestiones por regex sobre `LOWER(detalle)` + `categoria_general_id`
legacy (`v_informe_cooperativas`) por **tres catálogos estructurados y editables en runtime**
(`priv_categorias`, `priv_programas`, `priv_areas`), elegibles como desplegables al cargar/editar
una gestión, y administrables desde un panel. Agregar los campos `ok_gobernador` / `ok_ministro`.

## 2. Alcance

### Incluido
- Tablas `priv_categorias`, `priv_programas`, `priv_areas` (patrón `viv_cc_estados` de svc-vivienda).
- Columnas nullable en `priv_gestiones`: `categoria_id`, `programa_id`, `area_id` (FK a las 3
  tablas), `ok_gobernador`, `ok_ministro`.
- Migración `0002` en `db_privada`: crea las 3 tablas + columnas + seed inicial de `priv_categorias`
  con las 9 categorías + **backfill** de `categoria_id` desde `categoria_general_id` (mapa de
  compatibilidad, Anexo A de este spec).
- CRUD de los 3 catálogos (`GET` `ROLES_LECTURA`; `POST/PATCH/DELETE` `ROLES_TRANSICION`), con guard
  de integridad en `DELETE` → `409 {"code":"CATEGORIA_EN_USO" | "PROGRAMA_EN_USO" | "AREA_EN_USO"}`
  contando referencias en `priv_gestiones`, en `priv_gestiones_eventos`/`priv_gestion_derivaciones`
  y (para categoría) en el mapa de clasificación del informe.
- Schemas `POST /gestiones` y `PATCH /gestiones/{id}` ganan `categoria_id`, `programa_id`, `area_id`,
  `ok_gobernador`, `ok_ministro`, `derivado_a`, `acciones_implementadas` (aditivo, no-breaking).
- `acciones_implementadas` pasa a persistirse en `priv_gestiones` (hoy sólo va a `metadata_json` —
  RE-10 del spec padre).
- Frontend: modal(es) "Gestionar categorías / programas / áreas" (copia de `GestionarEstadosModal`
  de `CordonCunetaPage.tsx:619`), 3 desplegables en el alta/edición con "+ nueva opción" inline,
  campos `Ok Gobernador`/`Ok Ministro` en el formulario y como filtros de la lista.

### Fuera de alcance
- Re-apuntar `v_informe_cooperativas` / `tema_informe` al modelo estructurado →
  `spec-privada-informe-cooperativas-v2.md` (RE-1: reclasifica gestiones y mueve totales; requiere
  doble corrida + sign-off).
- El DAG de flujo → `spec-privada-flujo-derivaciones.md` (consume `priv_areas` de este spec).
- Máquina de estados formal / validación de transiciones (ADR-009: se mantiene laxa).
- Un mapa relacional obligatorio Categoría→Programa→Área (D-1: los tres son independientes).

## 3. Modelo de datos (`db_privada`)

### `priv_categorias` / `priv_programas` / `priv_areas` (misma forma; patrón `viv_cc_estados`)

| Columna | Tipo | Notas |
|---|---|---|
| `id` | BigInteger PK, `autoincrement=False` | client-generada `int(time.time()*1000)` |
| `label` | String(200) | NOT NULL |
| `orden` | Integer | posición en el desplegable |
| `activo` | Boolean | `server_default true` |
| `bg` / `text_color` | String(10) | opcional — sólo `priv_categorias` si se quiere chip de color |
| `created_at` / `updated_at` | TIMESTAMPTZ | |
| `updated_by` | String(200) | email del actor |

`priv_programas` puede sumar `codigo VARCHAR UNIQUE` (normalizado) para correlación string-keyed con
los programas de Vivienda (ADR-011) y un reporte de "programa sin categoría / huérfano".
`priv_areas` es **compartida con `spec-privada-flujo-derivaciones.md`** como set de nodos del DAG
(ADR-013); sembrado híbrido (curado + `SELECT DISTINCT` sobre `gestiones` — Anexo F del spec padre).

### Columnas nuevas en `priv_gestiones`

| Columna | Tipo | Notas |
|---|---|---|
| `categoria_id` / `programa_id` / `area_id` | BigInteger FK nullable | a las 3 tablas |
| `ok_gobernador` / `ok_ministro` | VARCHAR(20) + CHECK (`SI`/`NO`/`PENDIENTE`) | `server_default 'PENDIENTE'` (precedente `viv_ml_proyectos.ok_gob`) |
| `acciones_implementadas` | TEXT nullable | hoy sólo en `metadata_json` |

`categoria_general_id`, `tipo_gestion`, `canal_origen` **se conservan** hasta que
`spec-privada-informe-cooperativas-v2.md` confirme paridad.

## 4. Endpoints (`/api/v1/privada`)

| Método + path | roles |
|---|---|
| `GET /categorias` · `GET /programas` · `GET /areas` | `ROLES_LECTURA` |
| `POST /categorias` · `POST /programas` · `POST /areas` | `ROLES_TRANSICION` |
| `PATCH /{catalogo}/{id}` · `DELETE /{catalogo}/{id}` | `ROLES_TRANSICION` |

`DELETE` → `409` con `{"code": "...EN_USO", "message": "...N gestiones, M entradas de historial..."}`.

## 5. Frontend (`src/modules/privada/`)

- Modal admin (uno con 3 tabs, o 3 modales) clonado de `GestionarEstadosModal`: edición inline por
  fila, "+ Nueva", `orden`, `activo`, mensaje 409 inline, `queryClient.invalidateQueries`.
- Alta/edición de gestión: 3 `<select>` (categoría, programa, área) cada uno con opción
  "＋ agregar…" que abre un mini-form inline → `POST` → usa el id devuelto.
- `Ok Gobernador` / `Ok Ministro`: `<select>` tri-estado en el form; chips en la tabla; filtros en
  la barra de `GestionesListPage`.
- Permisos: `ROLES_TRANSICION` para el panel de administración y para el "+ agregar" inline.

## 6. Riesgos

- **RE-1** (heredado): la reclasificación del informe se difiere a su propio spec.
- **RE-4**: el guard de `DELETE` debe contar también las referencias del mapa de clasificación del
  informe, si no el informe pierde un tema en silencio.
- **RE-5**: alta inline de `priv_programas` → sprawl + colisión de `codigo` con Vivienda. Mitigar:
  `POST` restringido a `ROLES_TRANSICION` (no Operador), `codigo` único normalizado, reporte de
  huérfanos, documentar códigos reservados.
- **RE-10**: cablear `acciones_implementadas`/`derivado_a` cambia lo que `cambiar-estado` persiste.
  Landear el schema aditivo primero; los inputs de UI después.

## 7. Decisiones abiertas (para el área / relevamiento)

- `orden` y colores de las 9 categorías; sugerencia inicial de programa/área por categoría
  (opcional, no obligatoria) — **Anexo A**. La única instancia concreta de esta sugerencia
  implementada hasta ahora no es la tabla general que se dejó abierta acá, sino un mapeo
  puntual y hardcodeado (`caso_tipo` de Vivienda → categoría/programa/área), como parte de
  `docs/files/spec-vinculacion-vivienda-privada.md` (ADR-020).
- Semántica de `Ok Gobernador`/`Ok Ministro`: default = tri-estado `SI/NO/PENDIENTE`, seteable por
  Admin+Supervisor, independientes, en filtros. Confirmar.
- ¿`priv_programas.codigo` obligatorio o sólo para los que correlacionan con Vivienda?

## 8. Criterios de aceptación

- [ ] Migración `0002` crea las 3 tablas + columnas + seed de 9 categorías + backfill de
      `categoria_id`; `alembic current` = head.
- [ ] CRUD de los 3 catálogos con guard 409 verificado por test.
- [ ] Alta de gestión con los 3 desplegables + "+ agregar" inline funcionando.
- [ ] `ok_gobernador`/`ok_ministro` persisten, se filtran en la lista y se muestran en la tabla.
- [ ] `acciones_implementadas` persiste en `priv_gestiones` y sigue apareciendo en el evento.
- [ ] `categoria_general_id` intacto (no se toca en este spec).
- [ ] Audit log en cada escritura de catálogo (`resource_type` `priv_categoria`/`priv_programa`/`priv_area`).
- [ ] Gateway: nuevos paths `/categorias`/`/programas`/`/areas` + `options:` CORS; nueva config.

## Anexo A — Mapa de compatibilidad `categoria_general_id` → categoría nueva

**Completado 2026-09-14** con los datos reales del catálogo legacy (`B_cat_categoria_general.json`,
generado con `scripts/generar_anexos.sh`). Es una transcripción de `_LEGACY_A_CAT` en
`scripts/backfill_categorias.py` — **ya corre en prod** como fallback del backfill (cuando
`tema_informe(...)` no matchea nada); este Anexo documenta lo que el código ya decide, no introduce
un mapeo nuevo. `CAT_OTROS` (catch-all, "incluye `SIN_DEFINIR` y valores no mapeados") queda
deliberadamente sin mapear — el backfill lo deja `categoria_id = NULL` para que el área lo
complete a mano desde el panel.

| `categoria_general_id` legacy | Categoría nueva (Campo de Trabajo) |
|---|---|
| `CAT_GESTION_MUNICIPAL_INSTITUCIONAL` | Pedidos Administrativos |
| `CAT_AGUA_Y_SANEAMIENTO` | Obras de Recursos Hídricos |
| `CAT_INFRAESTRUCTURA_VIAL` | Cordón Cuneta y adoquinado |
| `CAT_OBRAS_PUBLICAS` | Otras Obras |
| `CAT_OBRA_ELECTRICA_ENERGIA` | Otras Obras |
| `CAT_OBRA_DE_GAS` | Otras Obras |
| `CAT_EDUCACION` | Ayudas a instituciones |
| `CAT_SALUD` | Ayudas a instituciones |
| `CAT_DESARROLLO_SOCIAL` | Ayudas a instituciones |
| `CAT_AYUDA_A_INSTITUCIONES` | Ayudas a instituciones |
| `CAT_COOPERATIVAS_Y_MUTUALES` | Ayudas a instituciones |
| `CAT_CULTURA_EVENTOS` | Otras Obras |
| `CAT_DEPORTES` | Otras Obras |
| `CAT_OTROS` (catch-all) | — (sin mapear; `categoria_id` queda `NULL`) |

Nota: el backfill real (`categoria_para()`) prioriza `tema_informe(categoria_general_id, detalle,
ministerio_agencia_id)` — que también usa regex sobre `detalle` — y sólo cae a esta tabla cuando
`tema_informe` no matchea nada. Por eso, p. ej., una gestión `CAT_INFRAESTRUCTURA_VIAL` puede
terminar en "Vivienda" o "Loteos" en vez de "Cordón Cuneta y adoquinado" si el `detalle` matchea esa
regex primero (reglas 6-7 de `app/informe/clasificacion.py`, "cualquier categoría"). Esta tabla es el
mapeo **de respaldo**, no el único criterio — ver `spec-privada-informe-cooperativas-v2.md` para el
análisis de esta interacción de cara al informe (E4).

## Anexo B — Reasignación masiva de "Otras Obras" (2026-10-06)

Operación de datos sobre `db_privada` (prod). No hubo cambios de código, de schema ni de contrato.

### Por qué

"Otras Obras" concentraba 484 de las 2.243 gestiones activas (el campo más cargado). La mayoría no
la había elegido el área: la asignó `scripts/backfill_categorias.py`, que manda ahí Gas, Kits
Solares, Luces LED, Infraestructura Eléctrica y los legacy `CAT_OBRAS_PUBLICAS` /
`CAT_CULTURA_EVENTOS` / `CAT_DEPORTES` (Anexo A). Después del backfill el área creó campos
específicos desde el panel (RED ELECTRICA, VEHICULO, infraestructura vial, ANUNCIO GOBERNADOR,
OBRA DE GAS), pero las gestiones ya cargadas quedaron en "Otras Obras".

### Cómo se hizo

1. **Extracción** de sólo lectura de las gestiones activas y los 3 catálogos editables.
2. **Propuesta**: se leyeron `detalle`, `observaciones`, `subtipo_detalle` y
   `acciones_implementadas` de las 484, una por una (no por palabras clave). Se agruparon en 23
   reglas, cada fila con campo propuesto, confianza y motivo. Donde el texto no alcanzaba se usó
   `categoria_general_id` como respaldo, bajando la confianza. Para elegir el destino se tomó como
   precedente cómo el área ya venía usando los campos nuevos (p. ej. kits solares y luminarias ya
   estaban en RED ELECTRICA; escuelas y salud en Ayudas a instituciones).
3. **Validación** por el responsable en un Excel (`mapeo_otras_obras_privada.xlsx`), con corrección
   por regla o por fila. Resultado: 40 correcciones puntuales sobre la propuesta — 39 a PLAYON
   DEPORTIVO (propuestas en Ayudas a instituciones) y 1 a Pedidos Administrativos (propuesta en
   Pedidos por ATP).
4. **Aplicación** en una sola transacción, previa corrida de simulación (rollback). Por cada
   gestión se replicó lo que hace `gestiones/service.patch_gestion`: `UPDATE` de `categoria_id` /
   `updated_at` / `updated_by`, evento `ACTUALIZA_DATO` (`campo_modificado = 'categoria_id'`) en
   `priv_gestiones_eventos` y registro en `priv_audit_log`. Sólo se tocaban gestiones que seguían
   en "Otras Obras" y no estaban borradas.

### Resultado

450 gestiones reasignadas, 34 se mantienen en "Otras Obras", ninguna salteada.

| Campo de Trabajo destino | `priv_categorias.id` | Gestiones |
|---|---|---|
| OBRA DE GAS | 1790167961248 | 178 |
| RED ELECTRICA | 1788440474308 | 104 |
| Obras de Recursos Hídricos | 1756700000006 | 43 |
| **PLAYON DEPORTIVO** (nuevo) | 1791315842588 | 39 |
| infraestructura vial | 1789737561042 | 36 |
| Ayudas a instituciones | 1756700000009 | 29 |
| Pedidos Administrativos | 1756700000007 | 10 |
| Cordón Cuneta y adoquinado | 1756700000003 | 7 |
| ANUNCIO GOBERNADOR | 1789749098268 | 2 |
| Loteos | 1756700000002 | 2 |
| Otras Obras (sin cambio) | 1756700000008 | 34 |

Cómo reconocer estos cambios en la base: eventos y auditoría con usuario
`script:mapeo-otras-obras`, `metadata_json.origen = 'mapeo_otras_obras'`, timestamp
`2026-10-06 19:44:02 UTC`.

### Criterios que quedaron fijados

- **Gas** → OBRA DE GAS. 100 de las 178 no mencionan gas en el texto (sólo estado: "OBRA
  INAUGURADA", "EXPEDIENTE EN PROCESO"); fueron por su `categoria_general_id = CAT_OBRA_DE_GAS`.
- **Kits/paneles solares, luminarias, alumbrado, tendido, media tensión** → RED ELECTRICA.
- **Agua, perforaciones, mangueras, tanques, cloacas, saneamiento** → Obras de Recursos Hídricos.
- **Pavimento, rutas, accesos, puentes, caminos rurales, ciclovías** → infraestructura vial;
  cordón cuneta y adoquinado → su campo propio.
- **Polideportivos, playones, techados deportivos** → PLAYON DEPORTIVO (39). El campo cubre
  infraestructura deportiva en general, no sólo playones. 9 gestiones deportivas quedaron en Ayudas
  a instituciones por decisión de la validación.
- **Escuelas, salud, comisaría, clubes, cooperativas** → Ayudas a instituciones.
- **Festivales y eventos, trámites** → Pedidos Administrativos.
- **Se mantienen en "Otras Obras"** (34): plazas, balnearios, SUM, edificios municipales, parques
  industriales, centros culturales, iglesias (25); y lo que no tiene campo que lo describa —
  conectividad, residuos/ambiente, minería, o texto insuficiente (9). Criterio no uniforme
  heredado: hay 6 plazas cargadas por el área en Pedidos Administrativos.

### Consecuencias y advertencias

- **`scripts/backfill_categorias.py --force` no debe volver a correrse**: recalcula `categoria_id`
  con el mapa viejo y devolvería estas gestiones a "Otras Obras". Sin `--force` es inocuo (sólo
  escribe donde `categoria_id IS NULL`), pero sus mapas `_TEMA_A_CAT` / `_LEGACY_A_CAT` siguen
  apuntando Gas y Eléctrica a "Otras Obras": las 104 gestiones aún sin campo, si se backfillean,
  caerían ahí. Actualizar esos mapas antes de reutilizarlo.
- **El Anexo A queda desactualizado** para `CAT_OBRA_DE_GAS`, `CAT_OBRA_ELECTRICA_ENERGIA` y
  `CAT_DEPORTES`: documenta lo que el script decide, no la clasificación vigente en la base.
- **Sin efecto sobre el Tablero / informe de Cooperativas**: `app/informe/` clasifica por regex
  sobre `detalle` (`tema_informe`), no por `categoria_id`. Sí es insumo para E4
  (`spec-privada-informe-cooperativas-v2.md`), que ahora parte de un `categoria_id` más preciso.
- `updated_at` de las 450 gestiones quedó en la fecha de la operación.

### Reversión

El respaldo `mapeo_otras_obras_respaldo_20261006.json` (raíz del directorio de trabajo, fuera de
los repos) lista `id`, `categoria_id_anterior` y `categoria_id_nuevo` de las 450. Revertir es
volver cada `id` a `1756700000008`, registrando evento y auditoría igual que en la aplicación. El
campo PLAYON DEPORTIVO se puede borrar desde el panel una vez que no tenga gestiones (guard 409
`CATEGORIA_EN_USO`). Los scripts de extracción y aplicación fueron de un solo uso y no se
versionaron.
