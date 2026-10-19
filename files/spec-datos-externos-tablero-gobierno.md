# Spec: Integración del "Tablero - Min. de Gobierno" (Looker Studio) al sistema

**Estado**: approved
**Versión**: 1.4.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-19
**Servicios**: `svc-datos-externos` (amplía alcance, ADR-028) + `svc-gralgob` (ATP, ADR-029) + `svc-vivienda` (`resumen_territorial`, consumidor)

Spec paraguas. Define el inventario de fuentes, las decisiones ya tomadas y el orden de
incorporación. **No habilita implementación por sí solo**: cada fase se construye recién cuando su
sección (o un spec hijo) pase a `approved`.

## Changelog

- **1.4.1** (2026-10-19): el usuario confirma la recomendación de §3.9 (montos de la Vista) y
  decide que superficie y población 2010 se tomen de fuentes oficiales externas (§3.10). Las
  preguntas al área se enviaron.
- **1.4.0** (2026-10-19): **"0. Resumen General" no se puede compartir.** El área lo informó y
  en su lugar mandó "0. Resumen General - Vista" (una pestaña, 427 localidades). Se pierden como
  fuente las pestañas `DATOS LOC`, `ATP` (histórico), `FONDO FEDERAL`, `FOCOM PURO` y `ADELANTOS`
  (§2). La Fase 1 pasa a leer la Vista (spec hijo v1.2.0). Las columnas de montos de la Vista
  quedan sin usar hasta decidir qué hacer con ellas (§3.9).
- **1.3.0** (2026-10-13): primer cruce ATP Compromisos vs. ATP Interno contra datos reales
  (§3.3.1). Confirma superposición parcial y descarta el referente como marca.
- **1.2.0** (2026-10-13): corrección del usuario — todo lo marcado "solo `Autoridad`" lo ve también
  el rol `Admin` (§5, §7).
- **1.1.0** (2026-10-13): respuestas del área de Gobierno a las consultas (§3.8). Cambios de
  fondo: **ATP Interno y ATP Compromisos sí se superponen** (§3.3, nota en ADR-029); FOCOM pagado
  por ATP está cargado en ambos lados; BANCOR y Vinculación Comunitaria salen del plan;
  `RESUMENGRAL` no sirve como validación; la población de zona rural es una estimación.
- **1.0.0** (2026-10-13): aprobado por el usuario. Respuestas a las preguntas abiertas en §7;
  Fase 1 aprobada en su spec hijo (v1.0.0), que suma los datos institucionales de `DATOS LOC`.
- **0.2.0** (2026-10-13): relevamiento de las 21 planillas fuente por API (solo lectura).
  Reemplaza el inventario por página del Looker por el inventario real de planillas (§2),
  agrega hallazgos (§3), calidad de datos (§4) y reordena el plan (§6).
- **0.1.0** (2026-10-09): relevamiento del Looker en modo vista; ADR-027/028/029.

---

## 0. Origen

El área de Gobierno compartió el Looker Studio "Tablero - Min. de Gobierno"
(`https://datastudio.google.com/u/0/reporting/70162e1c-d89d-4e1f-b1d0-afeedaac82f5/page/SKu1D`)
y, el 2026-10-13, las planillas de Google Sheets que lo alimentan (listado en
`url_fuentes_externas.txt`, raíz del monorepo). El objetivo del usuario es:

1. Sumar a nuestro sistema la información de ese tablero que hoy no tenemos.
2. Reemplazar fuentes propias frágiles por las de Gobierno donde sean mejores.
3. Como horizonte, replicar el tablero dentro del sistema.
4. Más adelante, que las planillas (primero las sensibles) pasen a ser paneles editables
   propios, para que las áreas no carguen información sensible en Google Sheets.

### 0.1 Cómo se relevó y qué límites tiene

- Lectura por la API de Sheets con la cuenta de servicio `svc-datos-externos@` (impersonada por
  `infraestructura.coop@gmail.com`, permiso `serviceAccountTokenCreator` otorgado el 2026-10-13
  para este relevamiento). Solo lectura: metadatos, encabezados, conteos y agregados.
- La planilla de Compromisos del Gobernador no es legible con esa cuenta (está compartida con
  `svc-gralgob@`, que es la que la sincroniza). No hizo falta: ya está en el sistema.
