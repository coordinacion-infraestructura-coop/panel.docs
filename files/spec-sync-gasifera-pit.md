# Spec: Sincronización Google Sheet "SEC. GAS PIT" → `svc-gasifera` (Fase 0)

**Estado**: approved
**Versión**: 1.7.0
**Servicio**: `svc-gasifera` (sync + panel preliminar de solo lectura + rollup territorial, sin panel de negocio)
**Última actualización**: 2026-09-23

---

## Changelog

- **1.7.0** (2026-09-23): agrega §16 — notificación a svc-vivienda cuando el
  sync detecta una acción territorial NUEVA (ADR-023). Ver §16.
- **1.6.0** (2026-09-23): **bug real encontrado y corregido** —
  `monto_inversion_usd` (`app/gas_pit/sync.py`) **multiplicaba**
  `monto_inversion_solicitado` por `settings.tipo_cambio_usd` (1460) en vez de
  **dividir**, desde la Fase 0 original. Pasó desapercibido porque nadie había
  mirado ese campo con datos reales hasta federarlo a `resumen_territorial`
  (ADR-021) — ahí saltaron montos "USD" en billones. Confirmado con datos
  reales: `monto_inversion_solicitado` está en ARS (mismo orden de magnitud
  que `importe_obra_actualizado` de `gas_pit_obras`, ej. obras de ~1.900
  millones de ARS) — dividir por 1460 da cifras de USD plausibles (ej. la
  mayor pasó de "2.48 billones" a ~1.16 millones de USD). El test
  correspondiente (`test_sync_inserta_accion_territorio_y_calcula_usd`)
  codificaba el cálculo erróneo como si fuera correcto — también corregido.
  Redeployado (`svc-gasifera-00005-svw`), sync real re-corrido para recalcular
  las 296 filas con monto, `resumen_territorial` recomputado con los valores
  correctos. **Lección**: un test que sólo verifica "la fórmula que escribí
  hace lo que escribí" no detecta un error de signo/dirección — hace falta
  contrastar contra una magnitud de referencia independiente (acá,
  `gas_pit_obras`) al menos una vez con datos reales.
- **1.5.0** (2026-09-23): agrega §15 — endpoint interno
  `GET /internal/gasifera/rollup-territorial`, consumido por `resumen_territorial`
  de `svc-vivienda` (ADR-021, mismo patrón que ADR-016 usó para Privada). Ver
  §15 para el detalle completo.
- **1.4.0** (2026-09-23): **incidente encontrado y resuelto** — la sesión en
  paralelo que trabajó `svc-gralgob` generó una config de Gateway nueva
  (`ministerio-config-v20260922`) a partir de una copia de
  `infra/gateway/openapi.yaml` que **no tenía** los paths de `/api/v1/gasifera/**`
  (habían quedado sin commitear en `infra/` desde §12.5). Esa config quedó
  activa, dejando el panel de Gasífera respondiendo `404` en producción sin
  que nadie lo notara hasta que el usuario pidió confirmar el estado del
  deploy. Fix: se commitearon los paths de gasifera a `infra/` (commit
  `195e735`, sin tocar los de gralgob, ya commiteados aparte), se generó
  `ministerio-config-v20260923` desde el archivo ya completo (gasifera +
  gralgob) y se actualizó el gateway — verificado con `curl` que los 3 paths
  de gasifera vuelven a dar `401` (no `404`) y que gralgob/vivienda no se
  rompieron. **Lección**: las configs de Gateway no son incrementales — cada
  config nueva se genera del `openapi.yaml` completo en ese momento, así que
  cualquier cambio sin commitear/pushear a `infra/` se pierde en el próximo
  deploy de gateway que haga *cualquier* sesión, no solo la propia.

- **1.3.0** (2026-09-23): Cloud Scheduler configurado (`sync-gasifera-pit`, cada
  hora, self-invoke OIDC) — el sync deja de ser manual. Hicieron falta 2 grants
  IAM: `roles/run.invoker` de `svc-gasifera@` sobre sí mismo, y
  `roles/iam.serviceAccountTokenCreator` sobre esa misma SA para el agente de
  servicio de Cloud Scheduler (`service-276787280674@gcp-sa-cloudscheduler.iam.gserviceaccount.com`)
  — sin el segundo, la corrida falla en runtime con 403
  (`run.routes.invoke` denegado) aunque el invoker esté bien puesto; mismo
  gotcha ya documentado para `svc-gralgob`. Verificado end-to-end: disparo
  manual del job → `200 OK` en los logs de Cloud Run, próxima corrida
  programada sola. Ver §11.
