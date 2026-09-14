# Spec: svc-privada — Flujo de derivaciones de una gestión (vista DAG)

**Estado**: draft (taxonomía de áreas propuesta 2026-09-14 — Anexo F, pendiente de sign-off del área)
**Versión**: 0.2.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-14
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

- Taxonomía de `priv_areas`: **borrador listo para revisión**, ver Anexo F (abajo) — falta que el
  área confirme los nodos de confianza media/baja (especialmente el bucket "Personal interno").
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

## Anexo F — Taxonomía de `priv_areas` propuesta (borrador, 2026-09-14)

Construido con `scripts/generar_anexos.sh` contra `essential-haiku-482815-u4` (27 valores distintos
de `derivado_a`, 22 de `cat_ministerio_agencia`; `metadata_json.derivado_a` confirmado **vacío** en
los 166 eventos — no hay cadena histórica que reconstruir, ver §1). **Propuesta, no aprobada** — el
área tiene que confirmar los nodos de confianza `media`/`baja` antes de sembrarla en la migración.

### Hallazgo: `organismo_id` NO sirve como insumo de área

Los ~185 valores distintos de `F_organismo_id_distinct.json` son nombres de **destinatarios**
(escuelas, jardines, municipios, comunas — "IPEM 348 Gabriel García Marquez", "Municipalidad de
Balnearia", "COOP DE ELECTRICIDAD Y SERV PUBLICOS"), no áreas de gobierno por las que rutea una
gestión. Es el dato del *para quién es la obra*, no *quién la gestiona*. Se descarta como insumo de
`priv_areas`/`priv_area_alias` (queda disponible para un futuro cruce de "beneficiarios", fuera de
este spec).

### Nodos ya sembrados (migración `0002`) — se mantienen

| id | label | orden |
|---|---|---|
| `1756700002001` | DGV | 10 |
| `1756700002002` | Secretaría de Gestión y Vinculación de Infraestructura | 20 |
| `1756700002999` | Área desconocida (centinela) | 999 |

### Nodos nuevos propuestos

`confianza`: **alta** = mapea 1:1 a un `cat_ministerio_agencia` ya vigente y usado en prod (sin
ambigüedad); **media** = inferencia razonable, sin confirmar con el área; **baja** = requiere que el
área lo resuelva explícitamente.

| orden | label propuesto | alias de `derivado_a` que resuelve (ocurrencias) | confianza |
|---|---|---|---|
| — (usa el existente `1756700002001` DGV) | — | `VIVIENDA` (41) — "DGV" ya está sembrada y es el nombre que usa el resto del sistema (Córdoba Hogar/Cordón Cuneta/Mi Lugar · DGV) | **alta** |
| — (usa el existente `1756700002002`) | — | `SECRETARIA DE INFRAESTRUCTURA SOCIAL` (5), `SUAC DEL MINISTERIO DE INFRAESTRUCTURA Y SERVICIOS PUBLICOS MIYSP(OBRAS PUBLICAS)` (2), `SECRETARIA PRIVADA DE LA DIRECCION DE VIALIDAD` (2), `SECRETARIA DE COORDINACION DE INFRAESTRUCTURA MIYSP` (2), `DIRECCION DE JURISDICCION DE INFRAESTRUCTURA Y EQUIPAMIENTO` (1), `JEFATURA DE AREA GESTION Y SEGUIMIENTO DE OBRAS DE SANEAMIENTO` (1), `DEPARTAMENTO I CONSERVACION CAMINOS DE TIERRA 24/10/2025` (1), `SUBSECRETARIA DE INFRAESTRUCTURA ELECTRICA MIYSP` (1) — total 15 | **media** (asume que "Secretaría de Gestión y Vinculación de Infraestructura" = MIYSP/Vialidad; confirmar) |
| 30 | Secretaría General de Gobierno | `SECRETARIA DE GOBIERNO` (8), `Gobierno` (5), `GOBIERNO` (4) — total 17 | **alta** |
| 40 | Ministerio de Desarrollo Social | `COORDINACION DE LA SECRETARIA PRIVADA DEL MINISTERIO DE DESARROLLO SOCIAL` (4) | **alta** |
| 50 | Ministerio de Salud | `Ministerio de Salud` (1) | **alta** |
| 60 | Ministerio de Cooperativas y Mutuales | `SUAC MINISTERIO DE COOPERATIVAS Y MUTUALES` (1) | **alta** |
| 70 | Ministerio de Ambiente | `Secretaria General Ambiente` (1) | **alta** |
| 80 | Agencia Córdoba Cultura | `SUBDIRECCION DE JURISDICCION PRODUCCION DE LA AGENCIA CORDOBA CULTURA` (1) | **alta** |
| 90 | Programa Semilla / Hábitat y Desarrollo Emprendedor | `PROGRAMA SEMILLA SECRETARIA GENERAL DE HABITAT Y DESARROLLO EMPRENDEDOR` (1) | **media** (¿nodo propio o alias de Desarrollo Social?) |
| 100 | Administración / Rendición de Cuentas | `RENDICION DE CUENTAS SUBAREA ORDENES DE PAGO Y RDF` (1) | **baja** |
| 110 | Personal interno (pendiente de resolver a área) | `MOLINARI` (31), `FABRICIO DIAZ` (7), `CHESTA` (5), `TURLETTO` (1), `BORELLO` (1), `BENSO` (1), `ALTAMIRANO` (1) — total 47 (**~30% de las 158 ocurrencias totales**) | **baja** — son nombres de persona, no de área; el área tiene que decir a qué oficina pertenece cada uno (o confirmar que quedan agrupados acá) |

`"prueba"` (4 ocurrencias) no es un valor real — se descarta (resuelve al centinela "Área
desconocida", no se crea alias).

### `priv_area_alias` — de la propuesta de arriba

Cada valor de la columna "alias que resuelve" se inserta como una fila `(alias_normalizado,
area_id)` en `priv_area_alias`, apuntando al `area_id` de su fila (existente o nuevo). El backfill
de `priv_gestion_derivaciones` (§2) resuelve `gestiones.derivado_a_id` contra esta tabla; lo que no
matchea cae al centinela `1756700002999` con `confianza='baja'`.

### Qué falta confirmar con el área antes de sembrar esto

1. ¿"Secretaría de Gestión y Vinculación de Infraestructura" (ya sembrada) es el nodo correcto para
   el cluster MIYSP/Vialidad/Saneamiento, o hace falta separarlo?
2. A qué oficina/área pertenece cada persona del bucket "Personal interno" (`MOLINARI`,
   `FABRICIO DIAZ`, `CHESTA`, `TURLETTO`, `BORELLO`, `BENSO`, `ALTAMIRANO`) — es casi un tercio del
   volumen total, vale la pena resolverlo bien en vez de dejarlo en un bucket genérico.
3. ¿"Programa Semilla..." es un nodo propio o un alias de "Ministerio de Desarrollo Social"?
4. ¿"Administración / Rendición de Cuentas" es un área real del flujo, o vuelve al centinela?
