# Spec: Tablero de Gobierno nativo (reemplazo del Looker "Tablero - Min. de Gobierno")

**Estado**: draft
**Versión**: 0.3.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-19
**Servicios**: `frontend` (módulo nuevo) · `svc-datos-externos` (endpoints de panel, ADR-028) ·
`svc-gralgob` (ATP) · `svc-vivienda` / `svc-gasifera` (datos propios: expedientes, obras de gas)

Spec hijo de `spec-datos-externos-tablero-gobierno.md` (paraguas): es su **Fase 7 (réplica)**,
planificada desde ahora para que cada fase de datos llegue con su pantalla. Base del relevamiento:
`relevamiento-looker-tablero-gobierno.md`.

## Changelog

- **0.3.0** (2026-10-19): primera revisión de la maqueta por el usuario: 9 correcciones
  incorporadas a la maqueta y a la matriz de paridad (§2.5, §4). Capacitación pasa a llamarse
  "Capacitación IPCAM" y se suma el top 10 de personas con más certificados (nombre y
  localidad, sin DNI), lo que revierte la exclusión de 0.1.0.
- **0.2.0** (2026-10-19): respuestas del usuario a §7 — acceso solo `Admin` y `Autoridad` (todo
  el panel, por ahora); paneles separados del Resumen Territorial y unidos en la Ficha de
  Localidad; ATP Interno solo `Admin`/`Autoridad`; se arma una maqueta navegable. La pregunta de
  Expedientes se reformula (§7).
- **0.1.0** (2026-10-19): primera versión. Plan de trabajo, propuesta de presentación, matriz de
  paridad con el Looker y etapas.

---

## 1. Objetivo

Un panel dentro del sistema que **reemplace al Looker** con:

1. **Paridad funcional**: los mismos indicadores, gráficos, tablas, KPIs y cálculos que hoy tiene
   el Looker (catálogo en el relevamiento §2–§3), con los mismos filtros.
2. **Una sola fuente de verdad**: todo calculado en nuestro backend desde las planillas base (o,
   más adelante, desde paneles editables propios), no desde agregados manuales.
3. **Una presentación más simple y consistente** que la del Looker, con la estética del sistema.
4. **Acceso por rol**: **todo el panel es solo para `Admin` y `Autoridad`** (decisión del
   usuario 2026-10-19; más adelante se evaluará sumar otros perfiles). Las marcas 🔒 quedan
   como capa adicional para el día que se abra a otros roles.

Criterio de "reemplazo": para cada página del Looker, nuestro panel muestra los mismos números con
los mismos filtros (diferencias explicadas y aceptadas por el área). Recién con todas las páginas
en paridad se propone dar de baja el Looker.

## 2. Propuesta de presentación

### 2.1 Qué cambia respecto del Looker

| Looker | Panel nativo |
|---|---|
| 19 pestañas planas | **6 secciones** con sub-pestañas, agrupadas por tema |
| Cada página repite filtros y botón VOLVER | Una barra de filtros global persistente (Departamento · Localidad · Año) que viaja en la URL |
| KPIs con estilos distintos por página | Un único componente de KPI, con comparación (vs. año anterior / % del total) cuando aplica |
| "Grupos por estado" armados a mano en cada fondo (FOCOM, Fondo Federal, Ambiental, Comu. Regionales, Expedientes) | **Un componente "embudo de estados"**: una franja de tarjetas por estado (monto · cantidad · localidades · % del total) que filtra la tabla al hacer clic |
| Tablas paginadas 1–50 sin exportar | Tablas con búsqueda, orden, paginación y exportación a Excel (`exportTable.ts`) |
| Sin mapa | Mapa coroplético por departamento (`CoropletiqueDepartamentos`) en la portada y en cada sección de montos |
| Fecha de actualización al pie | Indicador de frescura por fuente (último sync, y aviso si la fuente falla, como ATP del 9 al 19/10) |
| Datos personales visibles a quien tenga el link | Bloques sensibles solo para `Admin`/`Autoridad`, filtrados en el backend |

