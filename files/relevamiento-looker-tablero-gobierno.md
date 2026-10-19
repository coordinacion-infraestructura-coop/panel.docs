# Relevamiento: "Tablero - Min. de Gobierno" (Looker / Data Studio)

**Tipo**: documento de análisis (no es un spec; no habilita implementación)
**Versión**: 1.1.0
**Fecha**: 2026-10-19
**Responsable**: Pedro Bonafe
**Relacionado**: `spec-datos-externos-tablero-gobierno.md` (paraguas, inventario de planillas y
fases) · `spec-tablero-gobierno-panel.md` (plan del panel nativo que reemplaza al Looker)

## 0. Fuente y método

- Tablero: `https://datastudio.google.com/u/0/reporting/70162e1c-d89d-4e1f-b1d0-afeedaac82f5/page/SKu1D`,
  acceso de vista con `infraestructura.coop@gmail.com`.
- Lectura del 2026-10-09 en modo vista con Claude in Chrome, sin editar nada. Se tomó el texto de
  las 19 páginas (`get_page_text`). Este documento consolida esa lectura; la v0.1.0 del spec
  paraguas tenía un inventario resumido por página que quedó reemplazado por el inventario de
  planillas.
- **Límite**: las capturas fallaron en las páginas pesadas, así que de los gráficos se conoce el
  título y las dimensiones que aparecen como texto (leyendas, ejes), **no el tipo exacto de
  gráfico ni su diseño**. Se marca con "tipo a confirmar". Se completa con una pasada de capturas
  cuando la extensión esté conectada (§5).
- No se copian valores sensibles (nombres, teléfonos, DNI, partido por localidad, montos de ATP).
  Los datos personales que se vieron en el tablero se listan como **campos**, no como valores.

## 1. Estructura general

- **19 páginas** en una barra de pestañas: Tablero (portada), Resumen General, Obras, Visitas,
  Visitas Gobernador, COPA, FOFINDES, FODEMEEP, FOMMEP, ATP, Comp. Gober, Fondo Federal, Fondo
  Ambiental, VINC. COMU, FOCOM, Comu. Regionales, Expedientes (Ministerios), IPCAM, Elecciones.
- **Filtros globales** en todas las páginas: `LOCALIDAD` y `DEPARTAMENTO` (desplegables). Varias
  páginas suman un filtro de **AÑO** (2024–2026; COPA 2023–2026) con "Filtro predeterminado".
- **Navegación**: la portada tiene bloques con enlace a cada página ("INFORMACIÓN VINCULACIÓN",
  "INFORMACIÓN TRANSFERENCIAS", "INFORMACIÓN LOCALIDAD", "INFORMACIÓN CONVENIOS"); cada página
  tiene un botón **VOLVER**.
- **Pie**: "Última actualización de datos: <fecha hora>" en cada página.
- **Universo**: 427 gobiernos locales (271 municipios, 156 comunas), más comunidades regionales y
  "entes" en algunos fondos.
- Unidades: montos en pesos salvo **Resumen General (millones de pesos)** y **Obras (rotulado
  "millones de dólares", unidad en duda: paraguas §3.7)**.

## 2. Catálogo por página

Convenciones: **KPI** = tarjeta con un número; **Tabla**; **Gráfico** (tipo a confirmar salvo que
se indique). "Fuente" = planilla del inventario del paraguas §2.

### 2.1 Tablero (portada) — "TABLERO DE GESTIÓN"

Bloques de KPIs (todos responden a los filtros Departamento/Localidad):

| Bloque | KPIs | Cálculo |
|---|---|---|
| Localidad | Cantidad de Localidades · Población (2022) · Empleados · **Empleados por Pob** (%) · Electores (2023) | conteo; suma; suma; empleados ÷ población; suma |
| Reuniones en el Panal | Cantidad de Localidades · Cantidad de Reuniones | conteo distinto de localidades con reunión; conteo de reuniones |
| Visitas en Territorio | Cantidad de Localidades · Cantidad de Visitas | conteo distinto; conteo |
| ATP | Localidades con ATP · Cantidad de ATP · Total de monto otorgado ATP · Saldo disponible ATP | conteo distinto; conteo; suma; saldo informado por la fuente |
| FOCOM | Localidades con FOCOM · Cantidad de FOCOM · Total de monto FOCOM | conteo distinto; conteo; suma |
| Fondo Federal | Última cuota otorgada · Disponible Fondo Permanente · Pagado por etapas de FF en el año | valor de la fuente; saldo; suma del año |
| Préstamos BANCOR | Cantidad de Préstamos · Monto total calificado · Cantidad de Localidades | conteo; suma; conteo distinto |
| Fondo Ambiental | Cantidad de Loc./Entes · Monto total afectado | conteo distinto; suma |

