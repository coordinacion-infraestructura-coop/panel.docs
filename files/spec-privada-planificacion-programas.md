# Spec: Planificación de programas por localidad (Privada)

**Estado**: approved
**Versión**: 1.0.0
**Servicios**: `svc-privada` (dueño de los datos y de los endpoints), frontend
(`resumen-territorial`). Lectura IAM-only de `svc-vivienda`, `svc-gasifera`,
`svc-gralgob` y `svc-datos-externos` (§6).
**ADR**: ADR-036 (§11)
**Depende de**: `spec-privada-padron-oficial.md` (implemented — `priv_geo_localidades`
espejo del padrón), `spec-resumen-territorial.md`, `spec-ch-convenio-firmado.md`,
ADR-012, ADR-016/021/022/025
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-19

---

## 0. Por qué existe este spec

La "Tabla general" que está al final de `/resumen-territorial` no se usa. En
paralelo, la Secretaría Privada trabaja **por fuera del sistema** con planillas
por programa. Hay dos ejemplos:

- *"Cordón Cuneta OK.xlsx"*, pestaña **GENERAL** (Drive `1PRSBQ5xOl3Oms42PkHMHx8prF5DAYSdm`,
  analizada el 2026-11-19 y no modificada).
- La planilla de Córdoba Hogar (Sheet `1kS1P12xWPCgB9LU_qs7b_gwKxk2NGHzlnyFzjibAK_E`).
  **Pendiente de análisis**: el conector de Drive no la pudo abrir el
  2026-11-19 (ver §12.5).

En esas planillas se listan todas las localidades por departamento, con una
prioridad de color, los habitantes, si el gobernador las visitó o les anunció
algo, y se va controlando **a cuáles se les asigna qué monto y a cuáles no**.

Objetivo: una **pantalla genérica de planificación por programa**.

- Se elige un programa, de un catálogo que Privada administra, agrupado por área.
- Se ven **todas** las localidades del padrón oficial, ordenadas por la
  prioridad de ese programa.
- Hay columnas fijas que el sistema calcula solo: si la localidad ya está en el
  panel del programa, si tiene convenio firmado, la visita del gobernador, la
  coparticipación y el semáforo político.
- Hay columnas de trabajo editables: prioridad, estado, OK del gobernador, las
  columnas que vienen por defecto y **las columnas que el usuario agregue a mano**.
- Las prioridades, los estados y las columnas **se configuran por programa**
  desde un botón de parámetros en la misma pantalla.

Restricción fijada por el usuario: **los datos son de Privada y no pueden
impactar en `svc-vivienda`**. Ese servicio no recibe tablas, migraciones ni
escrituras (ver §6 sobre la única lectura que hace falta).

## 1. Lo que se relevó de la planilla de Cordón Cuneta (pestaña GENERAL)

| Columna de la planilla | En este spec |
|---|---|
| DEPARTAMENTO, LOCALIDAD | Padrón oficial (`priv_geo_localidades`) |
| (sin título, `1`…`5` / `X`) | **Prioridad del programa**, editable, con catálogo configurable por programa (§4.3) |
| TIPO (`M`/`C`) | `priv_localidades_info.tipo_localidad` (solo lectura) |
| HABITANTES | `priv_localidades_info.habitantes` (solo lectura) |
| ¿VISITA DEL GOBERNADOR? | Calculada, solo lectura (§5.1, fuente según §12.1) |
| ¿ANUNCIO DEL GOBERNADOR? | Columna editable por defecto |
| MONTO / MONTO FIRMADO O SUGERIDO | Columnas editables por defecto (`Monto sugerido`, `Monto firmado`) |
| REF GESTOR | Se reemplaza por **"¿Está en el panel?"**, un Sí/No automático (§5.1). No se carga a mano |
| OBSERVACIÓN | Columna editable por defecto |

Las otras pestañas (CELESTE, VERDE-NARANJA, AMARILLO-ROJO, por departamento,
contactos) son **vistas filtradas** del mismo listado. En el sistema se
resuelven con filtros.