Estética: paleta de gobierno (`gov-navy`, `gov-blue`, `gov-cyan`, `gov-orange`), Poppins, tarjetas
blancas con borde suave, mismo lenguaje que el Resumen Territorial y el Tablero analítico de
Vivienda. Componentes existentes a reutilizar: `KpiStrip`, `BarChart`, `LineChart`,
`DonutChart`, `CoropletiqueDepartamentos` (`src/shared/components/informe/`), y la Ficha de
Localidad (`FichaLocalidadPage.tsx`) como vista de detalle de una localidad.

### 2.2 Secciones

| # | Sección | Sub-pestañas | Páginas del Looker que reemplaza |
|---|---|---|---|
| 1 | **Inicio** | — | Tablero (portada) |
| 2 | **Transferencias** | Resumen general · Coparticipación · FOFINDES · FODEMEEP · FOMMEP | Resumen General, COPA, FOFINDES, FODEMEEP, FOMMEP |
| 3 | **Fondos y aportes** | ATP Compromisos · ATP Interno · FOCOM · Fondo Federal · Fondo Ambiental · Comunidades Regionales | Comp. Gober, ATP, FOCOM, Fondo Federal, Fondo Ambiental, Comu. Regionales |
| 4 | **Gestión territorial** | Obras · Visitas · Visitas del Gobernador · Reuniones · Convenios · Capacitación IPCAM | Obras, Visitas, Visitas Gobernador, IPCAM, bloques de portada (Reuniones, Convenios) |
| 5 | **Expedientes** | — | Expedientes (Ministerios) — armado con **nuestros** datos (flujo inverso) |
| 6 | **Político** 🔒 | Intendentes · Legisladores · Elecciones | Elecciones, tablas de intendentes/legisladores de la portada |

Fuera: VINC. COMU (en desuso) y Préstamos BANCOR (nunca se actualizó), salvo que el área los
reactive.

### 2.3 Esqueleto de una pantalla

