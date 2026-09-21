# Spec: svc-gralgob — Secretaría General de Gobierno

**Estado**: draft
**Versión**: 0.1.0
**Última actualización**: 2026-09-21

---

## Changelog

- **0.1.0** (2026-09-21): borrador inicial. Servicio nuevo, no existía en el
  plan original. Se crea a partir del análisis del Sheet real
  "ATP - Compromiso Gobernador" que el usuario proveyó — mismo criterio que
  `spec-svc-gasifera.md`: dominio inferido de datos reales, sin reunión con
  el área todavía.

## 0. Estado del proceso — leer antes de implementar nada de este spec

Este documento sigue en `draft` y **no debe pasar a `approved`** hasta que:
1. Se confirme la reunión con el área (agregar a
   `docs/context/areas/README.md`, hoy sin fila para esta secretaría).
2. Se valide
   `docs/context/areas/secretaria_Gral_Gobierno/contexto_detallado.md` (hoy
   borrador sin validar) contra lo que dice realmente el área.
3. Se resuelvan las preguntas abiertas del §10 (lista cerrada de
   ministerios/secretarías destino, alcance real del archivo — ¿solo ATP o
   también expedientes y otros programas de fondos del mismo Sheet?).

Lo único implementado hasta ahora es un **servicio de sync de solo lectura**
(`services/svc-gralgob/`, módulo `atp`), con su propio spec angosto
(`docs/files/spec-sync-atp-compromiso-gobernador.md`) — no requiere que este
spec esté aprobado porque no interpreta reglas de negocio ni expone nada a un
usuario final. **Nada de lo que sigue en este documento está implementado.**

## 1. Propósito

Servicio backend para la Secretaría General de Gobierno. Primer módulo:
**ATP (Aporte del Tesoro Provincial)** — registro de compromisos de fondos
anunciados por el Gobernador a localidades de la provincia, a qué
ministerio/secretaría se derivó su ejecución, y su cronograma de pago.
Reemplaza el Google Sheet "ATP - Compromiso Gobernador" (uno de al menos 16
que mantiene el área) que hoy se edita a mano. Objetivo declarado por el
usuario: que esta información sea consultable por localidad desde el panel
transversal Resumen Territorial (federación server-side a definir en spec/ADR
propios — no es parte de esta entrega, ver §7).

## 2. Módulos internos (propuesta, sujeta a validación con el área)

| Módulo | Descripción |
|--------|-------------|
| `atp` | ABM de compromisos ATP (hoy: espejo de solo lectura desde el Sheet, ver `spec-sync-atp-compromiso-gobernador.md`) |
| `catalogos` | Lista cerrada de ministerios/secretarías destino — a definir si se administra en el sistema o se hereda de un catálogo transversal (¿comparte con `portal_usuarios.secretarias`?) |

**No se proponen todavía** módulos para expedientes (`Estado de exp`) ni para
los otros programas de fondos vistos en el mismo Sheet (`ATP ACUMULADO`,
`COPA+FOFINDES`, `SALDO`, `BD NATALIO`) — no hay evidencia de que sean parte
del mismo requerimiento que "ATP - Compromiso Gobernador". Si el área los
confirma como necesarios, se agregan como spec hijo aparte (mismo patrón que
los child specs de `svc-privada`, ADR-010..016), no reescribiendo este
documento sin evidencia.

## 3. Modelos de datos — ya implementados (Fase 0, solo sync)

Ver `docs/files/spec-sync-atp-compromiso-gobernador.md §5` y
`services/svc-gralgob/app/atp/models.py` para el detalle completo:

- `atp_compromisos` — snapshot de compromisos ATP sincronizado desde `BD`.
- `atp_cronograma_pagos` — cronograma de pago mensual, 1 fila por
  compromiso×mes (normalizado desde las columnas mensuales del Sheet).
- `atp_sync_log` — log de corridas de sync.

Estas tablas son un **espejo de solo lectura** — no tienen todavía ningún
endpoint de escritura de negocio, `deleted_at`, ni auditoría, porque no son
editables por un usuario del sistema en esta fase.

## 4. Modelos de datos — futuros, NO implementados (requieren spec aprobado)

Cuando el ABM de negocio se aborde (post-reunión con el área, reemplazando la
carga en el Sheet — como pidió el usuario), el patrón más cercano en el repo
es **Checklist Técnico DGV** (`app/checklist_tecnico/`): tabla administrable
anclada a la entidad sincronizada, con historial de cambios separado de la
fila principal. No se detalla un modelo de datos definitivo acá — depende de
qué confirme la reunión real sobre qué necesita ser editable versus qué sigue
viniendo del Sheet durante la transición.

Preguntas de diseño pendientes de esa reunión (ver también
`contexto_detallado.md §7`):
- ¿El ABM reemplaza la carga en el Sheet de una, o convive con ella durante
  una transición (como el sync de CC convive con `viv_cordon_cuneta`)?
