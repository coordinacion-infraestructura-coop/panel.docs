# Spec: Integración del "Tablero - Min. de Gobierno" (Looker Studio) al sistema

**Estado**: draft
**Versión**: 0.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-09
**Servicios**: `svc-datos-externos` (amplía alcance, ADR-028) + `svc-gralgob` (ATP, ADR-029) + `svc-vivienda` (`resumen_territorial`, consumidor)

Spec paraguas de exploración. Define el inventario de fuentes, las decisiones ya tomadas y el
orden de incorporación. **No habilita implementación**: cada fase se construye recién cuando su
sección (o un spec hijo) pase a `approved`.

---

## 0. Origen

El área de Gobierno compartió el Looker Studio "Tablero - Min. de Gobierno"
(`https://datastudio.google.com/u/0/reporting/70162e1c-d89d-4e1f-b1d0-afeedaac82f5/page/SKu1D`,
acceso de vista con `infraestructura.coop@gmail.com`). El objetivo del usuario es:

1. Sumar a nuestro sistema la información de ese tablero que hoy no tenemos.
2. Reemplazar fuentes propias frágiles por las de Gobierno donde sean mejores.
3. Como horizonte, replicar el tablero dentro del sistema.
4. Más adelante, que las planillas que lo alimentan (primero las sensibles) pasen a ser paneles
   editables propios, para que las áreas no carguen información sensible en Google Sheets.

Relevamiento hecho el 2026-10-09 en modo vista, sin editar nada.

### 0.1 Límites del relevamiento

- **No se conocen las fuentes de datos reales** (qué Sheet y qué pestaña alimenta cada página):
  el modo vista no las muestra. Está pedido al área (§6).
- El contenido que es sólo gráfico no se relevó (comparativas de COPA, gráfico de IPCAM, mapas):
  las capturas de pantalla fallaron en las páginas pesadas y se leyó todo como texto.
- De cada tabla se leyeron las primeras filas visibles: alcanza para columnas y forma, no para
  volumen ni calidad de datos.

## 1. Decisiones ya tomadas (2026-10-09)

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
| Fuente | Sheet `1x3E73hlGwDPBiLc9fbQuWxWUxjaibs3glModHsxCMZM` | Planilla a pedir al área |
| Spec | `spec-sync-atp-compromiso-gobernador.md` (approved) | A abrir (spec hijo) |
| Sensibilidad | Media | Alta |
| Destino | Sigue como está | Módulo y panel nuevo en `svc-gralgob`; candidata prioritaria a cargarse sólo en el sistema |

Según informó el área en reunión, **los registros no se repiten** entre una y otra. Es un dato
informado, no verificado: antes de sumar ambas en cualquier total hay que correr un cruce de
control (localidad + fecha + monto) sobre los datos reales.

En el Looker hay dos páginas: "Comp. Gober" (1.173 compromisos) y "ATP" (≈6.000 pagos con fecha,
localidad y monto). A confirmar con el área si la página "ATP" es exactamente la planilla ATP
Interno Gobierno.

## 2. Inventario del tablero

19 páginas, filtros globales por Departamento y Localidad, universo de 427 localidades (coincide
con las 427 filas de `ext_geo_censo`).