## 2. Alcance

**Entra**
- Catálogo de programas a planificar, con área y vínculo opcional a un panel
  existente (CC, CH, ML, Gasífera, ATP).
- Una pantalla propia, accesible desde un **botón visible arriba** en
  `/resumen-territorial`.
- Columnas fijas calculadas, columnas fijas editables, columnas editables por
  defecto y columnas agregadas por el usuario.
- **Configuración de parámetros por programa** (botón en la pantalla):
  catálogo de prioridades, catálogo de estados y columnas.
- **Historial de cambios** por fila, en una columna desplegable.
- Filtros, orden, KPIs y exportación a Excel.
- Carga inicial de las planillas actuales de CC y CH (§9, Fase 5).
- Se saca la "Tabla general" vieja de la página.

**No entra (planificado aparte)**
- **Editar el semáforo político global** (`priv_localidades_info.color_semaforo`).
  En esta etapa **se muestra en solo lectura**. Su edición va a tener un panel
  propio en un spec posterior. ADR-012 queda como está.
- Visibilidad para otras secretarías o para el rol Autoridad.
- Cualquier escritura en los paneles de CC/CH/ML/Gasífera/ATP.

## 3. Acceso

- **Ver y editar valores**: Admin, y los roles `Operador`/`Supervisor` que tengan
  la secretaría `privada`. `Consulta` con `privada` solo ve.
- **Configuración de parámetros** (programas, prioridades, estados, columnas):
  Supervisor y Admin. Eliminar un programa: solo Admin.
- Los demás roles no ven el botón ni la pantalla. **Autoridad queda afuera a
  propósito**, aunque esté en `ROLES_LECTURA`. El backend aplica el gate con
  403. El frontend solo oculta el botón.

## 4. Modelo de datos (`db_privada`, Alembic `0005`)

Todas las tablas llevan `created_*`, `updated_*` y `deleted_at` (soft delete
universal). Cada escritura va además a `priv_audit_log`.

### 4.1 `priv_plan_programas` — catálogo de programas

| Columna | Tipo | Notas |
|---|---|---|
| `id` | serial PK | |
| `nombre` | varchar(200) | Único entre los no borrados |
| `area` | varchar(40) | `vivienda` · `gasifera` · `gralgob` · `privada` · `otra` |
| `fuente` | varchar(40) NULL | `cordon_cuneta` · `cordoba_hogar` · `mi_lugar` · `gas_pit` · `atp`. NULL = sin panel propio |
| `descripcion` | text NULL | |
| `orden` | int | |
| `activo` | bool | |

### 4.2 `priv_plan_estados` — estados por programa (editables)

| Columna | Tipo | Notas |
|---|---|---|
| `id` | serial PK | |
| `programa_id` | FK | |
| `nombre` | varchar(120) | Único dentro del programa |
| `color_bg`, `color_texto` | varchar(9) | Mismo esquema de colores que `viv_*_estados` |
| `orden` | int | |

### 4.3 `priv_plan_prioridades` — prioridades por programa (editables)

Misma forma que `priv_plan_estados`, más `codigo` (varchar(10), por ejemplo
`1`…`5` o `X`) y `excluye` (bool: "desestimado"; la fila se ordena al final y
se puede ocultar con un filtro). Cuando se crea un programa, se siembra la
escala de la planilla actual: 1 celeste, 2 verde, 3 naranja, 4 amarillo,
5 rojo y X desestimado. Desde la configuración se puede renombrar, recolorear,
reordenar, agregar o borrar.

**Borrar una prioridad o un estado que está en uso**: es un soft delete. Las
filas que lo tenían lo siguen mostrando tachado con la marca "(eliminado)"
hasta que alguien lo cambie. Nunca se reasignan en silencio.

### 4.4 `priv_plan_columnas` — columnas editables por programa