- **1.2.0** (2026-09-21): primera corrida real contra el Sheet en vivo, exitosa
  (316 filas leídas = 17 obras + 299 acciones, 0 errores — ver §9). Se agregan
  `ministerio` y `area` a `gas_pit_acciones_territorio` (migración `0002`) y se
  rediseña la tabla de acciones del panel siguiendo el esquema de
  vivienda (CC/CH: filtros por Departamento/Localidad/Estado + tabla), a pedido
  del usuario tras ver la primera versión (KPI + tablas planas) — no era
  práctica para trabajar con el dataset real. Ver §12.7.
- **1.1.0** (2026-09-21): agrega §12 — panel de visualización de solo lectura
  (3 endpoints públicos + página frontend) sobre los datos ya sincronizados.
  **Excepción explícita y documentada** a la regla de `CLAUDE.md` agregada el
  mismo día ("no agregar `/api/v1/gasifera/**` sin `spec-svc-gasifera.md`
  `approved`") — autorizada expresamente por el usuario, acotada a endpoints
  de solo lectura sin lógica de negocio nueva. Ver §12 para el detalle y el
  razonamiento completo.
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
- [x] Corrida real contra el Sheet en vivo (2026-09-21, disparada manualmente desde Cloud Shell con `gcloud auth print-identity-token` + `curl`, usando el `roles/run.invoker` otorgado a `infraestructura.coop@gmail.com` sobre `svc-gasifera`): `{"filas_leidas":316,"filas_insertadas":0,"filas_actualizadas":316,"filas_error":0,"errores":[]}` — **316 = 17 obras + 299 acciones, coincide exacto con el análisis manual del Excel**. Bloqueante real resuelto: el Sheet no estaba compartido con la SA correcta (`svc-gasifera@gestorcooperativo.iam.gserviceaccount.com`) — el primer intento devolvió 403 de la API de Sheets hasta compartirlo bien. La restricción de OAuth/impersonation del equipo local (§11, resuelta como no-bloqueante) nunca aplicó a Cloud Run en sí, solo a probar localmente.
- [ ] Cloud Scheduler — no configurado todavía. El sync corre bien manualmente pero no está automatizado (pendiente, ver §11).
- [x] `infra/gateway/openapi.yaml` — **sí aplica**: el alcance creció en v1.1.0/§12 (panel de solo lectura), que sí expone 3 paths públicos. Agregados y verificados (`ministerio-config-v20260921`, 401 no 404 sin token).

## 11. Pendiente

- ~~Cloud Scheduler~~ — **hecho (2026-09-23)**, ver §13.
- Backfill de `ministerio`/`area` en filas ya sincronizadas: no hace falta acción manual — la próxima corrida del sync los completa solo (UPSERT), ya verificado (316/316 actualizadas en la corrida del §9).
- Sigue pendiente: `SPIP` sin respuesta del área, y la segunda reunión para completar `contexto_detallado.md` y poder aprobar `spec-svc-gasifera.md`.

## 13. Cloud Scheduler (agregado 2026-09-23, v1.3.0)

Mismo patrón self-invoke que `sync-cc-checklist-tecnico` (`spec-sync-cc-checklist-tecnico.md §9`) y `sync-atp-compromiso-gobernador`:

```bash
# 1) svc-gasifera@ necesita invoker sobre sí mismo (self-invoke)
gcloud run services add-iam-policy-binding svc-gasifera \
  --region=southamerica-east1 --project=gestorcooperativo \
  --member="serviceAccount:svc-gasifera@gestorcooperativo.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# 2) el agente de servicio de Cloud Scheduler necesita poder emitir tokens
#    en nombre de esa SA — sin esto, la corrida falla en runtime con 403
#    (run.routes.invoke denegado) aunque el paso 1 esté bien hecho.
gcloud iam service-accounts add-iam-policy-binding \
  svc-gasifera@gestorcooperativo.iam.gserviceaccount.com \
  --member="serviceAccount:service-276787280674@gcp-sa-cloudscheduler.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator" \
  --project=gestorcooperativo

# 3) el job en sí
gcloud scheduler jobs create http sync-gasifera-pit \
  --location=southamerica-east1 \
  --schedule="0 * * * *" \
  --uri="https://svc-gasifera-iwni7vc2qq-rj.a.run.app/internal/sync/gasifera-pit" \
  --http-method=POST \
  --oidc-service-account-email="svc-gasifera@gestorcooperativo.iam.gserviceaccount.com" \
  --oidc-token-audience="https://svc-gasifera-iwni7vc2qq-rj.a.run.app" \
  --project=gestorcooperativo
```

Corre cada hora en punto (`0 * * * *`). Verificado con `gcloud scheduler jobs run sync-gasifera-pit` + logs de Cloud Run (`POST /internal/sync/gasifera-pit HTTP/1.1 200 OK`) — el primer disparo, inmediatamente después de otorgar los permisos, falló con 403 por demora de propagación de IAM (~60-90s); el segundo intento fue exitoso. `gcloud scheduler jobs describe sync-gasifera-pit` confirma `status.code` vacío (éxito) y la próxima corrida programada.

## 10. Pendiente / preguntas abiertas para la reunión con el área

Ver `docs/context/areas/secretaria_Gasifera/contexto_detallado.md` §7 para la lista completa. Actualizado tras la primera reunión (2026-09-21):
- **⏳ sin resolver**: ¿qué es `SPIP` realmente y por qué no es único? ¿Hay una clave de negocio mejor que `(nombre_obra, departamento)`? (se preguntó, el usuario va a re-consultar).
- ⏳ sin resolver: ¿el área reordena manualmente las filas de `ACCIONES TERRITORIO`? (afecta la estabilidad de la clave `sheet_row_number`).
- **✅ resuelto**: el catálogo de Departamento/Localidad vigente es `geo_localidades`/`info_localidades` (catálogos ya existentes del sistema), no ninguno de los dos que trae la hoja `Desplegables` del Sheet. No se integra todavía (ver §12).
- **✅ resuelto**: alcance confirmado como solo obras de gas por ahora (ver `contexto_detallado.md §2`).

## 12. Panel de visualización de solo lectura (agregado 2026-09-21, v1.1.0)

### 12.1 Por qué esto es una excepción documentada, no una violación silenciosa

El mismo día (2026-09-21), otra sesión de trabajo en paralelo agregó a `CLAUDE.md` (raíz) la regla: *"Don't add `/api/v1/gasifera/**` ... endpoints without that domain spec [`spec-svc-gasifera.md`] going `approved` first."* Al descubrir el conflicto, se le señaló explícitamente al usuario (siguiendo el hard rule de Spec Driven Development: señalar conflictos con specs/ADRs antes de implementar). El usuario, con conocimiento del conflicto, **decidió explícitamente anular esa regla para este caso puntual**: endpoints de **solo lectura**, sobre datos que **ya están sincronizados** (`gas_pit_*`, ya cubiertos por este spec `approved`), **sin interpretar ninguna regla de negocio nueva** — mismo criterio de acotamiento que ya usa este spec para el sync en sí (§0). No autoriza ningún endpoint de escritura, catálogo administrable, ni nada que dependa de las preguntas todavía abiertas del §10.

### 12.2 Alcance

**Incluido**: 3 endpoints GET públicos (vía API Gateway, con auth JWT estándar) que leen `gas_pit_obras`, `gas_pit_obras_localidades` y `gas_pit_acciones_territorio` tal cual están, más una página frontend que los muestra. Sin filtros de negocio nuevos, sin cálculos más allá de KPIs simples (conteos, sumas, promedios) sobre los mismos datos.

**Fuera de alcance**: cualquier escritura, el panel de negocio completo (`spec-svc-gasifera.md`, sigue `draft`), resolución de localidad/departamento contra el catálogo canónico `geo_localidades`/`info_localidades` (se muestra el texto crudo del Sheet — ver `contexto_detallado.md §7.2`), gráficos/mapas (decisión explícita del usuario, primera versión solo KPI strip + tablas).

### 12.3 Auth (nuevo — Fase 0 no tenía ningún endpoint público)

`app/auth.py` nuevo, modelado sobre `services/svc-privada/app/auth.py` (ADR-015) — `svc-gasifera` **no se conecta a `db_vivienda`**, resuelve rol + secretarías llamando a `GET {SVC_VIVIENDA_INTERNAL_URL}/internal/portal/usuarios/{email}` con un ID token (audience = esa URL), degradando a rol `invitado` ante cualquier falla (nunca 500). Roles: `ROLES_LECTURA = ("Admin","Supervisor","Operador","Consulta")` + pertenencia a la secretaría `"gasifera"` (ya presente en `SECRETARIAS_VALIDAS`, no requiere migración). Sin roles acotados nuevos tipo `TecnicoDGV`/`Autoridad`.

### 12.4 Endpoints

```
GET /api/v1/gasifera/obras?limit=200&offset=0
GET /api/v1/gasifera/acciones-territorio?limit=500&offset=0
GET /api/v1/gasifera/sync-estado
```
Router nuevo `app/gas_pit/router.py`, montado con prefijo `/api/v1/gasifera` en `main.py`. Los 3 requieren `Depends(get_current_user)` + rol de lectura + secretaría `gasifera`. `sync-estado` envuelve `gas_pit_sync.get_last_sync_status` (mismo dato que el endpoint interno de sync, expuesto de forma pública y de solo lectura para mostrar "Sincronizado hace X" en el panel).

### 12.5 Gateway e IAM (deploy real, pasos separados — ver criterios de aceptación)

Se agregan a `infra/gateway/openapi.yaml` (patrón idéntico a `/api/v1/vivienda/checklist-tecnico/catalogos`) + nueva config de gateway. Dos grants IAM nuevos: `svc-gasifera@` necesita `roles/run.invoker` sobre `svc-vivienda` (para el lookup de auth); `api-gateway-sa@` necesita `roles/run.invoker` sobre `svc-gasifera` (para poder enrutarle tráfico).

### 12.6 Criterios de aceptación

- [x] `app/auth.py` + tests — 21/21 en verde (`tests/test_gas_pit_router.py`): 200 con datos, 403 sin secretaría `gasifera`, 403 `invitado`, 401 sin token (vía dependency-override para los positivos, y un cliente sin override para el 401 real de JWT).
- [x] `app/gas_pit/router.py` con los 3 endpoints + `response_model`.
- [x] Frontend: `src/modules/gasifera/` (API client + `GasiferaPitPage.tsx`), activación en `DashboardPage.tsx`/`Layout.tsx`/`App.tsx`.
- [x] Redeploy de `svc-gasifera` con el código nuevo (revisión `svc-gasifera-00003-5lt` al cierre de esta entrega, tras el redeploy de §12.7).
- [x] IAM: los 2 `run.invoker` otorgados por el usuario (`svc-gasifera@`→`svc-vivienda`, `api-gateway-sa@`→`svc-gasifera`) — confirmados en la política IAM real.
- [x] Gateway actualizado con los 3 paths nuevos, config `ministerio-config-v20260921` activa (verificado: 401 no 404 en `/api/v1/gasifera/sync-estado` sin token).
- [x] Frontend deployado a producción (`gestorcooperativo.web.app`), 2 veces (versión inicial + rediseño de §12.7).
- [x] Verificación end-to-end en navegador por el usuario: login, panel visible con datos reales (316 filas). **No verificado por separado**: el 403 para un usuario sin la secretaría `gasifera` asignada — cubierto solo por el test automatizado (`test_sin_secretaria_gasifera_devuelve_403`), no repetido a mano en el navegador.

### 12.7 Rediseño de la tabla de acciones (2026-09-21, tras feedback del usuario)

La primera versión del panel (KPI strip + 2 tablas planas, sin filtros) resultó
poco práctica para trabajar con el dataset real, según el usuario tras probarlo
en producción. Pedido explícito: "más similar al excel", usando **vivienda
(CC/CH)** como modelo — filtros arriba + tabla, no gráficos.

- **Backend**: se agregan `ministerio`/`area` a `gas_pit_acciones_territorio`
  (migración `0002`, columnas nullable) — existían en el Sheet (`ACCIONES
  TERRITORIO`) pero la Fase 0 original no las persistía. `sync.py` las
  completa desde las columnas "Ministerio"/"Área"; el backfill de las 299
  filas ya sincronizadas fue automático (UPSERT) en la corrida siguiente, sin
  script aparte.
- **Frontend**: `GasiferaPitPage.tsx` rediseñada — la tabla de **acciones de
  seguimiento territorial** pasa a ser la información principal (antes era
  secundaria respecto a "obras"), con filtro bar (Departamento/Localidad/Estado,
  `<select>` con opciones derivadas de los datos vía `useMemo`, botón "Limpiar
  filtros") + tabla con columnas exactas: Fecha, Departamento, Localidad,
  Ministerio, Área, Acción, Detalle de la acción, Estado, Monto solicitado,
  Comentarios, Monto USD — mismo patrón visual (header navy, `font-size: 12px`,
  badges de estado) que `CordonCunetaPage.tsx`/`CordobaHogarPage.tsx`. La tabla
  de obras se mantiene debajo, sin cambios funcionales.
- Sin cambios de alcance respecto a §12.1/§12.2 — sigue siendo puramente
  visualización de datos ya sincronizados, cero lógica de negocio nueva (los
  filtros son client-side sobre datos ya traídos, no nuevos endpoints).

## 15. Rollup territorial — federación a `resumen_territorial` (agregado 2026-09-23, v1.5.0, ADR-021)

### 15.1 Endpoint

```
GET /internal/gasifera/rollup-territorial
```
`app/internal/router.py`, IAM-only (sin `Depends(get_current_user)`, no declarado en `infra/gateway/openapi.yaml`) — mismo criterio que el resto de este router. Consumido exclusivamente por `svc-vivienda` (`app/resumen_territorial/service.py::fetch_gasifera_lineas`), nunca por el frontend.

### 15.2 Qué agrega

`app/gas_pit/rollup.py::rollup_territorial(db)` — `GROUP BY UPPER(TRIM(departamento)), UPPER(TRIM(localidad))` sobre `gas_pit_acciones_territorio` (calcado de `services/svc-privada/app/gestiones/service.py::rollup_territorial`). **No incluye `gas_pit_obras`** — mismo alcance acotado que el rollup de Privada, que tampoco trae todas sus tablas, solo `gestiones`. Por fila: `departamento`, `localidad`, `total_acciones`, `cumplidas` (`estado = "CUMPLIDO"`), `en_curso` (resto), `monto_solicitado_sum`, `monto_usd_sum`, `fecha_max`.

### 15.3 Por qué texto normalizado y no un id de catálogo geográfico

Investigado antes de implementar (no asumido): **ni Vivienda ni Privada usan hoy `id_geo` como llave de join** para el matching territorial — todo el pipeline existente (`app/geo/matching.normalize_name` del lado Vivienda, `UPPER(TRIM(...))` en SQL del lado Privada) agrupa por texto normalizado `(departamento, localidad)`. Replicar "el mismo patrón" significa seguir ese criterio tal cual está hoy, no introducir una resolución por id que el resto del sistema tampoco tiene — eso queda fuera de alcance de este cambio (ver ADR-021 "Razón").

### 15.4 IAM

`roles/run.invoker` de `svc-vivienda@gestorcooperativo.iam.gserviceaccount.com` sobre el servicio Cloud Run `svc-gasifera` — dirección **inversa** al grant de ADR-015 (`svc-gasifera@` invoker sobre `svc-vivienda`, para que Gasífera resuelva `portal_usuarios`). Los dos grants coexisten sin conflicto (son sobre servicios distintos).

### 15.5 Criterios de aceptación

- [x] `app/gas_pit/rollup.py` + endpoint interno + tests (`tests/test_gas_pit_rollup.py`, 6 tests: agregación, normalización de estado, vacío, fecha máxima, endpoint sin JWT).
- [x] `svc-vivienda`: `fetch_gasifera_lineas`/`_map_gasifera_payload` (calcados de los de Privada) + `resumen_gasifera_estado`/`detalle_gasifera` en `aggregations.py` + tests (`tests/test_resumen_territorial.py`, 5 tests nuevos) — suite completa de `svc-vivienda` (294 tests) sigue en verde.
- [x] Frontend `ResumenTerritorialPage.tsx`: badge de área `gasifera` (label + color propio, `gov-orange`).
- [ ] IAM real otorgado (`svc-vivienda@` invoker sobre `svc-gasifera`) — pendiente, requiere confirmación antes de ejecutar (infra real).
- [ ] `services/cloudbuild.yaml`: sustituciones `_GASIFERA_FETCH_ENABLED`/`_SVC_GASIFERA_INTERNAL_URL` — pendiente.
- [ ] Redeploy de `svc-gasifera` (endpoint nuevo) y `svc-vivienda` (fetch nuevo) — pendiente, el de `svc-vivienda` es el de mayor riesgo por ser el servicio productivo principal.
- [ ] Verificación end-to-end: una localidad con obras/acciones de gas reales aparece en el snapshot de Resumen Territorial con `area: "gasifera"`, respetando visibilidad por secretaría.

## 16. Notificación a svc-vivienda cuando hay una acción territorial nueva (agregado 2026-09-23, v1.7.0, ADR-023)

### 16.1 Qué dispara la alerta

Cada vez que `sync_from_sheet` detecta una fila **nueva** en la hoja "ACCIONES TERRITORIO" (`is_new = True` en `_upsert_accion` — no una fila que ya existía y se actualizó), se llama, después de loguear la corrida en `GasPitSyncLog` y fuera del `SAVEPOINT` por fila, a `POST /internal/notificaciones` de `svc-vivienda` (ADR-019) con:

```
"La localidad {localidad}, {departamento} sumó obra de gas {accion}."
```

`destino_tipo="secretaria"`, `destino_valor="gasifera"`, `nivel="info"`, `origen="gas_pit_sync"`. Si falta `localidad`/`departamento`/`accion`, se sustituye por `"(sin localidad)"`/`"(sin departamento)"`/`"(sin detalle)"` — nunca se omite el envío por datos parciales. **Sólo la hoja "ACCIONES TERRITORIO" dispara esta alerta** — una obra nueva en "MATRIZ (NO TOMAR)" (`gas_pit_obras`) no notifica, porque el campo "acción" pedido no existe en esa tabla (`gas_pit_obras` no tiene un concepto de "acción", sólo estado/avance de obra).

### 16.2 Por qué después del loop, no dentro del SAVEPOINT

Atar la llamada HTTP a `svc-vivienda` a la transacción de cada fila arriesgaría: (a) sumarle latencia de red al batch completo, fila por fila; (b) que un fallo de red dispare el `except Exception` del loop y la fila se registre como error aunque el upsert haya sido válido. Se acumulan las acciones nuevas en una lista (`localidad`, `departamento`, `accion`) durante el loop de ACCIONES TERRITORIO y se notifican todas al final, ya con el log de sync persistido.

### 16.3 Implementación

`app/integrations/notificaciones_vivienda.py::notificar_accion_nueva(...)` — best-effort, nunca lanza (mismo criterio que el resto de las integraciones salientes del proyecto). Reusa `settings.svc_vivienda_internal_url` (ya sembrada para el lookup de `portal_usuarios`, ADR-015) — no hace falta una URL nueva. Gate propio: `settings.notificar_fila_nueva_enabled` (default `False` en `config.py`, `True` en prod vía `services/cloudbuild.yaml` — sustitución `_NOTIFICAR_FILA_NUEVA_ENABLED`, compartida con ATP/gralgob porque ambos servicios leen la misma env var). ID token de la SA de runtime, audiencia = URL base de `svc-vivienda` — mismo patrón que `app/auth.py::_fetch_portal_user`, pero en sentido inverso: acá `svc-gasifera` es quien llama.

### 16.4 IAM

Mismo grant que ya usa ADR-015 (`svc-gasifera@gestorcooperativo.iam.gserviceaccount.com` con `roles/run.invoker` sobre `svc-vivienda`) — no es un permiso nuevo, el mismo grant habilita tanto `GET /internal/portal/usuarios/{email}` como este `POST /internal/notificaciones`, porque ambos apuntan al mismo servicio destino.

### 16.5 Criterios de aceptación

- [x] `app/integrations/notificaciones_vivienda.py` + wiring en `app/gas_pit/sync.py::sync_from_sheet` (sólo en el loop de ACCIONES TERRITORIO).
- [x] Tests (`tests/test_gas_pit_notificaciones.py`, 4): acción nueva notifica con localidad/acción correctas; una acción ya existente no vuelve a notificar en una corrida posterior; una obra nueva en MATRIZ no dispara esta alerta; con el flag apagado no se instancia el cliente HTTP. Suite completa de `svc-gasifera` (30 tests) en verde.
- [x] `services/cloudbuild.yaml`: sustitución `_NOTIFICAR_FILA_NUEVA_ENABLED` (compartida con ATP/gralgob).
- [ ] Verificación end-to-end en producción (corrida real de sync con una acción nueva + confirmación visual de la alerta en el panel de notificaciones) — pendiente, análoga a la ya hecha para la vinculación Privada (`spec-vinculacion-vivienda-privada.md`).
