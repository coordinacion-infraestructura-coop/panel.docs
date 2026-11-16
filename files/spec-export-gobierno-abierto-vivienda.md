# Spec: Exportación de pagos de Vivienda (CC/CH/ML) a la planilla de Gobierno Abierto

**Estado**: approved
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-16
**Servicios**: `svc-vivienda`
**Specs relacionadas**: `spec-checklist-tecnico-dgv.md` (origen de los datos: hitos de obra)

## Changelog

- **1.0.0** (2026-11-16): aprobada por el usuario. Decisión registrada como ADR-034. Las
  preguntas abiertas del §9 no bloquean: se implementa con 100% = 0 y monto sobre el convenio
  actual, y se ajusta si Gobierno Abierto pide otra cosa.
- **0.1.0** (2026-11-16): primer borrador. Lo pidió el área de Gobierno Abierto: que cada pago
  de anticipo financiero de Cordón Cuneta, Córdoba Hogar o Mi Lugar se cargue solo en su
  planilla, y que después se completen los pagos de avance físico 40/70/100. Columnas, criterio
  de espejo y frecuencia acordados con el usuario.

## 1. Objetivo

Publicar en la planilla de Gobierno Abierto
(`1aRmMDPySFH62TVjO4t4cPUVmvuMO2E5dcQVm_dYYQtE`, pestaña **Vivienda**) una fila por obra de
Vivienda que ya cobró el **anticipo financiero**, con los pagos posteriores de **Avance físico
40% / 70% / 100%** (fecha y monto) a medida que se acreditan. Gobierno Abierto usa esa pestaña
para alimentar la pestaña **BD** con tablas dinámicas.

No se carga nada a mano: el dato ya existe en el sistema.

## 2. Origen de los datos

El Checklist Técnico DGV (`app/checklist_tecnico/`, ver `spec-checklist-tecnico-dgv.md` §5.5–5.6):

- `viv_checklist_obra_hitos (checklist_id, tipo ∈ anticipo|40|70|100, fecha_acreditado)`: la
  fecha de pago la carga el área económica con
  `PATCH /checklist-tecnico/{programa}/{entidad_id}/hitos/{tipo}`. Un hito está **pagado** cuando
  `fecha_acreditado` no es nula.
- El monto **no se guarda**. Se calcula con `catalog.monto_hito(tipo, entidad.monto)`:
  anticipo 50%, 40% → 25%, 70% → 25%, 100% → 0 (saldo). Es el mismo reparto que el Excel de la
  DGV.
- La entidad es `viv_cordon_cuneta` / `viv_cordoba_hogar` / `viv_ml_proyectos`, según
  `viv_checklist_tecnico.programa`. Localidad y departamento salen del padrón oficial
  `viv_geo_localidades` a través de `localidad_id` (ADR-026). Si la entidad no tiene vínculo,
  se usa el nombre libre de la entidad.

## 3. La pestaña Vivienda

### 3.1 Columnas

La fila 1 tiene los encabezados y los escribe el sistema, salvo A. Los datos empiezan en la
fila 2.

| Col | Encabezado | Contenido |
|---|---|---|
| A | *(la que nombró el área)* | **Vacía.** El sistema nunca la escribe. |
| B | Localidad | Nombre oficial del padrón (o el nombre libre si no hay vínculo) |
| C | Departamento | Departamento oficial (mismo criterio) |
| D | Programa | `Cordón Cuneta` · `Córdoba Hogar` · `Mi Lugar` |
| E | Fecha anticipo financiero | `fecha_acreditado` del hito `anticipo` |
| F | Monto anticipo | 50% del convenio |
| G | Cuotas restantes | Hitos sin fecha de pago, por ejemplo `Avance físico 40%, Avance físico 70%, Avance físico 100%`. `Ninguna` si están todos pagos. |
| H | Monto restante | Suma de los montos de los hitos sin pagar |
| I | Avance físico 40% — fecha pago | `fecha_acreditado` del hito `40` |
| J | Avance físico 40% — monto | 25% del convenio, solo si está pagado |
| K | Avance físico 70% — fecha pago | hito `70` |
| L | Avance físico 70% — monto | 25% del convenio, solo si está pagado |
| M | Avance físico 100% — fecha pago | hito `100` |
| N | Avance físico 100% — monto | 0 (saldo), solo si está pagado |
| O | Monto convenio | `entidad.monto` |
| P | Id sistema | `CC-<id>` · `CH-<id>` · `ML-<id> · <nombre del proyecto>` |