| Columna | Tipo | Notas |
|---|---|---|
| `id` | serial PK | |
| `programa_id` | FK | |
| `clave` | varchar(60) | slug estable, se genera del nombre al crear y no cambia si se renombra |
| `nombre` | varchar(120) | |
| `tipo` | varchar(20) | `texto` · `texto_largo` · `numero` · `monto` · `si_no` · `fecha` · `lista` |
| `opciones` | jsonb NULL | Solo para `lista` |
| `orden` | int | |
| `visible` | bool | Ocultar sin perder los datos |

Cada programa nuevo trae **cuatro columnas por defecto**: `Anuncio del
gobernador` (si_no), `Monto sugerido` (monto), `Monto firmado` (monto) y
`Observación` (texto_largo). Funcionan igual que las que agrega el usuario: se
pueden renombrar, ocultar o reordenar.

Cambiar el `tipo` de una columna que ya tiene datos: **no se permite** (409).
Hay que crear otra columna. Así se evita perder datos convirtiendo tipos.

### 4.5 `priv_plan_localidad` — una fila por (programa, localidad)

| Columna | Tipo | Notas |
|---|---|---|
| `id` | serial PK | |
| `programa_id` | FK | |
| `id_geo` | FK → `priv_geo_localidades.id_geo` | Padrón oficial, nunca texto libre (ADR-026) |
| `prioridad_id` | FK NULL → `priv_plan_prioridades` | |
| `estado_id` | FK NULL → `priv_plan_estados` | |
| `ok_gobernador` | bool NULL | Ver §5.2 |
| `valores` | jsonb | `{clave_columna: valor}`, validado contra `priv_plan_columnas.tipo` en `service.py` |
| `updated_at` | | Lock optimista → `409` |

UNIQUE parcial `(programa_id, id_geo) WHERE deleted_at IS NULL`. La fila se
crea **al primer cambio** (upsert). Una localidad sin fila se muestra igual, con
los campos editables vacíos.

### 4.6 `priv_plan_historial` — historial visible por fila

`id`, `programa_id`, `id_geo`, `campo` (`prioridad`, `estado`, `ok_gobernador`
o la `clave` de una columna), `valor_anterior` (text), `valor_nuevo` (text,
legible: el nombre del estado o de la prioridad, no el id), `usuario`, `fecha`.
Es append-only. Se escribe en la misma transacción que el cambio. Duplica a
propósito una parte de `priv_audit_log`: el audit log es para auditoría, esta
tabla es para leer en pantalla.

## 5. Columnas en pantalla

### 5.1 Fijas calculadas (solo lectura, siempre presentes)

| Columna | Fuente |
|---|---|
| Departamento · Localidad | `priv_geo_localidades` (agrupa y ordena) |
| Tipo (M/C) | `priv_localidades_info.tipo_localidad` |
| Habitantes | `priv_localidades_info.habitantes` |
| Coparticipación anual | **Monto anual que recibe la localidad por coparticipación**: suma del concepto `coparticipacion_ley_8663` (`svc-datos-externos`, `ext_transferencias`, ADR-025) de los **últimos 12 meses publicados**. El tooltip muestra el período (mes desde–hasta). Si hay menos de 12 meses cargados, se muestra lo que hay y se marca "parcial (N meses)" |
| Visita del gobernador | **Única fuente: la planilla "2. RECORRIDO Y VISITAS", pestaña `Visitas Gober`** (`svc-datos-externos`, `ext_gob_visita_gobernador`: `visitada` + `fecha_ultima`, por código INDEC → `id_geo`). Muestra "Sí · dd/mm/aaaa", "Sí (sin fecha)" o "No". Ya no se infiere de ATP (decisión del usuario del 2026-11-19, basada en el análisis de coherencia de §12.1) |
| Semáforo (político) | `priv_localidades_info.color_semaforo`. **Solo lectura en esta etapa** |
| ¿Convenio de Vivienda? | Sí/No: hay **fecha de firma de convenio** en el panel de la `fuente`. CH: `convenio_firmado`/`fecha_firma_convenio` (spec-ch-convenio-firmado). **CC y ML: la columna se muestra vacía ("—") hasta que el área defina qué dato cuenta como firma** (§12.3). No se usa `ok_gob` como sustituto. El tooltip muestra la fecha |
| ¿Está en el panel? | Sí/No: existe una fila no borrada en el panel de la `fuente` con ese `id_geo`. Si es Sí, también muestra el estado general con su color y la cantidad de filas si hay más de una. Las filas de la fuente sin `id_geo` se cuentan en un aviso y no se pierden en silencio |

