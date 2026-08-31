# Spec: Checklist Técnico DGV — panel editable por localidad y programa

**Estado**: approved
**Versión**: 1.1.1
**Servicio**: `svc-vivienda` (módulo nuevo `checklist_tecnico`, sin servicio nuevo)
**Responsable de spec**: Pedro Bonafe (revisado sección por sección con el usuario, 2026-08-26)
**Última actualización**: 2026-08-31

### Changelog
- **1.1.1 (2026-08-31)** — §8: se agrega `GET /api/v1/vivienda/programas-tablero` (KPIs
  agregados de CC/CH/ML), con `ROLES_LECTURA_TABLERO`. Motivo: §8 decía que `programas/router.py`
  "(Tablero/KPIs)" admitía `TecnicoDGV`, pero el Tablero real (`ProgramasPage.tsx`) calculaba
  todos sus KPIs en el cliente pidiendo los `GET` de panel completo de CC/CH/ML — vedados a ese
  rol. El endpoint nuevo devuelve solo los agregados; `GET /programas` (catálogo genérico) no
  servía para eso. Path con **guion** (`programas-tablero`, no `programas/tablero`): un literal
  en el mismo nivel de segmentos que `/programas/{programa_id}` rompe el ruteo de API Gateway
  (404) — misma convención que `cordon-cuneta-config`, `cordoba-hogar-config`, etc.
- **1.1.0 (2026-08-31)** — §6: se agrega `GET /checklist-tecnico/entidades` y
  `GET|POST /checklist-tecnico/{programa}/{entidad_id}/pedidos`. Motivo: la 1.0.0 decía
  reutilizar los `GET` de panel de `cordon_cuneta`/`cordoba_hogar`/`mi_lugar` para el selector
  y las observaciones, pero §8 veda esos `GET` a `TecnicoDGV` (403) — el único rol que usa esta
  pantalla. Contradicción interna: el selector nunca cargaba para el rol destino. Los endpoints
  nuevos viven dentro del módulo y usan `ROLES_LECTURA_CHECKLIST`/`ROLES_ESCRITURA_CHECKLIST`.
- **1.0.0 (2026-08-26)** — versión inicial aprobada.

---

## 0. Origen

El área técnica de la Dirección General de Vivienda (DGV) lleva su seguimiento documental de los 3 programas (Cordón Cuneta, Córdoba Hogar, Mi Lugar) en `DGV Programas 2026.xlsx`, editando fila por fila desde la pestaña `reporteLocalidades`. Se propuso y mockeó una interfaz que reemplace esa edición manual (artifact `Checklist Técnico DGV`, revisión 2) y **el equipo técnico del área (H) la aprobó**, incluidas las respuestas a las preguntas de diseño que quedaban abiertas (ver §3 y §4). Este spec traduce esa maqueta aprobada a un diseño de implementación concreto.

## 1. Propósito

Reemplazar la edición manual del Excel por un panel dentro del sistema: selector de localidad y programa → checklist de documentación con estado por ítem → "Estado del expediente" (workflow de 7 pasos) → hitos de ejecución de obra (Cordón Cuneta) → observaciones. Con autosave por campo (patrón Notion/Linear), sin el botón "Guardar" que usan hoy Cordón Cuneta y Córdoba Hogar — ese patrón fue, en palabras de H, "la manera casera que encontramos trabajando solo con una hoja de cálculo".

## 2. Alcance

### Incluido
- Checklist de documentación por localidad y programa (CC: 9 ítems + 10 sub-ítems del ítem 4; CH: 14 ítems; ML: 13 ítems + 6 sub-ítems del ítem 14), 5 valores posibles por ítem.
- "Estado del expediente de la localidad": catálogo de 7 valores, compartido entre los 3 programas, administrable por un usuario Admin.
- "Repartición" donde se radica el expediente: catálogo administrable por un usuario Admin, puede variar por programa.
- Fecha de radicación.
- Hitos de ejecución de obra con montos auto-calculados sobre el convenio (Anticipo 50% / Avance 40% → 25% / Avance 70% → 25% / Avance 100% → saldo) — **solo Cordón Cuneta** en esta entrega (es el único programa con datos reales de este tipo en la planilla).
- Nuevo rol `TecnicoDGV`, con navegación restringida a Tablero (KPIs) + Checklist Técnico — no ve Cordón Cuneta, Córdoba Hogar ni Mi Lugar como paneles completos.
- Semilla única de los datos ya sincronizados de Cordón Cuneta (ver §7) para no arrancar en blanco.

