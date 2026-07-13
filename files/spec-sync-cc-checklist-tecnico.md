# Spec: Sincronización Google Sheet "Base TOTAL" → Checklist Técnico de Cordón Cuneta

**Estado**: ✅ implementado y en producción (2026-07-13)
**Versión**: 1.0.0
**Servicio**: `svc-vivienda` (módulo `cordon_cuneta`, sin servicio nuevo)
**Última actualización**: 2026-07-13

## 0. Resultado de la primera corrida en producción

- Migración `0016` aplicada. Gateway: `ministerio-config-v20260713` (activa).
- Cloud Scheduler `sync-cc-checklist-tecnico` corriendo cada 15 min (`southamerica-east1`).
- Primera corrida real: `filas_leidas=48`, `filas_insertadas=46`, `filas_error=2`.
- **100% de matching** (`municipio_id`) sobre las 46 filas válidas — supera el mínimo del 80% del criterio de aceptación.
- Los 2 errores son esperados y persistentes hasta que el área técnica los corrija en el Sheet: "Deán Funes" y "La Laguna", altas nuevas sin `Departamento` cargado todavía (ver 3.2). **No indican una falla del sync** — si en `viv_cc_sync_log` aparecen siempre estos mismos dos, es correcto.

---

## 1. Propósito

El área técnica de Cordón Cuneta (Dirección General de Vivienda) lleva su seguimiento documental en un Google Sheet propio ("CORDÓN CUNETA Y ADOQUINADO", pestaña **"Base TOTAL"**), independiente del panel `viv_cordon_cuneta` que ya está en producción. No es viable migrarlos a nuestro sistema en el corto plazo. El objetivo de esta feature es **traer esa información al servicio** mediante sincronización periódica de solo lectura, para que los cargos jerárquicos puedan ver el avance del checklist técnico y las comunicaciones del área técnica **desde el panel existente**, sin que el área técnica cambie su forma de trabajo.

## 2. Alcance

### Incluido
- Lectura periódica (Cloud Scheduler, cada 15 min) de la pestaña "Base TOTAL" vía Google Sheets API v4.
- Upsert idempotente hacia dos tablas nuevas en `db_vivienda`.
- Vínculo best-effort de cada fila del Sheet con su municipio en `viv_cordon_cuneta` (tabla ya existente).
- Endpoint interno IAM-protegido que dispara la sincronización.
- Log de cada corrida (filas leídas/insertadas/actualizadas/con error).
- Tercera pestaña de solo lectura en el `DetailPanel` de `CordonCunetaPage.tsx`, mostrando el checklist técnico del municipio, inspirada en el reporte "REP Loc." del propio Sheet.

### Fuera de alcance (v1)
- Escritura hacia el Sheet (el sync es unidireccional: Sheet → Postgres).
- Vista agregada por departamento (equivalente a "REP Depart.") — se puede derivar client-side más adelante sin backend nuevo.
- Resolución manual de filas no vinculadas a un municipio — quedan logueadas en `viv_cc_sync_log` para revisión futura de calidad de datos.
- Sincronización de otras pestañas del Sheet (`REP Loc.`, `REP Depart.`, `Contactos intendentes`, `Validaciones`, etc.) — son reportes derivados, no fuente de datos.
- Migrar Córdoba Hogar a un patrón similar (si en el futuro aparece una necesidad análoga, se reevalúa aparte).

## 3. Fuente de datos: pestaña "Base TOTAL"

Analizada con `openpyxl` sobre la copia `.xlsx` provista (`docs/context/areas/secretaria_Vivienda/CORDON CUNETA Y ADOQUINADO.xlsx`).