Las dos últimas solo aparecen si el programa tiene `fuente`. Si no tiene, se
ocultan.

### 5.2 Fijas editables
- **Prioridad**: selector con el color del catálogo del programa (§4.3).
- **Estado**: selector con el catálogo del programa (§4.2).
- **OK gobernador** (Sí/No/—), del programa. Si la `fuente` ya tiene el dato
  (CC/CH tienen `ok_gob`) y la localidad está en el panel, se muestra el valor
  del panel en solo lectura con la marca "del panel". El valor propio de
  Privada se usa para las localidades que todavía no están en el panel.

### 5.3 Columnas editables
Las cuatro por defecto más las que agregue el usuario (§4.4), en el orden
configurado.

### 5.4 Historial de cambios
Columna angosta al final ("🕘", con la cantidad de cambios). Al hacer clic se
despliega debajo de la fila la lista de cambios (fecha, usuario, campo,
antes → después). Se carga bajo demanda, no viene con el listado.

### 5.5 Orden, filtros y KPIs
- Orden por defecto: prioridad (orden del catálogo, después las que excluyen,
  después las sin clasificar), departamento y localidad. Se puede ordenar por
  cualquier columna.
- Filtros: departamento (sincronizado con el "Ir a" del Resumen Territorial) ·
  prioridad (multi) · estado (multi) · ¿está en el panel? · ¿convenio? · visita
  del gobernador · OK gobernador · semáforo · búsqueda por localidad.
- Columnas fijas a la izquierda (departamento/localidad) y scroll horizontal
  para las demás.
- KPIs: localidades por prioridad y por estado, cuántas están en el panel,
  cuántas tienen convenio, y la suma de cada columna `monto` (total y por
  prioridad).

## 6. Fuentes del cruce y el único punto que roza a `svc-vivienda`

Hallazgo del 2026-11-19: el snapshot de `resumen_territorial` **no sirve** para un
usuario que solo tiene Privada. `service.filtrar_por_visibilidad` (svc-vivienda)
le saca todas las líneas cuya `area` no esté en sus secretarías. Además, el
snapshot no trae la fecha de firma del convenio ni `ok_gob`. Por eso el cruce se
arma **server-side en svc-privada**:

| Dato | De dónde lee svc-privada | Estado |
|---|---|---|
| CC/CH/ML: fila en el panel, estado general, convenio firmado + fecha, ok_gob | **Endpoint IAM-only nuevo** `GET /internal/planificacion/lineas?programa=…` en svc-vivienda. Lee en vivo (como ADR-030) y devuelve por `id_geo`: id, estado general, convenio_firmado, fecha_firma_convenio, ok_gob y monto | Aceptado por el usuario (2026-11-19). Es la única pieza nueva en svc-vivienda: solo lectura, sin tablas ni migraciones ni escrituras. Solo lo puede llamar `svc-privada@` por IAM (patrón de ADR-015) |
| Gasífera | `GET /internal/gasifera/rollup-territorial` (ya existe, ADR-021) | Falta dar `run.invoker` a `svc-privada@` |
| ATP | `GET /internal/atp/rollup-territorial` (ya existe, ADR-022) | Falta dar `run.invoker` a `svc-privada@` |
| Coparticipación, visitas del gobernador | `svc-datos-externos` por IAM (svc-privada ya tiene `integrations/datos_externos.py`, ADR-027) | Hay que definir o reusar el endpoint interno según §12.1 |

