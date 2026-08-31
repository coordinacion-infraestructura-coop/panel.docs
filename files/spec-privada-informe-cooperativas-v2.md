# Spec: svc-privada — Informe de Cooperativas v2 (clasificación sobre el modelo estructurado)

**Estado**: draft
**Versión**: 0.1.0 (candidato a enmienda de `spec-privada-categorias-programas.md`)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-08-31
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

- ¿`tema_informe` sigue siendo los 10 temas actuales (mapa de compatibilidad — **default**), o el
  informe adopta las 9 categorías nuevas como agrupación (rompe la serie histórica)?
- ¿Se dropea `categoria_general_id` tras el sign-off, o se conserva para auditoría histórica?

## 5. Criterios de aceptación

- [ ] Mapa `categoria_id` → `tema_informe` documentado y revisado por el área.
- [ ] `informe_service.py` clasifica por el mapa; `grep` por `REGEXP`/`LOWER(detalle)` en el módulo
      informe = 0.
- [ ] Reporte de doble corrida (regex vs estructurada) con lista de gestiones que cambian de tema;
      sign-off registrado.
- [ ] Los 4 endpoints devuelven totales consistentes con el sign-off para un rango de control.
- [ ] `categoria_general_id` congelada (no se escribe más) o dropeada por migración.