Tablas:

- **Intendentes**: LOCALIDAD · INTENDENTE · REELECTO · PARTIDO · CONT. · CELULAR · FOTO
  (paginada 50/427). *Sensible: datos personales y políticos.*
- **Legisladores**: DEPARTAMENTO · LEGISLADOR · PARTIDO_LEG. *Sensible: político.*
- **Matriz de convenios** ("INFORMACIÓN CONVENIOS"): DEPARTAMENTO · LOCALIDAD · ACUERDO FEDERAL ·
  EDUCACION · SEGURIDAD · CIDI · APROSS · AUTOMOTOR · URBANO · MONOTRIBUTO · INFORMACIÓN · OBRA ·
  RADIO · BALANCE · POLIDEP · PARQ. IND · COMP. VOL. Valores SI/NO/NC; algunas columnas numéricas
  (0) o vacías. Enlace "VER LISTADO DE LOCALIDADES POR FIRMAR".

Fuentes: "0. Resumen General" (`DATOS LOC`, no disponible), "1. REGISTRO REUNIONES",
"2. RECORRIDO Y VISITAS", "4. ATP 2026"/`ATP`, FOCOM, Fondo Federal, "8. BANCOR", Fondo
Ambiental, "6. CONVENIOS FIRMADOS Y RADIOS".

### 2.2 Resumen General — "RESUMEN POR LOCALIDAD (EN MILLONES DE PESOS)"

Dos tablas con la misma estructura:

1. **Por localidad**: LOCALIDAD · DEPARTAMENTO · PARTIDO · POBLACION · ATP · FOCOM · COPA ·
   FOFINDES · **ATP/COPA+FOFIN** · ADELANTO COPA · OBRAS PURAS · FONDO FEDERAL · FONDO AMBIENTAL ·
   **TOTAL** · **TOTAL x HABIT**. Fila de Total.
2. **Por departamento** (orden ▲): mismas columnas agregadas. Fila de Total.

Cálculos:

- `ATP/COPA+FOFIN` = ATP ÷ (COPA + FOFINDES), en %.
- `TOTAL` = ATP + FOCOM + COPA + FOFINDES + ADELANTO COPA + FONDO FEDERAL + FONDO AMBIENTAL.
  **Obras no entra en el total** ("Obras en millones de dólares (no incluidas en el total)").
- `TOTAL x HABIT` = TOTAL × 1.000.000 ÷ POBLACION (en pesos).
- Nota del tablero: "no se consideran los FOCOM o FONDO AMBIENTAL transferidos a comunidades
  regionales".

Fuente: hoy, "0. Resumen General - Vista" (paraguas §3.9). Es el agregado que el área califica
como no del todo confiable; el plan es calcularlo en nuestro backend desde cada fuente.

### 2.3 Obras

- **Tabla 1 — "OBRAS QUE ABARCAN UN SOLO DEPARTAMENTO"**: DEPARTAMENTO · TIPO · OBRA · ESTADO ·
  MES INICIO · MES FIN · AVANCE (%) · MONTO (orden ▼). Fila Total.
- **Tabla 2 — "OBRAS QUE ABARCAN MÚLTIPLES DEPARTAMENTOS"**: mismas columnas. Nota: "Los montos
  asociados a obras multidepartamentales están prorrateados para cada departamento en base a la
  cantidad de departamentos".
- **Gráfico "ESTADO OBRA"** (tipo a confirmar, probablemente torta/barras): obras por estado.
- Dominios: TIPO ∈ {HYG, CASISA, EPEC, VIALIDAD, ARQUITECTURA, …}; ESTADO ∈ {EN PROYECTO, A
  LICITAR, EN LICITACIÓN, EN PROCESO DE ADJUDICACIÓN, OBRA EN EJECUCION, FINALIZADA}.
