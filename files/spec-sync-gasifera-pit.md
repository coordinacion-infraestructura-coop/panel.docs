# Spec: Sincronización Google Sheet "SEC. GAS PIT" → `svc-gasifera` (Fase 0)

**Estado**: approved
**Versión**: 1.0.0
**Servicio**: `svc-gasifera` (nuevo — solo el módulo de sync en esta entrega, sin panel de negocio)
**Última actualización**: 2026-09-21

---

## Changelog

- **1.0.0** (2026-09-21): aprobado con el alcance descrito (espejo de solo
  lectura, sin reglas de negocio nuevas, sin pantallas — ver §0/§2). Aprobación
  puntual de este spec angosto, **no** sustituye la reunión real con el área ni
  aprueba `docs/files/spec-svc-gasifera.md` (dominio completo, sigue en
  `draft`) — las preguntas abiertas del §10 siguen sin resolver.
- **0.1.0** (2026-09-16): borrador inicial.

## 0. Por qué existe este spec (y por qué es angosto)

Este spec cubre **solo** el espejo de solo lectura del Sheet — mismo alcance y mismo criterio que tuvo `spec-sync-cc-checklist-tecnico.md` para Cordón Cuneta antes de que existiera el panel completo de Checklist Técnico DGV. **No sustituye la reunión real con el área** (pendiente, ver `docs/context/areas/README.md`) ni el spec de dominio completo (`docs/files/spec-svc-gasifera.md`, también draft) — no interpreta reglas de negocio nuevas, no agrega pantallas, solo refleja 1:1 lo que ya existe en el Sheet para poder empezar a construir sobre datos reales en vez de un modelo de escritorio.

## 1. Propósito

La Secretaría de Infraestructura Gasífera (y, más ampliamente, la Secretaría de Infraestructura de la Provincia) lleva su seguimiento de obras en un Google Sheet ("SEC. GAS PIT" — copia local: `docs/context/areas/secretaria_Gasifera/SEC. GAS PIT.xlsx`), uno de ~15 planillas que mantiene el área. El objetivo de esta feature es traer al sistema, mediante sincronización de solo lectura, el subconjunto de **obras de gas** y sus hitos de seguimiento territorial, como primer paso hacia reemplazar esos Sheets — igual que se hizo con Checklist Técnico DGV.

## 2. Alcance

### Incluido
- Lectura vía Google Sheets API v4 de dos pestañas del Sheet real:
  - `MATRIZ (NO TOMAR)`, filtrada a `SUB-TIPO DE OBRA = "E- OBRAS DE GAS"` (17 filas al momento del análisis).
  - `ACCIONES TERRITORIO`, completa (299 filas, ya 100% `Área = "Secretaría Gas"`).
- Upsert idempotente hacia 4 tablas nuevas en `db_gasifera` (base de datos propia, ADR-001).
- Un único endpoint interno IAM-protegido que dispara ambas sincronizaciones en una corrida.
- Log de cada corrida (filas leídas/insertadas/actualizadas/con error, por hoja).

### Fuera de alcance (v0.1.0)
- Escritura hacia el Sheet (unidireccional: Sheet → Postgres).
- Cualquier pantalla o endpoint de negocio (`/api/v1/gasifera/**`) — eso es el spec de dominio completo (`spec-svc-gasifera.md`), que requiere la reunión real con el área.
- `app/auth.py` / integración con `portal_usuarios` — no hace falta en esta fase porque no hay ningún endpoint alcanzable por un usuario final.
- Las otras 4 categorías de obra de `MATRIZ (NO TOMAR)` (vial, agua/cloaca, eléctrica, arquitectura) — pertenecen conceptualmente a una futura `svc-infraestructura`.
- Las otras pestañas del Sheet (`Obras a definir`, `Desplegables`, `monto actualizado `, `CONTROL LEGISLATURA 2026`, `Hoja 11`, `Hoja 13`, `Hoja 17`, `Inauguraciones`, `Anuncios`) — son catálogos, reportes derivados (pivots) o de otras categorías de obra, no fuente de datos de gas.
- Resolución de los problemas de calidad de datos de fondo (SPIP no único, catálogos de departamento/localidad duplicados y no sincronizados dentro del propio Sheet, etc.) — se documentan como preguntas abiertas para la reunión con el área (`contexto_detallado.md`), no se "arreglan" unilateralmente acá.
- Deploy real (Cloud SQL, Cloud Run, Cloud Scheduler, IAM) — se hace en una sesión aparte, explícitamente, con la skill `/deploy-servicio`.