- No se validó fila por fila la calidad de cada planilla; los conteos son de filas no vacías.
- El cruce ATP Interno vs. Compromisos (§3.3) quedó parcial: falta compararlo contra
  `atp_cronograma_pagos` de producción.

## 1. Decisiones ya tomadas

| Decisión | ADR |
|---|---|
| La autoridad del padrón político de localidades pasa a Gobierno | ADR-027 (supersede parcialmente ADR-012) |
| `svc-datos-externos` pasa a tener paneles de negocio propios | ADR-028 (amplía ADR-025) |
| ATP vive entero en `svc-gralgob`; son dos planillas distintas | ADR-029 |

### 1.1 Las dos planillas de ATP — no confundir

| | ATP Compromisos Gobernador | ATP Interno Gobierno |
|---|---|---|
| Qué es | Compromisos anunciados por el Gobernador | ATP de manejo discrecional/político de la Secretaría |
| Estado en el sistema | Ya sincronizada (`atp_compromisos`, `atp_cronograma_pagos`) | No está en el sistema |
| Fuente | Sheet `1x3E73hlGwDPBiLc9fbQuWxWUxjaibs3glModHsxCMZM` | Sheet `1h3FzAL-X0_tgIGNgnZ_DgKd5rDo8oLg0wNxY0gisBAU` ("4. ATP 2026"); el histórico estaba en "0. Resumen General", que no se puede compartir (§2) |
| Spec | `spec-sync-atp-compromiso-gobernador.md` (approved) | A abrir (spec hijo) |
| Sensibilidad | Media | Alta (incluye el referente político de cada pago) |
| Destino | Sigue como está | Módulo y panel nuevo en `svc-gralgob` |
| Relación | Anuncios del Gobernador en cada visita | Registro administrativo de pagos; **incluye también** pagos de los compromisos (§3.3) |

**Los montos de una y otra nunca se suman entre sí** (§3.3).

## 2. Inventario de planillas fuente

Todas las planillas comparten una pestaña `Validadores` (copia del padrón de localidades) y
varias tienen pestañas derivadas (`SUMA`, tablas dinámicas, copias). **Solo se sincroniza la
pestaña base** de cada una.