- Fuente: "12. Obras" (`BD-TABLERO`, prorrateo en `BD-VARIAS`).

### 2.4 Visitas

- KPIs: Cantidad de Localidades · Cantidad de visitas.
- **Tabla de visitas**: FECHA (▼) · LOCALIDAD · DEPARTAMENTO · FUNCIONARIO · TIPO DE VISITA
  (INSTITUCIONAL, FIRMA CONVENIO, …).
- **"Visitas en Territorio" — tabla por funcionario**: FUNCIONARIO · Total de actividades ·
  Localidades visitadas · como % del total · Municipios visitados · Comunas visitadas.
  Notas: TOTAL = visitas/actividades desde el inicio de la gestión; LOCALIDADES = localidades
  únicas; para un funcionario se cuentan actividades y no visitas.
- Enlace "VER LISTADO DE LOCALIDADES POR VISITAR".
- Calidad: fechas futuras y filas sin localidad son errores de carga (respuesta del área).
- Fuente: "2. RECORRIDO Y VISITAS" + "2.1 … - CALVO".

### 2.5 Visitas Gobernador

- KPI: Cantidad de Localidades (visitadas).
- **Tabla "LOCALIDADES VISITADAS"**: DEPARTAMENTO (▲) · LOCALIDAD · ÚLTIMA VISITA.
- **Tabla "LOCALIDADES AÚN NO VISITADAS"** con su KPI "Localidades NO visitadas".
- Fuente: "2. RECORRIDO Y VISITAS" pestaña `Visitas Gober`.

### 2.6 COPA — Coparticipación

KPIs:

| KPI | Cálculo |
|---|---|
| Total Coparticipación en el año · Total Neto Transferido en el año | suma bruta / neta del año filtrado |
| Último pago: Mes · Quincena · Fecha de Pago · Total coparticipación (ult. pago) · Total neto transferido (ult. pago) | registro con fecha de pago máxima |
| Último mes completo: Mes · Fecha de Pago · Total coparticipación (ult. mes completo) · Total neto transferido (ult. mes completo) | último mes con sus dos quincenas |
| **% de Coparticipación + FOFINDES TOTAL que se destina al pago de sueldos** | masa salarial ÷ (COPA bruta + FOFINDES bruto) del último mes completo |
| **% de Coparticipación + FOFINDES NETA transferida que se destina a sueldos** | masa salarial ÷ (COPA neta + FOFINDES neto) del último mes completo |

Nota del tablero: usa empleados y masa salarial de mayo 2025.

Gráficos (tipo a confirmar):

- "COMPARATIVA - ACUMULADO A <MES> DE CADA AÑO" (acumulado enero→mes, un valor por año).
- "COMPARATIVA - MES DE <MES> DE CADA AÑO".
- "COMPARATIVA CAJA/COPA - MES A MES" (serie mensual; "caja" = Caja de Jubilaciones descontada,
  a confirmar).

Filtro AÑO 2023–2026. Fuente: "11.1 COPARTICIPACION" (`COPA`, por quincena, con ~40 conceptos de
descuento). Nuestro scraping (`ext_transferencias`) queda como respaldo y control.

### 2.7 FOFINDES

- KPIs: Total FOFINDES en el año · Total FOFINDES Neto Transferido en el año · Último mes
  completo (MES, Fecha de Pago) · Total FOFINDES (ult. mes completo) · Total FOFINDES neto
  transferido (ult. mes completo).
- Gráfico/tabla "Monto Total transferido" por MES (tipo a confirmar).
- Filtro AÑO 2024–2026. Fuente: "11.1 FOFINDES".

### 2.8 FODEMEEP

- KPIs: Localidades con FODEMEEP · Total FODEMEEP presupuestado en la gestión · Total FODEMEEP
  transferido en la gestión · Último mes completo (MES, fecha) · Total transferido (ult. mes
  completo) · Total presupuestado (ult. mes completo).
- **Tabla**: DEPARTAMENTO (▲) · LOCALIDAD · NETO A TRANSFERIR por mes. Fila Total.
- **Gráfico por ESTADO de adhesión** (NORMAL, SUSPENDIDO, CORTADO, NO ADHERIDO) con tabla
  LOCALIDADES · ESTADO.