## 3. Fuente de datos

Analizada con `openpyxl` sobre la copia `.xlsx` provista. Ver el análisis completo (hallazgos de calidad de datos, hipótesis de dominio) referenciado desde `docs/context/areas/secretaria_Gasifera/contexto_detallado.md`.

### 3.1 `MATRIZ (NO TOMAR)` (451 filas totales, 17 de gas)

Headers en fila 1, datos desde fila 2. Sin marcador de fin de datos (se lee un rango generoso, `A1:AC500`, y se corta en la primera fila sin `NOMBRE DE OBRA`).

| Columna Sheet | Campo destino | Tipo |
|---|---|---|
| SPIP | `spip` | texto; `"-"` → NULL; ~80 casos que Excel autoconvirtió a fecha → NULL (no se adivina) |
| EXPEDIENTE | `expediente` | texto |
| DIVISIÓN | `division` | texto (ej. "HYG") |
| NOMBRE DE OBRA | `nombre_obra` | texto, NOT NULL |
| T. OBRA | `tipo_obra` | texto libre (ej. "GASODUCTOS") |
| SUB-TIPO DE OBRA | `sub_tipo_obra` | **filtro**: solo se procesa si es exactamente `"E- OBRAS DE GAS"` |
| CONTRATISTA | `contratista` | texto |
| ESTADO DE OBRA | `estado_obra` | enum, normalizado (ver 3.2) |
| LOCALIDAD | → `gas_pit_obras_localidades` (N:M) | texto multivalor, split por `" - "` |
| DEPARTAMENTO | `departamento` | texto |
| AVANCE | `avance` | fracción 0-1 |
| REPLA. INICIAL / FECHA LIC / VENCIMIENTO | fechas | DATE |
| PLAZO VIGENTE EN DÍAS / PLAZO ORIGINAL | enteros | INTEGER |
| CONTRATO BASE / AMPLIACION / ENMIENDA / IMPORTE DE OBRA ACTUALIZADO / IMPORTE EN DÓLAR | montos | NUMERIC(18,2) |
| PRIORIDAD | `prioridad` | enum, normalizado (ver 3.2) |
| CATEGORIA | `categoria` | 1-4 |
| REGION | `region` | enum |
| AUTORIZADA 2025 | `autorizada_2025` | enum |
| PIT | `pit` | booleano (`"PIT"` → true) |
| ESTADO-RESUMEN | `estado_resumen` | enum de 3 valores |

`DEPARTAMENTO MAPA` / `ID-DEPARTAMENTO` **no se sincronizan** — parecen ser un catálogo de apoyo (26 departamentos + id numérico) que conviene resolver contra un catálogo geográfico canónico más adelante (ver 3.3), no duplicarlo.

### 3.2 Calidad de datos observada (el sync debe tolerar esto sin frenarse)

- `SPIP` **no es único** (obras multi-tramo lo repiten) y frecuentemente vacío (`"-"`) → no sirve como clave natural. Tampoco `EXPEDIENTE`. La clave natural del upsert es `(nombre_obra, departamento)` normalizados — más frágil que la de CC (`localidad`+`departamento`), documentado como pregunta abierta para la reunión.
- `ESTADO DE OBRA` trae casi-duplicados por formato: `"PROC. DE ADJUDICACION"` vs `"EN PROCESO DE ADJUDICACIÓN"` → se colapsan al segundo.
- `PRIORIDAD` trae casi-duplicados: `"OBRA CON PRESUPUESTO"` vs `"OBRAS CON PRESUPUESTO"` → se colapsan al segundo.
- `LOCALIDAD` frecuentemente trae 2+ nombres concatenados (`"TANTI - EL DURAZNO"`) → se modela como relación N:M, no como texto plano.

