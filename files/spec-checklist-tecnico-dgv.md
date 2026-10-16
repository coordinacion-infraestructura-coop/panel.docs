# Spec: Checklist Técnico DGV — panel editable por localidad y programa

**Estado**: approved
**Versión**: 1.7.0
**Servicio**: `svc-vivienda` (módulo nuevo `checklist_tecnico`, sin servicio nuevo)
**Responsable de spec**: Pedro Bonafe (revisado sección por sección con el usuario, 2026-08-26)
**Última actualización**: 2026-10-16

### Changelog
- **1.7.0 (2026-10-16, aprobado por el usuario)** — Pedidos del área técnica sobre el panel (§6-ter):
  - **Documentación en dos botones (solo Cordón Cuneta):** "Checklist técnico" (los 10 sub-ítems
    del ítem 4, que carga el área técnica) y "Checklist legal y financiero" (los ítems 1 a 9,
    incluido el 4 como ítem general, que carga el área legal y financiera). Lo editan personas
    distintas. Córdoba Hogar y Mi Lugar siguen con una sola lista hasta que el área defina su división.
  - **Corrección de observaciones ya cargadas** (del expediente y de obra), con un lápiz como el de
    "Detalle" en Privada. Corrigen los mismos roles que cargan, quedan marcadas como "(editado)" y
    el texto anterior queda en auditoría. Migración `0041` y dos paths nuevos en el gateway.
- **1.6.1 (2026-10-16)** — Corrección: una localidad con varios expedientes del mismo programa
  mostraba uno solo en el selector (el frontend agrupaba por localidad y guardaba un expediente por
  programa). En producción dejaba 18 proyectos de Mi Lugar sin acceso, repartidos en 10 localidades
  (ej. 8 en Córdoba Capital). Ahora, si hay más de uno, debajo de la pestaña del programa aparecen
  botones para elegir el expediente. Además la página acepta `?programa=cc|ch|ml&entidad=<id>` para
  abrir un expediente puntual, que usa el enlace del Tablero analítico
  (`spec-vivienda-tablero-analitico.md` v1.1.0). Sin cambios de backend.
