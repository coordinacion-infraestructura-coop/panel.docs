# Spec: Vinculaciones entre servicios — Vivienda → Privada (gestiones)

**Estado**: approved (implementado, testeado y **activado en prod** 2026-09-14)
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-09-14
**Servicio**: `svc-vivienda` (módulo nuevo `app/integrations/`) + `svc-privada` (`app/gestiones/vivienda_sync.py` + endpoint nuevo en `app/internal/router.py`)
**Depende de**: `spec-checklist-tecnico-dgv.md` (patrón polimórfico `caso_tipo`/`caso_id`), `spec-resumen-territorial-ficha-localidad.md` (ADR-016, patrón de endpoint interno IAM-only cross-servicio), `spec-notificaciones.md` (ADR-019, canal de alertas), `spec-privada-categorias-programas.md` (E1, catálogos `priv_categorias`/`priv_programas`/`priv_areas`)
**ADRs**: ADR-020 (primera escritura cross-servicio), ADR-019 (consumidor del panel de notificaciones)

> Esta es la spec **paraguas** de vinculaciones cross-servicio en GestorCooperativo.
> La primera (y por ahora única) vinculación implementada es Vivienda → Privada
> (gestiones), pero el diseño — tabla de estado del vínculo + log append-only +
> endpoint interno de upsert idempotente por `id_legacy` — está pensado para
> reutilizarse con futuras vinculaciones entre otros paneles/servicios. Una
> vinculación nueva agrega una sección propia a este documento (o una spec hija
> que lo referencia desde acá), no un documento paralelo. Las specs de los
> módulos de origen (`spec-checklist-tecnico-dgv.md`, y cualquier spec futura de
> Cordón Cuneta / Córdoba Hogar / Mi Lugar) deben referenciar esta spec en su
> sección de integraciones en vez de repetir el diseño.

## 0. Origen

Pedido del usuario (2026-09-10/14): vincular los paneles DGV de `svc-vivienda`
(Cordón Cuneta, Córdoba Hogar, Mi Lugar) con el sistema de gestiones de
`svc-privada`, de forma que un caso cargado en cualquiera de los tres genere o
actualice automáticamente una gestión en Privada, con los campos de
clasificación (campo de trabajo, programa, área, ministerio, Ok
Gobernador/Ministro) ya completados — hoy esos dos sistemas registran el mismo
trámite territorial sin ningún vínculo entre sí.

En paralelo, el usuario estaba armando un panel de notificaciones internas
(`spec-notificaciones.md`, ADR-019) que resultó quedar implementado (backend +
frontend + gateway, pendiente sólo de deploy) *antes* de que esta spec se
escribiera — por eso esta vinculación lo consume directamente en vez de dejar
un mecanismo de alerta propio a resolver después (ver §3.6).

## 1. Propósito

Cuando se crea o edita un caso en los paneles DGV de `svc-vivienda`, crear o
vincular/sincronizar la gestión correspondiente en `svc-privada`, seteando un
conjunto acotado de campos derivados, sin bloquear ni romper el flujo de
Vivienda si la propagación falla, y avisando por el panel de notificaciones
cuando la sincronización corrige datos de una gestión ya cargada o cuando el
caso queda sin poder vincularse automáticamente.

## 2. Alcance

### Incluido (v1)
- Propagación saliente desde `crear_municipio`/`actualizar_municipio` (CC),
  `crear_localidad`/`actualizar_localidad` (CH), `crear_proyecto_ml`/
  `actualizar_proyecto_ml` (ML) hacia `POST /internal/privada/gestiones/sync`.
- Vínculo por `nro_expediente` (si el caso lo tiene) o `localidad+departamento`
  (si no), con desambiguación explícita — nunca se auto-vincula con 2+ candidatos.
- Creación de gestión nueva con campos derivados, si no hay match.
- Sincronización (sobrescritura acotada) de los campos derivados sobre una
  gestión ya vinculada, con diff calculado.
- Alertas en el panel de notificaciones (ADR-019) cuando la sincronización
  corrige campos de una gestión ya vinculada, o cuando un caso queda pendiente
  de revisión manual.
- Estado del vínculo consultable por backend (`viv_privada_vinculos`) y log
  técnico append-only de cada intento (`viv_privada_sync_log`) — sin pantalla
  propia en v1 (no se pidió una; el estado se ve por la notificación).

### Fuera de alcance
- Cualquier pantalla/endpoint de lectura dedicado sobre `viv_privada_vinculos`
  o `viv_privada_sync_log` (se agrega si hace falta más adelante).