- **Headers**: 3 filas combinadas (fila 1 = título, filas 2-3 = encabezados reales). Los datos empiezan en la **fila 6**. Las filas 4-5 son fórmulas agregadas (`SUM`, `COUNTIF`) — deben ignorarse en la lectura.
- **Rango de datos dinámico**: no hay una última fila fija. El área técnica agrega localidades nuevas insertando filas antes de un marcador literal en la planilla: `"☝🏻 AGREGAR NUEVAS LOCALIDADES ARRIBA DE ESTA FILA ☝🏻"`. El sync debe leer un rango generoso (ej. `A6:AR400`) y cortar el procesamiento al llegar a una fila totalmente vacía o a ese marcador.
- **Columnas relevantes** (44 totales, A→AR):

| Col | Campo | Tipo destino |
|---|---|---|
| A | Localidad/Proyecto | texto |
| B | Departamento | texto |
| C | Expediente N° | texto (con errores de formato conocidos) |
| D | N° Orden (del sheet) | entero |
| E | Municipio/Comuna | `C`/`M` |
| F | Intendente/Presidente comunal | texto |
| G | Teléfono M/C | texto |
| H | Correo electrónico | texto |
| I | Contacto técnico M/C | texto |
| J | Monto convenio | numérico |
| K | Cordón Cuneta (ml) | numérico |
| L | Adoquinado (m²) | numérico |
| M:AE | 19 ítems de checklist (ver 3.1) | enum |
| AF | Estado del expediente | enum |
| AG | Observaciones (bitácora de comunicaciones) | texto largo |
| AH | Fecha de radicación | fecha |
| AI | Repartición donde está radicado | enum |

### 3.1 Checklist (columnas M:AE) — 19 ítems, etiquetas obtenidas de comentarios de celda en fila 3

**Bloque 1 — Documentación inicial (items 1-4, cols M-P)**
1. Nota Solicitud de Financiamiento
2. Ordenanza y Decreto/Resolución comunal
3. DDJJ compromiso de ejecución de obra
4. Proyecto, Planos, Cómputo y Presupuesto

**Bloque 2 — Proyecto técnico (10 ítems con nombre propio en la fila 3, cols Q-Z)**
5. Descripción General
6. Memoria Técnica
7. Plazo de Ejecución
8. Cómputo y Presupuesto
9. Cronograma Avance Obra
10. Planimetría
11. Perfil Calzada
12. Detalle de Cordón Cuneta
13. Detalle de Badén
14. Paquete estructural de Calzada

**Bloque 3 — Documentación administrativa (numerada 5-9 en el sheet, pero son ítems 15-19 en nuestra secuencia, cols AA-AE)**
15. Nota a Contaduría — Cesión Coparticipación
16. N° CBU de M/C especial para depósito de fondos
17. N° CUIT Municipal/Comunal
18. DNI Intendente/Jefe Comunal
19. Acta de Proclamación Intendente/Presidente Comunal

> La numeración del propio Sheet es ambigua (reutiliza 1-9 en dos bloques distintos). Nuestro `item_num` (1-19) es una secuencia propia, estable, definida en el mapeo del código — no se lee del Sheet.

**Valores posibles por ítem** (enum de 5, con inconsistencias de espacios en el dato real que deben normalizarse con `.strip()`): `Completo OK`, `A corregir por M/C`, `En Evaluaciòn Jurìdico`, `En Evaluaciòn Tècnica`, `Sin Presentar`.

**AF (estado del expediente)**: `A INICIAR en DGV` (el dato real trae doble espacio — normalizar), `En CURSO en DGV`, `COMPLETO en DGV`, `APROBADO por TC`.

**AI (repartición)**: `_` (vacío/sin dato), `Técnica DGV`, `Jurídico DGV`, `Tribunal de Cuentas`, `Aprobado TC`.

