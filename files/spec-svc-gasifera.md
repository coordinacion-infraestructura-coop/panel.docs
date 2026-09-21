# Spec: svc-gasifera — Secretaría de Infraestructura Gasífera

**Estado**: draft
**Versión**: 0.2.0
**Última actualización**: 2026-09-16

---

## Changelog

- **0.2.0** (2026-09-16): reescritura completa. La v0.1.0 asumía un dominio
  ("conexión de gas en escuelas" / "asesoramientos legales-contables" /
  "créditos con amortización") que **no coincide con ningún dato real** — era
  un diseño de escritorio, nunca contrastado con el área ni con ninguna fuente.
  Al analizar en profundidad el Sheet real que el área mantiene hoy
  (`SEC. GAS PIT.xlsx`), se encontró que el dominio real es **seguimiento de
  obras de infraestructura de gas** (expedientes, contratista, montos, plazos,
  avance) + **acciones de seguimiento territorial** por localidad — sin
  ninguna mención a escuelas, asesoramientos o créditos con amortización en
  todo el archivo. Se descarta el modelo de datos de v0.1.0 en lugar de
  parchearlo. Ver el análisis completo referenciado desde
  `docs/context/areas/secretaria_Gasifera/contexto_detallado.md`.
- **0.1.0** (2025-01): versión original (dominio descartado, ver arriba).

## 0. Estado del proceso — leer antes de implementar nada de este spec

Este documento sigue en `draft` y **no debe pasar a `approved`** hasta que:
1. Se confirme la reunión con el área (`docs/context/areas/README.md` la marca
   "⏳ pendiente" para esta secretaría).
2. Se valide `docs/context/areas/secretaria_Gasifera/contexto_detallado.md`
   (hoy también borrador sin validar) contra lo que dice realmente el área.
3. Se resuelvan las preguntas abiertas del §10 (ownership real de `SPIP`,
   catálogos de localidad/departamento duplicados, alcance real — ¿solo gas o
   las 5 categorías del tablero general?).

Lo único implementado hasta ahora es un **servicio de sync de solo lectura**
(`services/svc-gasifera/`, módulo `gas_pit`), con su propio spec angosto
(`docs/files/spec-sync-gasifera-pit.md`) — no requiere que este spec esté
aprobado porque no interpreta reglas de negocio ni expone nada a un usuario
final. **Nada de lo que sigue en este documento está implementado.**

## 1. Propósito

Servicio backend para la Secretaría de Infraestructura Gasífera: gestión de
obras de infraestructura de gas (gasoductos, redes de gas, conexiones) por
localidad/departamento, con expedientes, contratista, montos, plazos y avance,
más el seguimiento de acciones territoriales asociadas (hitos de ejecución,
comunicación de estado a nivel localidad). Reemplaza el Google Sheet
"SEC. GAS PIT" (uno de ~15 que mantiene el área) que hoy se edita a mano.

## 2. Módulos internos (propuesta, sujeta a validación con el área)

| Módulo | Descripción |
|--------|-------------|
| `obras` | ABM de obras de gas (hoy: espejo de solo lectura desde el Sheet, módulo `gas_pit`) |
| `acciones_territorio` | Hitos de seguimiento por localidad (hoy: espejo de solo lectura, módulo `gas_pit`) |
| `catalogos` | Departamento/localidad, estado de obra, prioridad — a definir si se administran en el sistema o se heredan de un catálogo canónico externo (ver §10) |

**No se proponen** módulos de "escuelas", "asesoramientos" ni "créditos" — no
hay ninguna evidencia de esos dominios en la fuente de datos real. Si el área
los menciona en la reunión, se agregan como spec hijo aparte (mismo patrón que
los child specs de svc-privada, ADR-010..016), no reescribiendo este documento
de nuevo sin evidencia.

## 3. Modelos de datos — ya implementados (Fase 0, solo sync)

Ver `docs/files/spec-sync-gasifera-pit.md §5` y
`services/svc-gasifera/app/gas_pit/models.py` para el detalle completo:

- `gas_pit_obras` — snapshot de obras de gas sincronizado desde `MATRIZ (NO TOMAR)`.
- `gas_pit_obras_localidades` — relación N:M obra↔localidad.
- `gas_pit_acciones_territorio` — hitos sincronizados desde `ACCIONES TERRITORIO`.
- `gas_pit_sync_log` — log de corridas de sync.

Estas tablas son un **espejo de solo lectura** — no tienen todavía ningún
endpoint de escritura de negocio, `deleted_at`, ni auditoría, porque no son
editables por un usuario del sistema en esta fase.

## 4. Modelos de datos — futuros, NO implementados (requieren spec aprobado)