### Fuera de alcance (esta entrega)
- Los campos de convenio (expediente, monto, N° de orden) — son de solo lectura para el área técnica; se editan donde ya se editan hoy (`EditModal` de `CordonCunetaPage.tsx`/`CordobaHogarPage.tsx`, campo `viv_ml_proyectos` para Mi Lugar). No se duplican.
- Hitos de obra para Córdoba Hogar y Mi Lugar (sin datos reales de referencia todavía; extensible después si aparece la necesidad).
- Retiro del sync Excel→Postgres de Cordón Cuneta (`spec-sync-cc-checklist-tecnico.md`) — sigue corriendo en paralelo, sin tocar.
- Granularidad fina de permisos dentro del área (quién carga datos vs. quién administra estados/hitos/reparticiones vs. quién obtiene informes) — los jefes del área la definirán más adelante; esta entrega usa un único rol nuevo (`TecnicoDGV`) con permisos de lectura+escritura sobre el checklist.
- Integración con `informes/` (el `_COMPUTE_FN` de `app/informes/service.py` no incluye este módulo).

## 3. Definiciones confirmadas con el área (H, 2026-08-26)

Fuente: solapa `Validaciones` de `DGV Programas 2026.xlsx`, verificada directamente con `openpyxl` (filas 401-415) — coincide exactamente con lo que envió H.

**Estado de la documentación (por ítem)** — 5 valores, únicos, sin variantes informales adicionales:
1. Sin Presentar
2. En Evaluación Técnica
3. A corregir por M/C
4. En Evaluación Jurídico
5. Completo OK

**Estado del expediente de la localidad** — 7 valores, en este orden:
1. A INICIAR en DGV
2. En CURSO en DGV
3. COMPLETO en DGV
4. En CURSO en TC
5. APROBADO por TC
6. OBRA en EJECUCIÓN
7. OBRA TERMINADA

Este catálogo hoy se edita a mano en la solapa `Validaciones` — pasa a ser administrable por un usuario Admin desde el sistema (§6).

**Mi Lugar — ítem 14 ("Proyectos de infraestructura a realizar")**: se abre en 6 sub-ítems, cada uno independiente y con estado propio (confirmado — no alcanza una sola casilla para el ítem completo). Labels exactos, tomados de `Base TOTAL` fila 2, columnas 94-99:
1. Red Vial
2. Red de Agua Potable
3. Red Energía Eléctrica — Media y Baja Tensión
4. Red de Alumbrado Público
5. Nexo de Agua
6. Nexo Eléctrico

**Convenio (monto, expediente, N° de orden)**: de solo lectura para el área técnica — "en el área técnica no determinamos esos valores, ya vienen establecidos cuando nos llega el expediente" (H). Confirmado en la exploración: ya existen y son editables hoy vía `EditModal` de `CordonCunetaPage.tsx`/`CordobaHogarPage.tsx` — el panel nuevo los lee de ahí, no los duplica.

**Repartición**: no es un catálogo fijo hoy (no existe como tal en `Validaciones`) — puede variar por programa y debe administrarlo un usuario Admin.

**Roles y permisos**: los jefes del área definirán, más adelante, la granularidad completa (quién carga datos, quién administra estados/hitos/reparticiones, quién obtiene informes). Esta entrega no la implementa — ver §8.

## 4. Ítems de checklist por programa (etiquetas hardcodeadas, no administrables)

### Cordón Cuneta y Adoquinado — 9 ítems + 10 sub-ítems del ítem 4
(Idénticos a los ya usados por el sync existente, `spec-sync-cc-checklist-tecnico.md §3.1` — no se reinventan.)
1. Nota de Solicitud de Financiamiento
2. Ordenanza y Decreto / Resolución comunal
3. DDJJ compromiso de ejecución de obra
4. Proyecto, Planos, Cómputo y Presupuesto *(se abre en 10 sub-ítems, ver abajo)*
5. Nota a Contaduría — Cesión de Coparticipación
6. N° de CBU de M/C especial para depósito de fondos
7. N° de CUIT Municipal / Comunal
8. DNI Intendente / Jefe Comunal
9. Acta de Proclamación Int. / Pres. Comunal

Sub-ítems del ítem 4: Descripción General, Memoria Técnica, Plazo de Ejecución, Cómputo y Presupuesto, Cronograma de Avance de Obra, Planimetría, Perfil de Calzada, Detalle de Cordón Cuneta, Detalle de Badén, Paquete estructural de Calzada.

