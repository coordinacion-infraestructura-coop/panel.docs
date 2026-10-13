# Spec: Integración del "Tablero - Min. de Gobierno" (Looker Studio) al sistema

**Estado**: approved
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-13
**Servicios**: `svc-datos-externos` (amplía alcance, ADR-028) + `svc-gralgob` (ATP, ADR-029) + `svc-vivienda` (`resumen_territorial`, consumidor)

Spec paraguas. Define el inventario de fuentes, las decisiones ya tomadas y el orden de
incorporación. **No habilita implementación por sí solo**: cada fase se construye recién cuando su
sección (o un spec hijo) pase a `approved`.

## Changelog

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
| Fuente | Sheet `1x3E73hlGwDPBiLc9fbQuWxWUxjaibs3glModHsxCMZM` | Sheet `1h3FzAL-X0_tgIGNgnZ_DgKd5rDo8oLg0wNxY0gisBAU` ("4. ATP 2026") + histórico en "0. Resumen General" (§3.3) |
| Spec | `spec-sync-atp-compromiso-gobernador.md` (approved) | A abrir (spec hijo) |
| Sensibilidad | Media | Alta (incluye el referente político de cada pago) |
| Destino | Sigue como está | Módulo y panel nuevo en `svc-gralgob` |

## 2. Inventario de planillas fuente

Todas las planillas comparten una pestaña `Validadores` (copia del padrón de localidades) y
varias tienen pestañas derivadas (`SUMA`, tablas dinámicas, copias). **Solo se sincroniza la
pestaña base** de cada una.

| Planilla (título real) | Pestaña base | Filas | Contenido | Destino |
|---|---|---|---|---|
| 0. LOCALIDADES - NOMBRES | `LOCALIDADES` | 478 | Nomenclador: `ID_LOC` (INDEC), nombre, `ID_DPTO`, `ID_COPA`, tipo MU/CO | `svc-datos-externos` (§3.1) |
| 0. Resumen General | `DATOS LOC` | 452 | Superficie, población 2022 y 2010, electores, empleados, masa salarial, intendente, reelecto, partido, continuidad, legislador, contacto | `svc-datos-externos` |
| 0. Resumen General | `ATP` | 6.032 | Histórico de ATP Interno 2023–2026: fecha, localidad, monto, referente | `svc-gralgob` |
| 0. Resumen General | `FONDO FEDERAL` | 1.503 | Etapas por localidad: monto, disponible, estado | `svc-datos-externos` |
| 0. Resumen General | `FOCOM PURO` | 361 | FOCOM vigente: monto, concepto, categoría, estado, fecha de pago | `svc-datos-externos` |
| 0. Resumen General | `ADELANTOS` | 294 | Adelantos de coparticipación 2023–2025 | `svc-datos-externos` |
| 0. Resumen General | `RESUMENGRAL` | 427 | Tabla ya calculada por localidad (derivada; sirve para validar nuestra réplica) | No se sincroniza |
| 4. ATP 2026 | `ATP ACUMULADO` | 2.220 | ATP Interno del año en curso (planilla de carga) | `svc-gralgob` |
| 11.1 COPARTICIPACION | `COPA` | 28.182 | Por localidad y quincena, 2024–2026: bruto, neto y ~40 conceptos de descuento | `svc-datos-externos` |
| 11.1 FOFINDES | `FOFINDES` | 14.518 | Mensual 2024–2026: parte variable, excedentes, descuentos, neto | `svc-datos-externos` |
| 11.1 FODEMEP | `FODEMEP`, `ESTADO` | 14.785 / 427 | Mensual 2023–2026 y estado de adhesión con saldo | `svc-datos-externos` |
| 11.1 FOMMEP | `FROMMEP` | 854 | Anual: móviles y edificios por categoría | `svc-datos-externos` |
| 7. FONDO PERMANENTE | `FONDO PERM CUOTAS`, `IMPORT_Listo` | 962 / 865 | Préstamos a municipios en cuotas descontadas de coparticipación; destino del préstamo | `svc-datos-externos` |
| 8. BANCOR | `Prestamos` | 99 | Préstamos BANCOR: monto calificado, liquidado, concepto | `svc-datos-externos` |
| 9. FONDO AMBIENTAL | `Fondo` | 135 | Expediente, convenio, resolución, objeto, monto, estado | `svc-datos-externos` |
| 14. Vinculacion Comunitaria | `BD` | 388 | Aportes por beneficiario: monto, estado, expediente, rendición | `svc-datos-externos` |
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

**Sin fuente recibida**: IPCAM (capacitación municipal). Hay que pedirla.

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
- 366 pagos ($12.940 M) tienen como referente al Gobernador. Eso **no prueba** solapamiento con
  la planilla de Compromisos, pero obliga al cruce de control de ADR-029 antes de sumar ambas.
- Verificación puntual hecha: una localidad con $230 M de compromisos pagados en octubre de 2026
  no tiene ningún pago en ATP Interno ese año. Es consistente con que no se repiten, pero es un
  solo caso.

### 3.4 Coparticipación: la planilla es mucho más rica que el scraping

`COPA` tiene 60 columnas: además del bruto y el neto, cerca de 40 conceptos de descuento (Caja
de Jubilaciones, APROSS, Fondo Permanente, préstamos BANCOR y otros bancos, Redes de Gas,
Viviendas Semilla, EPEC, comunidades regionales, embargos). Cubre 427 localidades por quincena
desde enero de 2024. Nuestro scraping trae cinco conceptos mensuales.