### 3.3 `ACCIONES TERRITORIO` (299 filas, ya 100% gas)

Headers en fila 1, datos desde fila 2, sin marcador de fin — se lee `A1:O1100` y se cortan filas sin `Localidad` ni `Acción`.

| Columna Sheet | Campo destino | Notas |
|---|---|---|
| Fecha | `fecha` | DATE |
| Departamento / Localidad | `departamento` / `localidad` | texto |
| ID_ACCION | `id_accion` | texto |
| Acción | `accion` | texto |
| Detalle de la acción | `detalle_accion` | texto |
| Estado | `estado` | enum `{Cumplido, Pendiente, En ejecución}` — sin normalizar más, valores ya consistentes en el dato real |
| Monto Inversión solicitado | `monto_inversion_solicitado` | numérico |
| Comentarios | `comentarios` | texto |
| Monto Inversión USD | `monto_inversion_usd` | **NO se lee de esta columna** — se recalcula (`monto_inversion_solicitado × settings.tipo_cambio_usd`). La hoja auxiliar "monto actualizado " del Sheet real está acoplada por posición de fila a `ACCIONES TERRITORIO` (mismo `max_row`, sin clave) — antipatrón detectado en el análisis, no se replica |
| ALERTA_LOCALIDAD | `alerta_localidad` | informativo, no bloqueante (valor crudo del Sheet, ya desactualizado en el origen — no se recalcula acá) |
| DEPTO_SUGERIDO | — | no se sincroniza, columna vacía en el 100% de las filas del Sheet real |

Clave natural del upsert: `sheet_row_number` (única opción razonable — no hay combinación de columnas de negocio que garantice unicidad; una localidad puede tener varias acciones en fechas distintas). **Riesgo aceptado**: sensible a que el área reordene filas manualmente en el Sheet; documentado, no resuelto en v0.1.0.

## 4. Decisiones de arquitectura (confirmadas con el usuario)

1. **Sin Google Apps Script ni script local de un solo uso.** Toda la lógica vive en un servicio real, `svc-gasifera` — mismo patrón que el sync de CC, no un script ad-hoc (corrección hecha durante el planning: la primera versión de este plan proponía un script local; el precedente real del repo es un servicio en Cloud Run desde el día uno).
2. **Autenticación al Sheet**: ADC (`google.auth.default()`), scope `spreadsheets.readonly`. En producción, Service Account de runtime de `svc-gasifera` (a crear en el deploy real). En desarrollo local, la cuenta del usuario vía `gcloud auth application-default login`.
3. **Disparo**: en producción, Cloud Scheduler vía HTTP + token OIDC (a configurar en el deploy real, fuera de esta entrega). En desarrollo, `curl` manual.
4. **Endpoint interno** en el mismo servicio HTTP — el volumen (~316 filas) no justifica un Cloud Run Job separado.
5. **Autenticación del endpoint**: `POST /internal/sync/gasifera-pit` **no se agrega a `infra/gateway/openapi.yaml`**. En producción, el único control de acceso es IAM de Cloud Run (`--no-allow-unauthenticated` + `roles/run.invoker`). El endpoint no usa `Depends(get_current_user)` — de hecho, `svc-gasifera` en esta fase **no tiene `app/auth.py`** en absoluto, porque no hay ningún endpoint alcanzable por un usuario final.
6. **Escritura idempotente**: `SELECT` por clave natural → mutar o crear → `flush`, con `SAVEPOINT` por fila (`async with db.begin_nested()`) desde el primer commit — aplicando directamente la lección del incidente de producción de CC (`spec-sync-cc-checklist-tecnico.md §13.5`), no como fix posterior.
7. **Logging**: cada corrida registra contadores y detalle de errores (con la hoja de origen) en `gas_pit_sync_log`, sin abortar el batch por errores de fila individuales.