Tolerancia: si una fuente se cae, se aplica fail-open. La columna muestra "sin
dato" con un aviso y la planilla sigue editable. **Nunca se muestra "No" cuando
en realidad no se pudo consultar.** Caché en memoria de 5 minutos por fuente,
para no golpear a los demás servicios en cada recarga.

## 7. API (`svc-privada`, gateway: config nueva `ministerio-config-v{fecha}`)

Todas llevan `options:` con `security: []`. Prefijo
`/api/v1/privada/planificacion`.

| Método | Ruta | Rol |
|---|---|---|
| GET | `/programas` | lectura |
| POST · PATCH · DELETE | `/programas[/{id}]` | Supervisor/Admin (DELETE solo Admin) |
| GET · POST · PATCH · DELETE | `/programas/{id}/estados[/{estado_id}]` | GET lectura · resto Supervisor/Admin |
| GET · POST · PATCH · DELETE | `/programas/{id}/prioridades[/{prioridad_id}]` | idem |
| GET · POST · PATCH · DELETE | `/programas/{id}/columnas[/{columna_id}]` | idem |
| GET | `/programas/{id}/localidades` | lectura |
| PATCH | `/programas/{id}/localidades/{id_geo}` | escritura (upsert parcial por campo, `updated_at` → 409) |
| GET | `/programas/{id}/localidades/{id_geo}/historial` | lectura (paginado) |

`GET …/localidades`: `limit` por defecto y máximo 1000, `offset`. El padrón entra
en una sola página y el frontend filtra y ordena del lado del cliente. La
respuesta incluye `fuentes: {nombre: "ok"|"sin_dato"}` para los avisos.

Validaciones: `prioridad_id`/`estado_id` deben pertenecer al programa → si no,
422. Los `valores` se validan por tipo de columna; si la clave es desconocida
→ 422. Si `id_geo` no está activo → 404. Nombres duplicados → 409.

## 8. Frontend

- **Ruta propia**: `/resumen-territorial/planificacion`, página
  `PlanificacionProgramasPage.tsx` en `src/modules/resumen-territorial/pages/`,
  con sus componentes en `components/planificacion/` y la API en
  `api/planificacion.api.ts`. Se declara antes que la ruta
  `/resumen-territorial/:departamento/:localidad` para que no colisione.
- **Botón visible arriba** en `ResumenTerritorialPage.tsx`, en el encabezado,
  junto al título: "📋 Planificación de programas". Solo lo ven Privada y
  Admin (helper nuevo en `src/shared/utils/permisos.ts`).
- Encabezado de la página: selector de programa agrupado por área · botón
  "⚙ Configuración" (Supervisor/Admin) · Exportar Excel.
- **Modal de configuración**, con pestañas Programas · Prioridades · Estados ·
  Columnas. Cada pestaña permite alta, edición, reordenar (subir/bajar) y
  borrar, siempre del programa seleccionado.
- Edición en la celda con **autosave por campo**, igual que el Checklist
  Técnico, con indicador guardando / guardado / conflicto (recargar fila).
- Excel (`shared/utils/exportTable.ts`): todas las columnas visibles, con el
  formato de la planilla actual para no cortar el circuito de quienes hoy
  comparten el Sheet.
- **Resumen Territorial**: se saca la "Tabla general" vieja y sus filtros. Los
  botones que vivían ahí ("Ficha de municipio PDF/Excel" y "Exportar Excel"
  del resumen) se mueven junto al selector "Ir a", para que los demás roles
  no los pierdan.

## 9. Fases

1. Backend svc-privada: migración `0005`, catálogos (programas, estados,
   prioridades, columnas), planilla, historial y tests. Sin cruce.
2. Cruce Gasífera + ATP + datos externos (IAM).
3. Endpoint IAM-only de lectura en svc-vivienda + cruce CC/CH/ML.
4. Frontend + gateway + deploy. Verificación visual real con un usuario de Privada.
5. **Carga inicial** de las planillas de CC y CH: script idempotente en
   `services/svc-privada/scripts/seed_planificacion/` que lee el CSV ya
   emparejado (por agente, con revisión humana de los dudosos) y escribe por la
   API o por la capa de servicio, para que quede en el historial como "carga
   inicial". Las filas sin match no se cargan y quedan listadas en un
   informe. Los valores de la planilla que no entran en las columnas por
   defecto se cargan como columnas agregadas (por ejemplo "Monto" de la
   planilla, o la columna final sin título de CC), previa confirmación del
   usuario.

