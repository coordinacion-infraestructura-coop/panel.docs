# Spec: Privada adopta el padrón oficial de localidades

**Estado**: approved
**Versión**: 1.0.0
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

## 1.1 Medición en producción (2026-10-07)

Consulta de solo lectura a `db_vivienda` y `db_privada` (transacciones
`READ ONLY`, vía `cloud-sql-proxy`), autorizada por el usuario. Responde las
preguntas abiertas 1 y 2 de la v0.1.0.

**Los dos padrones son la misma lista.** `priv_geo_localidades` y
`viv_geo_localidades` tienen las mismas 544 filas con los mismos `id_geo`;
ninguna localidad existe en uno y no en el otro, y ningún nombre difiere una
vez normalizado. La copia de Privada sólo quedó vieja:

- 54 filas con la grafía anterior a la migración 0035 (minúsculas).
- 7 filas que el oficial desactivó por duplicadas (migración 0033: id 43, 78,
  81, 212, 213, 285, 555) siguen activas en Privada, así que el formulario de
  gestiones todavía las ofrece.

**Gestiones activas: 2.258.**

| Situación | Gestiones |
|---|---|
| `geo_id` correcto (coincide con el que resuelve el oficial por nombre) | 1.923 |
| `geo_id` sintético `BOOT\|DEPTO\|LOCALIDAD` heredado del sistema viejo — no existe en ningún padrón, pero el nombre resuelve | 325 |
| `geo_id` de una fila desactivada por duplicada (Charbonier 555 → 141; Eufrasio Loza 212 → 558) | 7 |
| `geo_id` equivocado: "LAS HIGUERAS" (Río Cuarto) apunta a 533 "La Higuera" (Cruz del Eje); corresponde 174 | 2 |
| Sin resolver por nombre | 2 |

Las 2 sin resolver: "PARAJE EL BARRIAL" (Tulumba, no está en el padrón) y
"MONTE CRISTO" cargada con departamento Colón (está en Río Primero; el
`geo_id` 392 guardado es correcto, lo que está mal es el departamento).

Además, 180 gestiones (38 combinaciones) tienen el texto de localidad con una
grafía que no es idéntica a la oficial — todas diferencias de mayúsculas.

**`priv_localidades_info`**: 430 filas, 429 resuelven contra el oficial; la
única que no es "SANTIAGO TEMPLE" (ausente del padrón, ya conocido).

**Vivienda** (para la pregunta abierta 4):

| Tabla | Registros | Con vínculo | Texto no idéntico al oficial | …de los cuales distinto aun sin tildes/mayúsculas |
|---|---|---|---|---|
| Cordón Cuneta | 89 | 89 | 11 | 1 (CHARRAS) |
| Córdoba Hogar | 221 | 221 | 24 | 3 (LUXARDO, GENERAL BALDISERA, CHARRAS) |
| Mi Lugar | 49 | 47 | 1 | 0 |

Los 2 proyectos de Mi Lugar sin vínculo son "Barrio Chingolo" y "Santa
Teresa" — el caso de los loteos por barrio de §0.

## 1.2 Decisiones del usuario (2026-10-07) — cierran las preguntas abiertas

1. **Gestiones ya cargadas: en lote.** Se ordenan todas de una vez (repunteo
   al `id_geo` oficial + nombre oficial), no una por una desde el panel.
2. **Lo nuevo que no resuelva: notificación + corrección manual.** Toda
   localidad nueva que no matchee el padrón se notifica en el panel de
   notificaciones (ya ocurre, `spec-normalizacion-localidades.md §4.11`) y
   tiene que poder corregirse a mano desde ahí. Ese panel de asignación manual
   es el pendiente ya registrado en `spec-normalizacion-localidades.md §9` —
   **queda como entrega siguiente** (§7), con spec propia.
3. **Nombre guardado en Cordón Cuneta / Córdoba Hogar / Mi Lugar: no se toca
   por ahora.** Queda documentado como pendiente para una próxima sesión (§7).
