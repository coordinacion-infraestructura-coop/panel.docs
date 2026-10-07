# Spec: Asignación manual de localidades sin resolver

**Estado**: approved
**Versión**: 1.0.0
**Servicios**: `svc-vivienda` (pendientes, alias, propagación, endpoints Admin),
`svc-privada` y `svc-datos-externos` (campo opcional `cantidad` en su cliente
del resolver), `frontend` (pantalla Admin enlazada desde Notificaciones),
`infra` (gateway)
**ADR**: ADR-026 (cierra su pendiente), ADR-024 (modifica `viv_geo_alias_manual`)
**Depende de**: `spec-normalizacion-localidades.md` (approved, v0.7.0),
`spec-privada-padron-oficial.md` (implemented, v1.3.0), `spec-notificaciones.md`
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-07

---

## Changelog

- **1.0.0** (2026-10-07): aprobado por el usuario, sin cambios de contenido.
- **0.2.0** (2026-10-07): incorpora las seis decisiones del usuario (§9). Único
  cambio respecto de lo recomendado en 0.1.0: los endpoints van en
  `/api/v1/geo/**` (transversal), no bajo `/vivienda`.
- **0.1.0** (2026-10-07): borrador inicial.

## 0. Por qué existe este spec

Es el pendiente de `spec-privada-padron-oficial.md §7` y
`spec-normalizacion-localidades.md §9`: hoy una localidad nueva que no matchea
contra el padrón oficial (`viv_geo_localidades`) sólo se puede corregir
escribiendo una migración que agregue una fila a `viv_geo_alias_manual`.
Se quiere que un Admin lo haga desde la aplicación.

## 1. Estado actual (relevado en el código, 2026-10-07)

- `app/geo/service.py::resolver_lote` es el único punto de resolución. Lo usan
  en proceso CC/CH/ML (`resolver_uno`, sólo al crear o editar un registro) y
  por HTTP (`POST /internal/geo/resolver-localidades`) Gasífera (`gas_pit`),
  Gral. de Gobierno (`atp`), Privada (`privada`) y datos externos
  (`datos_externos`). Los cuatro clientes ya mandan `origen`.
- **El alias se busca sólo por el texto de la localidad, sin departamento, y
  se prueba antes que el match exacto contra el padrón** (`_resolver_uno`).
  Un alias pisa a cualquier localidad homónima de cualquier departamento y de
  cualquier fuente, incluidos los "confirmado sin vínculo" (`id_geo NULL`).
- `_notificar_sin_match` crea una notificación de texto por cada corrida con
  algún `sin_match`. No guarda nada estructurado y no deduplica: Gasífera y
  ATP sincronizan cada hora, así que cada pendiente genera 24 avisos por día.
- Cómo re-resuelve cada fuente (define la propagación de §4):

  | Fuente | Dónde guarda el vínculo | Cuándo vuelve a resolver |
  |---|---|---|
  | CC / CH / ML | `localidad_id` (misma base) | Sólo al editar ese registro |
  | Gasífera, ATP | `id_geo` en su base | Todas las filas, en cada sync (cada hora) |
  | Datos externos | `id_geo` en `ext_transferencias` | Las filas del período que se sincroniza (mensual) |
  | Privada | `geo_id` de la gestión | Sólo con `POST /internal/privada/geo/normalizar-gestiones`; el rollup resuelve al vuelo lo que no tiene `geo_id` válido |

- Privada valida el alta de gestiones contra su espejo del padrón (400 si la
  localidad no está): no produce `sin_match` nuevos, sólo los heredados.
- Vivienda y Privada **pisan el texto con el nombre oficial** cuando hay
  vínculo (v0.7.0 / ADR-026). Consecuencia para el "deshacer": una vez
  propagado, cambiar el alias no alcanza, porque el registro ya no tiene el
  texto original.

## 2. Alcance

### Incluido

1. Tabla de pendientes estructurada, alimentada por el resolver, con dedup de
   la notificación.