## 10. Criterios de aceptación

- Un Operador de Privada ve el botón arriba en el Resumen Territorial, entra,
  elige "Cordón Cuneta" y ve las ~539 localidades ordenadas por prioridad. Las
  que están en el panel de CC muestran "Sí" con su estado general. Las que
  tienen fecha de firma muestran "Convenio: Sí".
- Un Supervisor agrega la prioridad "6 – gris", un estado "Notificado" y una
  columna "Expediente municipal" (texto). Los tres aparecen al instante solo en
  ese programa.
- Un cambio de valor persiste al recargar, aparece en el historial desplegable
  de esa fila y en `priv_audit_log`, y no hay botón Guardar.
- Dos usuarios editan el mismo campo: el segundo recibe 409 y un aviso.
- Un usuario sin `privada` (ni Admin), o con rol Autoridad, no ve el botón y
  recibe 403 en la API.
- Con una fuente caída, su columna muestra "sin dato" y la planilla sigue editable.
- `svc-vivienda`: sin migraciones, tablas ni escrituras nuevas. Solo el
  endpoint IAM de lectura.
- El semáforo político se ve y no se puede editar.

## 11. ADR-036 (registrado en `arquitectura.md`)

**ADR-036 — La planificación de programas por localidad es dato de Privada.**
Privada es dueña, en `db_privada`, de la prioridad, el estado, el OK del
gobernador y los valores de columnas configurables por (programa, localidad), y
de los catálogos por programa. Los paneles de cada programa siguen siendo la
fuente de verdad de lo que **ya existe**. Privada solo los lee, por IAM y
server-side, y nunca escribe en ellos. Para CC/CH/ML eso requiere un endpoint
IAM-only de lectura en svc-vivienda, que es la única excepción y fue aceptada
explícitamente. Se separa la **prioridad por programa** (este spec) del
**semáforo político global** (`color_semaforo`, que sigue en solo lectura según
ADR-012 hasta su propio spec).

## 12. Preguntas abiertas

1. ~~Visita del gobernador~~ → **resuelto el 2026-11-19**: única fuente, la
   planilla de visitas (`ext_gob_visita_gobernador`). El análisis de coherencia
   está en el scratchpad de la sesión (`coherencia_atp_visitas.md`). Resultado:
   "estar en ATP" deja afuera a 88 localidades visitadas (Córdoba Capital, Villa
   Carlos Paz, Alta Gracia, entre otras), y si se filtra por 2026 marca 71 de
   más. ATP Interno paga a 93 localidades nunca visitadas. **El mismo criterio se
   aplica al filtro y a la ficha del Resumen Territorial**: es un cambio aparte,
   E5g en `spec-resumen-territorial-ficha-localidad.md`. Quedan dos
   pendientes con el área: 8 localidades con anuncio ATP que no figuran como
   visitadas (en 2 hay fecha de anuncio, así que la planilla de visitas está
   atrasada), y confirmar que la planilla se sigue cargando (la última visita
   registrada es del 2026-09-20).
2. ~~Significado de la prioridad~~ → resuelto: se configura por programa (§4.3).
3. **Convenio firmado en CC y ML** → por ahora la columna queda vacía. El
   usuario lo consulta con el área. CH ya tiene `fecha_firma_convenio` (0046).
4. ~~Historial~~ → resuelto: columna desplegable (§5.4).
5. **Planilla de Córdoba Hogar**: está compartida con `infraestructura.coop@gmail.com`,
   pero el conector de Drive igual no la abre (2026-11-19), así que el conector
   usa otra cuenta. Hay que bajarla como .xlsx para relevarla y emparejarla.