### Córdoba Hogar — 14 ítems (planos)
Planimetría (Ubicación de Obra), Matrícula de cada lote, Estudio de Suelo, Certificado de No Inundabilidad, Factibilidad de Agua Potable, Factibilidad de Energía Eléctrica, Designación de Director Técnico de Obra, Memoria Técnica descriptiva, Planos Legajo Técnico, Pliego de Especificaciones Técnicas, Cómputo y Presupuesto, Acta de Medición - Certificado, Curva de Avance, Cronograma de Avance.

### Mi Lugar — 13 ítems + ítem 14 con 6 sub-ítems
Planimetría General, Plano de Altimetría, Plancheta Catastral, Títulos, Certificado de No Inundabilidad, Informe de Ministerio de Ambiente y Economía Circular, Factibilidad de Agua Potable, Factibilidad de Energía Eléctrica, Certificado de Prestación de Servicios, Factibilidad de Red Cloacal / Pozo Absorbente, Designación de Representante Técnico, Plano de Loteo Aprobado y Protocolizado, Ordenanza / Resolución, **14. Proyectos de infraestructura a realizar** *(sub-ítems en §3)*.

## 5. Modelo de datos

Prefijo `viv_checklist_` (distinto de `viv_cc_` del sync existente, para no confundirlos). Migración Alembic nueva, siguiente número disponible tras `0021_add_tipo_estados_monto_muni.py`.

### 5.1 `viv_checklist_estado_expediente` — catálogo compartido, administrable
```
id       BIGINT PK
label    VARCHAR(100) NOT NULL
orden    INTEGER NOT NULL
activo   BOOLEAN NOT NULL DEFAULT true
```
Seed: los 7 valores de §3, en orden.

### 5.2 `viv_checklist_reparticion` — catálogo administrable
```
id        BIGINT PK
programa  VARCHAR(2) NULL CHECK (programa IN ('cc','ch','ml'))   -- NULL = aplica a los 3
label     VARCHAR(200) NOT NULL
orden     INTEGER NOT NULL
activo    BOOLEAN NOT NULL DEFAULT true
```
Seed inicial: 3 valores de ejemplo (`Dirección de Regularización de Obras y Proyectos`, `Dirección Legal y Notarial`, `Área Coordinación Administrativa`) con `programa = NULL`. Es un punto de partida — no hay diferenciación real por programa en la fuente hoy; el catálogo queda abierto para que un Admin lo ajuste.

### 5.3 `viv_checklist_tecnico` — fila 1:1 por entidad existente
```
id                    UUID PK
programa              VARCHAR(2) NOT NULL CHECK (programa IN ('cc','ch','ml'))
entidad_id            VARCHAR(36) NOT NULL   -- id en viv_cordon_cuneta / viv_cordoba_hogar / viv_ml_proyectos según programa
estado_expediente_id  BIGINT NULL REFERENCES viv_checklist_estado_expediente(id)
fecha_radicacion      DATE NULL
reparticion_id        BIGINT NULL REFERENCES viv_checklist_reparticion(id)
created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
updated_by            VARCHAR(200) NULL

UNIQUE (programa, entidad_id)
```
`entidad_id` no lleva FK real de Postgres (apunta a 3 tablas distintas según `programa` — mismo patrón polimórfico que ya usa `viv_ml_proyectos.tipo`). `service.py` valida que la entidad exista en la tabla correspondiente antes de escribir.

### 5.4 `viv_checklist_items`
```
id             UUID PK
checklist_id   UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
item_num       SMALLINT NOT NULL      -- secuencia propia por programa, ver §4
sub_item_num   SMALLINT NULL          -- solo CC-ítem4 y ML-ítem14
valor          VARCHAR(30) NOT NULL CHECK (valor IN ('sin_presentar','eval_tecnica','a_corregir','eval_juridica','completo'))

UNIQUE (checklist_id, item_num, sub_item_num)
```

### 5.5 `viv_checklist_obra_hitos` — solo `programa = 'cc'` en esta entrega
```
id                 UUID PK
checklist_id       UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
tipo               VARCHAR(10) NOT NULL CHECK (tipo IN ('anticipo','40','70','100'))
monto              NUMERIC(18,2) NULL    -- calculado server-side sobre el convenio, no editable por el usuario
fecha_acreditado   DATE NULL

UNIQUE (checklist_id, tipo)
```
Fórmula de montos (confirmada empíricamente sobre la planilla): Anticipo = 50% del convenio, Avance 40% = 25%, Avance 70% = 25%, Avance 100% = saldo (0% fijo, el resto ya fue pagado). Se recalcula en cada `GET` a partir de `viv_cordon_cuneta.monto` — no se persiste un valor que pueda desincronizarse del convenio real.