La columna P es necesaria porque Mi Lugar puede tener varios proyectos en una misma localidad;
así cada fila es única y trazable.

### 3.2 Qué filas van

- Solo las obras con el **anticipo pagado**. El pago del anticipo es lo que hace aparecer la
  fila.
- Se excluyen las entidades dadas de baja (`deleted_at` no nulo).
- Orden: fecha del anticipo, después localidad y después Id.
- Fechas en formato fecha de Sheets y montos como número (se escriben con
  `valueInputOption=USER_ENTERED`), para que funcionen las tablas dinámicas.

### 3.3 Criterio de espejo

La pestaña la **maneja el sistema**: cada sincronización borra `Vivienda!B2:P` y la vuelve a
escribir completa desde la base. Consecuencias, aceptadas por el usuario:

- No hay duplicados. Si el área económica corrige o borra una fecha, la planilla se corrige
  sola. Si se borra la fecha del anticipo, la fila desaparece.
- **Nadie edita la pestaña Vivienda a mano**: cualquier cambio manual en B:P se pierde en la
  sincronización siguiente. Gobierno Abierto trabaja en la pestaña BD o en otras pestañas.
- Los montos salen del convenio **actual**. Si cambia el monto del convenio, los montos de la
  planilla se recalculan (es el mismo criterio que la pantalla del Checklist).

## 4. Cuándo se sincroniza

1. **Al instante**: después de guardar una fecha de hito (`PATCH .../hitos/{tipo}`), el router
   encola una sincronización en segundo plano (`BackgroundTasks`, con su propia sesión de base).
   Es *best-effort*: si falla Google, el PATCH igual responde 200 y el error queda en el log.
   Si llegan varias fechas seguidas, se juntan en una sola corrida (unos segundos de espera).
2. **Cada hora**: Cloud Scheduler `sync-gobabierto-vivienda` llama a
   `POST /internal/gobierno-abierto/vivienda/sync`. Esa ruta es solo IAM, no pasa por el
   gateway, y usa OIDC con `svc-vivienda@`. Corrige lo que no se haya podido escribir al
   instante.

Respuestas del endpoint interno:

| Código | Cuándo |
|---|---|
| 200 | Corrió bien. Devuelve la cantidad de filas escritas. |
| 503 | El flag está apagado. |
| 502 | Falló la escritura en Google. |

## 5. Implementación

- **Escritura en Sheets**: `app/integrations/google_sheets.py` agrega `clear_values` y
  `update_values`, con un cliente propio de scope `https://www.googleapis.com/auth/spreadsheets`
  (el cliente de solo lectura no cambia). Es ADC del service account de Cloud Run, sin claves, y
  usa un cliente por hilo (lección de `spec-tablero-gobierno-actualizar.md` 1.0.2).
- **Módulo `app/gobierno_abierto/`**:
  - `construir_filas(...)` es pura y testeable.
  - `sincronizar(db, origen)` lee, escribe y registra.
  - Log append-only `viv_gobabierto_sync_log` (inicio, fin, filas, ok, error, origen
    `hito|scheduler|manual`), en la migración `0047`.
- **Configuración**:
  - `gobabierto_export_enabled`: `False` por defecto en `config.py` y `True` en
    `services/cloudbuild.yaml`, igual que `privada_fetch_enabled`.
  - `gobabierto_sheet_id`.
  - `gobabierto_sheet_tab = "Vivienda"`.