```
┌───────────────────────────────────────────────────────────────────────────┐
│ Tablero de Gobierno   [Departamento ▾] [Localidad ▾] [Año ▾]   ● datos al 19/10 │
├───────────────────────────────────────────────────────────────────────────┤
│ Inicio | Transferencias | Fondos y aportes | Gestión territorial | …       │
├───────────────────────────────────────────────────────────────────────────┤
│ [KPI] [KPI] [KPI] [KPI] [KPI]                                              │
│ ┌──────── gráfico principal ────────┐ ┌────── mapa por departamento ─────┐ │
│ └───────────────────────────────────┘ └──────────────────────────────────┘ │
│ [Pagado 120 · $X · 80 loc] [Listo p/pagar …] [Trib. Cuentas …] [En trámite …] │  ← embudo de estados
│ ┌──────────────────── tabla (buscar · ordenar · exportar) ─────────────────┐ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

Al elegir una localidad, cada sección muestra sus números de esa localidad y un acceso a su
Ficha de Localidad, donde se suman los bloques de Gobierno (fondos, convenios, visitas).

### 2.4 Inicio (portada)

- Fila de KPIs: Localidades · Población 2022 · Total transferido en el año (y por habitante) ·
  Localidades visitadas (y %) · Compromisos del Gobernador (monto y % pagado).
- Mapa coroplético: total transferido por habitante por departamento (alternable: ATP,
  coparticipación, visitas).
- Tarjetas resumen por sección, cada una con su KPI principal y enlace (reemplazan los bloques
  "INFORMACIÓN …" del Looker).
- Matriz de convenios (SI/NO/NC por localidad) con conteo por convenio.
- 🔒 Intendentes y legisladores: se mueven a "Político".

### 2.5 Correcciones de la primera revisión de la maqueta (2026-10-19)

| # | Pantalla | Pedido del usuario | Cómo queda |
|---|---|---|---|
| 1 | Transferencias › Resumen general | Semáforo en la columna ATP/COPA+FOFIN, como el Looker | Punto de color + valor del ratio: verde > 1, amarillo ≈ 1 (banda ±5 %, a confirmar contra el Looker), rojo < 1. También en el KPI y la fila de total. La columna se muestra como ratio (1,23) y no en %. |
| 2 | Gestión › Obras | Faltan las tablas a nivel localidad | Selector "Por departamento / Por localidad", por defecto departamento (las dos tablas del Looker). "Por localidad": resumen por localidad (obras, en ejecución, finalizadas, avance promedio, monto) y listado de obras con su localidad. Si hay una localidad elegida en el filtro global, se fuerza el nivel localidad. Las obras de varios departamentos no tienen localidad única: solo en la vista por departamento. **A confirmar con el área**: que la planilla 12 trae la localidad de cada obra. |
| 3 | Gestión › Visitas | Falta el ratio de localidades visitadas y el desagregado comunas / municipios | KPIs en % con "X de Y" (total, municipios, comunas); panel de cobertura con barras de progreso; tabla de cobertura por departamento (visitadas, %, municipios y comunas "X de Y"), de menor a mayor. La misma cobertura se agrega a Visitas del Gobernador. |
| 4 | Transferencias › Coparticipación | Sumar "COMPARATIVA - ACUMULADO A <MES> DE CADA AÑO" y "COMPARATIVA - MES DE <MES> DE CADA AÑO" | Panel "Comparativa interanual" debajo de la serie mensual, con los dos gráficos lado a lado y un selector de mes (por defecto, el último mes completo; el Looker lo fija en febrero). Resalta el año del filtro. |
| 5 | Transferencias › FODEMEEP | "Neto a transferir por mes" relativo al total | Barras apiladas: la barra completa es el total convenido del mes, verde lo transferido, rojo lo que falta, % transferido arriba. KPI nuevo "Pendiente de transferir" (monto y % de lo convenido). *El usuario lo pidió en FOFINDES, pero la serie "Neto a transferir por mes" es de FODEMEEP (relevamiento §2.8): se aplicó ahí.* |
| 6 | Transferencias › FOMMEP | Falta el filtro por año | Filtro propio "Toda la gestión / 2024 / 2025 / 2026" (por defecto la gestión, como el Looker) que mueve KPIs y grupos; gráfico de total por año. |
| 7 | Fondos › ATP Interno | Falta la tabla con el detalle por localidad | Tabla por localidad: departamento, localidad, cantidad de pagos, monto, % del total, último pago y referente (🔒), con total. |
| 8 | Fondos › FOCOM | Falta la torta de cómo se pagó | Torta "Cómo se pagó" sobre el monto: pagado directo (FOCOM), pagado vía ATP, pendiente vía ATP y sin pagar (resto de estados). **A confirmar** contra la captura del Looker qué cortes usa. |
| 9 | Gestión › Capacitación | Renombrar a "Capacitación IPCAM"; faltan el gráfico por año e iniciativa y el top de personas con más certificados y su localidad | Renombrado. Barras apiladas de personas capacitadas por año e iniciativa (2024 = cuatrimestre ago–nov). Top 10 de personas con más certificados: nombre, localidad y cantidad, respeta los filtros. **Nunca el DNI** (se usa solo para contar en el backend). Datos personales: solo `Admin`/`Autoridad`, como el resto del panel. |

## 3. Arquitectura

### 3.1 Backend

- **Cálculos en el backend**, en un módulo puro sin acceso a base (`aggregations.py`, como
  `informes/` y `resumen_territorial/`), con un test por fórmula del relevamiento §3. El frontend
  no recalcula montos: dibuja.
- **`svc-datos-externos`** sirve las secciones 2, 3 (salvo ATP), 4 y 6 bajo
  `/api/v1/datos-externos/tablero/{seccion}`. Requiere lo que ADR-028 dejó pendiente: auth
  (`get_current_user` contra `svc-vivienda`, como Privada por ADR-015), roles, paths en
  `infra/gateway/openapi.yaml` con su `options` y config de gateway nueva. Es la **Etapa 0**.
- **`svc-gralgob`** sirve ATP Compromisos (ya existe la lectura) y, en la Fase 4, ATP Interno. El
  frontend los consulta directo; **no se suman** con otros ATP (paraguas §3.3).
- **Expedientes**: se arma con `svc-vivienda` (Cordón Cuneta) y `svc-gasifera` (Gas). No se lee
  la planilla "15. Programas visados".
- Respuesta por sección: `{ kpis, series, estados, filas, frescura }` con filtros
  `departamento`, `id_geo`, `anio`. Listados paginados (`limit`/`offset`).
- Visibilidad: los bloques 🔒 se filtran en el backend con una lista propia de roles
  (`Admin`, `Autoridad`), fail-closed, como `ficha_gobierno` (spec Fase 1 §7).

### 3.2 Frontend

- Módulo nuevo `src/modules/tablero-gobierno/` (una página por sección, componentes compartidos
  en su `components/`): `FiltrosGobierno`, `KpiCard`, `EmbudoEstados`, `TablaGobierno`,
  `ComparativaAnual`, `IndicadorFrescura`.
- Ruta `/tablero-gobierno/:seccion/:subseccion?` con filtros en query string (compartible).
- Entrada en el menú solo para `Admin` y `Autoridad`; el backend responde 403 a cualquier otro
  rol en todos los endpoints del panel.

## 4. Matriz de paridad con el Looker

Cada fila es un criterio de aceptación: "igual que el Looker con el mismo filtro".

| Página Looker | Componentes a replicar (relevamiento) | Sección | Datos | Etapa |
|---|---|---|---|---|
| Tablero | 22 KPIs, intendentes, legisladores, matriz de convenios | Inicio + Político | parcial | E1 → E6 |
| Resumen General | 2 tablas (localidad/depto), ATP/COPA+FOFIN, TOTAL, TOTAL x HABIT | Transferencias › Resumen | Vista (control) → planillas base | E2 → E3 |
| COPA | 9 KPIs, % a sueldos (bruto/neto), 3 comparativas (acumulado a mes y mismo mes de cada año con selector de mes, §2.5 #4) | Transferencias › Coparticipación | scraping → planilla COPA | E1 → E2 |
| FOFINDES | 5 KPIs, serie mensual | Transferencias › FOFINDES | scraping → planilla | E1 → E2 |
| FODEMEEP | 6 KPIs + pendiente de transferir, tabla mensual, neto a transferir apilado transferido/pendiente (§2.5 #5), estados de adhesión | Transferencias › FODEMEEP | planilla | E2 |
| FOMMEP | 2 KPIs + 4 grupos (móviles, edificios 1–3), filtro de año propio (§2.5 #6) | Transferencias › FOMMEP | planilla | E2 |
| Comp. Gober | 5 KPIs, tabla, por ministerio | Fondos › ATP Compromisos | **ya en el sistema** | E1 |
| ATP | 4 KPIs, tabla, gráfico, detalle por localidad (§2.5 #7) | Fondos › ATP Interno 🔒 | 4. ATP 2026 | E4 |
| FOCOM | 3 KPIs + 8 estados, torta de forma de pago (§2.5 #8), tabla | Fondos › FOCOM | falta fuente | E3 |
| Fondo Federal | 7 KPIs + estados con %, tabla de etapas | Fondos › Fondo Federal | falta fuente | E3 |
| Fondo Ambiental | 5 KPIs + 4 estados, tabla | Fondos › Fondo Ambiental | planilla 9 | E3 |
| Comu. Regionales | 5 KPIs + 6 estados, tablas | Fondos › Comunidades Regionales | planilla 17 | E3 |
| Obras | 2 tablas (1 y varios deptos), estado, prorrateo; nivel localidad con selector (§2.5 #2) | Gestión › Obras | planilla 12 (moneda en duda) | E5 |
| Visitas | KPIs con ratio total / municipios / comunas, cobertura por departamento (§2.5 #3), tabla, tabla por funcionario | Gestión › Visitas | planillas 2 y 2.1 | E5 |
| Visitas Gobernador | KPIs con ratio y cobertura por tipo, visitadas / no visitadas | Gestión › Visitas del Gobernador | planilla 2 | E5 |
| IPCAM | 4 KPIs, gráfico por año e iniciativa, top 10 por certificados (nombre y localidad, sin DNI), certificados | Gestión › Capacitación IPCAM | no recibida | E5 |
| Expedientes | 6 KPIs, por programa y estado, tabla | Expedientes | **datos propios** | E1 |
| Elecciones | info localidad, tabla municipal, % por fuerza | Político 🔒 | planilla 18 | E6 |
| VINC. COMU / BANCOR | — | fuera | — | — |

## 5. Etapas

Cada etapa se cierra con su conciliación contra el Looker (§4) y la verificación visual del
usuario. Las etapas E2–E6 dependen de las fases de datos del paraguas; el panel avanza a medida
que llegan las fuentes.

| Etapa | Contenido | Depende de |
|---|---|---|
| **E0 — Base** | Maqueta navegable aprobada (§7.1); ADR-028 en `svc-datos-externos` (auth, roles, gateway); módulo frontend con filtros, componentes compartidos e indicador de frescura | Aprobación de este spec |
| **E1 — Con lo que ya hay** | Inicio parcial (población, ATP Compromisos, transferencias del scraping); Fondos › ATP Compromisos; Transferencias › Coparticipación y FOFINDES desde el scraping (sin % a sueldos); Expedientes con datos propios | E0 |
| **E2 — Transferencias completas** | Coparticipación (quincenas, descuentos, 3 comparativas), FOFINDES, FODEMEEP, FOMMEP desde planillas; % a sueldos (necesita masa salarial: ficha restringida pedida al área); Resumen general calculado + control contra la Vista | Paraguas Fase 2 |
| **E3 — Fondos** | Fondo Ambiental y Comunidades Regionales; FOCOM, Fondo Federal y adelantos cuando lleguen sus fuentes (o como dato provisorio de la Vista, paraguas §3.9) | Paraguas Fase 3 |
| **E4 — ATP Interno** 🔒 | Página ATP; regla de no duplicación con Compromisos | Paraguas Fase 4 |
| **E5 — Gestión territorial** | Obras, visitas, visitas del Gobernador, reuniones, convenios, IPCAM | Paraguas Fase 5 |
| **E6 — Político** 🔒 | Intendentes, legisladores, elecciones | Paraguas Fase 6 + ADR-027 |
| **E7 — Reemplazo** | Paridad completa; período de convivencia; baja del Looker acordada con el área | E1–E6 |

Las páginas de un fondo comparten forma (KPIs + embudo de estados + tabla), así que después de la
primera (E1, ATP Compromisos) cada fondo nuevo es sobre todo trabajo de datos.

## 6. Tests y verificación

- Backend: un test por fórmula (§3 del relevamiento), filtros combinados, visibilidad 🔒 para cada
  rol (fail-closed), paginación.
- Conciliación: por cada página, una tabla "Looker vs. sistema" con el mismo filtro (provincia,
  un departamento, una localidad), guardada en este spec al cerrar la etapa.
- Frontend: `npm run build` y verificación visual del usuario en producción.

## 7. Preguntas abiertas

Resueltas por el usuario (2026-10-19):

- **Maqueta**: sí, se arma una maqueta navegable con datos de ejemplo antes de la Etapa 0.
  Publicada el 2026-10-19 (privada): https://claude.ai/artifact/TJh9BKJ53D29jpP35u98rF.
  Primera revisión del usuario: "muy bien", con 9 correcciones (§2.5), incorporadas en la
  versión 2. Pendiente de aprobación final; su aprobación abre la Etapa 0.
- **Usuarios**: por ahora solo `Admin` y `Autoridad`. Más adelante se verá si se suma algo.
- **Relación con el Resumen Territorial**: paneles separados; se unen en la Ficha de Localidad.
- **ATP Interno**: solo `Admin` y `Autoridad`.

Siguen abiertas:

1. **Expedientes (reformulada)**: la página "Expedientes (Ministerios)" del Looker la alimenta una
   planilla que el área carga a mano copiando expedientes de Cordón Cuneta y Gas, que **ya están
   en nuestro sistema**. Nuestro panel la arma con nuestros datos. ¿Se le avisa al área que puede
   dejar de cargar esa planilla y mirar el panel? Hoy el panel es solo para `Admin`/`Autoridad`,
   así que la pregunta aplica cuando se abra a otros usuarios.
2. Pendientes del relevamiento (§5): capturas de los gráficos, "CAJA" en la comparativa, período de
   los montos de la Vista, moneda de Obras.

## 8. Criterios de aceptación (del spec completo)

1. Las 17 páginas en alcance (19 menos VINC. COMU y BANCOR) tienen su sección y pasan la
   conciliación de §6.
2. Ningún endpoint del panel responde datos a usuarios fuera de `Admin`/`Autoridad` (403).
3. Cada fuente muestra su frescura y un aviso cuando su último sync falló.
4. El área valida el panel y acuerda la baja del Looker.
