# Spec: svc-privada — Categorías, Programas y Áreas editables + panel de administración

**Estado**: **approved** (backend + migración implementados 2026-09-02; frontend en curso)
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-02
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
  (opcional, no obligatoria) — **Anexo A**.
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

## Anexo A — Mapa de compatibilidad `categoria_general_id` → categoría nueva (a completar)

| `categoria_general_id` legacy | Categoría nueva sugerida |
|---|---|
| `CAT_INFRAESTRUCTURA_VIAL` (+ regex cordón/adoquín) | Cordón Cuneta y adoquinado |
| `CAT_AGUA_Y_SANEAMIENTO` | Obras de Recursos Hídricos |
| `CAT_AYUDA_A_INSTITUCIONES` | Ayudas a instituciones |
| … (resto) | … a definir con el área |
