# Spec: Resumen Territorial — Ficha de localidad + federación server-side de Privada

**Estado**: E5b (ficha, drawer) implementado · E5a (federación server-side) DESPLEGADO 2026-09-02 · exportables del RT rediseñados 2026-09-03 · E5c (Gasífera + ATP en la ficha de municipio) y E5d (filtro "Visita del gobernador") IMPLEMENTADOS 2026-09-23
**Versión**: 0.4.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-23

> **Implementado 2026-09-01 (E5b, drawer)** — `panel.front`:
> `src/modules/resumen-territorial/api/fichaLocalidad.api.ts` + componente `FichaDemografica`
> dentro del `DetailDrawer` de `ResumenTerritorialPage.tsx`. La ficha se consulta **on-demand con
> el token del usuario** a `GET /api/v1/privada/localidades-info` y `.../departamentos-info` (no se
> embebe en el snapshot; no depende de E5a). Muestra: habitantes, electores, semáforo (chip de
> color), intendente + partido, tipo de localidad, y los legisladores + electores/habitantes del
> departamento. Cacheada en React Query (5 min). Degrada a "Sin datos de padrón cargados" cuando
> la fila `priv_localidades_info` está vacía, y a un mensaje de error si la llamada falla — sin
> romper el resto de la ficha.
>
> **E5b completado 2026-09-02/03** — además del drawer:
> - `GET /api/v1/privada/localidades-info/all` (bulk) + columnas de demografía en el **export
>   Excel** y la **vista de impresión** del RT + botón **"⤓ PDF"** del RT.
> - **Filtro de localidad con autocompletar** en la barra del RT (combobox `<input list>`, scope
>   por departamento).
> - **"Ficha de municipio" imprimible** (`fichaMunicipio.ts`): PDF + Excel por municipio que junta
>   demografía + Córdoba Hogar + Cordón Cuneta + Mi Lugar + Gestiones (con último evento por
>   gestión). PDF A4 con membrete (escudo re-teñido a navy), franja de KPIs, chips de estado,
>   gestiones bloque (≤6) / tabla (>6). `avance` es estimación por posición del estado en el
>   catálogo — validar con el área si hay fórmula oficial.
>
> **Rediseño de exportables del RT — 2026-09-03** (`panel.front`):
> - **Búsqueda libre separada de los filtros.** El `<input type="search">` (localidad/departamento)
>   pasó a una barra propia con ícono arriba de la toolbar; abajo, la fila de filtros ahora rotulada
>   "Filtros". Orden nuevo: KPIs → búsqueda → toolbar (unidad + alcance + exportar) → filtros → tabla.
> - **Se eliminaron los botones "⤓ Excel", "⤓ PDF" y "⎙ Imprimir"** y el documento oculto
>   `.rt-print-doc` + su bloque `@media print` en `src/index.css` (era el primer `@media print` del
>   proyecto; ya no hay ninguno). Los tres compartían `fichaCols()`, que cruzaba el snapshot con
>   `priv_localidades_info` por `norm(departamento)|norm(localidad)`: como los dos lados traen el
>   departamento de fuentes distintas (padrón geo vs. Excel de Privada) y `norm` no reconcilia
>   abreviaturas, el `Map` colisionaba y muchas filas salían con la demografía de otra localidad
>   (el usuario reportó "siempre carga Villa Sarmiento"). **La demografía por localidad se sirve
>   ahora sólo desde la "Ficha de municipio", que la trae exacta para un municipio.**
> - **Único exportable nuevo: "⤓ Exportar Excel"** (`exportResumen.ts`, helper
>   `exportSheetsToXlsx` en `shared/utils/exportTable.ts`). `.xlsx` multi-hoja con los municipios
>   que cumplen los filtros aplicados: **Resumen** (alcance, filtros, totales, fecha) · **Programas**
>   (fila por localidad×programa: estado, sub-estados, checklist, monto, expediente, últ. com.) ·
>   **Checklists** (fila por ítem faltante de cada checklist de vivienda) · **Gestiones** (fila por
>   gestión de Sec. Privada de esas localidades, con Ok Gob/Min y nº de movimientos) ·
>   **Movimientos** (fila por evento de esas gestiones — el "trackeo por gestión" pedido).
>   Federación client-side con el token del usuario: 1 barrido paginado de `GET /gestiones/` +
>   `GET /gestiones/{id}/eventos` con concurrencia 8, tope de 300 gestiones para el detalle de
>   movimientos (por encima, aviso en pantalla y en la hoja Resumen). El match gestión↔localidad
>   es por `norm(depto)|norm(localidad)` — puede omitir gestiones cuyo texto libre no normaliza
>   igual que el padrón geo; falla "seguro" (fila de menos, nunca cruzada).
> - **`fichaLocalidadApi.todas()` y `GET /api/v1/privada/localidades-info/all`** quedan sin uso en
>   el front (los consumía el export viejo). El endpoint backend se conserva por si vuelve a hacer
>   falta un bulk.
> - **Sin tocar**: botón "Ficha (Excel)" / `fichaMunicipioXlsx` (rework en un paso siguiente).