| Página | Contenido | Estado en nuestro sistema | Servicio destino |
|---|---|---|---|
| Tablero (portada) | Población, electores, empleados municipales; intendente y legislador; matriz de convenios; KPIs de cada fondo; préstamos BANCOR; reuniones en el Panal | Parcial | `svc-datos-externos` |
| Resumen General | Una fila por localidad y por departamento: ATP, FOCOM, COPA, FOFINDES, adelanto de copa, obras, Fondo Federal, Fondo Ambiental, total y total por habitante | No existe | Se arma por federación (`resumen_territorial`) |
| Obras | Obras provinciales por tipo (HYG, CASISA, EPEC, Vialidad, Arquitectura, Inf. Eléctrica), estado, fechas, avance, monto | Sólo gas (`svc-gasifera`) | `svc-datos-externos` |
| Visitas | Visitas de funcionarios: fecha, localidad, funcionario, tipo | No existe | `svc-datos-externos` |
| Visitas Gobernador | Última visita por localidad | Parcial (filtro en la Ficha) | `svc-datos-externos` |
| COPA | Coparticipación bruta y neta, por quincena, 2023–2026; % destinado a sueldos | Parcial (`ext_transferencias`, mensual, por scraping) | `svc-datos-externos` |
| FOFINDES | Bruto y neto mensual, 2024–2026 | Parcial (concepto de `ext_transferencias`) | `svc-datos-externos` |
| FODEMEEP | Neto mensual por localidad; estado de adhesión | No existe | `svc-datos-externos` |
| FOMMEP | Montos y cantidades: móviles, edificios cat. 1 a 3 | No existe | `svc-datos-externos` |
| ATP | Pagos: fecha, localidad, monto; saldo disponible | No existe | `svc-gralgob` |
| Comp. Gober | Compromisos: anuncio, monto, pagado, derivado, destino | Existe | `svc-gralgob` |
| Fondo Federal | Etapas por localidad, otorgado, estado, disponible, saldo en cuenta | No existe | `svc-datos-externos` |
| Fondo Ambiental | Monto, estado, destino por localidad o ente | No existe | `svc-datos-externos` |
| VINC. COMU | Vinculación Comunitaria — **rota en el tablero** (error de conexión al conjunto de datos) | No existe | `svc-datos-externos` |
| FOCOM | Convenio, fecha, estado de pago, monto, concepto, estado de obra | No existe | `svc-datos-externos` |
| Comu. Regionales | Fondos para obras por departamento, cuotas, rendición | No existe | `svc-datos-externos` |
| Expedientes (Ministerios) | Expedientes visados de Cooperativas y Mutuales (Cordón Cuneta, Gas, Gas-Escuela) | **Es dato nuestro** (`svc-vivienda`, `svc-gasifera`) | Flujo inverso, ver §3.3 |
| IPCAM | Capacitación municipal: personas, certificados, por localidad | No existe | `svc-datos-externos` |
| Elecciones | Resultados por localidad: 2021, 2023 (legislativas y gobernador), 2025 | No existe | `svc-datos-externos` |

## 3. Análisis

### 3.1 Información nueva de más valor

- **Resumen General**: todo lo que la Provincia transfiere a cada localidad en una fila, con
  total por habitante. Es el objetivo natural de la réplica y se apoya en casi todas las demás
  fuentes.
- **Fondos por localidad con estado de pago** (ATP, FOCOM, Fondo Federal, Fondo Ambiental,
  FODEMEEP, FOMMEP, Comunidades Regionales): fuentes de forma parecida (localidad, monto,
  estado, fecha).
- **Empleados municipales y masa salarial**: habilita el indicador de qué porcentaje de la
  coparticipación se destina a sueldos.
- **Matriz de convenios**: 15 columnas SI/NO por localidad.
- **Obras provinciales** fuera de gas, y **visitas** de funcionarios.

Ninguna de estas fuentes cubre los huecos bloqueados de `spec-resumen-territorial-tablero-v2.md
§5` (población 2010, superficie/densidad, indicadores de necesidad, polígonos de localidad).

### 3.2 Fuentes propias candidatas a reemplazo

- **Scraping de PDFs de transferencias** (`ext_transferencias`): el tablero tiene coparticipación
  y FOFINDES con bruto y neto, por quincena, desde 2023. Si detrás hay una planilla, es más
  estable que el scraping (el sitio bloquea clientes automatizados) y trae más detalle. A
  decidir al ver la fuente: reemplazo total, o scraping como respaldo.
- **Padrón político de Privada** (`priv_localidades_info`): reemplazado como autoridad por
  ADR-027.

### 3.3 Flujo inverso

"Expedientes (Ministerios)" es una planilla que Gobierno mantiene a mano sobre programas que son
nuestros. La fuente real es nuestro sistema: en vez de leerla, corresponde ofrecerle el dato al
área (exportación o lectura directa). No es una fuente a incorporar.