- **1.6.0 (2026-10-15, approved)** — Pedidos del área de Vivienda
  (`docs/context/areas/secretaria_Vivienda/Observaciones y pedidos para la App GestorCoop 2026 10 02.pdf`,
  §2 "Incorporaciones": *Obra en ejecución* y *Estados*). El resto de los pedidos del documento
  queda fuera de esta versión (se evalúan aparte). Migración `0039`. Decisiones tomadas con el
  usuario el 2026-10-15:
  1. **Avance físico de obra — historial de mediciones** (§5.6, tabla nueva
     `viv_checklist_avance_medicion`): cada carga del área técnica agrega una fila
     (fecha de medición + % de 0 a 100 + usuario). La "barra de avance" muestra la medición de
     fecha más reciente. Se descartó el valor único que se pisa (se perdía el ritmo de obra).
  2. **La fecha de medición de los hitos 40% / 70% / 100% se deriva del historial** (§5.5): es la
     fecha de la primera medición que alcanzó ese umbral. Separa medición (área técnica) de pago
     (`fecha_acreditado`, área económica), que es lo que pide la pág. 3 del PDF — sin campo nuevo
     por hito ni doble carga. El Anticipo financiero no tiene medición.
  3. **Fecha de finalización de obra** (`viv_checklist_tecnico.fecha_fin_obra`, §5.3): manual,
     autosave, independiente. Si se carga con la última medición < 100% la UI avisa, pero no
     bloquea. **No** cambia el "Estado del expediente" (`OBRA TERMINADA` sigue siendo manual).
  4. **Estado de documentación "No Proyectado / No Aplica"** (§3, §5.4-bis): nueva fila del
     catálogo con un flag nuevo `excluye_completitud`. Un ítem en ese estado **no cuenta ni como
     faltante ni en el total** al medir si la documentación está completa (ej.: una localidad
     que sólo hace adoquines no debe arrastrar los ítems de cordón cuneta como pendientes para
     siempre). Aplica a los 3 programas (catálogo compartido).
  5. **Avance visible en los paneles CC / CH / ML** (§6-bis), siempre de **solo lectura**:
     columna en la tabla (mini barra + fecha de medición), columnas en el Excel exportado,
     barra + fecha de fin en el `DetailPanel` (en CC, dentro de la pestaña Checklist Técnico) y
     filtro cliente por situación de obra (sin medición / en ejecución / terminada).
  6. **Hitos de obra en los listados** (PDF pág. 4, "Listados": "que salga la información de
     avance de obra cuando la obra ya está iniciada — Anticipo Financiero, 40%, 70%, 100%"):
     los paneles CC / CH / ML muestran, también de solo lectura, el estado de los 4 hitos
     (medido / acreditado) en una columna compacta, en el Excel y en el detalle (§6-bis).
  7. Gateway: 2 paths nuevos (`.../avance`, `.../avance/{medicion_id}`) → nueva config
     `ministerio-config-v{YYYYMMDD}`.
- **1.5.1 (2026-09-25)** — Resuelto el pendiente de la 1.5.0: se decidió dar de baja el sync
  viejo (no restaurar el acceso al Sheet). Se eliminó el Cloud Scheduler
  `sync-cc-checklist-tecnico` (`gcloud scheduler jobs delete`) y se sacó del panel de Cordón
  Cuneta el indicador "Checklist técnico: sincronizado hace X" + botón "🔄 Actualizar ahora"
  (apuntaban al sync roto). **No se tocaron** los endpoints (`GET/POST /cordon-cuneta-checklist-tecnico/*`,
  `GET /cordon-cuneta/{municipio_id}/checklist-tecnico`), `checklist_sync.py`, ni las tablas
  `viv_cc_checklist_tecnico`/`viv_cc_checklist_items`/`viv_cc_sync_log` — quedan como código y
  datos muertos pero inofensivos, recuperables si hiciera falta retomar el espejo del Sheet.
- **1.5.0 (2026-09-25)** — Se retira el espejo viejo de Google Sheet ("Checklist Técnico" dentro
  del panel de Cordón Cuneta, `spec-sync-cc-checklist-tecnico.md`): el Sheet `DGV Programas 2026`
  perdió el acceso compartido (con la cuenta que usa el equipo y, con altísima probabilidad, con
  `svc-vivienda@gestorcooperativo.iam.gserviceaccount.com`) en algún momento entre el 2026-09-18
  14:30 y 14:45 UTC — el sync automático (`sync-cc-checklist-tecnico`, cada 15 min) viene
  fallando el 100% de las corridas desde entonces con `SheetReadError` → 502, sin que nadie lo
  notara. Coincide con que el área técnica migró a cargar los datos acá, en este módulo, en vez
  de en esa planilla.
  1. **La pestaña "Checklist Técnico" de `CordonCunetaPage.tsx` ahora lee de este módulo**
     (`GET /checklist-tecnico/cc/{municipio_id}` — mismo id que `viv_cordon_cuneta.id`, sin
     tabla de vínculo nueva) en vez del viejo `GET /cordon-cuneta/{municipio_id}/checklist-tecnico`
     (`viv_cc_checklist_tecnico`, sync de Sheet). Es un **resumen de solo lectura** para el área
     administrativa — se sigue editando únicamente desde `/vivienda/checklist-tecnico`. Sin
     cambios de backend ni de gateway: reutiliza endpoints ya existentes desde v1.0.0-v1.4.0.
  2. Pendiente de decisión (no resuelto en esta versión, ver `auditoria-codigo.md` #29):
     restaurar el acceso al Sheet viejo (si todavía hace falta ese espejo para algo más) vs.
     dar de baja el Cloud Scheduler `sync-cc-checklist-tecnico` y el indicador "🔄 Actualizar
     ahora" del panel de Cordón Cuneta, que siguen apuntando al sync roto.
- **1.4.0 (2026-09-09)** — El stepper de "Estado del expediente" ponía ✓ a TODOS los estados
  anteriores al actual, aunque el expediente nunca hubiera estado en ellos. `RECHAZADO por M/C`
  y `SIN AUTORIZACION MIN.GOB` **no son parte obligatoria del camino** — algunos expedientes
  pasan por ahí, otros no. Migración `0027`.
  1. **`viv_checklist_estado_expediente.en_ruta`** (BOOLEAN, default `true`): `false` = estado
     de excepción. Editable por Admin. Seed: `false` en los 2 de arriba.
  2. **`viv_checklist_estado_hist`**: registra cada cambio de estado del expediente. Backfill
     desde `viv_audit_log` + el estado actual. `GET .../{programa}/{entidad_id}` devuelve
     `estados_visitados: int[]` (estados por los que pasó de verdad).
  3. **Frontend**: los estados `en_ruta=false` se muestran **sueltos, a la izquierda del camino
     numerado, sin conectores**, y se marcan ✓ **solo si `estados_visitados` los incluye** (o si
     es el estado actual). El camino regular (7 pasos) mantiene el ✓ por posición (es lineal).
- **1.3.0 (2026-09-08)** — Ajustes de UX pedidos por el área tras probar la v1.2.0.
  Migración `0026`.
  1. **"Observaciones de obra" pasa de campo único a bitácora fechada** (`viv_checklist_obra_obs`,
     misma forma que `viv_*_pedidos`): una entrada por fecha, con el usuario que la cargó, y
     **botón "Guardar" explícito** — se acepta el botón aunque el patrón general del módulo sea
     autosave (pedido directo del área). Endpoints `GET|POST /checklist-tecnico/{programa}/{entidad_id}/obs-obra`.
     El campo `viv_checklist_tecnico.obs_obra` se elimina (su valor se migra a una entrada).
  2. **Ambas bitácoras registran y muestran el usuario** que hizo cada observación
     (`created_by` / `created_by_nombre`). La sección lateral se renombra a
     **"Observaciones del expediente"**; la de abajo queda como **"Observaciones de obra"**.
  3. Fixes de UI (auditoría, sin cambio de contrato): el desplegable de estado de un ítem ya no
     lo tapa la tarjeta siguiente (menú en `position:fixed`, se abre hacia arriba si no hay
     lugar); el `<select>` de repartición pasa a control custom que muestra la opción elegida
     completa (con salto de línea); `min-w-0` en el grid; `todayISO()` usa fecha local (no UTC);
     el disclosure "Detalle técnico" es por ítem; `Autoridad` no ve los campos habilitados; los
     catálogos admin muestran errores y no colisionan el `orden`.
- **1.2.0 (2026-09-07)** — Correcciones enviadas por el área técnica de la DGV
  (`docs/context/areas/secretaria_Vivienda/obervaciones AREA TECNICA.pdf` +
  `DGV Programas 2026.xlsx` solapa `Validaciones` filas 403-418). Migración `0024`.
  1. **§3 — "Estado del expediente" pasa de 7 a 9 valores**, renombrando `... en DGV` →
     `... en TÉCNICA` y `... TC` → `... TRIB.C`, y agregando `RECHAZADO por M/C` y
     `SIN AUTORIZACION MIN.GOB`. El catálogo ya era administrable; la migración solo
     reetiqueta in place (preserva los `estado_expediente_id` referenciados) e inserta 2 filas.
  2. **§3/§4/§5.4 — "Estado de la documentación" (por ítem) pasa de enum fijo a catálogo
     administrable** (`viv_checklist_item_estado`, con color `bg`/`text_color` y flag
     `es_completo`). Se **revierte** la decisión "no administrable" de la 1.0.0 — pedido
     explícito del área "para evitar futuros conflictos". `viv_checklist_items.valor` (str)
     → `item_estado_id` (FK). Labels nuevos: `A ESPERA de DOC.TÉCNICA`,
     `DIR. JUR. LEGAL Y NOTARIAL`, `En Evaluación TÉCNICA`, `A corregir por M/C`, `Completo OK`.
     Nuevos endpoints `POST|PATCH /checklist-tecnico/admin/item-estado[/{id}]` (`ROLES_ADMIN`).
  3. **§2/§5.5 — Hitos de ejecución de obra para los 3 programas** (antes solo Cordón Cuneta).
     Proporciones 50/25/25/0 confirmadas sobre el Excel para CC y CH; para ML se asume la misma
     a falta de datos de referencia.
  4. **§5.3 — `viv_checklist_tecnico.obs_obra`** (TEXT): observaciones de la etapa de obra
     (columna AT del Excel), un campo de texto único por localidad, separado de las
     observaciones del expediente (que siguen en `viv_*_pedidos`).
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

Reemplazar la edición manual del Excel por un panel dentro del sistema: selector de localidad y programa → checklist de documentación con estado por ítem → "Estado del expediente" (workflow de 9 pasos, v1.2.0) → hitos de ejecución de obra (los 3 programas, v1.2.0) → observaciones (del expediente y de obra, separadas). Con autosave por campo (patrón Notion/Linear), sin el botón "Guardar" que usan hoy Cordón Cuneta y Córdoba Hogar — ese patrón fue, en palabras de H, "la manera casera que encontramos trabajando solo con una hoja de cálculo".

## 2. Alcance

### Incluido
- Checklist de documentación por localidad y programa (CC: 9 ítems + 10 sub-ítems del ítem 4; CH: 14 ítems; ML: 13 ítems + 6 sub-ítems del ítem 14), 5 valores posibles por ítem.
- "Estado del expediente de la localidad": catálogo de 9 valores (v1.2.0), compartido entre los 3 programas, administrable por un usuario Admin.
- "Repartición" donde se radica el expediente: catálogo administrable por un usuario Admin, puede variar por programa.
- Fecha de radicación.
- Hitos de ejecución de obra con montos auto-calculados sobre el convenio (Anticipo 50% / Avance 40% → 25% / Avance 70% → 25% / Avance 100% → saldo) — **los 3 programas** (v1.2.0; la 1.0.0 lo limitaba a Cordón Cuneta). CC y CH usan la proporción confirmada en el Excel; ML asume la misma.
- "Observaciones de obra" (`obs_obra`): un campo de texto por localidad, separado de las observaciones del expediente (v1.2.0).
- Nuevo rol `TecnicoDGV`, con navegación restringida a Tablero (KPIs) + Checklist Técnico — no ve Cordón Cuneta, Córdoba Hogar ni Mi Lugar como paneles completos.
- Semilla única de los datos ya sincronizados de Cordón Cuneta (ver §7) para no arrancar en blanco.

### Fuera de alcance (esta entrega)
- Los campos de convenio (expediente, monto, N° de orden) — son de solo lectura para el área técnica; se editan donde ya se editan hoy (`EditModal` de `CordonCunetaPage.tsx`/`CordobaHogarPage.tsx`, campo `viv_ml_proyectos` para Mi Lugar). No se duplican.
- ~~Hitos de obra para Córdoba Hogar y Mi Lugar~~ — incorporados en v1.2.0 (ver §2 "Incluido").
- Retiro del sync Excel→Postgres de Cordón Cuneta (`spec-sync-cc-checklist-tecnico.md`) — sigue corriendo en paralelo, sin tocar.
- Granularidad fina de permisos dentro del área (quién carga datos vs. quién administra estados/hitos/reparticiones vs. quién obtiene informes) — los jefes del área la definirán más adelante; esta entrega usa un único rol nuevo (`TecnicoDGV`) con permisos de lectura+escritura sobre el checklist.
- Integración con `informes/` (el `_COMPUTE_FN` de `app/informes/service.py` no incluye este módulo).

## 3. Definiciones confirmadas con el área (H, 2026-08-26)

Fuente: solapa `Validaciones` de `DGV Programas 2026.xlsx`, verificada directamente con `openpyxl` (filas 401-415) — coincide exactamente con lo que envió H.

**Estado de la documentación (por ítem)** — 5 valores (v1.2.0, labels de `Validaciones!A403-407`).
Desde la v1.2.0 es un **catálogo administrable** (`viv_checklist_item_estado`, §5.4), ya no un
enum fijo:
1. A ESPERA de DOC.TÉCNICA
2. DIR. JUR. LEGAL Y NOTARIAL
3. En Evaluación TÉCNICA
4. A corregir por M/C
5. Completo OK  *(marcado `es_completo` — cuenta como documentación completa en Resumen Territorial / Tablero)*
6. No Proyectado / No Aplica  *(v1.6.0 — marcado `excluye_completitud`: el ítem no aplica a esta
   localidad/proyecto y se descarta al medir completitud — no es faltante ni suma al total)*

**Estado del expediente de la localidad** — 9 valores (v1.2.0, `Validaciones!A410-418`), en este orden:
1. A ESPERA de DOC.TÉCNICA
2. RECHAZADO por M/C
3. SIN AUTORIZACION MIN.GOB
4. En CURSO en TÉCNICA
5. COMPLETO en TÉCNICA
6. En CURSO en TRIB.C
7. APROBADO por TRIB.C
8. OBRA en EJECUCIÓN
9. OBRA TERMINADA

Este catálogo hoy se edita a mano en la solapa `Validaciones` — es administrable por un usuario Admin desde el sistema (§6).

> **Histórico (1.0.0-1.1.1):** el catálogo de documentación era el enum fijo
> `Sin Presentar / En Evaluación Técnica / A corregir por M/C / En Evaluación Jurídico / Completo OK`
> y el de expediente tenía 7 valores (`... en DGV` / `... TC`). Ver changelog 1.2.0.

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

## 4. Ítems de checklist por programa (etiquetas hardcodeadas)

> **v1.2.0:** solo las **etiquetas de los ítems** (las de abajo) siguen hardcodeadas. El
> **estado de cada ítem** ("Estado de la documentación", §3) pasó a ser un catálogo
> administrable (`viv_checklist_item_estado`, §5.4).


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
en_ruta  BOOLEAN NOT NULL DEFAULT true   -- v1.4.0 — false = estado de excepción, fuera del camino lineal
```
Seed: los valores de §3, en orden (9 desde v1.2.0; la migración `0024` reetiqueta los 7
originales in place e inserta 2). `en_ruta=false` en `RECHAZADO por M/C` y
`SIN AUTORIZACION MIN.GOB` (v1.4.0, migración `0027`).

### 5.1-bis `viv_checklist_estado_hist` — historial de estado del expediente (v1.4.0)
```
id                   UUID PK
checklist_id         UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
estado_expediente_id BIGINT NOT NULL REFERENCES viv_checklist_estado_expediente(id)
created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
created_by           VARCHAR(200) NULL
```
Se escribe solo cuando **cambia** el estado del expediente (no en cada PATCH). Permite saber si
un expediente transitó un estado de excepción. `GET .../{programa}/{entidad_id}` agrega
`estados_visitados: int[]` (ids distintos, más el actual). Backfill inicial desde `viv_audit_log`.

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
fecha_fin_obra        DATE NULL              -- v1.6.0 — fecha de finalización al 100%, manual
created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
updated_by            VARCHAR(200) NULL

UNIQUE (programa, entidad_id)
```
`entidad_id` no lleva FK real de Postgres (apunta a 3 tablas distintas según `programa` — mismo patrón polimórfico que ya usa `viv_ml_proyectos.tipo`). `service.py` valida que la entidad exista en la tabla correspondiente antes de escribir.
El campo `obs_obra` de la v1.2.0 fue **reemplazado en v1.3.0** por la bitácora §5.3-bis.

### 5.3-bis `viv_checklist_obra_obs` — bitácora de observaciones de obra (v1.3.0)
```
id                UUID PK
checklist_id      UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
descripcion       TEXT NOT NULL
fecha             DATE NOT NULL
created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
created_by        VARCHAR(200) NULL   -- email del actor
created_by_nombre VARCHAR(255) NULL
```
Misma forma que `viv_*_pedidos` (observaciones del expediente). Distinta de esas —
"son 2 etapas diferentes" (área técnica). Se lista más nueva primero (`fecha DESC, created_at DESC`).
La migración `0026` crea la tabla, migra el valor de `obs_obra` (si lo había) a una entrada y
elimina la columna.

### 5.4 `viv_checklist_items`
```
id             UUID PK
checklist_id   UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
item_num       SMALLINT NOT NULL      -- secuencia propia por programa, ver §4
sub_item_num   SMALLINT NULL          -- solo CC-ítem4 y ML-ítem14
item_estado_id BIGINT NOT NULL REFERENCES viv_checklist_item_estado(id)   -- v1.2.0 (antes: valor VARCHAR con CHECK)

UNIQUE (checklist_id, item_num, sub_item_num)
```

### 5.4-bis `viv_checklist_item_estado` — catálogo administrable (v1.2.0)
```
id          BIGINT PK
label       VARCHAR(120) NOT NULL
orden       INTEGER NOT NULL
activo      BOOLEAN NOT NULL DEFAULT true
bg          VARCHAR(10) NOT NULL DEFAULT '#f1f5f9'    -- color de fondo del chip
text_color  VARCHAR(10) NOT NULL DEFAULT '#64748b'
es_completo BOOLEAN NOT NULL DEFAULT false            -- estado "terminado" para Resumen Territorial / Tablero
excluye_completitud BOOLEAN NOT NULL DEFAULT false    -- v1.6.0 — "no aplica": fuera del total y de los faltantes
CHECK (NOT (es_completo AND excluye_completitud))
```
Seed: los 5 valores de §3 (colores tomados de `STATUS_META` del frontend; `es_completo=true`
solo en `Completo OK`). Un ítem nuevo arranca en el estado activo de menor `orden`.
`resumen_territorial` y `programas-tablero` cuentan "documentación faltante" como
`item_estado_id NOT IN (ids con es_completo)`, en vez del literal `'completo'`.

**v1.6.0 — "No Proyectado / No Aplica".** La migración `0039` agrega la columna y siembra la fila
`No Proyectado / No Aplica` (`excluye_completitud=true`, chip gris neutro `#e2e8f0`/`#475569`)
con `orden` = máximo + 1 — **nunca** el menor, para que no se vuelva el estado por defecto de
un ítem nuevo. Regla de completitud, en todos los lugares que la calculan
(`resumen_territorial/aggregations.items_faltantes`, contadores del frontend del checklist y de
la pestaña de CC, Ficha de localidad):
- `total` = ítems del programa − ítems en un estado con `excluye_completitud`.
- `faltan` = ítems que no están ni en `es_completo` ni en `excluye_completitud`.
- Un checklist con todos sus ítems en "No Aplica" queda `total=0, faltan=0` (se muestra "Sin
  documentación aplicable", no "completo").
- Los ítems nunca cargados siguen contando como faltantes (comportamiento actual sin cambios).

El flag es editable por `Admin` desde la pantalla de catálogos (como `es_completo`); el `CHECK`
impide marcar un mismo estado con ambos flags (`422` desde la API).

### 5.6 `viv_checklist_avance_medicion` — historial de avance físico de obra (v1.6.0)
```
id                UUID PK
checklist_id      UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
fecha_medicion    DATE NOT NULL
porcentaje        SMALLINT NOT NULL CHECK (porcentaje BETWEEN 0 AND 100)
created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
created_by        VARCHAR(200) NULL
created_by_nombre VARCHAR(255) NULL
deleted_at        TIMESTAMPTZ NULL      -- soft delete: corregir una carga errónea
```
Misma forma que la bitácora de obra (§5.3-bis). Botón **"Agregar"** explícito (es una bitácora,
igual que "Observaciones de obra" — no autosave). Reglas:
- `fecha_medicion` no puede ser futura (`422`). No se exige que el % sea creciente: si una carga
  baja respecto de la anterior la UI avisa ("menor que la medición del dd/mm") pero se acepta
  (una re-medición puede corregir un valor sobreestimado).
- **Avance vigente** = la medición no borrada de `fecha_medicion` más reciente (desempate por
  `created_at`). Sin mediciones → avance `null` ("Sin medición"), distinto de 0%.
- **Medición de hitos** (derivada, no persistida): para los hitos `40`, `70`, `100`,
  `fecha_medicion` = `min(fecha_medicion)` entre las mediciones no borradas con
  `porcentaje >= umbral`; `null` si ninguna lo alcanzó. `anticipo` no tiene medición (es
  financiero). Se expone en cada hito del `GET` (§6) junto a `fecha_acreditado`.
- Borrar es soft delete, permitido a `ROLES_ESCRITURA_CHECKLIST`, con audit log.
- **Situación de obra** (para el filtro de los paneles, calculada en el cliente):
  `fecha_fin_obra` cargada → *Terminada*; si no, avance vigente ≠ null → *En ejecución*;
  si no → *Sin medición*.

### 5.5 `viv_checklist_obra_hitos` — los 3 programas (v1.2.0; antes solo `cc`)
```
id                 UUID PK
checklist_id       UUID NOT NULL REFERENCES viv_checklist_tecnico(id) ON DELETE CASCADE
tipo               VARCHAR(10) NOT NULL CHECK (tipo IN ('anticipo','40','70','100'))
monto              NUMERIC(18,2) NULL    -- calculado server-side sobre el convenio, no editable por el usuario
fecha_acreditado   DATE NULL

UNIQUE (checklist_id, tipo)
```
Fórmula de montos: Anticipo = 50% del convenio, Avance 40% = 25%, Avance 70% = 25%,
Avance 100% = saldo (0% fijo). Confirmada sobre la planilla para CC y CH (`=$J*0.5` etc. y
`=$AV*0.5` etc.); para ML se asume la misma proporción (sin columnas de obra en el Excel).
Se recalcula en cada `GET` a partir del `monto` de la entidad del programa
(`viv_cordon_cuneta` / `viv_cordoba_hogar` / `viv_ml_proyectos`) — no se persiste un valor
que pueda desincronizarse del convenio real. `fecha_acreditado` se carga solo tras la visita
de obra → puede quedar `NULL`.
**v1.6.0:** `fecha_acreditado` es la fecha de **pago** (área económica). La fecha de
**medición** de cada hito de avance se deriva del historial de §5.6 — la tarjeta del hito
muestra ambas por separado ("Medido dd/mm" / "Pendiente de medición" y "Acreditado dd/mm" /
"Pendiente de pago"). Sobre los hitos se dibuja la barra de avance vigente con marcas en 40/70/100.

## 6. Endpoints

Base: `/api/v1/vivienda/checklist-tecnico`

```
GET    /catalogos                                    # estado_expediente + reparticion + items_estado (v1.2.0) + definición de ítems por programa (§4)
GET    /entidades                                     # [{programa, id, nombre, departamento}] de CC+CH+ML no borradas — para el selector (v1.1.0)
GET    /{programa}/{entidad_id}                       # checklist + items + hitos (v1.2.0: hitos para los 3 programas); crea la fila padre on-the-fly
PATCH  /{programa}/{entidad_id}                        # body: estado_expediente_id? / fecha_radicacion? / reparticion_id? / fecha_fin_obra? (v1.6.0)
GET    /{programa}/{entidad_id}/avance                 # historial de mediciones, más nueva primero (v1.6.0)
POST   /{programa}/{entidad_id}/avance                 # body: fecha_medicion, porcentaje (v1.6.0)
DELETE /{programa}/{entidad_id}/avance/{medicion_id}   # soft delete (v1.6.0)
PATCH  /{programa}/{entidad_id}/items/{item_num}        # body: item_estado_id (v1.2.0), sub_item_num?
PATCH  /{programa}/{entidad_id}/hitos/{tipo}            # body: fecha_acreditado — v1.2.0: los 3 programas
GET    /{programa}/{entidad_id}/pedidos                 # observaciones DEL EXPEDIENTE de la entidad (v1.1.0)
POST   /{programa}/{entidad_id}/pedidos                 # body: descripcion, fecha_pedido (v1.1.0)
GET    /{programa}/{entidad_id}/obs-obra                # observaciones DE OBRA — bitácora fechada con usuario (v1.3.0)
POST   /{programa}/{entidad_id}/obs-obra                # body: descripcion, fecha (v1.3.0)

GET    /admin/estado-expediente                         # catálogo completo (incluye inactivos)
POST   /admin/estado-expediente
PATCH  /admin/estado-expediente/{id}

GET    /admin/reparticion
POST   /admin/reparticion
PATCH  /admin/reparticion/{id}

POST   /admin/item-estado                               # catálogo "Estado de la documentación" (v1.2.0)
PATCH  /admin/item-estado/{id}
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

**v1.6.0 — cambios de contrato aditivos** (ningún campo existente cambia ni desaparece):
- `GET /{programa}/{entidad_id}` agrega `fecha_fin_obra`, `avance: {porcentaje, fecha_medicion} | null`
  (vigente) y, en cada hito, `fecha_medicion` (derivada, `null` en `anticipo`).
- `GET /catalogos` y `/admin/item-estado` agregan `excluye_completitud` a cada estado de ítem;
  `POST|PATCH /admin/item-estado` lo aceptan.
- Gateway: se declaran `/{programa}/{entidad_id}/avance` (get, post, options) y
  `/{programa}/{entidad_id}/avance/{medicion_id}` (delete, options) → config nueva.

## 6-bis. Avance de obra en los paneles CC / CH / ML (v1.6.0)

Mismo mecanismo que el Estado Técnico (`spec-estado-tecnico-desde-checklist.md`): los paneles
**leen** del checklist, nunca escriben. `app/checklist_tecnico/estado_tecnico.py` suma una
función `avance_por_entidad(db, programa) -> {entidad_id: {avance_pct, avance_fecha, fecha_fin_obra, hitos}}`
(una consulta por tabla y por programa — checklist, mediciones, hitos — agrupadas en Python,
sin N+1, nunca crea filas). Los `GET` de lista y detalle de `cordon_cuneta`, `cordoba_hogar` y
`mi_lugar` agregan, como campos de solo lectura:
```
avance_obra_pct     int | null    -- avance vigente (§5.6), null = sin medición
avance_obra_fecha   date | null   -- fecha de esa medición
fecha_fin_obra      date | null
hitos_obra          list | null   -- null si la entidad no tiene fila de checklist; si no, siempre 4,
                                  -- en orden anticipo, 40, 70, 100:
  - tipo              'anticipo' | '40' | '70' | '100'
  - monto             decimal | null   -- misma fórmula 50/25/25/0 de §5.5 sobre el monto del panel
  - fecha_medicion    date | null      -- derivada (§5.6); siempre null en 'anticipo'
  - fecha_acreditado  date | null      -- fecha de pago
```
La fórmula de montos se factoriza en una sola función (hoy vive en `service.py`) que usan tanto
el `GET` del checklist como `avance_por_entidad`, para que el monto de un hito no pueda
diferir entre el checklist y el panel. Un hito sin fila en `viv_checklist_obra_hitos` se
devuelve igual, con fechas `null`.

**Situación de cada hito** (calculada en el cliente, misma regla en las 4 vistas):
`fecha_acreditado` cargada → **Pagado** (verde); si no, `fecha_medicion` cargada → **Medido,
pendiente de pago** (ámbar); si no → **Pendiente** (gris). El anticipo sólo puede estar
Pagado o Pendiente.
Los schemas de create/update de los paneles **no** los aceptan. Frontend (los 3 paneles, se
replica en los hermanos — ver CLAUDE.md "Frontend structure"):
- Columna **"Avance de obra"** en la tabla: mini barra + `%` + fecha de medición; ordenable por %;
  "—" si no hay medición; "✓ Terminada dd/mm" si hay `fecha_fin_obra`.
- Columna **"Hitos"** en la tabla: 4 chips compactos `Ant · 40 · 70 · 100` coloreados según la
  situación de cada hito; tooltip con monto, fecha de medición y fecha de pago. Se muestra
  sólo si la obra está iniciada (hay alguna medición o algún hito con fecha); si no, "—".
- **Filtro** cliente "Obra": Todas / Sin medición / En ejecución / Terminada (regla de §5.6).
- **Excel** (`↓ Exportar`): columnas `Avance obra %`, `Fecha última medición`, `Fecha fin de obra`
  y, por hito, `Anticipo — monto`, `Anticipo — fecha pago`, `40% — monto`, `40% — fecha medición`,
  `40% — fecha pago` (ídem 70% y 100%) — 14 columnas en total, fechas como fecha de Excel.
- **DetailPanel**: barra completa con marcas 40/70/100 + fecha de fin + las 4 tarjetas de
  hito en solo lectura (monto, medido, pagado) — en CC, dentro de la pestaña "Checklist
  Técnico", que ya es el resumen de solo lectura del módulo.

## 6-ter. Secciones de documentación y corrección de observaciones (v1.7.0)

### 6-ter.1 Dos botones: "Checklist técnico" / "Checklist legal y financiero"
- **Clasificación**, fija en `catalog.py` junto a las etiquetas (§4), sin migración:
  - **Cordón Cuneta:** los sub-ítems 4.1 a 4.10 son `tecnica`; los ítems 1 a 9 son
    `legal_financiera`. El ítem 4 como fila general también es `legal_financiera`.
  - **Córdoba Hogar y Mi Lugar:** sin clasificación (`null`). Una sola lista, como hoy.
- **Catálogo:** `ItemDefinicion` e `ItemSubDefinicion` suman `seccion: "tecnica" | "legal_financiera" | null`
  en `GET .../catalogos`. Cambio aditivo.
- **Frontend (`ChecklistCard`):** si el programa tiene ítems con sección, la tarjeta muestra dos
  botones. Cada uno tiene su propio "X de Y completos" y "N no aplican" y lista solo sus ítems:
  - el técnico muestra los sub-ítems abiertos, sin el desplegable;
  - el legal y financiero muestra los ítems 1 a 9.
  
  Arranca en "Checklist técnico". El conteo general del encabezado y el del Resumen Territorial y
  los paneles no cambian: siguen contando toda la documentación.
- **Permisos:** no cambian. Los dos botones son una ayuda de edición, no una restricción por rol;
  `ROLES_ESCRITURA_CHECKLIST` puede editar las dos secciones.

### 6-ter.2 Corrección de observaciones
- **Alcance:** las dos bitácoras del panel:
  - "Observaciones del expediente", que son las comunicaciones de los paneles (`viv_cc_pedidos`,
    `viv_ch_pedidos`, `viv_ml_pedidos`);
  - "Observaciones de obra" (`viv_checklist_obra_obs`).
- **Endpoints nuevos**, con roles `ROLES_ESCRITURA_CHECKLIST`:
  - `PATCH /api/v1/vivienda/checklist-tecnico/{programa}/{entidad_id}/pedidos/{pedido_id}`, body
    `{descripcion?, fecha_pedido?}`;
  - `PATCH /api/v1/vivienda/checklist-tecnico/{programa}/{entidad_id}/obs-obra/{obs_id}`, body
    `{descripcion?, fecha?}`.
- **Reglas:**
  - la observación tiene que pertenecer a esa entidad; si no, 404;
  - descripción no vacía; si no, 422;
  - si no cambia nada, devuelve la fila sin tocarla;
  - las comunicaciones de otra secretaría (`secretaria` = `infraestructura` o `supervision`) no
    se corrigen desde acá y devuelven 403. Solo se corrigen las de Vivienda o sin secretaría.
- **Marca de edición:** la migración `0041` suma `editado_at TIMESTAMPTZ NULL`,
  `editado_por VARCHAR(200) NULL` y `editado_por_nombre VARCHAR(255) NULL` a las cuatro tablas.
  - Las respuestas (`ChecklistPedidoOut`, `ObraObsOut` y, por herencia, `PedidoResponse` de CC/CH y
    `PedidoMLOut`) las devuelven.
  - La bitácora muestra "(editado por X, fecha)".
- **Auditoría:** `log_audit` con `action="CORRECCION"`, el `resource_type` de la tabla y
  `payload = {anterior: {...}, nuevo: {...}}`.
- **Frontend (`BitacoraCard`):** cada entrada muestra un lápiz ✎ si el usuario puede editar. Al
  tocarlo abre un textarea y un campo de fecha en el lugar, con "Guardar corrección" y "Cancelar"
  (Esc cancela, Ctrl+Enter guarda), igual que en Privada. Las comunicaciones de otra secretaría
  no muestran el lápiz.
- **Paneles CC/CH/ML:** sus listas de comunicaciones devuelven también los campos `editado_*`,
  pero la corrección se hace desde el Checklist. Mostrar la marca o el lápiz en los paneles queda
  para otra entrega.
- **Gateway:** dos paths nuevos con su `options`, y config nueva.

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
- [ ] **(v1.2.0)** Migración `0024`: 9 estados de expediente (renombrados + reordenados), tabla
  `viv_checklist_item_estado` con 5 valores (`es_completo` en `Completo OK`),
  `viv_checklist_items.item_estado_id` poblado y `valor` eliminado, `viv_checklist_tecnico.obs_obra`,
  hitos backfill para las filas ch/ml existentes. `alembic downgrade -1` + `upgrade head` reversible.
- [ ] `GET /checklist-tecnico/{programa}/{entidad_id}` crea la fila padre on-the-fly la primera vez, sin 404.
- [ ] Actualizar un ítem, el estado del expediente, la repartición, un hito o las observaciones de obra persiste sin necesidad de un botón "Guardar" explícito (autosave).
- [ ] Los montos de hitos de obra se recalculan sobre el `monto` de la entidad del programa vigente, nunca se editan directamente; la `fecha_acreditado` puede cargarse y volver a vaciarse. **(v1.2.0)** Hitos disponibles en CC, CH y ML.
- [ ] **(v1.2.0)** Catálogo "Estado de la documentación" editable solo por `Admin`
  (`POST|PATCH /admin/item-estado`, `403` para el resto); `resumen_territorial` y
  `programas-tablero` cuentan documentación faltante contra `es_completo`, no contra un literal.
- [ ] Usuario con rol `TecnicoDGV`: `200` en `checklist-tecnico` y en `programas` (Tablero); `403` en `cordon-cuneta`, `cordoba-hogar`, `mi-lugar`.
- [ ] Usuario con rol `TecnicoDGV`: `200` en `GET /checklist-tecnico/entidades` (las 3 fuentes) y en `GET|POST /checklist-tecnico/{programa}/{entidad_id}/pedidos` — el selector y las observaciones del panel cargan sin `403` (v1.1.0).
- [ ] Usuario con rol `TecnicoDGV`: `200` en `GET /api/v1/vivienda/programas-tablero`; el Tablero de Programas renderiza sus KPIs sin `403` (v1.1.1).
- [ ] Usuario con rol `TecnicoDGV` en el frontend: la navegación de `vivienda` muestra solo Tablero y Checklist Técnico.
- [ ] Catálogos de estado del expediente y repartición: editables solo por `Admin` (`403` para el resto de los roles, incluido `TecnicoDGV`).
- [ ] Las 54 localidades de Cordón Cuneta con dato en el sync aparecen con su checklist pre-cargado tras el backfill; Córdoba Hogar y Mi Lugar arrancan vacíos.
- [ ] `viv_cc_checklist_tecnico`/`viv_cc_checklist_items` y su Cloud Scheduler siguen funcionando sin cambios (regresión de `spec-sync-cc-checklist-tecnico.md`).
- [ ] Tests unitarios de `checklist_tecnico` + test de regresión de permisos (§8) + test de `test_cc_checklist_sync.py` sigue en verde.
- [ ] `npm run build` sin errores con la página nueva y el nav condicional.
- [ ] **(v1.6.0)** Migración `0039`: tabla `viv_checklist_avance_medicion`,
  `viv_checklist_tecnico.fecha_fin_obra`, `viv_checklist_item_estado.excluye_completitud` + fila
  "No Proyectado / No Aplica" con `orden` máximo. Probada contra Postgres real (no sólo SQLite);
  `downgrade -1` + `upgrade head` reversible.
- [ ] **(v1.6.0)** `POST .../avance` con `porcentaje` fuera de 0..100 o fecha futura → `422`;
  `DELETE` oculta la medición y el avance vigente pasa a la anterior; `TecnicoDGV` puede
  cargar y borrar; `Consulta`/`Autoridad` → `403` en escritura.
- [ ] **(v1.6.0)** La fecha de medición de los hitos 40/70/100 se deriva correctamente
  (primera medición ≥ umbral; `null` si ninguna) y es independiente de `fecha_acreditado`.
- [ ] **(v1.6.0)** Un ítem en "No Proyectado / No Aplica" no suma al total ni a los faltantes en
  `resumen_territorial`, en el contador del checklist y en la pestaña de CC; todos en "No Aplica"
  → `total=0, faltan=0`. Un estado con ambos flags → `422`.
- [ ] **(v1.6.0)** Los `GET` de CC/CH/ML devuelven `avance_obra_pct`/`avance_obra_fecha`/`fecha_fin_obra`;
  los paneles muestran columna, filtro, Excel y detalle; los schemas de escritura de los
  paneles los ignoran/rechazan.
- [ ] **(v1.6.0)** Los `GET` de CC/CH/ML devuelven `hitos_obra` (4 hitos, mismo monto y
  mismas fechas que `GET /checklist-tecnico/{programa}/{entidad_id}` para la misma entidad;
  `null` sin fila de checklist); la columna "Hitos", el Excel (14 columnas de obra) y el
  detalle los muestran con la regla Pagado / Medido / Pendiente. Sin consultas N+1 (un test
  cuenta las queries del listado).
- [ ] **(v1.6.0)** Gateway con los 2 paths nuevos + `options:` desplegado en config nueva.
