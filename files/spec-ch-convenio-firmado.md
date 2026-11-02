# Spec: Córdoba Hogar — Convenio Firmado como dato propio

**Estado**: implemented
**Versión**: 1.0.1
**Servicio**: `svc-vivienda` (módulo `cordoba_hogar`) + frontend `vivienda` (`CordobaHogarPage.tsx`)
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-02

### Changelog
- **1.0.1 (2026-11-02)** — implemented. Desplegado: migración `0046` aplicada en producción (262 de 284 localidades marcadas, 177 con fecha), `svc-vivienda` `84516b3`, frontend `75f9d7e` en Firebase Hosting. Falta verificación visual.
- **1.0.0 (2026-11-02)** — aprobado por el usuario. A1 según la propuesta. A2 y A3 cerradas con la
  consulta de solo lectura a producción (§5 y §7): la carga inicial usa solo el listado Jurídico.
- **0.1.0 (2026-11-02)** — versión inicial, con decisiones abiertas (§7).

---

## 0. Origen

Pedido (2026-11-02): el KPI "Convenio Firmado" del panel de Córdoba Hogar cuenta solo las
localidades que están *hoy* en ese estado. Una localidad que firmó y avanzó su expediente deja de
contarse, aunque no perdió el hecho de haber firmado. Para contabilidad importa saber, de las
localidades que ingresaron el pedido, cuántas ya firmaron convenio.

## 1. Situación actual

- `CordobaHogarPage.tsx:963-966` calcula el KPI en el cliente: `ejuridico` cuyo label es
  exactamente `'Convenio Firmado'`.
- La firma no se guarda como dato: se deduce de un estado que está pensado para cambiar.
- El label es editable desde el administrador de estados: si alguien lo renombra, el KPI cae a 0
  sin avisar.
- La migración `0044` copió cada estado a varios listados, así que "Convenio Firmado" puede existir
  con ids distintos en Jurídico, Presupuestario y General. El KPI solo mira Jurídico.

## 2. Modelo de datos

Migración `0046`, columnas nuevas en `viv_cordoba_hogar`:

| Columna | Tipo | Regla |
|---|---|---|
| `convenio_firmado` | `BOOLEAN NOT NULL DEFAULT false` | Fuente única del KPI |
| `fecha_firma_convenio` | `DATE NULL` | Opcional: la carga inicial no siempre puede conocerla |

## 3. API

Sin paths nuevos, así que **no hace falta una config nueva del gateway**.

- `LocalidadResponse` agrega ambos campos.
- `LocalidadUpdate` (`PATCH` existente) los acepta.

Validaciones de `actualizar_localidad`, sobre el estado resultante del registro:

- Fecha con `convenio_firmado = false`: 422 `FECHA_FIRMA_SIN_CONVENIO`.
- Fecha futura: 422 `FECHA_FIRMA_FUTURA`.
- Pasar `convenio_firmado` a `false` sin enviar fecha borra la fecha guardada.

Cada cambio queda en `viv_audit_log` por el `log_audit` que ya hace el update.
`viv_ch_estado_historial` no se usa: sus filas exigen un `estado_nuevo_id`, y esto no es un estado.

## 4. Frontend (`CordobaHogarPage.tsx`)

- **KPI "Convenio Firmado"**: cuenta `convenio_firmado` y se muestra como `N / total`, sobre el
  total de localidades. Igual que los otros KPI de la franja, no depende de los filtros.
- **Columna "Convenio firmado"** en la tabla: `SI` / `NO`, con la fecha debajo si existe. Ordenable.
  El `colSpan` de la fila vacía pasa de 18 a 19.
- **Filtro** SI / NO, incluido en "Limpiar filtros".
- **`EditModal`**: casilla "Firmó convenio" y campo "Fecha de firma". La fecha se habilita solo con
  la casilla marcada; desmarcarla la limpia.
- **Excel**: columnas "Convenio firmado" y "Fecha firma convenio".

`AgregarLocalidadModal` no cambia: el pedido es sobre el panel de edición.

