# Spec: svc-privada — Flujo de derivaciones de una gestión (vista DAG)

**Estado**: draft
**Versión**: 0.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-08-31
**Servicio**: `svc-privada` (módulo `app/derivaciones/` o extensión de `app/gestiones/`)
**Depende de**: `spec-migracion-svc-privada.md` `approved`; `spec-privada-categorias-programas.md`
(provee `priv_areas` como set de nodos).
**ADRs**: ADR-013 (historial de derivaciones estructurado como fuente del DAG).

---

## 0. Origen

Pedido del usuario (2026-08-31), mejora 3: *"al seleccionar una gestión poder ver el flujo completo
de las áreas por donde pasó y dónde está ahora … algo similar a como se ve un flujo DAG"*, más
intuitivo que el timeline vertical actual (`GestionDetalleDrawer.tsx` → sección "Movimientos").

## 1. Estado actual (por qué hace falta esto)

- **No existe entidad de flujo.** `gestiones` tiene `ministerio_agencia_id` (FK a catálogo),
  `organismo_id` (texto libre, se setea al alta, nunca se actualiza) y `derivado_a_id` (texto libre,
  se **pisa** en cada `cambiar-estado`).
- **No hay taxonomía de áreas.** `derivado_a` es texto libre → sin un set de nodos estable no se
  puede dibujar un DAG legible. Valores reales (Anexo F): `VIVIENDA`, `MOLINARI`, `GOBIERNO`,
  `SECRETARIA DE GOBIERNO`, apellidos sueltos (`CHESTA`, `BORELLO`, `BENSO`), reparticiones largas
  (`SUAC DEL MINISTERIO DE INFRAESTRUCTURA Y SERVICIOS PUBLICOS…`). ~27 valores distintos, cola larga.
- **El histórico NO es reconstruible.** Comprobado en datos (Anexo G):
  - Sólo **166 eventos** en total para 2123 gestiones (159 `CREACION`, 3 `CAMBIO_ESTADO`, 2
    `ACTUALIZA_DATO`, 2 `ARCHIVO`). El grueso de las gestiones se importó por Excel sin eventos.
  - **`metadata_json.derivado_a` es siempre `null`** (`F_metadata_derivado_a_distinct` = `[]`).
  - ⇒ el backfill desde `metadata_json` **no produce aristas históricas**. Lo único disponible por
    gestión es el `derivado_a_id` **actual** (un solo valor, sin historia) y el ministerio de
    ingreso (`metadata_json.ministerio_agencia_id` de `CREACION`, sólo para las ~159 con evento).

**Implicancia de alcance:** el DAG es una feature **hacia adelante** — se llena a medida que
`svc-privada` escribe `priv_gestion_derivaciones` en cada `cambiar-estado` post-migración. Para las
gestiones históricas el "flujo" es a lo sumo 2 nodos (ingreso → área actual). Ver §5 (decisión E-4).

## 2. Alcance

### Incluido
- Tabla `priv_gestion_derivaciones` (append-only) como fuente del DAG.
- `priv_area_alias` — mapea variantes de texto libre observadas a un `priv_areas.id`.
- Escritura **runtime**: `cambiar-estado` (y un `POST .../derivar` si se separa) insertan una fila
  de derivación en la misma transacción que la mutación.
- Job de **backfill** best-effort — dado que `metadata_json.derivado_a` está vacío (§1), el backfill
  se reduce a: 1 fila por gestión con `area_hacia_id` = resolución de `gestiones.derivado_a_id`
  actual contra `priv_areas` + `priv_area_alias` (centinela si no matchea), `area_desde_id` =
  resolución del `ministerio_agencia_id` de ingreso (o `NULL`), `origen = 'backfill'`,
  `confianza = 'baja'`. No hay cadena histórica que reconstruir.
- `GET /api/v1/privada/gestiones/{id}/flujo` → `{ nodos: [{area_id, label, es_actual, primera_fecha,
  ultima_fecha}], aristas: [{desde, hacia, fecha, estado, usuario, confianza, origen}], actual:
  {area_id, estado} }`.