- Fuente: "11.1 FODEMEP" (`FODEMEP`, `ESTADO`).

### 2.9 FOMMEP

- KPIs: Localidades con FOMMEP · Total FOMMEP en la gestión.
- Cuatro grupos de KPI: Móviles (Total de FOMMEP para móviles · Cant. de móviles), Edificios
  Cat. 1, Cat. 2 y Cat. 3 (total y cantidad cada uno).
- Tabla/gráfico con Total. Filtro AÑO. Fuente: "11.1 FOMMEP" (`FROMMEP`).

### 2.10 ATP (ATP Interno de Gobierno)

- KPIs: Localidades con ATP · Cantidad de ATP · Total de monto ATP · Saldo disponible ATP.
- **Tabla**: FECHA (▼) · LOCALIDAD · MONTO.
- Gráfico "ATP" (tipo a confirmar). Filtro AÑO.
- *Sensible*: la fuente trae el referente político de cada pago (la página no lo muestra en la
  tabla leída). Fuente: "4. ATP 2026" + histórico en "0. Resumen General" (no disponible).
- **No se suma con Comp. Gober** (superposición confirmada, paraguas §3.3).

### 2.11 Comp. Gober — "ATP - Compromisos Gobernador"

- KPIs: Localidades con Compromisos · Cantidad de Compromisos · Total de monto comprometido ·
  Total de monto pagado · Total de saldo.
- **Tabla**: Fecha de anuncio (▼) · LOCALIDAD · DEPARTAMENTO · Monto · Pagado (SI/NO) · Derivado
  (ministerio) · Destino (texto libre).
- Gráfico por **Ministerio** (tipo a confirmar).
- Fuente: Sheet de Compromisos — **ya está en el sistema** (`svc-gralgob`).

### 2.12 Fondo Federal — "FONDO FEDERAL - ACUERDO FEDERAL"

- KPIs: Disponible etapas de Fondo Federal · Saldo en cuenta Fondo Federal · Última cuota pagada
  · Última cuota habilitada · Pagado por etapas de FF · Aportes Copa del año · Repago de
  préstamos del año.
- **Grupos por estado de etapa** (Pedidas · Disponibles · Listo para pagar · Pagado · …), cada uno
  con: Total de Fondo Federal para transferir · Cant. de localidades · como % del total.
- **Tabla "DETALLE DE ETAPAS POR LOCALIDAD Y ESTADO"**: DEPARTAMENTO · LOCALIDAD (▲) · ETAPA ·
  OTORGADO · ESTADO · DISPONIBLE ETAPA.
- Gráfico "ESTADO DE ETAPAS FONDO FEDERAL" (ESTADO - SIMPLE).
- Fuente: pestaña `FONDO FEDERAL` de "0. Resumen General" (no disponible) — falta la planilla de
  origen.

### 2.13 Fondo Ambiental

- KPIs: Loc. con Fondo Ambiental · Entes con Fondo Ambiental · Total de Fondos Ambiental · Total
  afectado · Total disponible Fondo Ambiental.
- **Grupos por estado** (Pagados · Listo para pagar · Tribunal de Cuentas · En trámite), cada uno
  con Total de monto y Cantidad de Loc/Ente.
- **Tabla**: DEPARTAMENTO · LOCALIDAD (▲) · ESTADO · MONTO · DESTINO (equipamiento: camión,
  chipeadora, galpón, …).
- Filtro AÑO. Fuente: "9. FONDO AMBIENTAL".

### 2.14 VINC. COMU — Vinculación Comunitaria

- KPIs previstos: Localidades beneficiadas · Cantidad de transferencias · Total de monto
  transferido. **La página está rota** ("Error de configuración del conjunto de datos").
- El área informó que está en desuso → **fuera del panel**.

### 2.15 FOCOM

- KPIs: Localidades con FOCOM · Cantidad de FOCOM · Total de monto FOCOM.
- **Grupos por estado** (Pagados · Listo para pagar · Tribunal de Cuentas · En trámite · Fondos
  Solicitados · Sin documentación · **Pagados ATP** · **Pendiente ATP**), cada uno con Total de
  monto · Cant. de FOCOM · Loc. con FOCOM.
