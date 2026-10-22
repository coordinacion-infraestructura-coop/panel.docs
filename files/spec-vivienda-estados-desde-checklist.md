# Spec: Estados Jurídico y Presupuestario de CC / CH / ML derivados del Checklist

**Estado**: draft — **bloqueado hasta definir las reglas con el área (§4)**
**Versión**: 0.1.0
**Servicio**: `svc-vivienda` (módulos `checklist_tecnico`, `cordon_cuneta`, `cordoba_hogar`, `mi_lugar`) + frontend `vivienda`
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-21
**Depende de**: `spec-vivienda-estados-por-area.md` (listados por área)

### Changelog
- **0.1.0 (2026-10-21)** — planificación inicial. Documenta lo que ya existe, el mecanismo
  propuesto y las preguntas para la reunión con el área. No se implementa nada.

---

## 0. Origen

Pedido (2026-10-21): que lo que se carga en el Checklist impacte también en los paneles CC, CH y
ML, como ya ocurre con el Estado Técnico, pero para los estados **Jurídico** y
**Presupuestario**. Hay que definir con el área qué avance de los ítems del checklist
corresponde a cada estado del panel, sobre todo en CC, que ya separa el checklist técnico del
legal y financiero.

## 1. Lo que ya existe (verificado 2026-10-21)

- **Estado Técnico** (`spec-estado-tecnico-desde-checklist.md`, en producción desde
  2026-10-07): los paneles leen `viv_checklist_tecnico.estado_expediente_id` a través de
  `app/checklist_tecnico/estado_tecnico.py`. Es solo lectura en los paneles, sin copia, y el
  historial del panel mezcla las entradas del checklist. `ChecklistTecnicoPage.tsx` invalida
  las queries del panel al cambiar el estado. **Es una correspondencia 1 a 1**: el área elige
  el estado del expediente a mano; no se deriva de los ítems.
- **Secciones en CC** (`spec-checklist-tecnico-dgv.md` v1.7.0 §6-ter.1): los sub-ítems 4.1–4.10
  son `tecnica` y los ítems 1–9 son `legal_financiera`. **Jurídico y financiero no están
  separados entre sí**, y CH y ML no tienen secciones.
- **Hitos de obra** (`viv_checklist_obra_hitos`, los tres programas): montos y acreditaciones
  por hito. Es el dato más cercano a un estado presupuestario que ya existe.
- **Estado de cada ítem**: catálogo `viv_checklist_item_estado` con `es_completo` y
  `excluye_completitud` ("No Aplica").

## 2. Diferencia de fondo con el Técnico

El Técnico se resolvió con un estado elegido a mano en el checklist. Para Jurídico y
Presupuestario se pide algo distinto: que el estado **se deduzca del avance de los ítems**.
Hay dos formas de hacerlo, a decidir con el área:

| Opción | Cómo funciona | A favor | En contra |
|---|---|---|---|
| **R1. Reglas sobre ítems** | Cada estado del listado tiene una condición ("ítems 2, 3 y 8 completos"). El panel muestra el estado más avanzado cuya condición se cumple. | Automático; no hay dato duplicado. | Las reglas tienen que cubrir todos los casos; un estado que no depende de documentación (p. ej. "en Contaduría") no se puede deducir. |
| **R2. Estado elegido en el checklist** | Igual que el Técnico: el checklist suma "Estado jurídico" y "Estado presupuestario" elegidos a mano, y el panel los lee. | Simple; mismo mecanismo ya probado. | No es automático: alguien tiene que cambiarlo. |
| **R3. Mixto** | R1 sugiere el estado; el usuario lo confirma o lo pisa. | Cubre los estados no documentales. | Más complejo; hay que mostrar cuándo está pisado. |

Recomendación: definir con el área estado por estado; si la mayoría se deduce de ítems, R3; si
no, R2.

## 3. Mecanismo propuesto (a confirmar)

- **Lectura derivada, sin copia**, igual que el Técnico: el panel calcula el estado al leer.
  Las columnas `ejuridico` / `efinanciero` quedan congeladas cuando el programa pase a derivarse
  (no antes).
- **Reglas en base, no en código** (si R1/R3): tabla
  `viv_estado_regla (programa, dimension, estado_id, items_requeridos[], orden)`, editable por
  `Admin`, para que el área ajuste sin deploy. Un estado del listado (`spec-vivienda-estados-por-area.md`)
  puede tener regla o no.
- **Activación por programa y por dimensión**: se puede encender CC Jurídico sin tocar CH.
- **Historial**: entradas `campo="ejuridico_checklist"` / `"efinanciero_checklist"` generadas al
  cambiar el estado derivado (R2: al elegirlo; R1: al cambiar un ítem que mueve el estado).
- **Clasificación jurídico / financiero de los ítems de CC**: hoy es una sola sección
  `legal_financiera`; hay que partirla (§4, P2).

## 4. Preguntas para la reunión con el área

**P1.** Para cada programa, listar los estados Jurídicos y Presupuestarios que usan hoy (salen
del switcher nuevo) y, para cada uno: ¿qué tiene que estar completo en el checklist para estar en
ese estado? ¿O depende de algo que no está en el checklist?

**P2.** CC — propuesta de partición de la sección legal y financiera, a validar:

| Ítem | Propuesta |
|---|---|
| 1. Nota de Solicitud de Financiamiento | Jurídico |
| 2. Ordenanza y Decreto / Resolución comunal | Jurídico |
| 3. DDJJ compromiso de ejecución de obra | Jurídico |
| 4. Proyecto, Planos, Cómputo y Presupuesto (fila general) | ? (sus sub-ítems son técnicos) |
| 5. Nota a Contaduría — Cesión de Coparticipación | Presupuestario |
| 6. N° de CBU de M/C especial | Presupuestario |
| 7. N° de CUIT Municipal / Comunal | Presupuestario |
| 8. DNI Intendente / Jefe Comunal | Jurídico |
| 9. Acta de Proclamación Int. / Pres. Comunal | Jurídico |

**P3.** CH y ML: ¿qué ítems son jurídicos y cuáles presupuestarios? Hoy son una sola lista
técnica. ¿Hace falta sumar ítems legales o financieros que hoy no están?

**P4.** ¿El estado Presupuestario se mueve con los hitos de obra (pedido de fondos, acreditado)?
¿Con qué hito pasa a cada estado?

**P5.** ¿Un ítem en "No Aplica" cuenta como cumplido para la regla?

**P6.** Si un ítem vuelve atrás (se observa un documento completo), ¿el estado retrocede?

**P7.** ¿Quién debe poder cambiar estos estados? (Se cruza con `spec-portal-areas-de-acceso.md`.)

## 5. Fuera de alcance

- Estado General: sigue 100 % manual.
- Implementación: nada se implementa hasta que este spec pase a `approved` con las respuestas
  de §4.