- Frontend: pestaña/vista **"Flujo"** en `GestionDetalleDrawer.tsx`, junto al timeline "Movimientos"
  existente (no lo reemplaza). DAG dibujado con SVG/flexbox (no hay componente reusable en el repo);
  badge "reconstruido" sobre aristas de `origen='backfill'` con `confianza != alta`.

### Fuera de alcance
- Máquina de estados formal / validación de transiciones (ADR-009).
- Editar el flujo a mano (es derivado de eventos + derivaciones runtime).
- Métricas de duración por etapa / SLA por área (posible v2).

## 3. Modelo de datos (`db_privada`)

### `priv_gestion_derivaciones` (append-only)

| Columna | Tipo | Notas |
|---|---|---|
| `id` | UUID PK | |
| `gestion_id` | UUID FK → `priv_gestiones.id` | |
| `area_desde_id` | BigInteger FK → `priv_areas.id` nullable | NULL en la primera (alta) |
| `area_hacia_id` | BigInteger FK → `priv_areas.id` | NOT NULL (centinela si no resuelve) |
| `estado` | VARCHAR | estado de la gestión al momento de la derivación |
| `fecha` | TIMESTAMPTZ | |
| `usuario` | VARCHAR (email) | |
| `evento_id` | UUID FK → `priv_gestiones_eventos.id` nullable | traza al evento que la originó |
| `origen` | VARCHAR CHECK (`runtime`/`backfill`) | |
| `confianza` | VARCHAR CHECK (`alta`/`media`/`baja`) | `alta` para `runtime` |
| `created_at` | TIMESTAMPTZ | |

### `priv_area_alias`

| Columna | Tipo |
|---|---|
| `id` | BigInteger PK |
| `alias` | VARCHAR (texto observado, normalizado) |
| `area_id` | BigInteger FK → `priv_areas.id` |

## 4. Riesgos

- **RE-2**: el backfill no tiene clave limpia (`derivado_a` texto libre). Mitigar: `confianza` por
  fila, nodo centinela, badge "reconstruido", y opción de configurar "DAG sólo para gestiones
  post-migración; timeline para históricas" si el área lo prefiere.
- **RE-3**: sin taxonomía de áreas estable el spec se re-trabaja → `priv_areas` + `priv_area_alias`
  deben estar poblados y revisados por el área **antes** de implementar (Anexo F del spec padre).
- **RE-11**: los endpoints nuevos aceptan sólo UUID (no `id_legacy`); el frontend debe estar 100%
  migrado a UUID antes de esta fase.

## 5. Decisiones abiertas (para el área)

- Taxonomía de `priv_areas`: híbrido (curado + `SELECT DISTINCT` sobre `gestiones`) — confirmar la
  lista final y los alias.
- Flujo histórico: default = DAG reconstruido best-effort con badge. Confirmar si prefieren "sólo
  gestiones nuevas".
- ¿El "área actual" sale de la última derivación, o de `area_id` de la gestión
  (`spec-privada-categorias-programas.md`)? Propuesta: de la última derivación; `area_id` es el área
  temática, la derivación es el ruteo.

## 6. Criterios de aceptación

- [ ] Migración crea `priv_gestion_derivaciones` + `priv_area_alias`; `alembic current` = head.
- [ ] `cambiar-estado` inserta una fila `runtime` en la misma transacción.
- [ ] Job de backfill corre idempotente; reporte de % resuelto vs centinela por `confianza`.
- [ ] `GET /gestiones/{id}/flujo` devuelve nodos/aristas/actual; sólo acepta UUID.
- [ ] Vista "Flujo" en el drawer, con el timeline "Movimientos" intacto al lado.
- [ ] Badge "reconstruido" en aristas `backfill` de confianza no-alta.
- [ ] Gateway: path `/gestiones/{id}/flujo` + `options:`; nueva config.