## 6. Endpoints

Base: `/api/v1/vivienda/checklist-tecnico`

```
GET    /catalogos                                    # estado_expediente + reparticion + definición de ítems por programa (§4, hardcodeado)
GET    /entidades                                     # [{programa, id, nombre, departamento}] de CC+CH+ML no borradas — para el selector (v1.1.0)
GET    /{programa}/{entidad_id}                       # checklist + items + hitos; crea la fila padre on-the-fly si no existe (estado_expediente_id NULL)
PATCH  /{programa}/{entidad_id}                        # body: estado_expediente_id? / fecha_radicacion? / reparticion_id?
PATCH  /{programa}/{entidad_id}/items/{item_num}        # body: valor, sub_item_num?
PATCH  /{programa}/{entidad_id}/hitos/{tipo}            # body: fecha_acreditado — solo programa=cc, 404 en ch/ml
GET    /{programa}/{entidad_id}/pedidos                 # observaciones de la entidad (v1.1.0)
POST   /{programa}/{entidad_id}/pedidos                 # body: descripcion, fecha_pedido (v1.1.0)

GET    /admin/estado-expediente                         # catálogo completo (incluye inactivos)
POST   /admin/estado-expediente
PATCH  /admin/estado-expediente/{id}

GET    /admin/reparticion
POST   /admin/reparticion
PATCH  /admin/reparticion/{id}
```

**Selector y observaciones (v1.1.0, corrige la 1.0.0):** `GET /entidades` y
`GET|POST /{programa}/{entidad_id}/pedidos` viven en este módulo y usan sus constantes locales
(`ROLES_LECTURA_CHECKLIST` / `ROLES_ESCRITURA_CHECKLIST`, §8). La 1.0.0 planteaba reutilizar los
`GET` de panel completo de `cordon_cuneta`/`cordoba_hogar`/`mi_lugar`, pero §8 los veda a
`TecnicoDGV` con `403` — así que el selector y las observaciones nunca funcionaban para el único
rol que abre esta pantalla. `/entidades` devuelve solo `{programa, id, nombre, departamento}`
(el resto del dato de convenio sigue viniendo de `GET /{programa}/{entidad_id}`). Los `pedidos`
delegan en el `service.py` del programa correspondiente (misma validación de entidad 404 y mismo
enmascarado de comunicaciones de infra/supervisión), pasando el actor `TecnicoDGV` tal cual.

Cada path nuevo necesita su `options:` con `security: []` en `infra/gateway/openapi.yaml` (CORS preflight), siguiendo la convención del resto de `vivienda`.

## 7. Semilla de datos para Cordón Cuneta

El sync de Excel ya pobló `viv_cc_checklist_tecnico`/`viv_cc_checklist_items` (solo lectura, sigue corriendo). Para que las localidades de CC no arranquen en blanco en el panel nuevo, la migración que crea las tablas de §5 hace, en un paso posterior al `upgrade`:

1. Por cada fila de `viv_cc_checklist_tecnico` con `municipio_id IS NOT NULL`: insertar en `viv_checklist_tecnico` (`programa='cc'`, `entidad_id=municipio_id`), mapeando `estado_expediente`/`reparticion` (texto) al `id` correspondiente en los catálogos nuevos por label normalizado (trim + comparación case-insensitive).
2. Por cada una de las 19 filas de `viv_cc_checklist_items` de esa localidad: insertar en `viv_checklist_items` con el mismo `item_num` (la numeración 1-19 ya es una secuencia propia estable, definida en el mapeo del sync — coincide con la numeración de este spec para CC).
3. Es un backfill de una sola vez, ejecutado dentro de la migración — no un proceso recurrente. El sync de Excel sigue escribiendo en sus propias tablas (`viv_cc_checklist_tecnico`), sin relación con las tablas nuevas después de este punto.

Córdoba Hogar y Mi Lugar no tienen tracking previo en base — arrancan sin filas en `viv_checklist_tecnico` (se crean on-the-fly al primer `GET`/`PATCH` de cada localidad).

## 8. Permisos

Rol nuevo: `TecnicoDGV`, agregado a `ROLES_VALIDOS` en `app/portal/schemas.py`.