- ¿Quién carga/edita un compromiso — un solo rol, o roles distintos por
  etapa (anuncio, derivación, seguimiento de pago)?
- ¿Hace falta un rol acotado nuevo (tipo `TecnicoDGV`/`Autoridad`) o alcanza
  con la jerarquía estándar (`Admin`/`Supervisor`/`Operador`/`Consulta`)?
- ¿`ministerio_destino` debe validarse contra la lista real de secretarías
  de `portal_usuarios`, o es un catálogo propio de `svc-gralgob`?

## 5. Endpoints — futuros, NO implementados

No se proponen todavía rutas `/api/v1/gralgob/**` concretas — depende del
modelo de datos del §4, que depende de la reunión con el área. Los únicos
endpoints que existen hoy son internos y de sync
(`spec-sync-atp-compromiso-gobernador.md §6`).

## 6. Reglas de negocio — futuras, NO implementadas

A definir junto con la reunión real. Preguntas abiertas concretas ya
detectadas en el análisis (ver `contexto_detallado.md §5`): si `Ministerio !=
"Gobierno"` y `Derivado = true` son siempre equivalentes o pueden divergir; si
`SALDO ATP` debería recalcularse en el sistema nuevo en vez de heredar la
fórmula del Sheet; qué pasa con el saldo de un compromiso derivado (¿se sigue
trackeando en algún lado?).

## 7. Integración con Resumen Territorial — futura, NO implementada

Objetivo final declarado por el usuario: que los compromisos ATP sean
consultables por localidad desde el panel transversal `resumen_territorial`
(`svc-vivienda`). El precedente directo es Privada (ADR-016): un endpoint
interno IAM-only en `svc-gralgob` (`GET /internal/atp/rollup-territorial` o
equivalente) que `resumen_territorial.service` de `svc-vivienda` consulte
server-side, con el mismo criterio de degradación tolerante (una caída de
`svc-gralgob` produce un snapshot parcial, no rompe el panel para todos).
**No implementado** — requiere spec/ADR propio una vez que el modelo de datos
de negocio (§4) esté definido, porque el rollup necesita saber qué campo
representa "estado" del compromiso para una localidad (¿monto total
anunciado? ¿saldo pendiente? ¿ambos?).

## 8. Eventos Pub/Sub — futuros, NO implementados

A definir junto con el modelo de datos del §4. Los módulos "panel" del resto
del sistema (`cordon_cuneta`/`cordoba_hogar`/`mi_lugar`, `gas_pit`) no emiten
eventos Pub/Sub — es razonable que `svc-gralgob` siga el mismo patrón salvo
que la reunión revele una necesidad real de integración con otro servicio.

## 9. Integración con el Sheet — implementada (Fase 0)

Ver `docs/files/spec-sync-atp-compromiso-gobernador.md` completo. Resumen:
sync de solo lectura de la hoja `BD`, Cloud Run + Cloud SQL (mismo patrón que
Checklist Técnico CC y `svc-gasifera`), sin deploy real todavía en esta
entrega.

## 10. Criterios de aceptación

### Fase 0 (esta entrega) — ver `spec-sync-atp-compromiso-gobernador.md §9`
Sync de solo lectura de `BD`, scaffold del servicio, tests. Deploy real
queda para una sesión aparte vía `/deploy-servicio`.

### Fase 1+ (futura, gated on reunión + spec aprobado)
- [ ] Reunión con el área realizada, `contexto_detallado.md` validado.
- [ ] Este spec (§2-§8) revisado con datos reales y pasado a estado `approved`.
- [ ] Modelo de datos de negocio definido y migrado.
- [ ] ABM implementado (backend + frontend), siguiendo el precedente de
      Checklist Técnico DGV.
- [ ] Federación hacia Resumen Territorial implementada (spec/ADR propio).

## 11. Preguntas abiertas (bloquean pasar este spec a `approved`)

Ver `docs/context/areas/secretaria_Gral_Gobierno/contexto_detallado.md §7`
para el detalle completo. Las que más impactan el diseño de este spec en
particular:

1. **Alcance real**: ¿"ATP - Compromiso Gobernador" es solo el programa ATP
   (hoja `BD`), o el área espera que el sistema termine cubriendo también
   expedientes (`Estado de exp`) y/o los otros programas de fondos del mismo
   libro (`ATP ACUMULADO`, `COPA+FOFINDES`, `SALDO`, `BD NATALIO`)?
2. **Catálogo de ministerios/secretarías destino**: ¿lista cerrada propia de
   `svc-gralgob`, o debería validarse contra `portal_usuarios.secretarias`
   (que ya administra `svc-vivienda`)?
3. **Transición Sheet → ABM**: ¿reemplazo total desde el día uno del ABM, o
   convivencia temporal (y por cuánto tiempo)?
4. **Federación a Resumen Territorial**: ¿qué campo(s) del compromiso importan
   para el panel (monto anunciado, saldo pendiente, cronograma de pago
   próximo), y con qué visibilidad por rol (mismo criterio que Privada,
   ADR-016, o distinto)?
