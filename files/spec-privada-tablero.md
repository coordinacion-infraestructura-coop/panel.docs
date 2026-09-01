# Spec: svc-privada — Tablero React nativo (reemplazo del iframe Looker Studio)

**Estado**: **implementado** (pendiente de deploy + paridad numérica vs Looker)
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-01

> **Implementado 2026-09-01** — `panel.front` `6791757`:
> `src/modules/privada/api/informe.api.ts` (los 4 endpoints ya portados:
> `resumen`/`temporal`/`por-departamento`/`puntos`) + `TableroPage.tsx` reescrito **sin `<iframe>`
> ni URLs de Looker/BigQuery** (`grep` de `lookerstudio`/`bigquery` en `src/` = 0). Contenido:
> KPIs (total / en curso / finalizadas / urgentes / archivadas), donut por tema, barras top-15
> departamentos, línea de evolución mensual, mapa de puntos (`MapaDualPuntos`); filtros
> `fecha_desde` / `fecha_hasta` + `tema`. Reutiliza `shared/components/informe/*` (mismo
> `chartSetup.ts` que los informes de Vivienda). **No** se necesitaron endpoints de agregación
> nuevos — los 4 existentes cubren el tablero. Cómputo en request (no snapshot; ~2100 filas).
>
> **Pendiente**: comparación de paridad numérica contra el informe Looker `f9dc4a4e-…` para un
> rango de control **antes** de apagar BigQuery (criterio de aceptación 2). El inventario formal
> del Anexo A no se hizo — el tablero cubre las dimensiones del informe (tema, departamento,
> tiempo, mapa); si el área detecta un KPI faltante se agrega en v1.1.
**Servicio**: `svc-privada` (endpoints de agregación) + frontend `src/modules/privada/pages/TableroPage.tsx`
**Depende de**: `spec-migracion-svc-privada.md` `approved` + Fase 6 (cutover) completada.
**ADRs**: ADR-014 (Tablero nativo, sin mirror de BigQuery).

---

## 0. Origen

`TableroPage.tsx` embebe hoy un iframe de Looker Studio (informe
`f9dc4a4e-a174-45a8-938c-385f4121f689`) que lee BigQuery `infra_gestion` directamente. Al apagar
BigQuery (decommission del sistema viejo, `spec-migracion §10`) ese iframe muere. Decisión D-4 /
ADR-014: **reconstruir el tablero nativo en React**, sin mirror de BigQuery.

Este spec es **prerequisito del apagado de BigQuery** (gate del decommission), no del cutover.

## 1. Propósito

Dar al área un tablero analítico equivalente al informe Looker actual, servido por endpoints de
agregación de `svc-privada` sobre `db_privada`, sin ninguna dependencia del frontend con BigQuery.

## 2. Alcance

### Incluido
- Inventario de los gráficos/KPIs del informe Looker actual (captura previa — Anexo A).
- Endpoints de agregación bajo `/api/v1/privada/informe/**` — se reutilizan los 4 que ya porta la
  migración (`resumen`, `temporal`, `por-departamento`, `puntos`) y se agregan los que falten para
  cubrir el Looker (p. ej. por estado, por urgencia, por categoría/tema, por ministerio).
- Si el cómputo resulta caro, patrón snapshot append-only (`viv_informe_snapshot` de vivienda como
  precedente) con `POST .../actualizar` + `POST /internal/...` para Cloud Scheduler. Con ~523 filas
  probablemente **no** haga falta — cómputo en request.
- `TableroPage.tsx`: reemplaza el `<iframe>` por componentes de `shared/components/informe/`
  (`KpiStrip`, `BarChart`, `LineChart`, `DonutChart`, `CoropletiqueDepartamentos`, `MapaDualPuntos`)
  ya usados por los informes de Vivienda.
- Filtros de fecha (`fecha_desde`/`fecha_hasta`) equivalentes a los del Looker.

### Fuera de alcance
- Mantener Looker Studio o cualquier mirror de BigQuery (rechazado por ADR-014).
- Reportes nuevos que el Looker no tiene (posible v2).
- La reclasificación `tema_informe` → categorías nuevas (ese es `spec-privada-informe-cooperativas-v2.md`);
  el Tablero consume lo que el informe exponga en el momento.

## 3. Endpoints

Base `/api/v1/privada/informe/cooperativas` (ya portados por la migración) + los que el inventario
del Anexo A determine que faltan. Todos `ROLES_LECTURA`, paginación no aplica (son agregados).

## 4. Frontend

- Un archivo `TableroPage.tsx` (monolítico, consistente con el resto del módulo).
- `useQuery` por endpoint con `staleTime` generoso.
- Reutiliza `chartSetup.ts` (registro único de Chart.js) y los componentes de
  `shared/components/informe/`.
- Vista imprimible opcional (reutilizar `.rt-print-doc` si aplica).

## 5. Riesgos

- El informe Looker puede tener fórmulas/campos calculados no triviales — el inventario del Anexo A
  debe capturarlos antes de estimar.
- Paridad numérica: comparar cada KPI del tablero nativo contra el Looker para un rango de control
  **antes** de retirar el iframe.

## 6. Criterios de aceptación

- [~] Inventario del Looker documentado (Anexo A) — no formal; el tablero cubre tema / departamento /
      tiempo / mapa. Gap a completar con el área si aparece.
- [ ] Todos los KPIs/gráficos del Looker tienen equivalente nativo; **paridad numérica verificada
      para un rango de fechas de control** — pendiente, gate del apagado de BigQuery.
- [x] `TableroPage.tsx` sin `<iframe>` y sin ninguna URL de `lookerstudio.google.com` / `bigquery`.
- [x] `grep` del frontend por `bigquery`/`lookerstudio` = 0 resultados.
- [x] Gateway: no hicieron falta paths nuevos (los 4 de informe ya estaban en la config
      `ministerio-config-v20260901`).
- [ ] Marca el checklist "Tablero nativo en producción" de `spec-migracion §10` — al desplegar +
      validar paridad.

## Anexo A — Inventario del informe Looker (a completar)

Capturar del informe `f9dc4a4e-...`: páginas, cada gráfico (tipo, dimensión, métrica, filtro),
KPIs de cabecera, y cualquier campo calculado. Fuente: el propio Looker + el área.