> **Implementado 2026-09-23 (E5c — Gasífera + ATP en la Ficha de municipio, y E5d — filtro
> "Visita del gobernador")** — `panel.front`, pedido directo del usuario (Pedro Bonafe, también
> responsable de spec). Extiende la "Ficha de municipio" (`fichaMunicipio.ts`, PDF + Excel) más
> allá de lo que cubría §2 originalmente (demografía + CH/CC/ML + Gestiones Privada) para sumar
> las dos fuentes que se agregaron al sistema después de escrita esta spec (`svc-gasifera`,
> `svc-gralgob` — ver `../CLAUDE.md` § Servicios). Ambas son **client-side, contra los carve-outs
> de solo lectura ya aprobados** de cada servicio (`spec-sync-gasifera-pit.md §12`,
> `spec-sync-atp-compromiso-gobernador.md §12`) — sin endpoints nuevos, sin lógica de negocio
> nueva en el backend.
> - **Gasífera** (`GET /api/v1/gasifera/acciones-territorio`, dataset completo — el endpoint no
>   filtra por departamento/localidad — filtrado client-side): Área, Acción, Etapa
>   (`detalle_accion`), Estado, Monto solicitado. Una entrada por acción en el municipio.
> - **ATP** (`GET /api/v1/gralgob/compromisos` + `.../compromisos/{id}/cronograma`, mismo
>   criterio de filtrado client-side): Ministerio Destino, Fecha de anuncio, Destino, Monto, y
>   las entregas (cronograma de pagos: período + monto, más el total entregado ya calculado con
>   el mismo criterio que `AtpPage.tsx` — `abs(total_pagado)`).
> - **Matching (departamento, localidad)**: como ninguno de los dos servicios expone filtro por
>   ubicación en su endpoint de lectura, se trae la lista completa y se filtra en el cliente por
>   localidad normalizada **y** departamento normalizado cuando la fuente lo trae — no sólo por
>   localidad, a propósito: evita repetir el bug documentado más arriba (join viejo del RT por
>   sólo-nombre que "siempre cargaba Villa Sarmiento" al cruzar `priv_localidades_info`).
> - **E5d — filtro "Visita del gobernador" (SI/NO) en `ResumenTerritorialPage.tsx`**: estar en
>   ATP (`programas` con `area === 'gralgob'`) para una localidad implica que el gobernador la
>   visitó y anunció algo ahí — mismo hecho que ya usa el badge ATP del panel (`resumen_atp_estado`
>   en `aggregations.py`, ADR-022). El filtro es de **localidad**, no de programa: filtra qué
>   localidades aparecen en la tabla sin ocultar sus otros programas (CH/CC/ML/Privada/Gasífera)
>   cuando "SI" está activo — se evalúa contra los `programas` originales de la localidad, antes
>   del filtro por área/programa/estado/checklist, para no dar falsos "NO" cuando el usuario ya
>   tiene otro filtro de área puesto. Sin cambios de backend: los datos ya estaban en el snapshot
>   desde la federación ATP (ADR-022, 2026-09-23, ver `../CLAUDE.md`).
> - **Fix de paso**: `AREA_LABEL`/`AREA_DOT_COLOR` (`ResumenTerritorialPage.tsx`) y el tipo
>   `AreaResumen` (`types/resumenTerritorial.types.ts`) no tenían la entrada `gralgob` — omisión
>   de la sesión que desplegó ADR-022 (la propia nota de `../CLAUDE.md` decía "no confirmado
>   visualmente en la UI del RT"). El badge de área ATP mostraba el string crudo `"gralgob"` sin
>   color propio; corregido de paso (`'Sec. Gral. de Gobierno'`, `#172c3f`) porque el filtro nuevo
>   lo hacía visible de inmediato.
**Servicio**: `svc-vivienda` (módulo `app/resumen_territorial/`) + frontend
`src/modules/resumen-territorial/`
**Depende de**: `spec-migracion-svc-privada.md` Fase 2 (endpoints `rollup-territorial` y
`departamentos-info` de `svc-privada`).
**ADRs**: ADR-016 (federación server-side de Privada), ADR-012 (datos territoriales propiedad de
svc-privada, read-only vía gateway).

---

## 0. Origen

Pedido del usuario (2026-08-31), mejora 2: el Resumen Territorial del sistema nuevo debe cargar una
**pestaña con la ficha de la localidad** — cantidad de electores, color de semáforo, nombre y
partido del intendente, habitantes, etc. — todo lo que muestra el resumen territorial del sistema
viejo (`GET /gestiones/resumen-territorial` → `territorio_info`).

Se aprovecha para resolver ADR-016: con `svc-privada` migrado al mismo proyecto GCP, la federación
de las líneas de Privada pasa del browser al servidor.

## 1. Estado actual

- `resumen_territorial` (spec `spec-resumen-territorial.md` approved v0.2.0) NO tiene demografía de
  localidad en `ResumenLocalidad`. No existe `localidades_info`/`color_semaforo`/electores en el
  sistema nuevo (único `intendente` = `viv_cc_checklist_tecnico.intendente`, CC-only, sin partido).
- Las líneas de Privada se federan **en el browser** (`api/privadaGestiones.ts` pagina
  `GET /api/v1/privada/gestiones/`) — "plan B" de `spec-resumen-territorial.md §3.3`.
- El frontend ya tiene un drawer titulado **"Ficha de localidad"** (`DetailDrawer`, `detalleLoc`),
  y un toggle segmentado `unidad` = `localidad | departamento` (no hay tab bar).

## 2. Alcance

### E5a — Federación server-side de Privada (ADR-016)
- `settings.privada_fetch_enabled = True`; `service.fetch_privada_lineas()` llama
  `GET /api/v1/privada/gestiones/rollup-territorial` con el ID token de la SA de `svc-vivienda`
  (audience = gateway; `svc-privada` en el mismo proyecto ⇒ el token se acepta).
- `_map_privada_payload` se ajusta a la forma real del `rollup-territorial`.
- Se elimina `frontend/src/modules/resumen-territorial/api/privadaGestiones.ts` y el merge en el
  `useMemo` de `ResumenTerritorialPage.tsx` (se deja detrás de un flag un release; RE-7).
- La regla de visibilidad (`filtrar_por_visibilidad`) sigue aplicándose server-side sobre el
  snapshot completo — igual que hoy.

### E5b — Ficha de localidad
- Bloque nuevo `ficha` en `ResumenLocalidad` (o consulta lazy separada al abrir la ficha):
  - **Localidad** (de `svc-privada` `GET /api/v1/privada/localidades-info`): `habitantes`,
    `electores`, `intendente_jefe_comunal`, `partido_politico`, `tipo_localidad`, `color_semaforo`,
    `updated_at`, `updated_by`.
  - **Departamento** (de `svc-privada` `GET /api/v1/privada/departamentos-info`):
    `legislador_departamental`, `partido_politico`, `legislador_sabana1/2` + partidos, `habitantes`,
    `electores`.
- La demografía es **padrón público** → visible a todos los roles del panel, sin enmascarado.
- Frontend: tercer segmento `ficha` en el toggle `unidad` (o tab strip real), y/o expandir el
  `DetailDrawer` existente a vista full-width. Chip de color para `color_semaforo`
  (verde/amarillo/rojo). Extender el Excel export y la vista `@media print` (`.rt-print-doc`) con
  las columnas de la ficha.
- `resumen_territorial` obtiene la ficha por **llamada HTTP read-only a `svc-privada`** (ADR-012),
  no por join cross-DB. Se cachea con el snapshot (append-only) o se consulta on-demand — decidir
  según coste (551 localidades).

### E5c — Gasífera y ATP en la Ficha de municipio
- Fuera del alcance original de esta spec (Gasífera/`svc-gralgob` no existían al escribirla) —
  agregado 2026-09-23 a pedido directo del usuario. Ver nota de cabecera para el detalle completo.
- `fichaMunicipio.ts` suma dos bloques nuevos (PDF y Excel), client-side, sobre los carve-outs de
  solo lectura ya aprobados de cada servicio — sin endpoints ni lógica de negocio nueva:
  - **Gasífera**: Área, Acción, Etapa, Estado, Monto solicitado (`GET /api/v1/gasifera/acciones-territorio`).
  - **ATP**: Ministerio Destino, Fecha de anuncio, Destino, Monto, entregas/cronograma
    (`GET /api/v1/gralgob/compromisos` + `.../cronograma`).
- Filtrado por `(departamento, localidad)` client-side (ninguno de los dos endpoints lo ofrece
  server-side) — localidad normalizada **y** departamento normalizado cuando la fuente lo trae.

### E5d — Filtro "Visita del gobernador" (ATP) en el Resumen Territorial
- Agregado 2026-09-23, mismo pedido. Nuevo filtro SI/NO en `ResumenTerritorialPage.tsx`: una
  localidad "visitada por el gobernador" es una que tiene al menos una línea ATP (`area === 'gralgob'`)
  en el snapshot — mismo hecho de negocio que ya usa el badge ATP (`resumen_atp_estado`).
- Sin cambios de backend — los datos ya estaban federados desde ADR-022 (2026-09-23). Filtro de
  **localidad**, evaluado contra los `programas` originales antes de aplicarse los filtros de
  área/programa/estado/checklist (para no dar falsos "NO" con otro filtro de área activo).

### Fuera de alcance
- Edición de `localidades_info`/`departamentos_info` desde el panel transversal (el `PUT` de
  `svc-privada` sigue siendo la única vía de edición; `tipo_localidad`/`color_semaforo` read-only).
- Mapa coroplético por semáforo (posible v2, reutilizando `CoropletiqueDepartamentos`).

## 3. Riesgos

- **RE-7**: la federación server-side cambia el modo de falla — una caída de Privada produce un
  snapshot parcial **para todos** hasta el próximo recálculo (antes era aviso por-usuario).
  Mitigar: conservar `generado_para_areas`, flag para volver al path browser un release, monitorear
  el éxito del fetch en el job de cómputo.
- **RE-8**: `tipo_localidad`/`color_semaforo` vienen de un Excel one-off (`match_semaforo/`); nadie
  los refresca post-migración → la ficha muestra semáforos envejecidos. Documentar el procedimiento
  de recarga; decidir si se agrega edición o un job de re-import.

## 4. Decisiones abiertas

- ¿La ficha va como tercer segmento del toggle `unidad`, como tab strip nuevo, o sólo como drawer
  full-width? (propuesta: tab strip — es la primera vez que el panel necesita >2 vistas).
- ¿La demografía se embebe en el snapshot o se consulta on-demand al abrir la ficha? (propuesta:
  on-demand para no inflar el snapshot; cachear en React Query).
- `departamentos_info`: read-only (default) o se agrega edición.

## 5. Criterios de aceptación

**E5c — Gasífera + ATP en la Ficha de municipio** — implementado 2026-09-23:
- [x] El PDF y el Excel de "Ficha de municipio" incluyen un bloque Gasífera (Área, Acción, Etapa,
      Estado, Monto solicitado) y un bloque ATP (Ministerio Destino, Fecha de anuncio, Destino,
      Monto, entregas/cronograma) por cada acción/compromiso del municipio.
- [x] El match `(departamento, localidad)` es client-side, exige localidad normalizada y
      departamento normalizado cuando la fuente lo trae (evita el bug de cruce documentado en E5b).
- [x] `npm run build` sin errores de tipos.
- [ ] Verificación visual en el navegador con un municipio real con acciones de gas y/o
      compromisos ATP conocidos — pendiente.

**E5d — filtro "Visita del gobernador"** — implementado 2026-09-23:
- [x] Nuevo `<select>` en la barra de filtros del RT (SI/NO/cualquiera), integrado a
      `hayFiltros`/`limpiar`/`filtrosTexto` como el resto de los filtros existentes.
- [x] El filtro es de localidad (no oculta programas de otras áreas de una localidad que sí
      visitó el gobernador) y no se ve afectado por otros filtros de área/programa activos.
- [x] Fix de paso: `AREA_LABEL`/`AREA_DOT_COLOR`/`AreaResumen` no tenían `gralgob` — el badge de
      área ATP mostraba el string crudo sin color propio.
- [ ] Verificación visual en el navegador — pendiente (mismo pendiente que ADR-022 traía arrastrado).

**E5b — ficha (drawer)** — implementado 2026-09-01:
- [x] La ficha muestra electores, `color_semaforo` (con chip de color), intendente + partido,
      habitantes, `tipo_localidad`, y los legisladores + electores/habitantes del departamento.
- [x] Visible para todos los roles del panel, sin enmascarado (consulta con el token del usuario;
      demografía = padrón público).
- [x] Una caída de `svc-privada` (o fila vacía) no rompe el drawer — degrada a mensaje y el resto
      de la ficha (programas) sigue funcionando.
- [~] Excel y vista de impresión incluyen las columnas de la ficha → **descartado 2026-09-03**: el
      join snapshot↔`priv_localidades_info` por nombre normalizado cruzaba datos entre localidades
      ("siempre carga Villa Sarmiento"). La demografía por localidad la da ahora sólo la "Ficha de
      municipio" (exacta, un municipio); los exportables masivos del RT se unificaron en un único
      "⤓ Exportar Excel" multi-hoja sin demografía (ver nota de cabecera, 2026-09-03).

**E5a — federación server-side (ADR-016)** — **código listo, pendiente de deploy** (`panel.backend`
`2f48a55`, `panel.front` `6791757`):
- [x] Camino de auth resuelto — **NO** por el gateway (svc-privada rechaza el ID token de la SA),
      sino por un endpoint **IAM-only** nuevo en svc-privada: `GET /internal/privada/rollup-territorial`
      (sin `/api/v1`, sin `get_current_user`), que svc-vivienda consume con un ID token cuyo audience
      es la URL de Cloud Run de svc-privada. Simétrico al `/internal/portal/usuarios/{email}` que
      svc-privada ya usa contra svc-vivienda.
- [x] `svc-vivienda`: `config.svc_privada_internal_url` + `fetch_privada_lineas` reescrito +
      `_map_privada_payload` entiende la forma del `rollup-territorial`.
- [x] `cloudbuild.yaml`: `_PRIVADA_FETCH_ENABLED` / `_SVC_PRIVADA_INTERNAL_URL` (default off).
- [x] Frontend: flag `VITE_PRIVADA_CLIENT_FEDERATION (opt-in de rollback; OFF por default)` — cuando `='true'` no se federa en el browser
      (RE-7: el código cliente `privadaGestiones.ts` queda un release detrás del flag, sin borrar).
- [x] Tolerancia: `fetch_privada_lineas` ya devuelve `[]` ante cualquier fallo → el snapshot se
      guarda sólo con Vivienda y `generado_para_areas` refleja eso (comportamiento preexistente).
- [ ] **Deploy** (runbook en `infra/DEPLOY-svc-privada.md` §E5a): `run.invoker` de la SA
      `svc-vivienda@` sobre svc-privada → redeploy svc-privada (endpoint interno) → redeploy
      svc-vivienda con `PRIVADA_FETCH_ENABLED=true` + `SVC_PRIVADA_INTERNAL_URL` → deploy frontend
      con `VITE_PRIVADA_CLIENT_FEDERATION (opt-in de rollback; OFF por default)=true` → recomputar snapshot → verificar
      `generado_para_areas` incluye `privada` y no hay doble conteo.