6. ~~Coparticipación~~ → resuelto: monto anual del concepto coparticipación,
   últimos 12 meses publicados (§5.1).
7. ~~Visibilidad de las visitas del gobernador~~ → **aceptado por el usuario
   (2026-11-19)**: `visitada` + `fecha_ultima` se muestran en esta pantalla
   (Privada + Admin) y en el Resumen Territorial (todos sus usuarios). Nunca se
   exponen `INTENDENTE` ni `PARTIDO`.

## 13. Carga inicial — estado del emparejamiento (2026-11-19)

`services/svc-privada/scripts/seed_planificacion/` (sin commitear):
`cordon_cuneta_general.csv`, `README.md` y `emparejar_cordon_cuneta.py`.

- **CC, pestaña GENERAL:** 424 filas → 397 exactas, 18 por alias, 6 por parecido
  del nombre (todas confirmadas por el usuario) y 2 confirmadas a mano por el
  usuario (Parque Calmayo → 2 CALMAYO; Villa Sta. Rosa de Río I → 208 VILLA SANTA
  ROSA). Queda 1 pendiente: Santiago Temple, alta de la migración 0042, cuyo
  `id_geo` hay que tomar del espejo `priv_geo_localidades` al momento de cargar.
- **Montos:** la columna "MONTO FIRMADO O SUGERIDO" está **en millones de
  pesos** (por ejemplo `200` = $200.000.000, y "150M" = $150.000.000, según el
  usuario). El script de carga multiplica por 1.000.000 todo valor menor a
  100.000. La columna "MONTO" ya viene en pesos.
- **Hallazgos sobre el padrón y los alias de svc-vivienda** (no se corrigen en
  este spec; van por migración de svc-vivienda en un cambio aparte):
  - Hay un "AGUA DE ORO" en Río Segundo (`id_geo` 241). El usuario confirmó que
    está mal: la localidad es de Colón (22).
  - El alias "SANTA ROSA DE RIO PRIMERO" → 206 RIO PRIMERO, de la migración 0034,
    marcado de confianza media, es incorrecto: el usuario confirmó que es 208
    VILLA SANTA ROSA.
  - "KILÓMETRO 658" (alta de la 0042) puede estar duplicando a 195 PEDRO E. VIVAS.
    **No se corrige**: el usuario lo revisa a mano. Contexto: Vivienda pidió que
    sus paneles admitan más de un programa por localidad (cambio futuro, otro
    spec), y es probable que hoy estén cargando filas nuevas con el nombre de la
    localidad modificado para distinguirlas. Por eso "¿Está en el panel?" cuenta
    **todas** las filas por `id_geo` y muestra la cantidad (§5.1).

## Changelog

- **1.0.0 (2026-11-19)** — **Approved** por el usuario. Visibilidad de las visitas
  aceptada. Kilómetro 658 / Pedro E. Vivas queda para revisión manual. ADR-036
  registrado.

- **0.3.0 (2026-11-19)** — Respuestas del usuario: visita del gobernador desde
  la planilla de visitas como única fuente; coparticipación anual (últimos 12
  meses del concepto coparticipación); convenio de CC/ML vacío hasta consultar
  con el área; emparejamiento de CC confirmado y montos en millones. Se agrega
  §13 (estado de la carga inicial) y la pregunta abierta 12.7 (visibilidad de
  las visitas).

- **0.2.0 (2026-11-19)** — Observaciones del usuario: pantalla propia con botón
  arriba en el Resumen Territorial; columnas fijas (tipo, habitantes,
  coparticipación, visita, OK gobernador, convenio, está en el panel, semáforo
  en solo lectura); estado y prioridad con catálogos editables por programa
  desde un botón de configuración; columnas agregables a mano; historial
  desplegable por fila; endpoint IAM-only de lectura en svc-vivienda aceptado;
  carga inicial de las planillas de CC y CH.
- **0.1.0 (2026-11-19)** — Borrador inicial a partir de la planilla de Cordón
  Cuneta.