- Propagación en sentido inverso (Privada → Vivienda).
- Reconciliación proactiva/batch de vínculos `PENDING_REVIEW` (job periódico) —
  se resuelven recién en la próxima sincronización de ese caso (una edición
  posterior, o un reintento manual).
- Resolución manual asistida de un `PENDING_REVIEW` (elegir a mano cuál de los
  candidatos ambiguos vincular) — hoy se resuelve editando los datos en
  cualquiera de los dos sistemas para que deje de haber ambigüedad.
- Cualquier vinculación que no sea Vivienda → Privada (el diseño es reusable,
  pero esta versión de la spec sólo cubre esta instancia).

## 3. Decisiones de arquitectura

Ver ADR-020 en `arquitectura.md` para el resumen ejecutivo; acá el detalle.

### 3.1 Mecanismo: llamada HTTP inline, best-effort
Se evaluaron tres mecanismos — llamada HTTP inline síncrona, `FastAPI
BackgroundTasks`, y Pub/Sub con subscriber — y se eligió el primero. Razones:
volumen bajo (altas manuales de operador, pocas por semana: 54 municipios CC,
43 localidades CH, un puñado de proyectos ML), latencia irrelevante para una
operación manual esporádica, cero infraestructura nueva, reusa exactamente el
patrón ya existente (`resumen_territorial.fetch_privada_lineas`, ADR-016) y
permite QA local sin emuladores. `BackgroundTasks` habría evitado el riesgo de
huérfano pre-commit (§8) a costa de un patrón nuevo en el repo y de requerir
una sesión de DB propia; Pub/Sub habría dado reintentos automáticos a costa de
infraestructura que `svc-privada` deliberadamente no tiene (ADR-009: "sin
Pub/Sub") para un volumen que no lo justifica.

### 3.2 Dónde vive el estado del vínculo: en `svc-vivienda`
`viv_privada_vinculos` + `viv_privada_sync_log` viven en `svc-vivienda`, no en
`svc-privada` ni duplicados en ambos — es Vivienda quien "sabe" a qué gestión
está atado cada uno de sus casos; `svc-privada` no modela ese concepto, sólo
guarda el `id_legacy` que recibe.

### 3.3 Idempotencia vía `id_legacy` determinístico
`svc-vivienda` calcula `id_legacy = f"vivienda:{caso_tipo}:{caso_id}"` sin
consultar nada. `svc-privada` resuelve "¿ya sincronicé este caso?" con un único
`SELECT` sobre `priv_gestiones.id_legacy` (`UNIQUE` desde la migración `0001`)
— un reintento, o cualquier edición posterior del mismo caso, nunca duplica.

### 3.4 Algoritmo de matching (`app/gestiones/vivienda_sync.py`, svc-privada)
Orden estricto, corta en el primer resultado:
0. **`id_legacy` propio** — ya reclamado en una sincronización anterior de este
   caso. Es el camino que toma todo reintento y toda edición posterior.
1. **`nro_expediente`** (si el caso lo tiene) — identidad fuerte, sin filtrar
   por estado (una gestión `FINALIZADA`/`ARCHIVADO` con el mismo expediente
   sigue siendo la gestión correcta).
   - 0 candidatos → crea una gestión nueva.
   - 1 candidato sin `id_legacy` → vincula y sincroniza.
   - 1 candidato con `id_legacy` de otro caso → `PENDING_REVIEW`
     (`EXPEDIENTE_YA_VINCULADO_OTRO_CASO`), no se toca nada.
   - 2+ candidatos → `PENDING_REVIEW` (`EXPEDIENTE_AMBIGUO`), no se toca nada.
2. **`localidad + departamento`** (sólo si el caso no tiene expediente) — sólo
   gestiones no cerradas (`estado` fuera de `FINALIZADA`/`ARCHIVADO`) y sin
   `id_legacy` de otro caso (para no robarle el vínculo a otro caso de
   Vivienda).
   - 0 candidatos → crea. 1 candidato → vincula. 2+ → `PENDING_REVIEW`
     (`LOCALIDAD_DEPARTAMENTO_AMBIGUO`) — mismo riesgo de colisión ya
     documentado en `spec-resumen-territorial-ficha-localidad.md` (ahí, el
     join por nombre normalizado colisionaba porque los dos lados traían el
     departamento de fuentes distintas). Acá el riesgo es menor (mismo
     catálogo `priv_geo_localidades` que valida ambos lados) pero se aplica
     la misma regla defensiva: nunca auto-vincular con ambigüedad.

### 3.5 Escritura acotada (whitelist)
El sync sólo puede escribir 7 campos "derivados" de `priv_gestiones`:
`categoria_id`, `programa_id`, `area_id`, `ministerio_agencia_id`,
`ok_gobernador`, `ok_ministro`, y `nro_expediente` (este último sólo si el caso
de Vivienda tiene uno — nunca lo vacía). Nunca toca `detalle`, `observaciones`,
`urgencia`, `estado`, ni ningún otro campo que un operador de Privada haya
cargado a mano.

### 3.6 Alertas: se reusa el panel de notificaciones existente (ADR-019)
El plan original era dejar un mecanismo de alerta propio (una tabla de
"pendiente de futuro panel") porque, al momento de diseñar esta vinculación, el
usuario todavía estaba construyendo el panel de notificaciones por separado y
avisaría cuando estuviera listo. Al revisar el código se encontró que ese panel
**ya estaba implementado** (ADR-019, migración `0028`, `spec-notificaciones.md`
en estado `approved`, backend+frontend+gateway completos, sólo pendiente de
deploy) — con exactamente el canal que hacía falta: `POST
/internal/notificaciones` (IAM-only). Se confirmó con el usuario reusarlo en
vez de duplicar el mecanismo. Como el módulo de notificaciones vive en el
mismo servicio (`svc-vivienda`), `privada_sync.py` llama a
`notificaciones_service.crear(...)` **en proceso** (import directo, misma
sesión de DB) en vez de un loopback HTTP contra su propio endpoint interno.

Se generan dos tipos de alerta, ambos con `destino_tipo="secretaria"`,
`destino_valor="privada"` (para que el área de Privada vea que Vivienda tocó su
gestión, o que hay un caso sin poder vincularse):
- **Corrección de campos** (`LINKED_EXISTING` con diff no vacío) — nivel
  `info`, detalla qué campos cambiaron.
- **Pendiente de revisión manual** (`PENDING_REVIEW`) — nivel `advertencia`.

Deliberadamente **no** se notifica por un `ERROR` transitorio (timeout, 5xx) —
sería ruido; el vínculo ya `LINKED` no se degrada por eso (§3.7) y el log
técnico (`viv_privada_sync_log`) alcanza para diagnosticarlo.

### 3.7 Regla de no-flapping ante errores transitorios
Si un caso ya está `LINKED` (tiene `gestion_id`) y una sincronización posterior
falla por un problema de infraestructura (timeout, 5xx, `svc-privada` caído),
`viv_privada_vinculos.estado_vinculo` **no** se degrada a `ERROR` — sólo se
actualiza `ultimo_intento_at`. Motivo: un timeout puntual no debería hacer
aparecer una alerta de "pendiente de revisión" sobre un caso que en realidad
está bien vinculado.

### 3.8 Endpoint interno IAM-only, mismo patrón que ADR-015/ADR-016
`POST /internal/privada/gestiones/sync` se agrega al router IAM-only existente
de `svc-privada` (`app/internal/router.py`, `prefix="/internal/privada"`) —
sin `get_current_user`, sin declarar en `infra/gateway/openapi.yaml`,
protegido sólo por `roles/run.invoker` de la SA de `svc-vivienda` sobre
`svc-privada` (mismo grant que ya usa `GET /internal/privada/rollup-territorial`,
ADR-016 / E5a). Resultados de negocio (`PENDING_REVIEW`, geo inválida) viajan
en el body de una respuesta 200, no como error HTTP — sólo un payload inválido
(422) o una excepción no prevista (500) dan un status distinto.

## 4. Modelo de datos

### `svc-vivienda` (migración `0029`)

**`viv_privada_vinculos`** — snapshot, 1 fila por caso (upsert):

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `varchar(36)` PK | uuid |
| `caso_tipo` | `varchar(2)` | `'cc'\|'ch'\|'ml'`, CHECK — mismo patrón que `ChecklistTecnico.programa` |
| `caso_id` | `varchar(36)` | sin FK real (polimórfico) |
| `id_legacy` | `varchar(100)` UNIQUE | `vivienda:{caso_tipo}:{caso_id}` |
| `gestion_id` | `varchar(36)` NULL | id de `priv_gestiones`, si resuelto |
| `estado_vinculo` | `varchar(20)` | `LINKED\|PENDING_REVIEW\|ERROR`, CHECK |
| `motivo` | `varchar(60)` NULL | código de motivo si no está `LINKED` |
| `ultimo_intento_at` | `timestamptz` | cada sync, exitoso o no |
| `ultimo_ok_at` | `timestamptz` NULL | último resultado `LINKED_NEW`/`LINKED_EXISTING` |
| `created_at`/`updated_at` | `timestamptz` | |

`UNIQUE(caso_tipo, caso_id)`.

**`viv_privada_sync_log`** — append-only, 1 fila por intento (log técnico, no
la alerta de usuario — precedente: `viv_cc_sync_log`/`viv_informe_snapshot`):

| Columna | Tipo | Notas |
|---|---|---|
| `id` | `varchar(36)` PK | |
| `caso_tipo`, `caso_id`, `id_legacy` | — | |
| `resultado` | `varchar(20)` | `LINKED_NEW\|LINKED_EXISTING\|PENDING_REVIEW\|ERROR` |
| `gestion_id` | `varchar(36)` NULL | |
| `motivo` | `varchar(60)` NULL | |
| `diff_json` | `json` NULL | `{"campo": {"antes":.., "despues":..}}` |
| `http_status` | `integer` NULL | |
| `error_detalle` | `text` NULL | |
| `created_at` | `timestamptz` | |

`INDEX(caso_tipo, caso_id)`.

### `svc-privada`
Sin cambios de esquema — `priv_gestiones.id_legacy` ya era `UNIQUE` nullable
desde la migración `0001`; `categoria_id`/`programa_id`/`area_id`/
`ministerio_agencia_id`/`ok_gobernador`/`ok_ministro`/`nro_expediente` ya
existían (E1, migración `0002`, y el esquema original).

## 5. Endpoints

### `POST /internal/privada/gestiones/sync` (svc-privada, nuevo)

Request (`GestionSyncFromVivienda`, `app/internal/schemas.py`):

```
caso_tipo: "cc" | "ch" | "ml"
caso_id: str
id_legacy: str                       # "vivienda:{caso_tipo}:{caso_id}"
nro_expediente: str | null
localidad: str
departamento: str
categoria_id: int | null
programa_id: int | null
area_id: int | null
ministerio_agencia_id: str | null
ok_gobernador: "SI" | "PENDIENTE"
ok_ministro: "SI" | "PENDIENTE"
detalle: str | null                  # si no viene, el server arma un default
```

Response (`GestionSyncResult`):

```
resultado: "LINKED_NEW" | "LINKED_EXISTING" | "PENDING_REVIEW" | "ERROR"
id_legacy: str
gestion_id: str | null
motivo: str | null                   # "GEO_INVALIDO" | "EXPEDIENTE_AMBIGUO" |
                                      # "EXPEDIENTE_YA_VINCULADO_OTRO_CASO" |
                                      # "LOCALIDAD_DEPARTAMENTO_AMBIGUO"
candidatos: [str] | null             # gestion_id ambiguos
diff: {campo: {antes, despues}} | null
```

IAM-only, sin JWT de Firebase. No se agregan endpoints nuevos en svc-vivienda.

## 6. Frontend
Ninguno en esta iteración — la visibilidad es a través del panel de
notificaciones ya existente (`src/modules/notificaciones/`, ADR-019), que no
necesita cambios: recibe estas alertas como cualquier otra.

## 7. Permisos
Ninguno nuevo. La llamada es server-to-server (SA `svc-vivienda@` con
`roles/run.invoker` sobre `svc-privada`, ya otorgado para
`resumen_territorial`/E5a). Los endpoints de escritura existentes en los 3
paneles DGV mantienen su protección actual (`ROLES_ESCRITURA`).

## 8. Riesgos

| # | Riesgo | Mitigación |
|---|---|---|
| R-1 | Colisión localidad+departamento (2+ gestiones candidatas) | Nunca se auto-vincula con ambigüedad — `PENDING_REVIEW`, no se toca nada (§3.4) |
| R-2 | Pisar ediciones manuales de un operador de Privada | Escritura acotada a 7 campos (whitelist, §3.5) |
| R-3 | Gestión huérfana pre-commit — `svc-privada` crea la gestión pero el request de Vivienda hace rollback después por una falla no relacionada | Riesgo bajo; idempotencia vía `id_legacy` — la próxima sincronización de ese caso reclama la misma gestión en vez de duplicarla |
| R-4 | `svc-privada` caído o lento | Best-effort — no bloquea ni rompe el alta/edición de Vivienda (§3.1); un vínculo ya `LINKED` no se degrada (§3.7) |
| R-5 | Ruido de alertas por errores transitorios | Nunca se notifica por `ERROR` (§3.6) — sólo por corrección real o `PENDING_REVIEW` |
| R-6 | ~~`ministerio_agencia_id="MIN_GOBIERNO"` incorrecto contra el catálogo real~~ | Confirmado contra el catálogo real (§9.1) — cerrado |

## 9. Decisiones abiertas / pendientes antes de activar en prod

1. ~~**`ministerio_agencia_id="MIN_GOBIERNO"`**: a confirmar contra el catálogo
   real.~~ **Confirmado 2026-09-14** contra `docs/data/cat_ministerio_agencia.json`
   (volcado de `priv_cat_ministerio_agencia`): `MIN_GOBIERNO` = "Ministerio de
   Gobierno", `activo=true`, `orden=10`. `services/cloudbuild.yaml` enciende
   `PRIVADA_SYNC_GESTIONES_ENABLED=true` por default en cada deploy de CI (mismo
   criterio que `_PRIVADA_FETCH_ENABLED`, ADR-016).
2. **Categoría "Vivienda" (singular) vs "Viviendas" (plural)**: el pedido
   original usaba "Viviendas" para Córdoba Hogar; el catálogo semilla
   `priv_categorias` tiene `"Vivienda"` (singular, id `1756700000001`, migración
   `0002`). Se usó el id existente en vez de crear una categoría casi-duplicada
   — a confirmar con el usuario si prefiere renombrar la categoría semilla.
3. **Ok Gobernador/Ministro cuando `ok_gob != "SI"`**: se mapea a `"PENDIENTE"`
   (nunca `"NO"`) — confirmado con el usuario (reflejar el valor de Vivienda en
   ambos campos de la gestión).
4. `_geo_lookup` en `svc-privada` matchea `(departamento, localidad)` exacto
   por `upper(trim(...))`, sin fold de tildes — un caso de Vivienda cuya
   localidad no esté exactamente en `priv_geo_localidades` cae en `ERROR`
   (`GEO_INVALIDO`) al intentar crear una gestión nueva. No es un riesgo nuevo
   de esta spec (mismo comportamiento que el resto de `svc-privada`), pero
   conviene revisar la cobertura del catálogo antes de activar.

## 10. Criterios de aceptación

- [x] Crear un caso CC con expediente que no matchea ninguna gestión existente
      → se crea una gestión nueva con `categoria_id=1756700000003`,
      `programa_id=1756700001003`, `area_id=1756700002001`, `id_legacy` seteado.
- [x] Crear/editar un caso con expediente que matchea una gestión existente sin
      `id_legacy` → se vincula, se sincronizan los 7 campos derivados, queda un
      diff persistido y una notificación con `destino_valor="privada"`.
- [x] Caso sin expediente, localidad+departamento con 1 sola gestión candidata
      activa → se vincula igual que el punto anterior.
- [x] Ídem con 2+ candidatas → no se crea ni se modifica ninguna gestión;
      `viv_privada_vinculos.estado_vinculo="PENDING_REVIEW"`, se notifica.
- [x] Departamento/localidad inválidos (sólo aplica al crear) →
      `resultado="ERROR"`, `motivo="GEO_INVALIDO"`, el alta del caso en
      Vivienda se completa igual, sin alerta (ruido).
- [x] `svc-privada` no responde (timeout/5xx) → el alta/edición del caso en
      Vivienda se completa igual; fila `ERROR` en `viv_privada_sync_log`, sin
      alerta; un vínculo previamente `LINKED` no se degrada.
- [x] Reintentar la sincronización del mismo caso nunca duplica la gestión.
- [x] La sincronización nunca sobrescribe `nro_expediente` de la gestión si el
      caso de Vivienda no tiene uno.
- [x] La sincronización nunca toca `detalle`/`observaciones`/`urgencia`/`estado`.
- [x] Con el flag `privada_sync_gestiones_enabled=False` (default), no hay
      ningún cambio de comportamiento en los paneles DGV — toda la suite
      existente de `svc-vivienda` pasa sin modificaciones (272 tests).

Cubierto por `services/svc-privada/tests/test_vivienda_sync.py` (10 tests) y
`services/svc-vivienda/tests/test_privada_sync.py` (8 tests, incluye un
end-to-end real vía `POST /api/v1/vivienda/cordon-cuneta`).
