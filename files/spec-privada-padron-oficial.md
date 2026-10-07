# Spec: Privada adopta el padrón oficial de localidades

**Estado**: draft
**Versión**: 0.1.0
**Servicios**: `svc-privada` (padrón espejo, gestiones, rollup), `svc-vivienda`
(endpoint interno de lectura del padrón)
**ADR**: ADR-026 (reemplaza parcialmente ADR-012, cierra el pendiente de ADR-024)
**Depende de**: `spec-normalizacion-localidades.md` (approved, v0.5.0)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-07

---

## 0. Por qué existe este spec

Criterio fijado por el usuario (2026-10-07): **Vivienda, Privada, Gral. de
Gobierno, Gasífera y Resumen Territorial tienen que usar la misma base oficial
de localidades**, para que el dato sea consistente y no haya errores por mala
escritura. Con dos salvedades ya conocidas:

1. **Gasífera y Gral. de Gobierno** cargan desde planillas externas que
   escriben algunos nombres distinto — se resuelven con el mapeo previo a la
   carga (`viv_geo_alias_manual` + resolución en sync-time, ADR-024).
2. **Mi Lugar** tiene proyectos que son loteos cuyo "nombre de localidad" es en
   realidad un barrio de Córdoba Capital — no van a matchear contra el padrón y
   eso es correcto, no un error a corregir.

Relevamiento del mismo día contra producción (rollups internos de cada servicio
comparados con el padrón):

| Fuente | Localidades | Vinculadas al padrón oficial | Con grafía distinta (resueltas por mapeo) | Sin vincular |
|---|---|---|---|---|
| Gasífera | 195 | 193 | 19 | Kilómetro 658, Santiago Temple |
| Gral. de Gobierno (ATP) | 259 | 258 | 30 | Santiago Temple |
| Privada | 379 | 378 (*) | 4 | Paraje El Barrial |

(*) En Privada el vínculo **no está persistido**: se resuelve por nombre recién
al armar el rollup territorial. Es el único módulo que no cumple el criterio.

## 1. Estado actual de Privada

- `priv_geo_localidades` (~551 filas, `id_geo` propio `String(30)`): copia
  traída del sistema viejo. Alimenta los desplegables del formulario de
  gestión (`GET /api/v1/privada/catalogos/departamentos|localidades|geo`) y la
  validación de alta/edición (`gestiones/service._geo_lookup` → 400 si la
  localidad no está en esa tabla).
- `priv_gestiones` guarda `geo_id` (id de `priv_geo_localidades`),
  `departamento`, `localidad` (texto) y `lat`/`lon`.
- `priv_localidades_info` / `priv_departamentos_info` (padrón demográfico y
  electoral): clave primaria por **texto** `(departamento, localidad)`.
- El rollup `GET /internal/privada/rollup-territorial` agrupa por texto y
  resuelve el `id_geo` oficial al vuelo llamando a
  `POST /internal/geo/resolver-localidades` de `svc-vivienda`
  (`app/integrations/geo_resolver.py`), sin persistirlo.

Consecuencia: Privada valida contra una lista que puede divergir del padrón
oficial (altas, bajas, correcciones de nombre que se hagan en uno no llegan al
otro), y el cruce con el resto de las áreas depende de un matching por texto.

## 2. Decisión de diseño

**`priv_geo_localidades` pasa a ser un espejo de solo lectura del padrón
oficial** (`viv_geo_localidades`, `svc-vivienda`): mismos `id_geo`, mismos
nombres, mismo `activo`. Privada lo sincroniza desde `svc-vivienda`; nadie lo
edita en Privada.

Se descartó que Privada consulte a `svc-vivienda` en cada request (desplegables
y validación): ata la disponibilidad del formulario de gestiones a otro
servicio. El espejo respeta database-per-service (ADR-001): no hay join
cross-DB, la fuente de verdad es una sola y Privada lee una copia local.

## 3. Alcance

### Incluido

1. **`svc-vivienda`**: endpoint interno IAM-only nuevo
   `GET /internal/geo/padron` — devuelve todas las filas de
   `viv_geo_localidades` (`id_geo`, `departamento`, `localidad`, `lat_centro`,
   `lon_centro`, `activo`). Sin gateway, mismo patrón que el resto de
   `/internal/**`. `svc-privada@` ya tiene `run.invoker` sobre `svc-vivienda`
   (ADR-015).
2. **`svc-privada` — sincronización del espejo**:
   `POST /internal/privada/geo/sync` (IAM-only) reemplaza el contenido de
   `priv_geo_localidades` con el padrón oficial (upsert por `id_geo`; lo que
   deja de existir se marca `activo=false`, no se borra). Cloud Scheduler
   diario + disparo manual. Si la lectura falla, el espejo queda como estaba
   (nunca se vacía).
3. **`svc-privada` — migración de datos de gestiones** (una vez, script o
   migración según volumen, ~2.250 filas): para cada gestión, resolver
   `(departamento, localidad)` contra el padrón oficial con el mismo algoritmo
   que el resto (`exacto` / `alias` / `manual`) y:
   - con match: `geo_id` = `id_geo` oficial, `departamento`/`localidad` = nombre
     oficial, `lat`/`lon` = centroide oficial. El texto anterior queda en un
     evento de auditoría de la gestión.
   - sin match: `geo_id = NULL`, el texto se conserva tal cual, y se lista para
     revisión (no se inventa un vínculo).