4. **Barrios de Mi Lugar: "confirmado sin vínculo".** "Barrio Chingolo" y
   "Santa Teresa" se registran en `viv_geo_alias_manual` con `id_geo = NULL`
   (mismo mecanismo que "Santiago Temple"), para que dejen de generar avisos.
   Se suma "Paraje El Barrial" (Privada), ausente del padrón.

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
3. **`svc-privada` — normalización en lote de las gestiones**:
   `POST /internal/privada/geo/normalizar-gestiones` (IAM-only, idempotente,
   `dry_run=true` por defecto: devuelve el resumen de lo que cambiaría sin
   escribir). No es una migración Alembic porque necesita consultar el
   resolver de `svc-vivienda` (alias manuales). Para cada gestión activa
   resuelve `(departamento, localidad)` contra el padrón oficial con el mismo
   algoritmo que el resto (`exacto` / `alias` / `manual`); si el texto no
   resuelve pero el `geo_id` guardado es una fila activa del padrón, vale ese
   `geo_id` (caso "Monte Cristo" cargada con otro departamento). Después:
   - con match: `geo_id` = `id_geo` oficial, `departamento`/`localidad` = nombre
     oficial. Por cada gestión modificada se escribe un evento
     `ACTUALIZA_DATO` con el valor anterior y el nuevo (mismo criterio que la
     reasignación masiva de "Otras Obras", `spec-privada-categorias-programas.md`
     Anexo B). `lat`/`lon` de la gestión no se tocan (son del punto cargado, no
     del centroide).
   - sin match: `geo_id = NULL`, el texto se conserva tal cual, y se lista para
     revisión (no se inventa un vínculo).
4. **`priv_localidades_info`**: se agrega la columna `id_geo` (oficial,
   nullable) resuelta con el mismo criterio. La clave primaria por texto no se
   toca en esta entrega (el `PUT` existente sigue funcionando igual).
5. **Rollup territorial**: usa el `geo_id` persistido cuando es una fila
   activa del espejo y consolida en una sola línea las gestiones de un mismo
   `id_geo`. La resolución al vuelo de `geo_resolver.py` queda sólo como
   respaldo para las filas sin `geo_id` válido (robusto durante la transición
   y ante datos nuevos sin vínculo).
6. **`svc-vivienda` — "confirmado sin vínculo"** (decisión 4 de §1.2):
   migración que agrega a `viv_geo_alias_manual` "Barrio Chingolo", "Santa
   Teresa" y "Paraje El Barrial" con `id_geo = NULL`.
7. **Documentar las dos salvedades** de §0 en
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

- **Cambio de nombre visible** en gestiones ya cargadas (180 gestiones, todas
  diferencias de mayúsculas — §1.1): filtros guardados o links con
  `?localidad=` con el nombre viejo dejan de coincidir. Mitigación: los filtros
  de Privada ya comparan sin mayúsculas; evaluar comparar también por alias.
- **Padrón desincronizado** si el job falla en silencio: el sync registra cada
  corrida y expone su estado, como los sync de Gasífera/ATP.
- **Homónimos**: la resolución exige departamento + localidad (mismo criterio
  de `geo/service.py`), nunca sólo localidad.

## 5. Preguntas abiertas

Ninguna — las cinco de la v0.1.0 quedaron resueltas por la medición (§1.1) y
las decisiones del usuario (§1.2).

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

## 7. Pendientes para próximas entregas

- **Panel de asignación manual de localidades sin resolver** (decisión 2 de
  §1.2): desde el feed de notificaciones, un Admin ve las localidades que no
  matchearon y las vincula a una fila del padrón o las marca "confirmado sin
  vínculo", sin pasar por una migración. Requiere un endpoint de alta en
  `viv_geo_alias_manual` gateado a Admin y una pantalla. Spec propia.
- **Nombre guardado en Cordón Cuneta / Córdoba Hogar / Mi Lugar** (decisión 3
  de §1.2): 36 registros no tienen el texto idéntico al oficial (§1.1), 4 de
  ellos con diferencia real ("CHARRAS" ×2, "LUXARDO", "GENERAL BALDISERA").
  Pasarlos al nombre oficial cuando hay vínculo cambia
  `spec-normalizacion-localidades.md §2.6`.
- **Clave de `priv_localidades_info`**: sigue siendo texto; evaluar pasarla a
  `id_geo` cuando la columna nueva esté poblada y estable.