- **Aviso de fallas**: si la corrida horaria falla, se crea una notificación para Admin con
  `notificaciones.service.crear`, igual que `integrations/privada_sync.py`.
- **Sin cambios** en el gateway ni en el frontend.

## 6. Permisos (precondición)

El dueño de la planilla (`infraestructura.coop@gmail.com`) la comparte como **Editor** con:

**`svc-vivienda@gestorcooperativo.iam.gserviceaccount.com`**

En Google Sheets: Compartir → pegar la cuenta → rol **Editor** → desmarcar "Notificar" →
Compartir. Conviene proteger la pestaña Vivienda (Datos → Proteger hojas y rangos) para que solo
la edite esa cuenta y el dueño.

Es la **primera vez que el sistema escribe en una planilla**. Hasta ahora solo leía (ver
"Cómo compartir una planilla" en `pedidos_al_cliente/pendientes_gobAbierto_2026-10-21.md`, que
hay que actualizar). La decisión está registrada como ADR-034 en `arquitectura.md`.

## 7. Propuesta para la pestaña BD

Para tablas dinámicas conviene un formato **largo**: **una fila por pago**, en lugar de una fila
por obra con columnas repetidas:

| Localidad | Departamento | Programa | Concepto | Fecha pago | Monto | Id | Año | Mes |
|---|---|---|---|---|---|---|---|---|

Se arma con una sola fórmula en `BD!A2`, sin que el sistema escriba en BD. Está escrita para una
planilla en configuración regional Argentina (separador `;`), así que hay que probarla en una
copia antes de entregarla:

```
=LET(f; FILTER(Vivienda!B2:P; Vivienda!B2:B<>"");
 bloque; LAMBDA(conc; cf; cm;
   FILTER(HSTACK(CHOOSECOLS(f;1;2;3); MAP(CHOOSECOLS(f;1); LAMBDA(x; conc));
                 CHOOSECOLS(f;cf;cm;15)); CHOOSECOLS(f;cf)<>""));
 t; VSTACK(IFERROR(bloque("Anticipo financiero";4;5));
           IFERROR(bloque("Avance físico 40%";8;9));
           IFERROR(bloque("Avance físico 70%";10;11));
           IFERROR(bloque("Avance físico 100%";12;13)));
 t2; FILTER(t; CHOOSECOLS(t;1)<>"");
 HSTACK(t2; YEAR(CHOOSECOLS(t2;5)); MONTH(CHOOSECOLS(t2;5))))
```

Con esa base salen directo, por ejemplo, el monto pagado por departamento y mes, el monto por
programa y los anticipos del año.

Alternativa a futuro, fuera de este alcance: que el sistema escriba también una pestaña
`Vivienda_pagos` en formato largo, si prefieren no mantener la fórmula.

## 8. Pruebas

En `tests/test_gobierno_abierto_export.py`:

- Armado de filas:
  - solo aparecen las obras con anticipo;
  - cuotas restantes y monto restante;
  - el 100% lleva monto 0;
  - Mi Lugar con dos proyectos en la misma localidad;
  - las entidades dadas de baja se excluyen;
  - fallback cuando no hay `localidad_id`.
- Con el cliente de Sheets simulado:
  - se borra y escribe el rango B2:P;
  - la columna A no se toca.
- El PATCH del hito responde 200 aunque falle Sheets.

Verificación manual:

- **Local**: contra una **copia** de la planilla, cargar fechas en el Checklist y ver la fila
  aparecer, completarse y desaparecer.
- **Producción**: disparar el endpoint interno una vez y revisar la pestaña y el log.

## 9. Preguntas abiertas para Gobierno Abierto

1. ¿Está bien que el monto del 100% sea 0 (saldo), según el reparto 50/25/25 de la DGV?
2. ¿Prefieren el monto calculado sobre el convenio actual (se recalcula si el convenio cambia)
   o el monto congelado a la fecha de pago? El sistema hoy no guarda montos pagados.
3. ¿Les sirve la columna P (Id sistema) o prefieren otro identificador?