**Importante**: `app/auth.py` define constantes compartidas (`ROLES_LECTURA`, `ROLES_ESCRITURA`, etc.) que ya importan tal cual `cordon_cuneta/router.py`, `cordoba_hogar/router.py`, `mi_lugar/router.py` y `programas/router.py`. Agregar `TecnicoDGV` a esas constantes compartidas lo filtraría automáticamente a los 3 paneles completos — lo opuesto de lo aprobado por H. Por eso:

- `checklist_tecnico/router.py` define constantes **locales**: `ROLES_LECTURA_CHECKLIST = ROLES_LECTURA + ("TecnicoDGV",)`, `ROLES_ESCRITURA_CHECKLIST = ROLES_ESCRITURA + ("TecnicoDGV",)`. Catálogos admin (`/admin/...`) usan `ROLES_ADMIN` sin agregar `TecnicoDGV`.
- `programas/router.py` (Tablero/KPIs) cambia su `require_roles(*ROLES_LECTURA)` por una constante local `ROLES_LECTURA_TABLERO = ROLES_LECTURA + ("TecnicoDGV",)`.
  **(v1.1.1)** Además expone `GET /api/v1/vivienda/programas-tablero` con esa misma constante:
  KPIs agregados de CC/CH/ML (`app/programas/tablero.py`, replica 1:1 el cálculo que hacía el
  frontend). El `ProgramasPage.tsx` pasa a consumir ese endpoint en vez de pedir los 5 `GET`
  de panel completo, que le daban `403` a `TecnicoDGV`. `GET /programas` (catálogo genérico de
  `viv_programas`) no cambia y no alcanzaba para los KPIs.
- `cordon_cuneta/router.py`, `cordoba_hogar/router.py`, `mi_lugar/router.py`: **sin cambios** — al no agregarse `TecnicoDGV` a las constantes compartidas de `app.auth`, quedan automáticamente vedados para ese rol (`403 PERMISO_INSUFICIENTE`).

Frontend (`Layout.tsx`): `SECRETARIA_NAV['vivienda']` deja de ser un array fijo — se filtra según `portalUser.rol`; `TecnicoDGV` ve solo `Tablero` y `Checklist Técnico`.

## 9. Criterios de aceptación

- [ ] Migración aplicada: 5 tablas nuevas + catálogos sembrados (7 estados, 3 reparticiones) + backfill de CC ejecutado.
- [ ] `GET /checklist-tecnico/{programa}/{entidad_id}` crea la fila padre on-the-fly la primera vez, sin 404.
- [ ] Actualizar un ítem, el estado del expediente, la repartición o un hito persiste sin necesidad de un botón "Guardar" explícito (autosave).
- [ ] Los montos de hitos de obra se recalculan sobre `viv_cordon_cuneta.monto` vigente, nunca se editan directamente.
- [ ] Usuario con rol `TecnicoDGV`: `200` en `checklist-tecnico` y en `programas` (Tablero); `403` en `cordon-cuneta`, `cordoba-hogar`, `mi-lugar`.
- [ ] Usuario con rol `TecnicoDGV`: `200` en `GET /checklist-tecnico/entidades` (las 3 fuentes) y en `GET|POST /checklist-tecnico/{programa}/{entidad_id}/pedidos` — el selector y las observaciones del panel cargan sin `403` (v1.1.0).
- [ ] Usuario con rol `TecnicoDGV`: `200` en `GET /api/v1/vivienda/programas-tablero`; el Tablero de Programas renderiza sus KPIs sin `403` (v1.1.1).
- [ ] Usuario con rol `TecnicoDGV` en el frontend: la navegación de `vivienda` muestra solo Tablero y Checklist Técnico.
- [ ] Catálogos de estado del expediente y repartición: editables solo por `Admin` (`403` para el resto de los roles, incluido `TecnicoDGV`).
- [ ] Las 54 localidades de Cordón Cuneta con dato en el sync aparecen con su checklist pre-cargado tras el backfill; Córdoba Hogar y Mi Lugar arrancan vacíos.
- [ ] `viv_cc_checklist_tecnico`/`viv_cc_checklist_items` y su Cloud Scheduler siguen funcionando sin cambios (regresión de `spec-sync-cc-checklist-tecnico.md`).
- [ ] Tests unitarios de `checklist_tecnico` + test de regresión de permisos (§8) + test de `test_cc_checklist_sync.py` sigue en verde.
- [ ] `npm run build` sin errores con la página nueva y el nav condicional.