### 3.2 Calidad de datos observada (el sync debe tolerar esto sin frenarse)
- Expedientes con typos (`"043-079116/2026"` en vez de `"0423-..."`, dígitos faltantes).
- Expediente con placeholder no numérico (`"PEDIR BETI"`).
- Expediente vacío en filas existentes.
- Filas de alta parcial (solo localidad cargada, sin departamento/expediente aún) — deben excluirse del upsert y loguearse como fila incompleta, no como error fatal.
- Nombres de departamento **no coinciden textualmente** entre las 3 fuentes del proyecto (Sheet: `"Presidente Roque Sáenz Peña"`; `viv_cordon_cuneta`: `"Pdte R. Sáenz Peña"`; `viv_geo_localidades`: `"PTE ROQUE SAENZ PEÑA"`). Esto descarta cualquier estrategia de vínculo basada en comparar departamento textualmente.

## 4. Decisiones de arquitectura (confirmadas)

1. **Sin Google Apps Script.** Toda la lógica vive en `svc-vivienda`.
2. **Autenticación al Sheet**: Application Default Credentials de la Service Account de runtime ya existente, `svc-vivienda@gestorcooperativo.iam.gserviceaccount.com`. **No se crea ninguna clave ni se usa Secret Manager para esto** — el Sheet se comparte como Viewer con esa cuenta de servicio. Se reutiliza para no multiplicar identidades (confirmado por el usuario).
3. **Disparo**: Cloud Scheduler, cada 15 min (ajustable), vía HTTP + token OIDC.
4. **Endpoint interno** en el mismo servicio HTTP (no Cloud Run Job) — el volumen (~50 filas) y tiempo de ejecución (segundos) no justifican un Job separado. Se reevalúa si el alcance crece a múltiples sheets/secretarías.
5. **Autenticación del endpoint**: el path `POST /internal/sync/cordon-cuneta-checklist` **no se agrega a `infra/gateway/openapi.yaml`** — queda invisible para API Gateway. Cloud Scheduler invoca directamente la URL de Cloud Run (que ya corre con `--no-allow-unauthenticated`) con un token OIDC emitido por una Service Account con `roles/run.invoker` sobre `svc-vivienda`. El endpoint no usa `Depends(get_current_user)` — su único control de acceso es IAM a nivel de Cloud Run, igual que ya protege el resto del servicio de invocaciones no autorizadas.
6. **Escritura idempotente**: `INSERT ... ON CONFLICT DO UPDATE` sobre clave natural propia de la tabla nueva.
7. **Logging**: cada corrida registra contadores y detalle de errores en `viv_cc_sync_log`, sin abortar el batch por errores de fila individuales.

## 5. Modelo de datos

Todas las tablas nuevas, prefijo `viv_cc_` (mismo dominio funcional que Cordón Cuneta), migración Alembic `0016`.

### 5.1 `viv_cc_checklist_tecnico`
Snapshot actual (no histórico) por localidad del Sheet.

```
id                  UUID PK
localidad           VARCHAR(150) NOT NULL   -- crudo, columna A
departamento        VARCHAR(100)             -- crudo, columna B
expediente          VARCHAR(60)              -- crudo, puede tener errores de formato
orden_sheet         INTEGER                  -- columna D, solo informativo
tipo                VARCHAR(1)               -- 'C' / 'M', columna E
intendente          VARCHAR(200)
telefono            VARCHAR(60)
email               VARCHAR(200)
contacto_tecnico    VARCHAR(300)
monto_convenio      NUMERIC(18,2)
cordon_cuneta_ml    NUMERIC(10,2)
adoquinado_m2       NUMERIC(10,2)
estado_expediente   VARCHAR(50)              -- columna AF, normalizado (trim)
observaciones       TEXT                     -- columna AG
fecha_radicacion    DATE                     -- columna AH
reparticion         VARCHAR(50)              -- columna AI
municipio_id        VARCHAR(36) NULL REFERENCES viv_cordon_cuneta(id)  -- vínculo best-effort, ver 6
sheet_row_number    INTEGER NOT NULL         -- fila original en el Sheet, para debug
last_synced_at      TIMESTAMPTZ NOT NULL
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

Índice único parcial (clave natural del upsert, mismo patrón que `uq_cc_municipio_depto_activo`):
```sql
CREATE UNIQUE INDEX uq_cc_checklist_localidad_depto
  ON viv_cc_checklist_tecnico (lower(trim(localidad)), lower(trim(departamento)));