| Planilla (título real) | Pestaña base | Filas | Contenido | Destino |
|---|---|---|---|---|
| 0. LOCALIDADES - NOMBRES | `LOCALIDADES` | 478 | Nomenclador: `ID_LOC` (INDEC), nombre, `ID_DPTO`, `ID_COPA`, tipo MU/CO | `svc-datos-externos` (§3.1) |
| 0. Resumen General - Vista | `Hoja 1` | 427 | Población 2022 de Gobierno y partido por localidad, más totales de montos por fondo (§3.9). Reemplaza a "0. Resumen General" | `svc-datos-externos` (población y partido, Fase 1) |
| ~~0. Resumen General~~ | `DATOS LOC` | 452 | Superficie, población 2022 y 2010, electores, empleados, masa salarial, intendente, reelecto, partido, continuidad, legislador, contacto | **No disponible** (2026-10-19) |
| ~~0. Resumen General~~ | `ATP` | 6.032 | Histórico de ATP Interno 2023–2026: fecha, localidad, monto, referente | **No disponible** |
| ~~0. Resumen General~~ | `FONDO FEDERAL` | 1.503 | Etapas por localidad: monto, disponible, estado | **No disponible** |
| ~~0. Resumen General~~ | `FOCOM PURO` | 361 | FOCOM vigente: monto, concepto, categoría, estado, fecha de pago | **No disponible** |
| ~~0. Resumen General~~ | `ADELANTOS` | 294 | Adelantos de coparticipación 2023–2025 | **No disponible** |
| ~~0. Resumen General~~ | `RESUMENGRAL`, `RESUMENGRAL-BRANDAN` | 427 | Tablas agregadas a pedido de dos despachos. El área informa que **no son confiables** (montos sin corroborar) | No se sincroniza ni se usa para validar |
| 4. ATP 2026 | `ATP ACUMULADO` | 2.220 | ATP Interno del año en curso (planilla de carga) | `svc-gralgob` |
| 11.1 COPARTICIPACION | `COPA` | 28.182 | Por localidad y quincena, 2024–2026: bruto, neto y ~40 conceptos de descuento | `svc-datos-externos` |
| 11.1 FOFINDES | `FOFINDES` | 14.518 | Mensual 2024–2026: parte variable, excedentes, descuentos, neto | `svc-datos-externos` |
| 11.1 FODEMEP | `FODEMEP`, `ESTADO` | 14.785 / 427 | Mensual 2023–2026 y estado de adhesión con saldo | `svc-datos-externos` |
| 11.1 FOMMEP | `FROMMEP` | 854 | Anual: móviles y edificios por categoría | `svc-datos-externos` |
| 7. FONDO PERMANENTE | `FONDO PERM CUOTAS`, `IMPORT_Listo` | 962 / 865 | Préstamos a municipios en cuotas descontadas de coparticipación; destino del préstamo | `svc-datos-externos` |
| 8. BANCOR | `Prestamos` | 99 | Préstamos BANCOR: monto calificado, liquidado, concepto. **Nunca se actualizó** (depende de un pedido al banco) | No se sincroniza hasta que el área la actualice |
| 9. FONDO AMBIENTAL | `Fondo` | 135 | Expediente, convenio, resolución, objeto, monto, estado | `svc-datos-externos` |
| 14. Vinculacion Comunitaria | `BD` | 388 | Aportes por beneficiario: monto, estado, expediente, rendición. **En desuso** según el área | No se sincroniza |
| 17. Comunidades Regionales | `Fondo` | 131 | Fondos de funcionamiento y obra por comunidad regional, cuotas, rendición | `svc-datos-externos` |
| 12. Obras | `BD-TABLERO` | 1.063 (886 obras) | Obra, división, estado, avance, fechas, monto; `BD-VARIAS` trae el prorrateo de obras compartidas | `svc-datos-externos` |
| 12. Obras | `Parques industriales` (oculta) | 88 | Parques industriales por localidad | `svc-datos-externos` |
| 2. RECORRIDO Y VISITAS | `Recorrido`, `Visitas Gober` | 429 / 427 | Visitas de funcionarios; última visita del Gobernador por localidad | `svc-datos-externos` |
| 2.1 RECORRIDO Y VISITAS - CALVO | `Recorrido` | 365 | Actividades del ministro (misma estructura) | `svc-datos-externos` |
| 1. REGISTRO REUNIONES | `Reuniones`, `Intendentes` | 578 / 452 | Reuniones con temas y resolución; padrón de intendentes con contacto | `svc-datos-externos` |
| 6. CONVENIOS FIRMADOS Y RADIOS | `Convenios`, `Compromisas`, `Polideportivos` | 427 / 427 / 50 | Matriz de convenios; compromiso voluntario de datos; polideportivos | `svc-datos-externos` |
| 10. Polideportivos y Vivienda Semilla | `Polideportivo` | 50 | Duplicado de la pestaña anterior | No se sincroniza |
| 13. EMPLEADOS | `Activos MyC` | 1.125 | Empleados, salario promedio y masa salarial por localidad y año | `svc-datos-externos` |
| 18. Resultados elecciones 21-25 | `2021`, `2023`, `2025`, `Municipales - 2023` | 634 circuitos / 427 | Votos por circuito electoral; resultado municipal 2023 | `svc-datos-externos` |
| 15. Programas visados por Gobierno | `Fondo` | 132 | Expedientes visados de otros ministerios | Flujo inverso (§3.6) |
| 5. FOCOM - Deprecated | — | 332 | Versión vieja de FOCOM | No se sincroniza |
| 0. Responsables BD - Tablero | `Hoja 1` | 12 | Quién mantiene cada base y si se actualiza sola o a pedido | Referencia |

**Sin fuente recibida** (el área las va a compartir): IPCAM (capacitación municipal) y la
planilla de origen de Fondo Federal (la pestaña `FONDO FEDERAL` de Resumen General es una copia).

**"0. Resumen General" no se puede compartir** (informado por el área el 2026-10-19; dejó de ser
legible el 2026-10-13). Las pestañas tachadas arriba dejan de ser fuente. Lo que eso afecta:

