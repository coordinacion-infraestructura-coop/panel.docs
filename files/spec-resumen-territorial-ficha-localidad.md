# Spec: Resumen Territorial — Ficha de localidad + federación server-side de Privada

**Estado**: draft
**Versión**: 0.1.0 (candidato a enmienda v0.3.0 de `spec-resumen-territorial.md`)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-08-31
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

- [ ] `privada_fetch_enabled=True`; las líneas de Privada aparecen en el snapshot server-side; se
      eliminó `privadaGestiones.ts` (o quedó tras flag).
- [ ] `GET /api/v1/privada/rollup-territorial` y `.../departamentos-info` consumidos por
      `svc-vivienda` con la SA (sin error de token).
- [ ] La ficha muestra electores, `color_semaforo` (con chip de color), intendente + partido,
      habitantes, `tipo_localidad`, y los legisladores del departamento.
- [ ] Visible para todos los roles del panel, sin enmascarado.
- [ ] Excel y vista de impresión incluyen las columnas de la ficha.
- [ ] Una caída simulada de `svc-privada` no rompe el `GET /api/v1/resumen-territorial` (degrada a
      snapshot sin Privada, con `generado_para_areas` correcto).