```

### 5.2 `viv_cc_checklist_items`
19 filas por localidad — tabla normalizada (confirmado, no JSONB) para poder iterar en el frontend.

```
id              UUID PK
checklist_id    UUID NOT NULL REFERENCES viv_cc_checklist_tecnico(id) ON DELETE CASCADE
item_num        SMALLINT NOT NULL   -- 1..19, secuencia propia (no la del Sheet)
item_label      VARCHAR(150) NOT NULL   -- ej. "Memoria Técnica" — hardcodeado en el mapeo del código
valor           VARCHAR(50) NOT NULL    -- uno de los 5 valores enum, normalizado

UNIQUE (checklist_id, item_num)
```

Estrategia de upsert de items: dentro de la misma transacción del upsert del padre, `DELETE FROM viv_cc_checklist_items WHERE checklist_id = :id` seguido de reinsert de los 19 valores actuales. Más simple que 19 upserts individuales; el volumen (19 × ~50 filas por corrida) es trivial para Postgres.

### 5.3 `viv_cc_sync_log`
Una fila por corrida de sincronización.

```
id                  UUID PK
started_at          TIMESTAMPTZ NOT NULL
finished_at         TIMESTAMPTZ
filas_leidas        INTEGER NOT NULL DEFAULT 0
filas_insertadas    INTEGER NOT NULL DEFAULT 0
filas_actualizadas  INTEGER NOT NULL DEFAULT 0
filas_error         INTEGER NOT NULL DEFAULT 0
errores             JSONB    -- [{"fila": 27, "motivo": "expediente invalido: PEDIR BETI"}, ...]
triggered_by        VARCHAR(50)   -- 'cloud-scheduler' | 'manual'
```

## 6. Estrategia de vínculo `municipio_id` (best-effort, no bloqueante)

Al procesar cada fila del Sheet, en este orden:
1. Normalizar el expediente crudo (trim, mayúsculas) y buscar match exacto contra `viv_cordon_cuneta.expediente` (también normalizado) `WHERE deleted_at IS NULL`.
2. Si no hay match, normalizar la localidad (trim, minúsculas, sin acentos vía `unicodedata`) y buscarla contra `viv_cordon_cuneta.municipio` normalizado de la misma forma, **ignorando departamento** (los nombres de departamento no son comparables entre fuentes, ver 3.2).
3. Si ninguna de las dos estrategias matchea, `municipio_id = NULL`. La fila se sincroniza igual (con sus propios datos), pero queda sin vínculo visible desde el panel hasta que se resuelva.
4. Las filas con `municipio_id = NULL` no generan error de sync — es un estado válido y esperado dado el estado actual de los datos. Queda documentado como deuda de calidad de datos para una mejora futura (no se construye herramienta de resolución manual en v1).

## 7. Endpoint interno

```
POST /internal/sync/cordon-cuneta-checklist
```
- Router nuevo `app/internal/router.py`, montado en `main.py` **sin** prefijo `/api/v1/vivienda` y **sin** `Depends(get_current_user)`.
- No se declara en `infra/gateway/openapi.yaml` (intencional — ver decisión 5).
- Ejecuta: leer Sheet → validar/castear filas → upsert `viv_cc_checklist_tecnico` + `viv_cc_checklist_items` → registrar `viv_cc_sync_log` → responder `{ filas_leidas, filas_insertadas, filas_actualizadas, filas_error }`.
- Reutiliza `AsyncSessionLocal` existente; no requiere nueva conexión a DB.

## 8. Cliente de Google Sheets

`app/integrations/google_sheets.py` — cliente genérico y reutilizable (no acoplado a Cordón Cuneta), pensado para si en el futuro otra secretaría necesita el mismo patrón.
- `google.auth.default()` para credenciales (ADC de la SA de runtime).
- `googleapiclient.discovery.build('sheets', 'v4', credentials=creds)`.
- Método `get_values(spreadsheet_id, range_name) -> list[list[Any]]` sobre `spreadsheets().values().get(...)`.
- Nueva dependencia en `pyproject.toml`: `google-api-python-client` (ya está `google-auth==2.35.0`).
- `spreadsheet_id` y el rango van en `config.py` (`Settings`), no hardcodeados en el service.

## 9. Cloud Scheduler + IAM

```bash
# Habilitar API si falta
gcloud services enable sheets.googleapis.com --project=gestorcooperativo