| Pestaña perdida | Afecta a | Alternativa |
|---|---|---|
| `DATOS LOC` | Fase 1: superficie, población 2010, zona rural y ficha institucional | Vista (solo población y partido); resto en spec Fase 1 §11.1 |
| `ATP` (histórico) | Fase 4: ATP Interno 2023–2025 y el cruce histórico con Compromisos | "4. ATP 2026" cubre solo el año en curso. Preguntar si hay planillas anuales |
| `FONDO FEDERAL` | Fase 3 | Ya estaba pendiente la planilla de origen; ahora es la única vía |
| `FOCOM PURO` | Fase 3 | "5. FOCOM - Deprecated" no sirve. Pedir la fuente original (administración) |
| `ADELANTOS` | Fase 3 | Pedir la planilla de origen |

## 3. Hallazgos

### 3.1 Todas las planillas comparten una clave: el código INDEC de gobierno local

`ID_LOC` es el código INDEC de gobierno local (6 dígitos, p. ej. `142007`) — **el mismo que ya
tenemos en `ext_geo_censo.codigo_indec`**. No hace falta resolver localidades por nombre para
ninguna de estas fuentes: el vínculo a `id_geo` sale de un join por código.

El nomenclador de Gobierno tiene 478 filas: 427 gobiernos locales (271 municipios, 156 comunas),
26 comunidades regionales (ids 428–453) y 25 filas "zona rural" por departamento (`nnZR`). Trae
además `ID_COPA` (código de la localidad en las planillas de coparticipación) e `ID_DPTO`.

**Decisión confirmada por el usuario (2026-10-13)**: el padrón oficial sigue siendo
`viv_geo_localidades` (ADR-024/026). El código INDEC no lo reemplaza: se usa como **llave
alternativa** para incorporar planillas de Gobierno. Dos consecuencias:

- Las filas de `ext_geo_censo` que quedaron sin `id_geo` por homónimos (el Cuadro 1.6 del Censo
  no trae departamento) ahora se pueden cerrar, porque el nomenclador sí trae departamento.
- Comunidades regionales y zonas rurales **no son localidades**: necesitan un concepto propio
  ("ente") y no entran al padrón.

### 3.2 `DATOS LOC` destraba dos huecos del tablero v2

> **2026-10-19**: `DATOS LOC` no está disponible (§2). Lo que sigue describe lo que tenía; los
> dos huecos del tablero v2 siguen abiertos (spec Fase 1 v1.2.0 §11.1).