## 5. Carga inicial (migración `0046`)

Marca `convenio_firmado = true` en las localidades no borradas que estén o hayan estado en
"Convenio Firmado" del listado Jurídico (label comparado sin distinguir mayúsculas). Fuentes:

1. `ejuridico` actual.
2. Entradas `campo = 'ejuridico'` de `viv_ch_estado_historial`, como `estado_nuevo_id` o
   `estado_anterior_id`. El anterior cuenta porque esas localidades ya estaban firmadas cuando
   empezó el historial.

`fecha_firma_convenio` toma la fecha de la **primera** entrada que llega al estado. Queda `NULL`:

- si no hay entrada de llegada, o
- si esa primera entrada es de `created_by = 'importado-html'`, porque su fecha es la de la
  importación y no la de la firma.

No se inventan fechas. Las demás son la fecha del cambio de estado, no necesariamente la de la
firma, y el área puede corregirlas.

Cada localidad marcada deja una fila en `viv_audit_log`, de donde sale el downgrade.

### 5.1 Resultado esperado en producción (consulta del 2026-11-02)

- De 284 localidades activas, **262 quedan marcadas**; el KPI pasa de 3 a 262.
- Unas 48 quedan sin fecha porque solo aparecen como estado anterior, más las importadas.
- El orden del catálogo no refleja el circuito real: 166 de las 276 salidas de "Convenio Firmado"
  van a "Notificado". Por eso, estar hoy en un estado de orden menor **no** es una reversión.

### 5.2 Casos para que revise el área

Se marcan igual, porque la regla acordada es "alguna vez tuvo el estado", pero pueden ser cargas
por error. El área puede desmarcarlos:

- Estuvieron menos de 24 h en el estado: General Fotheringham, Idiazábal, Las Acequias, Los Molinos,
  Marull, Plaza Luxardo y Río Bamba.
- Hubo 3 salidas de "Convenio Firmado" de vuelta a "Para Firma de Convenio".

Quedan **sin marcar**, porque no tienen rastro del estado: Alejandro Roca y Noetinger, cuyas notas
dicen "FALTA CONVENIO FIRMADO".

**Límite conocido:** lo que pasó por el estado sin dejar rastro no se puede detectar. El área lo
completa a mano.

## 6. Fuera de alcance

- **Cordón Cuneta y Mi Lugar.** Mi Lugar no tiene este KPI. El Tablero de Programas sí muestra
  "Convenio firmado" para CC, pero lo calcula con `ok_gob == "SI"`
  (`app/programas/tablero.py:70`): ahí "OK" y "firmado" se tratan como lo mismo. Queda como
  hallazgo a revisar aparte.
- **`informes/` y `tablero_analitico`** de Vivienda: no usan este KPI.

## 7. Decisiones

- **A1 — Cambio de estado a "Convenio Firmado".** Al pasar `ejuridico` a ese estado (al editar o al
  crear), el backend marca la casilla aunque el formulario la mande desmarcada. Si no había fecha,
  usa `fecha_cambio` o, sin ella, la fecha de hoy. Nunca desmarca: si después se desmarca a mano con
  el estado sin cambiar, queda desmarcada.
- **A2 — Listados de la carga inicial.** Solo Jurídico. Presupuestario no tiene ningún rastro, y
  todas las localidades con rastro en General también lo tienen en Jurídico.
- **A3 — Audit log como fuente.** No se usa: no aporta ninguna localidad que el historial no tenga.

## 8. Pruebas

`tests/test_cordoba_hogar.py`:

- Marcar y desmarcar el convenio; desmarcar borra la fecha.
- Las dos validaciones 422.
- Cambiar `ejuridico` a otro estado no desmarca el convenio.
- A1: cambiar `ejuridico` a "Convenio Firmado" marca la casilla con `fecha_cambio`, también al crear.
- La migración se prueba contra Postgres real (SQLite oculta errores de tipos en `viv_audit_log`).

Frontend: `npm run build` y verificación manual en el navegador.