# Permitir que la SA de runtime invoque el propio Cloud Run (para Scheduler)
gcloud run services add-iam-policy-binding svc-vivienda \
  --member="serviceAccount:svc-vivienda@gestorcooperativo.iam.gserviceaccount.com" \
  --role="roles/run.invoker" --region=southamerica-east1 --project=gestorcooperativo

# Cloud Scheduler job
gcloud scheduler jobs create http sync-cc-checklist-tecnico \
  --location=southamerica-east1 \
  --schedule="*/15 * * * *" \
  --uri="https://svc-vivienda-xxx.a.run.app/internal/sync/cordon-cuneta-checklist" \
  --http-method=POST \
  --oidc-service-account-email="svc-vivienda@gestorcooperativo.iam.gserviceaccount.com" \
  --oidc-token-audience="https://svc-vivienda-xxx.a.run.app" \
  --project=gestorcooperativo
```
Compartir el Google Sheet con `svc-vivienda@gestorcooperativo.iam.gserviceaccount.com` (rol Viewer) antes del primer sync.

## 10. Frontend — tercera pestaña en `DetailPanel`

`CordonCunetaPage.tsx` — el `DetailPanel` pasa de 2 a 3 tabs: `Comunicaciones` | `Cambios de estado` | **`Checklist Técnico`**. Cambio puramente aditivo: no toca los dos tabs existentes ni las mutaciones actuales del panel.

Contenido de la pestaña nueva, inspirado en el reporte "REP Loc." del propio Sheet (misma información, estética del sistema):
- Header: expediente, monto convenio, Cordón Cuneta (ml), Adoquinado (m²).
- Tres bloques de checklist (Documentación inicial / Proyecto técnico / Documentación administrativa) — cada ítem como badge de color según valor (`Completo OK` = verde, `A corregir por M/C` = ámbar, `En Evaluación Jurídica/Técnica` = azul, `Sin Presentar` = gris).
- Observaciones del área técnica (texto largo, solo lectura).
- Expediente radicado en: fecha + repartición.
- Pie: "Sincronizado hace X min" (`last_synced_at`), estado de solo lectura (sin botones de edición — el dato vive en el Sheet, no se edita desde acá).
- Estado vacío si `municipio_id` no tiene checklist vinculado: "Sin datos técnicos sincronizados para este municipio."

Nuevo endpoint de lectura (dentro del router existente `cordon_cuneta/router.py`, no uno nuevo):
```
GET /api/v1/vivienda/cordon-cuneta/{municipio_id}/checklist-tecnico
```
Sí se agrega a `openapi.yaml` (con su `options:` de CORS) — es de lectura, mismo patrón/roles (`ROLES_LECTURA`) que el resto del panel.

## 11. Tests

- `tests/test_cc_checklist_sync.py`: mock de la respuesta de `google_sheets.get_values` (fixture con filas representativas, incluyendo casos de calidad de datos: expediente typo, fila incompleta, valores con espacios) + DB de test (sqlite/aiosqlite, patrón ya usado en `conftest.py`). Verifica: insert inicial, update en corrida siguiente (idempotencia), fila inválida no frena el batch, `viv_cc_sync_log` refleja contadores correctos, vínculo `municipio_id` por expediente y por fallback de nombre.
- `tests/test_internal_router.py`: el endpoint responde 200 con el resumen de la corrida; no requiere `Depends(get_current_user)` (verificar que no hay chequeo de rol Firebase en este path).
- Test de lectura del endpoint nuevo `GET .../checklist-tecnico` con los roles existentes.

## 12. Criterios de aceptación

- [x] Migración `0016` aplicada: 3 tablas nuevas + índice único.
- [x] `POST /internal/sync/...` no accesible sin identidad IAM válida (verificado: solo `api-gateway-sa` y `svc-vivienda@...` tienen `roles/run.invoker`; el path no está en `openapi.yaml`).
- [x] `POST /internal/sync/...` no aparece en `openapi.yaml` ni es alcanzable vía Gateway.
- [x] Corridas repetidas del sync no duplican filas (UPSERT verificado en tests y en producción).
- [x] Una fila con dato inválido (expediente `"PEDIR BETI"` en tests; falta de departamento en producción) no frena el procesamiento de las demás; queda reflejada en `filas_error` y en `errores` de `viv_cc_sync_log`.
- [x] Al menos el 80% de las localidades del Sheet quedan vinculadas a `municipio_id` — **100% real** (46/46).
- [x] Tercera pestaña visible en el panel, sin afectar las dos existentes ni las mutaciones actuales de `viv_cordon_cuneta`.
- [x] Cloud Scheduler configurado y corriendo cada 15 min en producción (`sync-cc-checklist-tecnico`, `southamerica-east1`).

## 13. Pendiente / fuera de esta entrega

- Vista agregada por departamento (equivalente "REP Depart.") — derivable client-side, sin backend nuevo, si se pide más adelante.
- Herramienta de resolución manual para filas con `municipio_id = NULL`.
- Extender el patrón de sync a otras secretarías/pestañas si aparece la necesidad.

## 14. Alertas (agregado 2026-07-13, post-entrega inicial)

Una falla **total** del sync (no puede leerse el Sheet — API deshabilitada, credenciales revocadas, Sheet desvinculado de la SA, cuota excedida) es distinta de un error de fila individual (ver §3.2/§6): esta última nunca frena el batch ni genera alerta, es un estado esperado. Una falla total sí debe avisar a un humano.

**Código** (`app/cordon_cuneta/checklist_sync.py`):
- Excepción `SheetReadError`: si `google_sheets.get_values()` explota, se loguea igual una fila en `viv_cc_sync_log` (`filas_error=1`, motivo con el detalle de la excepción) y se re-lanza.
- `app/internal/router.py` traduce `SheetReadError` a `HTTPException(502, {"code": "SHEET_SYNC_FALLIDO", ...})` — un error de fila individual nunca llega a este código, solo la falla total.

**Infraestructura** (Cloud Monitoring, proyecto `gestorcooperativo`):
- Métrica basada en logs `cc_checklist_sync_errors`: cuenta respuestas `httpRequest.status>=400` de Cloud Run en el path `/internal/sync/cordon-cuneta-checklist`.
- Canales de notificación (email): `projects/gestorcooperativo/notificationChannels/16305525817428725684` (bonafepedro@gmail.com), `projects/gestorcooperativo/notificationChannels/5332100367370625933` (infraestructura.coop@gmail.com).
- Política de alerta `projects/gestorcooperativo/alertPolicies/3550730852890842105` ("Sync checklist tecnico CC - fallo"): dispara con 1 sola respuesta de error en una ventana de 15 min (`COMPARISON_GT` sobre 0, `alignmentPeriod=900s`), auto-cierra a los 7 días si no vuelve a ocurrir.

**Por qué log-based metric y no la métrica nativa de Cloud Scheduler** (`cloudscheduler.googleapis.com/job/...`): la métrica sobre logs de Cloud Run es más directa de verificar (se ve el filtro funcionando en Logs Explorer antes de crear la alerta) y no depende de conocer de memoria el schema exacto de labels de la métrica nativa de Scheduler.