Trae **superficie** y **población 2010** completas para las 452 filas. Son los dos datos que
`spec-resumen-territorial-tablero-v2.md §5` tenía bloqueados sin fuente (densidad y crecimiento
intercensal). También trae electores, empleados y masa salarial (base del indicador "% de
coparticipación destinado a sueldos"). La columna "CONT." del Looker es `CONTINUIDAD`.

Sigue sin fuente: indicadores de necesidad. Polígonos: la planilla de visitas tiene una pestaña
oculta con geometría de radios municipales, pero solo para 175 localidades.

### 3.3 ATP Interno: dónde está y qué se sabe del solapamiento

- "4. ATP 2026" es la planilla de carga del año en curso (2.220 filas, $76.209 M).
- La pestaña `ATP` de "0. Resumen General" es el **histórico acumulado** 2023–2026 (6.032 pagos,
  $146.825 M) e incluye las mismas filas de 2026. La página "ATP" del Looker lee de ahí.
- Cada pago tiene un **referente** (funcionario al que se atribuye). Es el dato más sensible.
- 366 pagos ($12.940 M) tienen como referente al Gobernador.

**Solapamiento con Compromisos — informado por el área el 2026-10-13; contradice lo dicho en la
reunión que originó ADR-029.** La planilla de Compromisos registra los anuncios del Gobernador en
cada visita. Con eso se hace un registro administrativo que impacta en "4. ATP 2026". Por lo
tanto **hay pagos repetidos por localidad en las dos planillas**, y el área no puede
identificarlos: la base de administración no marca cuáles vienen de un compromiso. El área elevó
la consulta a su autoridad.

Consecuencias:

- **Los montos de las dos fuentes no se suman nunca**, en ningún total, hasta tener una regla de
  identificación acordada con el área. Se muestran como dos líneas separadas.
- El cruce de control de ADR-029 deja de ser una verificación y pasa a ser un entregable de la
  Fase 4: medir cuánto de `atp_cronograma_pagos` aparece en ATP Interno (localidad + mes +
  monto) y proponerle al área la regla.

#### 3.3.1 Primer cruce contra datos (2026-10-13)

Lectura de solo lectura, autorizada por el usuario: `atp_cronograma_pagos` de producción (montos
negativos = pagado) contra la pestaña `ATP ACUMULADO` de "4. ATP 2026". Solo 2026: el histórico
está en "0. Resumen General", que ese día no era legible.

| | Pagos | Monto |
|---|---|---|
| Compromisos pagados en 2026 | 952 | $26.993 M |
| ATP Interno 2026 | 2.205 | $76.058 M |
| Pagos de Compromisos con un pago de ATP Interno de **igual localidad y monto**, mismo mes | 135 | $5.450 M |
| Ídem, con un mes de diferencia | 123 | $5.274 M |
| Ídem, en otro mes de 2026 | 139 | $4.834 M |
| Sin pago de igual monto en la localidad | 555 | $11.436 M |

Lectura:

- **Hay superposición real y es parcial.** Al menos el 40 % del monto pagado de Compromisos en
  2026 ($10.724 M) aparece idéntico en ATP Interno dentro de un mes de diferencia; contando
  cualquier mes, el 58 %.
- **No todo se explica pago a pago.** Para el 42 % restante no hay un pago de igual monto. Puede
  estar agrupado o partido: mirando totales por localidad y mes, 159 de 511 combinaciones dan
  exactamente lo mismo en las dos fuentes y en 275 ATP Interno es mayor o igual.
- **El referente no sirve como marca.** De los 397 pagos de ATP Interno que coincidieron, solo 73
  tienen referente Gobernador; los otros 324 están atribuidos a otros funcionarios. A la inversa,
  82 de los 137 pagos con referente Gobernador coinciden con un compromiso.
- Límite del método: los montos redondos pueden coincidir por azar; las coincidencias en "otro
  mes" son las menos confiables. De las 255 localidades con compromisos, 252 también tienen pagos
  en ATP Interno 2026.
- Corrige el dato previo de este spec (una localidad con compromisos pagados y sin pagos en ATP
  Interno): estaba mal leído el signo del cronograma.

Conclusión para la Fase 4: no hay una llave ni una marca que permita deduplicar con certeza.
Se mantiene la regla de no sumar las dos fuentes. La regla definitiva la tiene que dar el área
(por ejemplo, una columna en la base de administración que identifique el compromiso de origen).

### 3.4 Coparticipación: la planilla es mucho más rica que el scraping

`COPA` tiene 60 columnas: además del bruto y el neto, cerca de 40 conceptos de descuento (Caja
de Jubilaciones, APROSS, Fondo Permanente, préstamos BANCOR y otros bancos, Redes de Gas,
Viviendas Semilla, EPEC, comunidades regionales, embargos). Cubre 427 localidades por quincena
desde enero de 2024. Nuestro scraping trae cinco conceptos mensuales.

**Actualización**: COPARTICIPACIÓN y FOFINDES se cargan a pedido, con una semana de demora como
máximo según el área. El scraping queda como respaldo permanente y control cruzado. BANCOR nunca
se actualizó.

### 3.5 Fondo Permanente y Fondo Federal son cosas distintas

- **Fondo Federal**: aportes por etapas (pestaña `FONDO FEDERAL` de Resumen General).
- **Fondo Permanente**: préstamos a municipios que se devuelven en cuotas descontadas de la
  coparticipación (planilla propia). Es un dato nuevo que no habíamos identificado en el Looker.

### 3.6 Flujo inverso

"15. Programas visados por Gobierno" lista expedientes de Cordón Cuneta, Gas, Gas-Escuela,
Saneamiento y Loteo que Gobierno carga a mano. La fuente real es nuestro sistema para los tres
primeros. No se incorpora: corresponde ofrecerle el dato al área.

### 3.7 Pendiente con el área

- **Obras**: la columna de monto "debería ser en USD"; el área va a revisar si hay un error de
  carga en los valores de miles de millones. La Fase 5 no muestra montos de obras hasta que lo
  confirmen.
- **IPCAM** y **planilla de origen de Fondo Federal**: el área las va a compartir.
- **ATP**: respuesta de la autoridad del área sobre la superposición (§3.3).
- **Histórico de ATP Interno 2023–2025**: quedó sin responder si la pestaña `ATP` de Resumen
  General es la fuente autoritativa o si hay planillas anuales anteriores. Volver a preguntar.

### 3.9 Columnas de montos de "0. Resumen General - Vista" (2026-10-19)

La Vista trae por localidad, en **millones de pesos**: `ATP`, `FOCOM PURO`, `COPA`, `FOFINDES`,
`ADELANTO COPA`, `OBRAS`, `FONDO FEDERAL`, `FONDO AMBIENTAL`, `GUARDIAS` (vacía) y `TOTAL`; más dos
derivadas, `ATP/COPA+FOFIN` (proporción) y `TOTAL x HABIT` (en pesos, `TOTAL × 1.000.000 /
POBLACION`). Es la única fuente disponible hoy para FOCOM, Fondo Federal y adelantos por localidad.

**No se usan todavía.** Por qué:

- Son totales agregados del mismo tipo que `RESUMENGRAL`, que el área calificó como no del todo
  confiables y para los que pidió calcular en nuestro backend desde cada planilla base (§3.8). La
  Vista vuelve a ofrecer justamente ese agregado.
- No se sabe **qué período** cubren (¿2024–2026 acumulado?, ¿año en curso?).
- `ATP` no aclara si incluye los pagos de Compromisos del Gobernador (§3.3). Si se mostrara junto
  a la línea de Compromisos, podría contarlos dos veces.
- `OBRAS` arrastra la duda de moneda (§3.7).

Opciones, a decidir por el usuario:

1. **Usarlas como dato provisorio** de las Fases 3 y 7 (total por fondo y por localidad), rotulado
   "agregado informado por Gobierno, período X", sin sumar `ATP` con Compromisos. Requiere que el
   área confirme período y moneda, y un spec hijo.
2. **Usarlas solo como control**: guardar los totales y compararlos con lo que calculemos desde
   las planillas base (COPA y FOFINDES ya tienen la suya), sin mostrarlas.
3. **No usarlas** y esperar las planillas base.

Recomendación: 2 ahora para COPA y FOFINDES, que tienen planilla base; para FOCOM, Fondo Federal y
adelantos, 1 recién con período y moneda confirmados por el área, porque no hay otra fuente.

**Decisión del usuario (2026-10-19): se adopta la recomendación.**

- **COPA y FOFINDES**: los totales de la Vista se guardan **solo como control** contra lo que se
  calcule desde sus planillas base. Nunca se muestran. Se implementa con la Fase 2, que es la que
  trae las planillas base; sin ellas no hay contra qué comparar.
- **FOCOM, Fondo Federal y adelantos**: se usan como **dato provisorio** de la Fase 3, rotulado como
  agregado de Gobierno con su período, **recién cuando el área confirme período y moneda**. Hasta
  entonces no se sincronizan.
- `ATP` de la Vista no se usa hasta que el área aclare si incluye Compromisos (§3.3).
- Pendiente del área (enviado el 2026-10-19): período y moneda de los montos; si `ATP` incluye
  Compromisos; zonas rurales; fuentes para el histórico de ATP, FOCOM, Fondo Federal y adelantos.

### 3.10 Superficie y población 2010: fuentes oficiales externas (2026-10-19)

Decisión del usuario: superficie del radio municipal y población 2010 **no se le piden al área**;
se toman de las fuentes oficiales públicas (las mismas que el área citó: superficie del polígono
del radio según IDECOR, población del Censo 2010 de INDEC). Se configura **en otra sesión**, con su
propio spec hijo (fuente, formato, frecuencia, vínculo por código INDEC o `id_geo`). Al cargarse,
completan `superficie_km2` y `poblacion_2010` de `ext_gob_localidades`, o columnas equivalentes que
defina ese spec, y destraban densidad y crecimiento intercensal sin cambios en el frontend
(spec Fase 1 §8).

**Spec hijo (2026-10-19)**: `spec-datos-externos-fuentes-oficiales.md` (approved). Fuente elegida:
Estadística Córdoba (ArcGIS REST, Censo 2010/2022 por gobierno local con el mismo código INDEC);
la superficie para la densidad es la del Censo, no el radio legal de IDECOR. Suma además
indicadores de condiciones de vida, IPS, RENABAP, equipamientos y estadísticas vitales.

La **ficha institucional restringida** (intendente, electores, masa salarial, contacto, etc.) la
vuelve a pedir el usuario al área, por ser importante. El modelo y el frontend ya la soportan.

### 3.8 Respuestas del área (2026-10-13)

| Tema | Respuesta | Efecto |
|---|---|---|
| Superficie | km², polígono del radio de cada localidad, fuente IDECOR | Confirma la etiqueta "radio municipal" |
| Población 2010 | Censo 2010, INDEC | Sirve para crecimiento intercensal |
| Diferencias de población 2022 | Villa San Isidro: ajuste manual a pedido de la localidad (se unió con La Quintana). Cañada de Machado y Los Pozos: probablemente también ajustes manuales | Se mantiene el Censo (decisión del usuario); la diferencia queda como anomalía informada |
| Zona rural | Es la población sin gobierno local, pero **repartida por departamento según la participación general**, porque el Censo 2022 no la desagrega | Es una estimación: se rotula como tal |
| COPA / FOFINDES | Una semana de demora como máximo | §3.4 |
| BANCOR | Nunca se actualizó | Sale de la Fase 3 |
| FOCOM | `FOCOM PURO` es la fuente original filtrada por tipo de pago; la carga es de administración | Se usa `FOCOM PURO` |
| FOCOM pagado por ATP | Está cargado **en ambos lados** (FOCOM y ATP); por eso existe "FOCOM puro" | Los FOCOM con estado "Pagado ATP" / "Pendiente ATP" no se suman a ATP |
| Fondo Federal | Viene de otra planilla, todavía no compartida | Fase 3 espera esa planilla |
| ATP | Hay registros duplicados con Compromisos, sin forma de identificarlos | §3.3 |
| Obras | Debería ser USD; a revisar | §3.7 |
| Vinculación Comunitaria | En desuso | Sale de la Fase 3 |
| Visitas | Las fechas futuras y filas sin localidad son errores de carga | El sync las descarta y las informa como anomalía |
| `RESUMENGRAL` / `-BRANDAN` | Agregados pedidos por dos despachos; no confiables | Todo cálculo se hace en nuestro backend desde cada planilla base |
| Cambios de estructura | Avisan antes | — |

## 4. Calidad de datos observada (el sync debe tolerarla)

- `ID_LOC` con separador de miles según la planilla (`140.105`, `142,007`): leer solo dígitos.
- `ID_LOC` en `#N/A` cuando el nombre no matcheó el validador de la planilla.
- Formatos numéricos mezclados entre planillas (`$12.400.000` y `$12,400,000`); años como
  `2.023`.
- Fechas en `d/m/aaaa`, `dd/mm/aa` y texto (`24-ene-24`, `30 de enero de 2025`).
- Texto libre con variantes y espacios finales en referentes (un mismo apellido escrito de dos
  formas).
- Filas de total y filas de relleno con id pero sin datos dentro del rango de datos.
- Fórmulas rotas (`#REF!`) en Vinculación Comunitaria — es lo que rompe esa página del Looker.
- En Comunidades Regionales, la columna de destino viene como fecha serial en parte de las filas.

Leer siempre con valores sin formato (`UNFORMATTED_VALUE`) donde se pueda, para no depender de la
configuración regional de cada planilla.

## 5. Sensibilidad de datos

| Nivel | Fuentes | Tratamiento |
|---|---|---|
| Alto — datos personales | Contacto de intendentes (`DATOS LOC`, `Intendentes`); IPCAM | Se sincroniza; visible solo para los roles `Admin` y `Autoridad` |
| Alto — político | ATP Interno (referente); Elecciones; detalle de Reuniones; intendente, partido y legislador | Visible solo para los roles `Admin` y `Autoridad` |
| Medio — financiero | Descuentos de coparticipación, Fondo Permanente, BANCOR, saldos | Roles de lectura del módulo |
| Bajo | Obras, visitas, convenios, polideportivos, nomenclador, demografía | Roles de lectura del módulo |

## 6. Plan por fases

Cada fase requiere su sección o spec hijo en `approved` antes de escribir código. Todas usan el
patrón de sync de Sheets ya probado en `svc-gasifera`/`svc-gralgob` (ADC, SAVEPOINT por fila, log
de corrida append-only, Cloud Scheduler), con vínculo por código INDEC.

| Fase | Qué | Por qué en este orden |
|---|---|---|
| 1 | **Nomenclador + demografía** (`spec-datos-externos-fase1-nomenclador-demografia.md`, approved v1.2.0): tabla puente INDEC ↔ `id_geo` (con `ID_COPA`, tipo y entes), cierre de los `id_geo` nulos del censo, y población 2022 de Gobierno + partido desde la Vista. Superficie, población 2010 y ficha institucional quedan sin fuente (spec hijo §11.1) | Es la base de todas las demás y destraba densidad y crecimiento intercensal en el tablero ya construido. Baja sensibilidad |
| 2 | **Transferencias automáticas**: COPA con descuentos, FOFINDES, FODEMEP, FOMMEP. Conciliación mes a mes contra el scraping | Mayor volumen y mayor valor fiscal; define si el scraping se retira |
| 3 | **Fondos por localidad**: FOCOM (`FOCOM PURO`), Fondo Federal (cuando llegue su planilla de origen), Fondo Ambiental, Fondo Permanente, Adelantos, Comunidades Regionales. Fuera: BANCOR (sin actualizar) y Vinculación Comunitaria (en desuso) | Forma parecida entre sí; completan el "total transferido por localidad" |
| 4 | **ATP Interno** en `svc-gralgob` (spec hijo propio). Empieza por el cruce contra Compromisos y la regla de no duplicación (§3.3); sin esa regla, las dos fuentes se muestran separadas y sin total conjunto | Sensible; la superposición está confirmada por el área |
| 5 | **Gestión territorial**: obras, visitas, reuniones, convenios, polideportivos, parques industriales, empleados | Depende de aclarar la unidad de montos de obras |
| 6 | **Político**: padrón de intendentes y legisladores (ADR-027) y elecciones | Sensible; requiere el mecanismo de ADR-027 y roles |
| 7 | **Réplica**: Resumen General nativo, calculado en nuestro backend desde las planillas base (no se valida contra `RESUMENGRAL`, §3.8), luego páginas por fondo | Necesita fases 1 a 4 |
| 8 | **Paneles editables** que reemplazan planillas, empezando por las sensibles | ADR-028: auth, gateway, roles y audit log en `svc-datos-externos` |

## 7. Preguntas abiertas

Resueltas por el usuario el 2026-10-13:

- **Código INDEC como llave alternativa**: confirmado, sin tocar el padrón oficial (§3.1).
- **Coparticipación**: la planilla de Gobierno pasa a ser la fuente principal; el scraping queda
  como **respaldo permanente** y control cruzado.
- **Roles**: ATP Interno, Elecciones y el detalle de Reuniones los ven **solo los roles
  `Admin` y `Autoridad`** (corregido el 2026-10-13: `Admin` ve todo siempre).
- **Contacto de intendentes**: se sincroniza; lo ven solo los roles `Admin` y `Autoridad`.
- **Escritura hacia las planillas**: ninguna por ahora. El sistema solo lee.

Siguen abiertas:

1. ADR-027: ¿espejo sincronizado en Privada (como ADR-026) o lectura federada? ¿Qué pasa con la
   edición manual de `habitantes` y `electores` en Privada? Hasta definirlo, los datos de
   Gobierno y los de Privada conviven por separado (spec Fase 1 §7.3).
2. Los puntos de §3.7, pendientes del área de Gobierno.
3. ATP: regla para no contar dos veces los pagos que están en Compromisos y en ATP Interno
   (§3.3). Depende del cruce de datos y de la respuesta de la autoridad del área.
4. Qué hacer con las columnas de montos de la Vista (§3.9).
5. Fuentes que reemplacen las pestañas perdidas de "0. Resumen General" (§2): histórico de ATP
   Interno, FOCOM, Fondo Federal, adelantos; y para la Fase 1, superficie, población 2010, zona
   rural y ficha institucional.

## 8. Criterios de aceptación

A definir por fase en su spec hijo. Este spec paraguas no tiene entregables de código.
