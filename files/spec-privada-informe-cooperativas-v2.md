# Spec: svc-privada — Informe de Cooperativas v2 (clasificación sobre el modelo estructurado)

**Estado**: draft (análisis de compatibilidad `categoria_id`→`tema_informe` 2026-09-14 — §4, pendiente de sign-off del área)
**Versión**: 0.2.0 (candidato a enmienda de `spec-privada-categorias-programas.md`)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-14
**Servicio**: `svc-privada` (módulo `app/informe/`)
**Depende de**: `spec-privada-categorias-programas.md` `approved` + su migración `0002` aplicada
(las gestiones ya tienen `categoria_id` backfilleado).

---

## 0. Origen

La migración-paridad (`spec-migracion-svc-privada.md §4.6`) porta la clasificación de
`v_informe_cooperativas` **tal cual**: un `CASE WHEN` de 10 prioridades sobre `categoria_general_id`
+ `REGEXP_CONTAINS(LOWER(detalle), …)` que asigna `tema_informe` ∈ {`Cordón Cuneta + Adoquinado`,
`Kits Solares`, `Luces LED`, `Gas`, `Bombeo Solar`, `Vivienda`, `Lotes`,
`Infraestructura Eléctrica`, `Préstamos y Fortalecimiento`, `Otras Obras`, NULL}.

Con `priv_categorias` estructurado (`spec-privada-categorias-programas.md`), la clasificación por
regex sobre texto libre queda obsoleta y frágil. Este spec la re-apunta al campo estructurado.

## 1. Propósito

Que los 4 endpoints `/api/v1/privada/informe/cooperativas/**` (`resumen`, `temporal`,
`por-departamento`, `puntos`) y el Tablero nativo (`spec-privada-tablero.md`) deriven `tema_informe`
de `categoria_id` / `programa_id` / `area_id`, no de regex sobre `detalle`.

## 2. Alcance

### Incluido
- Mapa de compatibilidad **`categoria_id` (nueva) → `tema_informe` (10 valores actuales)**, para no
  romper los 4 endpoints ni las comparaciones históricas. Default: los 10 temas se mantienen; cada
  categoría nueva mapea a uno.
- Re-implementar la función de clasificación (`informe_service.py`) para usar el mapa en vez del
  regex. Quitar la dependencia de `LOWER(detalle)`.
- **Doble corrida + diff**: correr la clasificación vieja (regex) y la nueva (estructurada) sobre
  todas las gestiones migradas; producir un reporte de gestiones que cambian de tema; **sign-off del
  área** antes de retirar el regex.
- Tras el sign-off: congelar `categoria_general_id` (dejar de escribirla) y, opcionalmente, dropearla
  en una migración posterior.

### Fuera de alcance
- Cambiar la lista de 10 temas por las 9 categorías nuevas como agrupación del informe — es una
  **decisión abierta** (§4); el default es mantener los 10 temas.
- Nuevos reportes.

## 3. Riesgos

- **RE-1**: mover de regex a `categoria_id` **reclasifica** gestiones (el regex captura matices que
  las categorías no, y viceversa) → los totales del informe se mueven y las comparaciones
  históricas se rompen. Mitigación central de este spec: mapa de compatibilidad + doble corrida +
  diff + sign-off del área **antes** de retirar el regex.

## 4. Decisiones abiertas (para el área)

### Borrador del mapa `categoria_id` → `tema_informe` (2026-09-14) — y por qué no cierra limpio

Con `B_cat_categoria_general.json` + `_TEMA_A_CAT`/`_LEGACY_A_CAT` de `scripts/backfill_categorias.py`
(el mapeo que **ya corre en prod** para poblar `categoria_id`), se puede reconstruir qué `tema_informe`
histórico alimenta a cada una de las 9 categorías nuevas:

| Categoría nueva (`categoria_id`) | `tema_informe` que la alimentan | Problema |
|---|---|---|
| Vivienda | Vivienda (único) | ninguno — 1:1 limpio |
| Loteos | Lotes (único) | ninguno — 1:1 limpio |
| Cordón Cuneta y adoquinado | Cordón Cuneta + Adoquinado (único) | ninguno — 1:1 limpio |
| **Obras de Recursos Hídricos** | Bombeo Solar (regla estrecha) **+** el grueso de `CAT_AGUA_Y_SANEAMIENTO` que nunca matchea esa regla (fallback legacy, sin tema propio hoy) | la mayoría de sus gestiones **no tienen tema hoy** (agua/saneamiento genérico nunca fue un tema de los 10) — asignarle "Bombeo Solar" como tema sería engañoso |
| **Otras Obras** | Gas + Kits Solares + Luces LED + Infraestructura Eléctrica + Otras Obras (5 temas distintos) **+** `CAT_OBRAS_PUBLICAS`/`CAT_CULTURA_EVENTOS`/`CAT_DEPORTES` (fallback) | **colapsa 5 temas en 1** — si el informe deriva el tema de `categoria_id`, Gas/Solar/LED/Eléctrica dejan de distinguirse entre sí |
| **Ayudas a instituciones** | Préstamos y Fortalecimiento (único tema) **+** `CAT_EDUCACION`/`CAT_SALUD`/`CAT_DESARROLLO_SOCIAL`/`CAT_AYUDA_A_INSTITUCIONES`/`CAT_COOPERATIVAS_Y_MUTUALES` (fallback, sin tema propio hoy) | igual que Recursos Hídricos: la mayoría de sus gestiones no tienen tema hoy; "Préstamos y Fortalecimiento" sería un nombre engañoso para educación/salud/desarrollo social |
| Pedidos Administrativos | ninguno (`CAT_GESTION_MUNICIPAL_INSTITUCIONAL` no dispara ningún tema) | hoy **no entra al informe** salvo que `ministerio_agencia_id = MIN_COOPERATIVAS_MUTUALES` (regla 9-10) |
| Pedidos por ATP | — (categoría sin precedente: no aparece en `_LEGACY_A_CAT` ni en ningún `categoria_general_id` legacy) | sin dato histórico para inferir nada — la resuelve el área desde cero |
| Pedidos NorOeste y Sur Sur | — (ídem) | ídem |

**Hallazgo central**: la taxonomía nueva (9 categorías, pensada para *quién gestiona qué campo de
trabajo*) y la vieja (10 temas, pensada para *qué agrupación mostrar en el informe de Cooperativas*)
**no son jerarquías compatibles** — no es que una refine a la otra, agrupan según criterios distintos.
Forzar un mapa `categoria_id → tema_informe` 1:1 es honesto sólo para 3 de las 9 categorías (Vivienda,
Loteos, Cordón Cuneta); las otras 6 requieren una decisión real, no una transcripción.

**Recomendación** (a confirmar con el área, no es una decisión tomada): dado el hallazgo de arriba,
la **Opción C** original del spec ("el informe adopta las 9 categorías nuevas como agrupación") es
arquitectónicamente más simple que forzar el mapa a los 10 temas viejos — evita mantener dos
taxonomías paralelas y las categorías nuevas ya son la fuente de verdad de "campo de trabajo" en el
resto del sistema (alta de gestión, filtros, catálogo editable). El costo (que reconoce el spec) es
que rompe la comparación histórica con los 10 temas de antes; se puede mitigar documentando el corte
("desde tal fecha, el informe usa las 9 categorías") en vez de intentar reconciliar retroactivamente.

- ¿`tema_informe` sigue siendo los 10 temas actuales (mapa de compatibilidad forzado — **default
  original del spec, pero ver el hallazgo de arriba**), o el informe adopta las 9 categorías nuevas
  como agrupación (rompe la serie histórica, **recomendado** tras este análisis)?
- Si se mantienen los 10 temas: ¿qué hacemos con Gas/Kits Solares/Luces LED/Infraestructura Eléctrica
  colapsando en "Otras Obras"? ¿Conservar el regex de `detalle` **sólo** para gestiones de esa
  categoría (híbrido), o aceptar el colapso?
- ¿Cómo se clasifican "Pedidos por ATP" y "Pedidos NorOeste y Sur Sur" — sin ningún precedente
  histórico, las tiene que definir el área desde cero (no hay diff posible contra el regex viejo)?
- ¿Se dropea `categoria_general_id` tras el sign-off, o se conserva para auditoría histórica?

## 5. Criterios de aceptación

- [ ] Mapa `categoria_id` → `tema_informe` documentado y revisado por el área.
- [ ] `informe_service.py` clasifica por el mapa; `grep` por `REGEXP`/`LOWER(detalle)` en el módulo
      informe = 0.
- [ ] Reporte de doble corrida (regex vs estructurada) con lista de gestiones que cambian de tema;
      sign-off registrado.
- [ ] Los 4 endpoints devuelven totales consistentes con el sign-off para un rango de control.
- [ ] `categoria_general_id` congelada (no se escribe más) o dropeada por migración.