## 5. Modelo de datos

Todas las tablas nuevas, prefijo `gas_pit_`, migración Alembic `0001` (primera migración del servicio nuevo).

### 5.1 `gas_pit_obras`
Snapshot actual (no histórico) de cada obra de gas. Ver `services/svc-gasifera/app/gas_pit/models.py` para el detalle completo de columnas — resumen de las más relevantes: `spip`, `expediente`, `nombre_obra` (+`nombre_obra_norm`/`departamento_norm` para la clave del upsert), `sub_tipo_obra` (siempre `"E- OBRAS DE GAS"` en esta fase), `estado_obra`, `estado_resumen`, `avance`, montos (`contrato_base`/`ampliacion`/`enmienda`/`importe_obra_actualizado`/`importe_dolar`), `prioridad`, `categoria`, `region`, `pit`, `sheet_row_number`, `last_synced_at`.

Índice único (clave natural del upsert): `(nombre_obra_norm, departamento_norm)`.

### 5.2 `gas_pit_obras_localidades`
Relación N:M obra↔localidad (`obra_id`, `localidad`), `UNIQUE(obra_id, localidad)`. Estrategia de upsert: `DELETE` + reinsert de las localidades vigentes en cada corrida (mismo patrón que `viv_cc_checklist_items`).

### 5.3 `gas_pit_acciones_territorio`
Una fila por hito/acción territorial. Columnas: `fecha`, `departamento`, `localidad`, `id_accion`, `accion`, `detalle_accion`, `estado`, `monto_inversion_solicitado`, `comentarios`, `monto_inversion_usd` (recalculado, ver 3.3), `alerta_localidad`, `sheet_row_number` (UNIQUE), `last_synced_at`.

### 5.4 `gas_pit_sync_log`
Una fila por corrida, cubriendo ambas hojas: `started_at`, `finished_at`, `filas_leidas`, `filas_insertadas`, `filas_actualizadas`, `filas_error`, `errores` (texto JSON, `[{"fila", "hoja", "motivo"}]`), `triggered_by`.

## 6. Endpoint interno

```
POST /internal/sync/gasifera-pit
GET  /internal/sync/gasifera-pit/estado
```
Router `app/internal/router.py`, montado en `main.py` sin prefijo `/api/v1` y sin `Depends(get_current_user)`. No se declara en `infra/gateway/openapi.yaml`.

## 7. Cliente de Google Sheets

`app/integrations/google_sheets.py` — mismo cliente mínimo que `svc-vivienda` (no se comparte código entre servicios; database-per-service implica también deploy independiente). `spreadsheet_id` y los dos rangos van en `config.py` (`Settings`), no hardcodeados.

## 8. Tests

- `tests/test_gas_pit_sync.py`: mock de `google_sheets.get_values` (responde según el rango pedido, no una lista fija — soporta correr `sync_from_sheet()` más de una vez en el mismo test de idempotencia). Cubre: filtro por `SUB-TIPO DE OBRA`, normalización de enums casi-duplicados, `SPIP` ambiguo → NULL, split de localidades multivalor, idempotencia (UPSERT), aislamiento por `SAVEPOINT` (regresión forzando un `IntegrityError` real en la fila del medio de un batch), falla total de lectura → `SheetReadError` + log.
- `tests/test_internal_router.py`: el endpoint responde 200 con el resumen de la corrida y no requiere JWT; 502 si falla la lectura del Sheet.

## 9. Criterios de aceptación (esta entrega)

