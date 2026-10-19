# Spec: Tablero de Gobierno nativo (reemplazo del Looker "Tablero - Min. de Gobierno")

**Estado**: draft
**Versión**: 0.1.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-19
**Servicios**: `frontend` (módulo nuevo) · `svc-datos-externos` (endpoints de panel, ADR-028) ·
`svc-gralgob` (ATP) · `svc-vivienda` / `svc-gasifera` (datos propios: expedientes, obras de gas)

Spec hijo de `spec-datos-externos-tablero-gobierno.md` (paraguas): es su **Fase 7 (réplica)**,
planificada desde ahora para que cada fase de datos llegue con su pantalla. Base del relevamiento:
`relevamiento-looker-tablero-gobierno.md`.

## Changelog

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
4. **Acceso por rol**: lo sensible solo para `Admin` y `Autoridad` (paraguas §5).

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
| 4 | **Gestión territorial** | Obras · Visitas · Visitas del Gobernador · Reuniones · Convenios · Capacitación (IPCAM) | Obras, Visitas, Visitas Gobernador, IPCAM, bloques de portada (Reuniones, Convenios) |
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
- Entrada en el menú para la secretaría General de Gobierno y para `Admin`/`Autoridad`.

## 4. Matriz de paridad con el Looker

Cada fila es un criterio de aceptación: "igual que el Looker con el mismo filtro".

| Página Looker | Componentes a replicar (relevamiento) | Sección | Datos | Etapa |
|---|---|---|---|---|
| Tablero | 22 KPIs, intendentes, legisladores, matriz de convenios | Inicio + Político | parcial | E1 → E6 |
| Resumen General | 2 tablas (localidad/depto), ATP/COPA+FOFIN, TOTAL, TOTAL x HABIT | Transferencias › Resumen | Vista (control) → planillas base | E2 → E3 |
| COPA | 9 KPIs, % a sueldos (bruto/neto), 3 comparativas | Transferencias › Coparticipación | scraping → planilla COPA | E1 → E2 |
| FOFINDES | 5 KPIs, serie mensual | Transferencias › FOFINDES | scraping → planilla | E1 → E2 |
| FODEMEEP | 6 KPIs, tabla mensual, estados de adhesión | Transferencias › FODEMEEP | planilla | E2 |
| FOMMEP | 2 KPIs + 4 grupos (móviles, edificios 1–3) | Transferencias › FOMMEP | planilla | E2 |
| Comp. Gober | 5 KPIs, tabla, por ministerio | Fondos › ATP Compromisos | **ya en el sistema** | E1 |
| ATP | 4 KPIs, tabla, gráfico | Fondos › ATP Interno 🔒 | 4. ATP 2026 | E4 |
| FOCOM | 3 KPIs + 8 estados, tabla | Fondos › FOCOM | falta fuente | E3 |
| Fondo Federal | 7 KPIs + estados con %, tabla de etapas | Fondos › Fondo Federal | falta fuente | E3 |
| Fondo Ambiental | 5 KPIs + 4 estados, tabla | Fondos › Fondo Ambiental | planilla 9 | E3 |
| Comu. Regionales | 5 KPIs + 6 estados, tablas | Fondos › Comunidades Regionales | planilla 17 | E3 |
| Obras | 2 tablas (1 y varios deptos), estado, prorrateo | Gestión › Obras | planilla 12 (moneda en duda) | E5 |
| Visitas | 2 KPIs, tabla, tabla por funcionario | Gestión › Visitas | planillas 2 y 2.1 | E5 |
| Visitas Gobernador | KPIs, visitadas / no visitadas | Gestión › Visitas del Gobernador | planilla 2 | E5 |
| IPCAM | 4 KPIs, gráfico por año e iniciativa, certificados (sin DNI) | Gestión › Capacitación | no recibida | E5 |
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

1. **Maqueta**: ¿se aprueba la estructura de 6 secciones y el esqueleto de §2.3? Propuesta: armar
   una maqueta navegable con datos de ejemplo antes de la Etapa 0.
2. **Usuarios**: ¿quiénes lo usan además de `Admin`/`Autoridad`? ¿Usuarios de la secretaría General
   de Gobierno con rol `Consulta`? Define qué ve cada uno fuera de lo 🔒.
3. **Relación con el Resumen Territorial**: propuesta de mantenerlos separados (el Resumen es "qué
   programas tiene cada localidad"; este tablero es "qué recibe cada localidad del Gobierno") y
   unirlos en la Ficha de Localidad. ¿De acuerdo?
4. **Expedientes**: ¿se le ofrece al área esta página en lugar de su planilla "15. Programas
   visados" (flujo inverso)?
5. ATP Interno: ¿lo ven solo `Admin`/`Autoridad` (paraguas §5) o también la secretaría?
6. Pendientes del relevamiento (§5): capturas de los gráficos, "CAJA" en la comparativa, período de
   los montos de la Vista, moneda de Obras.

## 8. Criterios de aceptación (del spec completo)

1. Las 17 páginas en alcance (19 menos VINC. COMU y BANCOR) tienen su sección y pasan la
   conciliación de §6.
2. Los bloques 🔒 no llegan en la respuesta de la API a usuarios fuera de `Admin`/`Autoridad`.
3. Cada fuente muestra su frescura y un aviso cuando su último sync falló.
4. El área valida el panel y acuerda la baja del Looker.
