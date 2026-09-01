# Spec: Resumen Territorial — Ficha de localidad + federación server-side de Privada

**Estado**: E5b (ficha) **implementado parcial** (drawer, on-demand) · E5a (federación server-side) draft
**Versión**: 0.2.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-01

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
> **Pendiente de E5b**: columnas de demografía en el **export Excel** y en la **vista de impresión**
> (`.rt-print-doc`) — requieren la demografía de *todas* las localidades del snapshot, o sea un
> endpoint bulk en svc-privada (`GET /localidades-info/all`) o el embebido server-side de E5a.
> El `GET /localidades-info` actual es lookup de a una. No hacer N+1 de 551 llamadas desde el browser.
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

**E5b — ficha (drawer)** — implementado 2026-09-01:
- [x] La ficha muestra electores, `color_semaforo` (con chip de color), intendente + partido,
      habitantes, `tipo_localidad`, y los legisladores + electores/habitantes del departamento.
- [x] Visible para todos los roles del panel, sin enmascarado (consulta con el token del usuario;
      demografía = padrón público).
- [x] Una caída de `svc-privada` (o fila vacía) no rompe el drawer — degrada a mensaje y el resto
      de la ficha (programas) sigue funcionando.
- [ ] Excel y vista de impresión incluyen las columnas de la ficha → **diferido** (necesita endpoint
      bulk o el embebido de E5a; ver nota de cabecera).

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
- [x] Frontend: flag `VITE_PRIVADA_SERVER_FEDERATION` — cuando `='true'` no se federa en el browser
      (RE-7: el código cliente `privadaGestiones.ts` queda un release detrás del flag, sin borrar).
- [x] Tolerancia: `fetch_privada_lineas` ya devuelve `[]` ante cualquier fallo → el snapshot se
      guarda sólo con Vivienda y `generado_para_areas` refleja eso (comportamiento preexistente).
- [ ] **Deploy** (runbook en `infra/DEPLOY-svc-privada.md` §E5a): `run.invoker` de la SA
      `svc-vivienda@` sobre svc-privada → redeploy svc-privada (endpoint interno) → redeploy
      svc-vivienda con `PRIVADA_FETCH_ENABLED=true` + `SVC_PRIVADA_INTERNAL_URL` → deploy frontend
      con `VITE_PRIVADA_SERVER_FEDERATION=true` → recomputar snapshot → verificar
      `generado_para_areas` incluye `privada` y no hay doble conteo.