- **Tabla** (433 filas): DEPARTAMENTO · LOCALIDAD · CONVENIO · FECHA CONVENIO (▲) · ESTADO ·
  MONTO · CONCEPTO · ESTADO OBRA (NO INICIADA, EN PROCESO, TERMINADA, INAUGURADA).
- Filtro AÑO. Fuente: `FOCOM PURO` de "0. Resumen General" (no disponible).
- Regla: los FOCOM "Pagados ATP"/"Pendiente ATP" están cargados también en ATP → no sumarlos dos
  veces (paraguas §3.8).

### 2.16 Comu. Regionales — Comunidades Regionales

- KPIs: Total de Fondos para obras · Crédito Vigente (año) · Saldo · Total de monto Cuotas · Cant.
  de Cuotas.
- **"Cuotas de fondos por estado"** (Pagados · Listo para pagar · Tribunal de Cuentas · En trámite
  · Fondos Solicitados · A pagar / Debe rendir): monto y cantidad por estado.
- **Tabla**: DEPARTAMENTO · AÑO (▲) · DESTINO · ESTADO (PAGADO RENDIDO, PAGADO SIN RENDIR, LISTO
  PARA PAGAR, ANULADO, …) · CUOTA · MONTO CUOTA · FECHA PAGO CUOTA.
- Tabla "FONDOS PARA OBRAS" (13 filas). Fuente: "17. Comunidades Regionales".

### 2.17 Expedientes (Ministerios) — "EXPEDIENTES VISADOS"

- KPIs: Total de Expedientes · Monto total visado · Monto total Remitido · Monto total En Trámite
  · Total de Localidades · Total de Departamentos.
- **"Expedientes por programa y estado"**: por programa (Cordón Cuneta, Gas, Gas - Escuela, …) y
  estado (Remitido, En Trámite): monto y cantidad.
- **Tabla** (132 filas): ESTADO · N° EXPEDIENTE · MINISTERIO · PROGRAMA · LOCALIDAD · Fecha ·
  MONTO.
- **Flujo inverso**: la fuente real de Cordón Cuneta y Gas es nuestro sistema (paraguas §3.6).

### 2.18 IPCAM — Instituto Provincial de Capacitación Municipal