### 3.4 Inconsistencias a confirmar con el área

- **Obras**: los títulos dicen "en millones de dólares" pero hay montos de miles de millones para
  obras chicas. La unidad no cierra.
- **Población**: la portada da 3.840.905 y el Resumen General 3.770.861 — dos bases distintas.
- **Visitas**: hay fechas futuras y filas sin localidad.
- **Columna "CONT."** del padrón de intendentes: significado desconocido.
- **FOCOM**: tiene estados "Pagado ATP" / "Pendiente ATP" — hay FOCOM que se pagan por ATP.
  Riesgo de doble conteo entre FOCOM y ATP al armar totales por localidad.

## 4. Sensibilidad de datos

| Nivel | Fuentes | Tratamiento |
|---|---|---|
| Alto — datos personales | Contactos de intendentes; DNI de personas capacitadas (IPCAM) | No se exponen en rollups, exportaciones generales ni en `resumen_territorial`. Evaluar si IPCAM necesita el DNI o alcanza con agregados |
| Alto — político | ATP Interno Gobierno; Elecciones; partido cruzado con montos transferidos | Visibilidad acotada por rol; fuera de la vista de `Consulta`/`Operador` por defecto |
| Medio — financiero | Saldos de cuenta y disponibles de fondos | Roles de lectura del módulo |
| Bajo | Obras, visitas, convenios, fondos ya publicados | Roles de lectura del módulo |

El orden para pasar de planilla a panel editable propio (Fase D) sigue esta tabla de arriba hacia
abajo.

## 5. Plan por fases

Cada fase requiere su sección o spec hijo en `approved` antes de escribir código.

| Fase | Qué | Depende de |
|---|---|---|
| 0 | Obtener las planillas fuente y aclarar §3.4 | Respuesta del área (§6) |
| A | Sync de solo lectura de los fondos por localidad en `svc-datos-externos` (patrón `sync-sheets-to-cloudsql`, vínculo por `id_geo` vía ADR-024) y federación al Resumen Territorial. ATP Interno en `svc-gralgob`, con spec propio | Fase 0 |
| B | Reemplazos: coparticipación/FOFINDES desde la fuente de Gobierno; padrón político según ADR-027 | Fase 0; definición del mecanismo de ADR-027 |
| C | Réplica del tablero: primero Resumen General, después páginas por fondo | Fases A y B |
| D | Paneles editables que reemplazan planillas, empezando por las sensibles | ADR-028: auth, gateway, roles, audit log en `svc-datos-externos` |

## 6. Pedido al área (Fase 0)

Compartir cada planilla como **Lector** con:

- `infraestructura.coop@gmail.com` — para relevar columnas y pestañas.
- `svc-datos-externos@gestorcooperativo.iam.gserviceaccount.com` — cuenta de servicio que lee en
  cada sync. **Pendiente de verificar contra GCP** que exista con ese nombre exacto.
- `svc-gralgob@gestorcooperativo.iam.gserviceaccount.com` — sólo para la planilla ATP Interno
  Gobierno (ya tiene acceso a la de Compromisos).

Y pedir, por cada página del tablero: qué planilla y pestaña la alimenta, si alguna no sale de un
Sheet, y quién la mantiene.

## 7. Preguntas abiertas

1. ¿La página "ATP" del Looker es exactamente la planilla ATP Interno Gobierno?
2. ADR-027: ¿espejo sincronizado en Privada (como ADR-026) o lectura federada? ¿Qué pasa con la
   edición manual de `habitantes` y `electores` en Privada?
3. Coparticipación: ¿reemplazo total del scraping o scraping como respaldo?
4. ¿Qué roles ven ATP Interno y Elecciones? ¿Hace falta un rol nuevo o alcanza con
   `Admin`/`Autoridad`?
5. IPCAM: ¿se necesita el dato nominal o alcanza con agregados por localidad?
6. Los puntos de §3.4.

## 8. Criterios de aceptación

A definir por fase al pasar cada una a `review`. Este spec en `draft` no tiene entregables de
código.