2. Alias por texto + departamento, además del alias global actual.
3. Endpoints Admin para listar pendientes, resolverlos (vincular o "confirmado
   sin vínculo"), descartarlos, deshacer y consultar los alias existentes.
4. Propagación del vínculo a lo ya cargado.
5. Pantalla Admin, enlazada desde cada notificación del resolver.

### Excluido

- ABM del padrón (`viv_geo_localidades`): dar de alta una localidad que falta
  (ej. "Santiago Temple") sigue siendo por migración.
- Fusión de registros duplicados de CC/CH/ML (sigue manual,
  `spec-normalizacion-localidades.md §3.5`).
- Reescribir el texto de las planillas de Gasífera/ATP (ADR-026, salvedad 1).
- Reversión de gestiones de Privada al deshacer un vínculo (§5).

## 3. Modelo de datos (`svc-vivienda`, una migración)

### 3.1 Tabla nueva `viv_geo_pendientes`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `String(36)` PK | uuid |
| `origen` | `String(50)` | `cordon_cuneta`, `cordoba_hogar`, `mi_lugar`, `gas_pit`, `atp`, `privada`, `datos_externos` |
| `departamento_original`, `localidad_original` | `String(200)` | tal cual llegó (última grafía vista) |
| `departamento_normalizado` | `String(200)`, NOT NULL, default `''` | `normalize_departamento()` |
| `texto_normalizado` | `String(200)` | `normalize_name()` de la localidad |
| `cantidad` | `Integer`, nullable | registros afectados en la última corrida que la informó (§3.3) |
| `primera_vez`, `ultima_vez` | timestamptz | |
| `estado` | `String(20)` | `pendiente` / `resuelta` / `descartada` |
| `alias_id` | `String(36)`, nullable | alias que la resolvió |
| `resuelta_at`, `resuelta_by` | | |

Única por `(origen, departamento_normalizado, texto_normalizado)`.

`resolver_lote`, al terminar, hace upsert de cada `sin_match` con localidad no
vacía (actualiza `ultima_vez`, grafía y `cantidad`), dentro de un `SAVEPOINT`
best-effort: una falla acá nunca interrumpe el flujo que llamó al resolver
(mismo criterio que la notificación hoy). Una fila `descartada` o `resuelta`
que vuelve a aparecer como `sin_match` se reabre.

**Notificación**: se emite sólo cuando el lote crea o reabre al menos un
pendiente (deja de repetirse en cada corrida). Lleva
`enlace = "/admin/localidades-sin-resolver"`. Cambia
`spec-normalizacion-localidades.md §4.11` ("no dedup").

**Carga inicial**: la migración inserta los pendientes de CC/CH/ML (registros
activos con `localidad_id NULL` y sin alias). Gasífera y ATP se cargan solos
en el primer sync; Privada, en el próximo recálculo de Resumen Territorial;
datos externos, en su próximo sync.

### 3.2 Cambios en `viv_geo_alias_manual`

- `departamento_normalizado String(200) NOT NULL DEFAULT ''` — `''` = alias
  global (todas las filas actuales quedan así, sin cambio de comportamiento).
- La unicidad pasa de `(texto_normalizado)` a
  `(texto_normalizado, departamento_normalizado)` entre filas no borradas.
  Fundamento (usuario, 2026-10-07): hay localidades homónimas, pero nunca dos
  con la misma localidad + departamento.
- `updated_at`, `updated_by`, `deleted_at` (hoy la tabla no tiene baja lógica).

`_resolver_uno` busca primero el alias `(texto, departamento)` y después el
global `(texto, '')`. El resto del orden (`manual` → `exacto` → `alias` →
`sin_match`) no cambia. Modifica `spec-normalizacion-localidades.md §4.2`.

**Alias globales existentes**: la misma migración acota por departamento los
"confirmado sin vínculo" que hoy tapan homónimos ("Santa Teresa" y "Barrio
Chingolo" → Capital). La lista exacta se arma revisando las filas actuales
contra el padrón y se confirma con el usuario antes de correrla.

### 3.3 "Cuántos registros afecta"

- CC/CH/ML: se cuenta en vivo al listar (misma base).
- Gasífera/ATP: mandan todas sus filas en cada sync, así que la cantidad es la
  de apariciones del par en el lote.
- Privada y datos externos deduplican antes de llamar: `ResolverItem` suma un
  campo opcional `cantidad` (compatible hacia atrás) y esos dos clientes lo
  mandan (gestiones del grupo / filas de transferencias). Si falta, vale la
  cuenta de apariciones.

## 4. Propagación al resolver un pendiente

Todo lo de `svc-vivienda` ocurre en la misma transacción que el alta del alias.

| Fuente | Qué pasa |
|---|---|
| CC / CH / ML | Se re-resuelven los registros activos sin vínculo cuyo texto y departamento coinciden: `localidad_id`, `localidad_match_tipo = "manual"`, y nombre + departamento oficiales. Una fila de `viv_audit_log` por registro con los valores anteriores. |
| Gasífera / ATP | Nada inmediato: toman el vínculo en el próximo sync horario (hasta 1 h). El texto de la planilla se conserva. |
| Datos externos | El período vigente lo toma en su próximo sync; los períodos anteriores, sólo si se re-sincronizan. |
| Privada | `svc-vivienda` llama a `POST /internal/privada/geo/normalizar-gestiones?dry_run=false` después del commit (best-effort; ya tiene `run.invoker`). |
| Resumen Territorial | Se recalcula el snapshot al final (best-effort). |

Si el vínculo deja dos registros activos de CC o CH con el mismo `id_geo`, se
aplica igual y se avisa en la confirmación y en la respuesta; la fusión sigue
siendo manual.

"Confirmado sin vínculo" no modifica registros: sólo crea el alias con
`id_geo NULL`, con lo que el resolver deja de devolver `sin_match`.

La respuesta informa, por fuente, qué se aplicó ya y qué queda para el próximo
sync, y cualquier llamada best-effort que haya fallado.

## 5. Auditoría y deshacer

- Alta, cambio y baja de alias: `viv_audit_log` (`resource_type =
  "geo_alias_manual"`, `resource_id` = id del alias, payload con antes/después,
  motivo y pendiente de origen). El `motivo` es obligatorio en la pantalla.
- **Deshacer**: baja lógica del alias, reapertura del pendiente y restauración
  de los registros de CC/CH/ML desde los valores anteriores guardados en
  `viv_audit_log`. Un registro editado después de la propagación no se
  restaura (se informa en la respuesta).
  - Gasífera/ATP/datos externos se corrigen solos en el siguiente sync.
  - **Privada: limitación documentada.** `normalizar-gestiones` conserva el
    `geo_id` guardado aunque el texto deje de resolver, y el texto original
    sólo queda en los eventos `NORMALIZACION_LOCALIDAD`. Deshacer no revierte
    gestiones de Privada; la respuesta lo avisa.
- Corregir un vínculo mal hecho = deshacer + volver a resolver.

## 6. Endpoints (todos `Admin`)

Transversales, prefijo `/api/v1/geo` (sin `/vivienda`, mismo criterio que
`portal` y `notificaciones`, ADR-007). Router nuevo en `app/geo/`, servido por
`svc-vivienda`. `GET /api/v1/vivienda/geo/duplicados` queda donde está.

```
GET  /api/v1/geo/pendientes?estado=&origen=&limit=&offset=
POST /api/v1/geo/pendientes/{id}/resolver
       body: { id_geo: str | null, alcance: "departamento" | "global", motivo: str, dry_run: bool }
POST /api/v1/geo/pendientes/{id}/descartar
POST /api/v1/geo/pendientes/{id}/deshacer
GET  /api/v1/geo/alias?limit=&offset=
```

- `resolver` con `dry_run=true` devuelve el impacto (registros de CC/CH/ML que
  cambiarían, con nombre actual y oficial; duplicados que se generarían) sin
  escribir. La pantalla lo usa como paso de confirmación.
- `alcance` por defecto: `departamento`.
- `id_geo` debe ser una fila activa del padrón (400 si no). Si ya existe un
  alias para ese texto y alcance: 409.
- `descartar`: saca el pendiente de la lista sin crear alias (ej. la fuente ya
  corrigió la grafía); se reabre si vuelve a aparecer.
- El desplegable de localidades reusa `GET /api/v1/vivienda/cordon-cuneta/geo`.

**Gateway**: 5 paths nuevos, cada uno con su `options:` (`security: []`), y
una configuración nueva `ministerio-config-v{YYYYMMDD}`, armada sobre la
activa (§10).

## 7. Frontend

- `src/modules/admin/pages/LocalidadesSinResolverPage.tsx`, ruta
  `/admin/localidades-sin-resolver`, `ProtectedRoute roles={['Admin']}`.
- Tabla: localidad, departamento, fuente (etiqueta legible), registros
  afectados, primera y última vez vista. Filtros por fuente y estado.
- Por fila: "Vincular" (desplegable de departamento preseleccionado con el de
  la fila → localidad; alcance; motivo; confirmación con el impacto de
  `dry_run`), "Confirmar sin vínculo", "Descartar". En la vista de resueltas:
  "Deshacer".
- `NotificacionesPage.tsx` no cambia: ya muestra "Ver" cuando la notificación
  trae `enlace`. Se agrega un acceso a la pantalla para Admin en `Layout.tsx`.

## 8. Tests y orden de despliegue

**Tests (`svc-vivienda`)**: upsert y reapertura de pendientes; una sola
notificación por pendiente nuevo, ninguna en corridas repetidas; precedencia
alias por departamento > alias global > exacto; un alias por departamento no
captura al homónimo de otro departamento; `cantidad` informada vs. contada;
propagación a CC/CH/ML con audit; aviso de duplicado; `dry_run` no escribe;
deshacer restaura y respeta registros editados después; 403 fuera de Admin.
Migración verificada contra Postgres real (upgrade + downgrade).
`svc-privada` / `svc-datos-externos`: el cliente manda `cantidad`.
Frontend: `npm run build`.

**Despliegue**: (1) migración de `svc-vivienda` en Cloud Shell — el código
nuevo depende de las columnas y la tabla; (2) push a `main` de `panel.backend`
(CI despliega `svc-vivienda`, `svc-privada` y `svc-datos-externos`; el campo
`cantidad` es opcional, no hay orden entre ellos); (3) configuración nueva del
gateway; (4) frontend.

## 9. Decisiones del usuario (2026-10-07)

1. **Alias por texto + departamento**: sí.
2. **Cantidad para Privada y datos externos**: campo opcional `cantidad`.
3. **Privada**: `normalizar-gestiones` se dispara automáticamente; el deshacer
   de Privada queda como limitación documentada.
4. **Duplicado de CC/CH al vincular**: se aplica y se avisa; fusión manual.
5. **Prefijo**: `/api/v1/geo/**`, transversal.
6. **Gasífera y ATP**: esperan al sync horario.

## 10. Verificaciones pendientes antes de implementar

- **Configuración activa del gateway**: `ministerio-config-v20260928`
  (verificado con `gcloud api-gateway gateways describe`, 2026-10-07). Volver
  a verificar el día del deploy: otra sesión puede haberla cambiado.
- **Número de migración**: la última conocida es `0037`; confirmar el head.
- **Lista de alias globales a acotar** (§3.2), contra datos reales.

## 11. Criterios de aceptación

- [ ] Un `sin_match` nuevo aparece en la pantalla con departamento, fuente y
      cantidad, y genera una sola notificación con enlace.
- [ ] Corridas siguientes con el mismo pendiente no generan notificaciones.
- [ ] Vincular desde la pantalla deja los registros de CC/CH/ML con vínculo y
      nombre oficial en el momento, y los de Gasífera/ATP tras el sync.
- [ ] "Confirmado sin vínculo" saca el pendiente y deja de notificarse.
- [ ] Deshacer restaura CC/CH/ML y reabre el pendiente.
- [ ] Toda alta, cambio o baja de alias queda en `viv_audit_log`.
- [ ] Sólo `Admin` accede (403 para el resto, pantalla y endpoints).