- [x] Servicio `svc-gasifera` nuevo, scaffold completo (`pyproject.toml`, `app/`, `alembic/`, `tests/`, `docker-compose.dev.yml`, `Dockerfile`, `README.md`).
- [x] Migración `0001` crea las 4 tablas nuevas.
- [x] `pytest` en verde sobre SQLite in-memory (13/13), sin requerir Postgres.
- [x] `POST /internal/sync/gasifera-pit` no usa `Depends(get_current_user)`; no se declara en `openapi.yaml`.
- [x] Corridas repetidas no duplican filas (UPSERT verificado en tests).
- [x] Una fila con error real de DB no frena el resto del batch ni impide el log final (test de regresión con `IntegrityError` forzado).
- [x] Migración `0001` verificada contra Postgres real (Docker local, `docker-compose.dev.yml`): `upgrade head` / `downgrade base` / `upgrade head` limpios, tablas y constraints (`uq_gas_pit_obra_nombre_depto`, `uq_gas_pit_obra_localidad`, `sheet_row_number` UNIQUE, FK con `ON DELETE CASCADE`) verificadas con `\d`.
- [x] Deploy real a Cloud Run (2026-09-21, build manual `gcloud builds submit`, revisión `svc-gasifera-00001-9zv`, 100% tráfico, `Ready=True`). Infra (SA `svc-gasifera@`, roles IAM, `db_gasifera`, `user_gasifera`, secret `svc-gasifera-db-url`) ya estaba pre-provisionada de una corrida anterior — solo faltó correr la migración `0001` contra Cloud SQL real (vía `cloud-sql-proxy` local) y el build/deploy. Servicio en `https://svc-gasifera-iwni7vc2qq-rj.a.run.app`, `--no-allow-unauthenticated` (IAM-only, verificado: llamadas sin `roles/run.invoker` devuelven 401). **No se otorgó `run.invoker` a nadie todavía** (ni a `api-gateway-sa` — no aplica, no hay paths en el Gateway en esta fase — ni a Cloud Scheduler — no configurado todavía) — el servicio está desplegado pero no hay ningún caller autorizado a disparar el sync en producción hasta el próximo paso.
- [ ] Corrida real contra el Sheet en vivo — **bloqueada en el equipo local** por política de organización de GCP (OAuth de apps no verificadas + impersonation de Service Accounts, ambas restringidas para `infraestructura.coop@gmail.com`/`pedrobonafe.data@gmail.com`). El Sheet ya está compartido como Viewer con `svc-gasifera@gestorcooperativo.iam.gserviceaccount.com`, así que una vez que exista un caller autorizado (Cloud Scheduler u otro) debería funcionar sin este problema (la SA usa su identidad nativa en Cloud Run, no impersonation).
- [ ] Cloud Scheduler — no configurado todavía (pendiente, ver §11).
- [ ] `infra/gateway/openapi.yaml` — no aplica en esta fase (endpoint IAM-only, sin exposición pública).

## 11. Pendiente inmediato tras el deploy

- Otorgar `roles/run.invoker` sobre `svc-gasifera` a quien vaya a disparar el sync (Cloud Scheduler con su propia SA, siguiendo el patrón de `sync-cc-checklist-tecnico` — ver `spec-sync-cc-checklist-tecnico.md §9`) — no se hizo en esta sesión porque requiere una acción de otorgamiento de permisos IAM que el harness bloqueó por precaución; queda para una sesión donde el usuario la autorice explícitamente paso a paso.
- Con eso configurado, correr una primera sincronización real y comparar conteos contra el análisis manual (17 obras de gas + 299 acciones territoriales).

## 10. Pendiente / preguntas abiertas para la reunión con el área

Ver `docs/context/areas/secretaria_Gasifera/contexto_detallado.md` §7 para la lista completa. Las más relevantes para este sync en particular:
- ¿Qué es `SPIP` realmente y por qué no es único? ¿Hay una clave de negocio mejor que `(nombre_obra, departamento)`?
- ¿El área reordena manualmente las filas de `ACCIONES TERRITORIO`? (afecta la estabilidad de la clave `sheet_row_number`).
- ¿Cuál de los dos catálogos de Departamento/Localidad de la hoja `Desplegables` es el vigente?