Cuando el panel de negocio se aborde (post-reunión con el área), el patrón más
cercano en el repo es **Checklist Técnico DGV** (`app/checklist_tecnico/`):
tablas administrables ancladas a la entidad sincronizada, con historial de
cambios de estado separado de la fila principal, y catálogos editables para lo
que el área vaya a necesitar reconfigurar sin deploy (ver
`docs/files/spec-checklist-tecnico-dgv.md` como precedente de diseño). No se
detalla un modelo de datos definitivo acá — depende de qué confirme la reunión
real sobre qué necesita ser editable versus qué sigue viniendo del Sheet.

Preguntas de diseño pendientes de esa reunión (ver también
`contexto_detallado.md §7`):
- ¿El panel reemplaza la carga en el Sheet, o convive con ella (como el sync de
  CC convive con `viv_cordon_cuneta`)?
- ¿Quién edita qué campo — un solo rol o varios (ej. técnico vs. financiero vs.
  comunicación, dado que el Sheet mezcla esos tres procesos, ver
  `contexto_detallado.md §2`)?
- ¿Hace falta un rol acotado nuevo (tipo `TecnicoDGV`/`Autoridad`) o alcanza con
  la jerarquía estándar (`Admin`/`Supervisor`/`Operador`/`Consulta`)?

## 5. Endpoints — futuros, NO implementados

No se proponen todavía rutas `/api/v1/gasifera/**` concretas — depende del
modelo de datos del §4, que depende de la reunión con el área. Los únicos
endpoints que existen hoy son internos y de sync (`spec-sync-gasifera-pit.md §6`).

## 6. Reglas de negocio — futuras, NO implementadas

Ninguna regla de negocio de la v0.1.0 (amortización de créditos, mora, agenda
de asesoramientos) tiene base en la fuente de datos real y se descarta en
bloque. Las reglas reales (ej. transiciones válidas de `ESTADO DE OBRA`, si
`AVANCE` se calcula o se carga a mano) son una de las preguntas abiertas del
§10 — no se puede especificar sin confirmar con el área si esos valores
siguen algún flujo formal o son informativos libres (ver
`contexto_detallado.md §5`).

## 7. Eventos Pub/Sub — futuros, NO implementados

A definir junto con el modelo de datos del §4. `cordon_cuneta`/`cordoba_hogar`/
`mi_lugar` (patrón "panel module") no emiten eventos Pub/Sub — es razonable
que `svc-gasifera` siga el mismo patrón salvo que la reunión revele una
necesidad real de integración con otro servicio.

## 8. Integración con el Sheet — implementada (Fase 0)

Ver `docs/files/spec-sync-gasifera-pit.md` completo. Resumen: sync de solo
lectura, Cloud Run + Cloud SQL (mismo patrón que Checklist Técnico CC), sin
deploy real todavía en esta entrega.

## 9. Criterios de aceptación

### Fase 0 (esta entrega) — ver `spec-sync-gasifera-pit.md §9`
Sync de solo lectura, scaffold del servicio, tests, y **deploy real a Cloud Run**
(2026-09-21) — servicio corriendo en `https://svc-gasifera-iwni7vc2qq-rj.a.run.app`,
migración `0001` aplicada contra `db_gasifera` en Cloud SQL. Pendiente inmediato:
otorgar `roles/run.invoker` a un caller autorizado (Cloud Scheduler) para que el
sync corra de verdad en producción — ver `spec-sync-gasifera-pit.md §11`.

### Fase 1+ (futura, gated on reunión + spec aprobado)
- [ ] Reunión con el área realizada, `contexto_detallado.md` validado.
- [ ] Este spec (§2-§7) revisado con datos reales y pasado a estado `approved`.
- [ ] Modelo de datos de negocio definido y migrado.
- [ ] Panel de negocio implementado (backend + frontend), siguiendo el
      precedente de Checklist Técnico DGV.

## 10. Preguntas abiertas (bloquean pasar este spec a `approved`)

Ver `docs/context/areas/secretaria_Gasifera/contexto_detallado.md §7` para el
detalle completo. Las que más impactan el diseño de este spec en particular:

1. **Alcance**: ¿el sistema es solo para obras de **gas**, o el área espera
   cubrir las 5 categorías del tablero general (vial, agua/cloaca, eléctrica,
   arquitectura, gas) — lo cual correspondería más a una futura
   `svc-infraestructura` que a `svc-gasifera`?
2. **Ownership del catálogo geográfico**: el Sheet tiene 2 catálogos de
   departamento y 2 de localidad no sincronizados entre sí. `svc-privada` ya
   posee un catálogo canónico (`priv_localidades_info`/`priv_departamentos_info`,
   ADR-012) consumido read-only por otros servicios — ¿conviene que
   `svc-gasifera` lo reutilice en vez de mantener un tercer catálogo propio?
3. **Identificador real de obra**: `SPIP` no es único ni siempre está cargado.
   ¿Existe un identificador de negocio confiable, o el sistema nuevo introduce
   uno propio?
4. **Las otras ~14 planillas**: ¿tienen estructura similar a esta (mismo
   patrón de sync reutilizable) o son heterogéneas (requieren análisis
   independiente cada una)?