- KPIs: Localidades alcanzadas · Departamentos · Personas capacitadas · Certificados.
- Gráfico "Personas capacitadas por año e iniciativa".
- Tabla "Top 10 personas con más certificados" (**por DNI** — dato personal). Se replica con nombre y localidad, sin mostrar el DNI (decisión del usuario 2026-10-19, `spec-tablero-gobierno-panel.md` §2.5 #9).
- Tabla CERTIFICADOS: LOCALIDAD · DEPARTAMENTO · PROYECTO · INSTITUCIÓN/DISERTANTE · AÑO ·
  INICIATIVA.
- Nota: los datos 2024 son del cuatrimestre agosto–noviembre (lanzamiento del Instituto).
- Fuente: **no recibida** (el área la va a compartir).

### 2.19 Elecciones

- Bloque "INFORMACIÓN DE LA LOCALIDAD": ELECTORES · INTENDENTE · REELECTO · PARTIDO · CONT.
- **Tabla municipal**: LOCALIDAD · GANADOR · SEGUNDO · MARGEN.
- **Tablas provinciales/nacionales por localidad** con % por fuerza: gobernador 2023 (HxC, JxC,
  …), legislativas (LLA, Defendamos Córdoba, PU, Otros) y 2021.
- *Sensible: político.* Fuente: "18. Resultados elecciones 21-25".

## 3. Catálogo de cálculos a reproducir

| Indicador | Fórmula | Páginas |
|---|---|---|
| Empleados por población | Σ empleados ÷ Σ población | Tablero |
| % COPA+FOFINDES a sueldos (bruto) | masa salarial ÷ (COPA bruta + FOFINDES bruto), último mes completo | COPA |
| % COPA+FOFINDES a sueldos (neto) | masa salarial ÷ (COPA neta + FOFINDES neto), último mes completo | COPA |
| Último pago / último mes completo | máx(fecha de pago) / último mes con todas sus quincenas | COPA, FOFINDES, FODEMEEP |
| Comparativas anuales | acumulado ene→mes por año; mismo mes por año; serie mensual | COPA |
| ATP/COPA+FOFIN | ATP ÷ (COPA + FOFINDES) | Resumen General |
| TOTAL transferido | ATP + FOCOM + COPA + FOFINDES + Adelanto COPA + Fondo Federal + Fondo Ambiental (sin Obras; sin FOCOM/FA a comunidades regionales) | Resumen General |
| Total por habitante | TOTAL ÷ población | Resumen General |
| % de localidades visitadas | localidades únicas visitadas ÷ 427 | Visitas |
| Agregados por estado | Σ monto, conteo y conteo distinto de localidades por estado | FOCOM, Fondo Ambiental, Fondo Federal, Comu. Regionales, Expedientes |
| % del total por estado | monto del estado ÷ monto total | Fondo Federal |
| Saldo | comprometido − pagado | Comp. Gober, Comu. Regionales |
| Prorrateo de obras multidepartamentales | monto ÷ cantidad de departamentos | Obras |

Todos los indicadores se recalculan con los filtros Departamento / Localidad / Año.

## 4. Mapa de fuentes y estado en nuestro sistema

| Página | Fuente | Estado en el sistema (2026-10-19) |
|---|---|---|
| Tablero | varias | Parcial: población (Censo), partido (Vista), ATP Compromisos |
| Resumen General | Vista (agregado) | Población y partido; montos sin usar (paraguas §3.9) |
| Obras | 12. Obras | No; moneda en duda |
| Visitas / Visitas Gobernador | 2. / 2.1 RECORRIDO Y VISITAS | No |
| COPA / FOFINDES | 11.1 COPARTICIPACION / FOFINDES | Parcial: scraping mensual (`ext_transferencias`) |
| FODEMEEP / FOMMEP | 11.1 FODEMEP / FOMMEP | No |
| ATP | 4. ATP 2026 (+ histórico no disponible) | No (Fase 4) |
| Comp. Gober | ATP - Compromiso Gobernador | **Sí** (`svc-gralgob`) |
| Fondo Federal | pestaña de "0. Resumen General" (no disponible) | No; falta planilla de origen |
| Fondo Ambiental | 9. FONDO AMBIENTAL | No |
| VINC. COMU | 14. Vinculación Comunitaria | Fuera (en desuso) |
| FOCOM | `FOCOM PURO` (no disponible) | No; falta fuente |
| Comu. Regionales | 17. Comunidades Regionales | No |
| Expedientes | 15. Programas visados | Flujo inverso: el dato es nuestro |
| IPCAM | no recibida | No |
| Elecciones | 18. Resultados elecciones | No |
| BANCOR (portada) | 8. BANCOR | Fuera (nunca se actualizó) |
| Reuniones (portada) | 1. REGISTRO REUNIONES | No |
| Convenios (portada) | 6. CONVENIOS FIRMADOS Y RADIOS | No |

## 5. Pendientes del relevamiento

### 5.1 Segunda lectura (2026-10-19, Claude in Chrome, solo lectura)

Se recorrieron las 17 páginas en alcance contando los componentes de cada una (tablas, KPIs,
gráficos) y se miraron de cerca los gráficos. Lo único que se tocó fue el orden de una tabla
(estado de vista, no se guarda en el informe).

| Página | Componentes | Hallazgo |
|---|---|---|
| Resumen General | 2 tablas | **Semáforo de ATP/COPA+FOFIN** (color de fondo de la celda): verde ≥ 1,5 %, amarillo entre ~0,8 % y 1,5 %, rojo < ~0,8 % (0,9 % es amarillo y 0,7 % es rojo; el corte exacto no se ve en modo vista). Valores reales: 0,1 % a 108,8 % por localidad, 2,1 % a 51,1 % por departamento, total 18,4 % (la fila de total no lleva color). Rótulo "Obras en **miles de millones de pesos**" con valores "USD …". |
| COPA | 6 KPIs, 2 tablas (pagos por quincena y por mes), 2 velocímetros, 3 combinados | Los velocímetros son el **% de COPA + FOFINDES que va a sueldos** (sobre el total y sobre lo neto; este último pasa el 100 %). Las 3 comparativas **no son montos**: son **retenciones como % de la COPA bruta**. "COMPARATIVA CAJA/COPA – MES A MES": Caja de Jubilaciones ÷ COPA bruta por mes, una serie por año. "ACUMULADO A <MES>" y "MES DE <MES> DE CADA AÑO": barras horizontales por año con dos series, **CAJA / COPA BRUTA** y **APROSS / COPA BRUTA**. Resuelve el pendiente 2. |
| FOFINDES | 4 KPIs, 1 tabla | Sin gráfico. |
| FODEMEEP | 6 KPIs, 2 tablas, 1 tabla dinámica | Tabla "Último mes completo": DEPARTAMENTO · LOCALIDAD · MES · NETO A TRANSFERIR. Tabla dinámica de localidades por estado de adhesión. |
| FOMMEP | 10 KPIs | Sin tablas visibles; filtro AÑO. |
| ATP | 4 KPIs, 1 tabla | Sin gráfico (el relevamiento 1.0.0 lo daba "a confirmar"). |
| Comp. Gober | 5 KPIs, 1 tabla | Sin gráfico. |
| Fondo Federal | 23 KPIs, 2 tablas, 2 velocímetros | KPIs extra: última cuota habilitada, saldo en cuenta, última cuota pagada, disponible en etapas, pagado por etapas, **aportes COPA del año**, **repago de préstamos del año**. Velocímetros "Localidades que recibieron la **última** / **anteúltima** cuota otorgada (% del total)". Tablas: detalle de etapas por localidad y estado (con filtros ESTADO y ETAPA) y **detalle de etapas pagadas por localidad en el año** (otorgado). |
| Fondo Ambiental | 13 KPIs, 1 tabla, 1 torta | Torta "FONDO AMBIENTAL POR ESTADO DE PAGO". |
| FOCOM | 27 KPIs, 1 tabla, 1 torta | Los KPIs de arriba son "**excluyendo pagados por ATP**"; el bloque por estado es "incluyendo pagados por ATP". **Torta: PAGADO · PAGADO ATP · PENDIENTE ATP** (sobre el monto). La tabla colorea la celda de estado. |
| Comu. Regionales | 15 KPIs, 1 tabla, 1 torta | Torta "Cuotas de fondos por estado". |
| Obras | 5 tablas | Además de las 2 por departamento, tiene **3 tablas por localidad**: "obras que abarcan una sola localidad", "obras que abarcan múltiples localidades" (montos **no** prorrateados) y "resumen de obras generales" (sin departamento asignado). Columnas MES INICIO y MES FIN. La planilla 12 **sí trae localidad**. Rótulo "millones de dólares". |
| Visitas | 22 KPIs, 1 tabla, 3 velocímetros | Velocímetros: % de localidades visitadas, municipios visitados, comunas visitadas. Tabla "LOCALIDADES VISITADAS": FECHA · LOCALIDAD · FUNCIONARIO · TIPO DE VISITA. |
| Visitas Gobernador | 2 KPIs, 2 tablas | — |
| Expedientes | 18 KPIs, 1 tabla | — |
| IPCAM | 4 KPIs, 1 combinado, 1 tabla dinámica | Combinado = personas por año e iniciativa. La tabla dinámica es el top por certificados, **identificado por DNI** (con localidad); no hay nombre. |
| Elecciones | 5 tablas | Sin gráficos. |

### 5.2 Siguen pendientes

1. Corte exacto entre amarillo y rojo del semáforo (entre 0,7 % y 0,9 %): pedirlo al área o leerlo
   en modo edición si nos dan acceso de editor.
2. Moneda de Obras: el Looker dice "millones de dólares" en Obras y "miles de millones de pesos" en
   el Resumen General para la misma columna (pedido al área).
3. Período de los montos de la Vista (pedido al área).
4. IPCAM: la planilla identifica a las personas por DNI; para mostrar nombre hace falta que la
   fuente lo traiga (si no, se muestra solo localidad y cantidad).
5. Valores de referencia para la conciliación: al implementar cada página, comparar nuestros
   KPIs con los del Looker con el mismo filtro (criterio de aceptación del plan).