4. **`priv_localidades_info`**: se agrega la columna `id_geo` (oficial,
   nullable) resuelta con el mismo criterio. La clave primaria por texto no se
   toca en esta entrega (el `PUT` existente sigue funcionando igual).
5. **Rollup territorial**: agrupa por el `geo_id` persistido; se retira la
   resolución al vuelo de `geo_resolver.py`.
6. **Documentar las dos salvedades** de §0 en
   `spec-normalizacion-localidades.md` (hoy el comportamiento existe pero la
   excepción de los barrios de Mi Lugar no está escrita).

### Contrato HTTP (regla dura de `svc-privada`)

Las rutas, los parámetros y la forma de las respuestas de
`/api/v1/privada/**` **no cambian**. Cambian sólo valores:

- `GET /catalogos/localidades` devuelve los nombres oficiales (hoy difieren 4
  de 379 con datos, más lo que difiera en las localidades sin gestiones).
- `GET /catalogos/geo` y el campo `geo_id` de una gestión pasan a traer el
  `id_geo` oficial. El frontend lo trata como texto opaco.
- La validación de alta/edición sigue devolviendo el mismo 400 para una
  localidad que no está en el padrón.

### Excluido

- Mover `priv_localidades_info` / `priv_departamentos_info` fuera de Privada:
  la demografía sigue siendo de Privada (esa parte de ADR-012 sigue vigente).
- Pantalla de administración del padrón o de los alias (pendiente ya
  registrado en `spec-normalizacion-localidades.md §9`).
- Reescribir el texto libre de Cordón Cuneta / Córdoba Hogar / Mi Lugar con el
  nombre oficial — ver pregunta abierta 4.

## 4. Riesgos

- **Localidades del padrón de Privada que no existen en el oficial** (551 vs
  544 filas): gestiones que hoy validan y después de la migración quedarían sin
  vínculo. Hay que medirlo antes de migrar (pregunta abierta 1) y decidir caso
  por caso: alta en el padrón oficial, alias, o "sin vínculo" confirmado.
- **Cambio de nombre visible** en gestiones ya cargadas (las 4 grafías
  distintas y lo que aparezca al medir): filtros guardados o links con
  `?localidad=` con el nombre viejo dejan de coincidir. Mitigación: los filtros
  de Privada ya comparan sin mayúsculas; evaluar comparar también por alias.
- **Padrón desincronizado** si el job falla en silencio: el sync registra cada
  corrida y expone su estado, como los sync de Gasífera/ATP.
- **Homónimos**: la resolución exige departamento + localidad (mismo criterio
  de `geo/service.py`), nunca sólo localidad.

## 5. Preguntas abiertas

1. ¿Cuántas filas de `priv_geo_localidades` no tienen equivalente en
   `viv_geo_localidades`, y cuántas gestiones cuelgan de ellas? Requiere una
   consulta de solo lectura a `db_privada` y `db_vivienda` en producción.
2. ¿Los `id_geo` de `priv_geo_localidades` coinciden con los oficiales para las
   localidades comunes (vienen del mismo origen) o son otro espacio de
   identificadores? Define si la migración es un repunteo o una validación.
3. Las gestiones sin vínculo después de migrar: ¿se corrigen a mano una por
   una desde el panel, o se resuelven con alias en lote como se hizo con
   Gasífera/ATP?
4. **Texto libre en Vivienda**: `spec-normalizacion-localidades.md §2.6` dice
   que el nombre que guarda cada registro de CC/CH/ML no se pisa (se agrega
   `localidad_id` al lado). El criterio de §0 sugiere ir más lejos: que el
   nombre guardado sea siempre el oficial cuando hay vínculo, y que sólo quede
   texto libre en los casos sin vínculo (barrios de Mi Lugar). ¿Se incluye acá,
   en una entrega propia, o se deja como está?
5. Los proyectos de Mi Lugar que son barrios de Capital generan hoy una
   notificación de "localidad sin resolver" (§4.11 del spec de normalización).
   ¿Se marcan como "confirmado sin vínculo" para que dejen de notificar?

## 6. Criterios de aceptación

- [ ] `priv_geo_localidades` tiene exactamente las filas del padrón oficial
      (mismos `id_geo`, nombres y `activo`) después de un sync.
- [ ] Toda gestión con localidad del padrón tiene `geo_id` oficial y nombre
      oficial; las que no, quedan con `geo_id = NULL` y figuran en el listado
      de revisión.
- [ ] El rollup territorial de Privada no llama al resolver al vuelo y
      `resumen_territorial` agrupa las líneas de Privada por `id_geo`.
- [ ] Tests de contrato de `svc-privada` (`tests/test_contrato.py`) sin
      cambios de forma.
- [ ] Alta y edición de una gestión desde el frontend funcionan sin cambios en
      el formulario.
- [ ] Las salvedades de §0 quedan escritas en
      `spec-normalizacion-localidades.md`.