**Riesgo**: según "Responsables BD", COPARTICIPACIÓN, FOFINDES y BANCOR se actualizan **a
pedido**, no automáticamente. El resto figura como automática. Antes de reemplazar el scraping
hay que medir cuánto se atrasa la planilla; hasta entonces el scraping queda como respaldo y
como control cruzado.

### 3.5 Fondo Permanente y Fondo Federal son cosas distintas

- **Fondo Federal**: aportes por etapas (pestaña `FONDO FEDERAL` de Resumen General).
- **Fondo Permanente**: préstamos a municipios que se devuelven en cuotas descontadas de la
  coparticipación (planilla propia). Es un dato nuevo que no habíamos identificado en el Looker.

### 3.6 Flujo inverso

"15. Programas visados por Gobierno" lista expedientes de Cordón Cuneta, Gas, Gas-Escuela,
Saneamiento y Loteo que Gobierno carga a mano. La fuente real es nuestro sistema para los tres
primeros. No se incorpora: corresponde ofrecerle el dato al área.

### 3.7 Pendiente de aclarar con el área

- **Obras**: los montos más altos (miles de millones en obras rotuladas "en millones de
  dólares") no cierran con la unidad declarada.
- **Población**: `DATOS LOC` y el Censo cargado en `ext_geo_censo` pueden diferir; definir cuál
  manda.
- **IPCAM**: falta la planilla.
- **Histórico de ATP Interno**: confirmar que la pestaña `ATP` de Resumen General es la fuente
  autoritativa de 2023–2025 (o si existen planillas anuales anteriores).

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
| Alto — datos personales | Contacto de intendentes (`DATOS LOC`, `Intendentes`); IPCAM | Se sincroniza; visible solo para el rol `Autoridad` |
| Alto — político | ATP Interno (referente); Elecciones; detalle de Reuniones; intendente, partido y legislador | Visible solo para el rol `Autoridad` |
| Medio — financiero | Descuentos de coparticipación, Fondo Permanente, BANCOR, saldos | Roles de lectura del módulo |
| Bajo | Obras, visitas, convenios, polideportivos, nomenclador, demografía | Roles de lectura del módulo |

## 6. Plan por fases

Cada fase requiere su sección o spec hijo en `approved` antes de escribir código. Todas usan el
patrón de sync de Sheets ya probado en `svc-gasifera`/`svc-gralgob` (ADC, SAVEPOINT por fila, log
de corrida append-only, Cloud Scheduler), con vínculo por código INDEC.

| Fase | Qué | Por qué en este orden |
|---|---|---|
| 1 | **Nomenclador + demografía** (`spec-datos-externos-fase1-nomenclador-demografia.md`, approved): tabla puente INDEC ↔ `id_geo` (con `ID_COPA`, tipo y entes), cierre de los `id_geo` nulos del censo, y superficie / población 2010 / electores desde `DATOS LOC` | Es la base de todas las demás y destraba densidad y crecimiento intercensal en el tablero ya construido. Baja sensibilidad |
| 2 | **Transferencias automáticas**: COPA con descuentos, FOFINDES, FODEMEP, FOMMEP. Conciliación mes a mes contra el scraping | Mayor volumen y mayor valor fiscal; define si el scraping se retira |
| 3 | **Fondos por localidad**: FOCOM, Fondo Federal, Fondo Ambiental, Fondo Permanente, Adelantos, BANCOR, Comunidades Regionales, Vinculación Comunitaria | Forma parecida entre sí; completan el "total transferido por localidad" |
| 4 | **ATP Interno** en `svc-gralgob` (spec hijo propio), con el cruce de control contra Compromisos | Sensible; necesita definir roles antes |
| 5 | **Gestión territorial**: obras, visitas, reuniones, convenios, polideportivos, parques industriales, empleados | Depende de aclarar la unidad de montos de obras |
| 6 | **Político**: padrón de intendentes y legisladores (ADR-027) y elecciones | Sensible; requiere el mecanismo de ADR-027 y roles |
| 7 | **Réplica**: Resumen General nativo (validado contra `RESUMENGRAL`), luego páginas por fondo | Necesita fases 1 a 4 |
| 8 | **Paneles editables** que reemplazan planillas, empezando por las sensibles | ADR-028: auth, gateway, roles y audit log en `svc-datos-externos` |

## 7. Preguntas abiertas

Resueltas por el usuario el 2026-10-13:

- **Código INDEC como llave alternativa**: confirmado, sin tocar el padrón oficial (§3.1).
- **Coparticipación**: la planilla de Gobierno pasa a ser la fuente principal; el scraping queda
  como **respaldo permanente** y control cruzado.
- **Roles**: ATP Interno, Elecciones y el detalle de Reuniones los ve **solo el rol
  `Autoridad`**.
- **Contacto de intendentes**: se sincroniza; lo ve solo el rol `Autoridad`.
- **Escritura hacia las planillas**: ninguna por ahora. El sistema solo lee.

Siguen abiertas:

1. ADR-027: ¿espejo sincronizado en Privada (como ADR-026) o lectura federada? ¿Qué pasa con la
   edición manual de `habitantes` y `electores` en Privada? Hasta definirlo, los datos de
   Gobierno y los de Privada conviven por separado (spec Fase 1 §7.3).
2. Los puntos de §3.7, en consulta con el área de Gobierno.

## 8. Criterios de aceptación

A definir por fase al pasar cada una a `review`. Este spec en `draft` no tiene entregables de
código.
