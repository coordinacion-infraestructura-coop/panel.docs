# Spec: Fase 3 — Fondos por localidad desde las planillas de Gobierno

**Estado**: review
**Versión**: 0.9.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-20
**Servicio**: `svc-datos-externos`
**Spec padre**: `spec-datos-externos-tablero-gobierno.md` (approved 1.4.1), Fase 3
**Pantallas que habilita**: `spec-tablero-gobierno-panel.md`, Etapa 3

## Changelog

- **0.9.0** (2026-10-20): primera versión, a partir de la lectura de estructura de las tres
  planillas (solo lectura, con la cuenta `svc-datos-externos@`, autorizada por el usuario).

## 0. Alcance

Sincronizar en solo lectura tres planillas de Gobierno y exponerlas al Tablero de Gobierno:

| Planilla | Pestañas que se leen | Qué trae |
|---|---|---|
| 7. FONDO PERMANENTE | `FONDO PERM`, `FONDO PERM CUOTAS`, `IMPORT_Listo` | Etapas del fondo: última recibida y disponibles por localidad, saldo en cuenta; préstamos otorgados |
| 9. FONDO AMBIENTAL | `Fondo` | Convenios por localidad o comunidad regional: objeto, monto, estado; ingresos del año |
| 17. Comunidades Regionales | `Fondo`, `Control` | Cuotas de fondos de funcionamiento y de obra por comunidad regional; crédito vigente del año |

**Fuera de alcance**:
- **FOCOM, Fondo Federal (si es otro fondo, §8.1) y adelantos**: siguen sin fuente. Las pestañas
  estaban en "0. Resumen General", que el área ya no comparte. Por la decisión del 2026-10-19
  (paraguas §3.9), entran recién cuando el área mande la planilla de origen o confirme período y
  moneda de la Vista. Sus pantallas siguen "en preparación".
- **Columnas que nunca se leen**: `INTENDENTE` (Fondo Permanente), `CUIT` y `OBSERVACIONES`
  (Comunidades Regionales), `OBSERVACIONES` (Fondo Ambiental). Las observaciones son texto libre
  con nombres y datos de contacto. Ver §3 para cómo se evita leerlas.
- Pestañas que no se leen: `FONDO PERM - listo` (sus 9 filas ya están en `IMPORT_Listo`),
  `IMPORT_Observados` (rota, `#REF!`), `IMPORT_Disponible` (es la misma información que
  `FONDO PERM CUOTAS` en forma de matriz, §1.1), `Control - Import` (se recalcula en el backend) y
  `Validadores`.
- Cualquier escritura hacia las planillas.

## 1. Estructura relevada (2026-10-20)

Las tres usan `ID_LOC` = código INDEC del gobierno local (Fase 1). Las comunidades regionales
usan los códigos de ente `428`–`453`, que ya están en `ext_gob_entes`.

### 1.1 Fondo Permanente

Es un fondo de etapas (55 hasta hoy). En cada etapa se habilita un monto por localidad adherida;
la localidad lo pide como préstamo y lo devuelve en cuotas descontadas de la coparticipación.

- **`FONDO PERM`**: una fila por localidad (427) con `ULTIMA CUOTA` (última etapa recibida),
  `MONTO` (disponible: suma de sus etapas sin otorgar), `ULT_C_SOC` (1 si recibió la última etapa
  habilitada) y `AULT_C_S` (1 si recibió la anteúltima). Dos filas de control con `ID_LOC = 0`:
  `ULTIMA DISPONIBLE` (última etapa habilitada, hoy 55) y `Disponible` (saldo en cuenta, hoy
  7.444 M).
  - 220 localidades recibieron la etapa 55 y 123 la 54. Son los dos velocímetros del Looker.
- **`FONDO PERM CUOTAS`**: etapas **disponibles sin otorgar**, una fila por localidad y etapa con
  su monto (208 localidades). Tiene 31 filas repetidas idénticas; sin repetir son 752 filas y
  2.133 M.
  - Comprobado: la suma por localidad es igual al `MONTO` de `FONDO PERM` en 206 de 208. La primera
    etapa disponible es la siguiente a la última recibida en 206 de 208.
- **`IMPORT_Listo`**: préstamos otorgados, 865 filas (2023: 5, 2024: 806, 2025: 45; 9 sin fecha =
  listos para pagar). Columnas: `ETAPAS` (número o rango: `49 a 50`, `49 y 50`, `48 (b) a 50`),
  `FECHA`, `COPA DESC.` (mes de inicio del descuento, en texto), `MONTO TOTAL`, `CUOTA`, `PLAZO`
  (36 meses en 854), `ORD./RES.`, `STICKER`, `FECHA INGRESO`, `DESTINO DEL PRÉSTAMO`.
  - Comprobado: `CUOTA × PLAZO = MONTO TOTAL` en las 865 filas. Total 4.743 M.
  - 11 filas sin `ID_LOC`. 141 stickers repetidos (un expediente con varias filas): no es clave.

### 1.2 Fondo Ambiental

- `Fondo`: 135 convenios (2024: 47, 2025: 88), 128 localidades o entes distintos. **10 son de
  comunidades regionales** (1.652 M).
- Columnas: `N° EXPEDIENTE`, `AÑO`, `N° DE CONVENIO`, `FECHA DE CONVENIO`, `N° DE RESOLUCIÓN`,
  `FECHA DE RESOLUCION`, `OBJETO SEGÚN CONVENIO (GLOSARIO)`, `MONTO`, `ESTADO`, `FECHA ESTADO`.
- Estados hoy: `PAGADO` (133, 6.892 M) y `LISTO PARA PAGAR` (2, 49,6 M). El Looker muestra también
  "Tribunal de Cuentas" y "En trámite", hoy vacíos.
- `INGRESOS`: dos celdas, una por año (2024: 1.499 M; 2025: 5.553 M). Es el ingreso del fondo en
  el año. "Total disponible" del Looker = ingresos − afectado.
- El objeto trae un código de glosario al final (`CHIPEADORA (21)`) en 53 filas, y variantes de
  escritura (`PRENSA HIDRAULICA` / `PRENSA HIDRÁULICA`).

### 1.3 Comunidades Regionales

- `Fondo`: 131 filas, una por cuota, de 25 comunidades regionales. `TIPO` = `Funcionamiento`
  (2024: 50, 2025: 52, 2026: 16) u `Obras` (2025: 12, 2026: 1).
- Columnas: `AÑO`, `N° EXPEDIENTE`, `DESTINO`, `MONTO`, `CUOTA`, `MONTO CUOTA`, `ESTADO`,
  `FECHA PAGO CUOTA`, `N° RESOLUCIÓN`, `FECHA RENDICIÓN CUOTA`.
- Estados: `PAGADO RENDIDO` 95, `ANULADO` 17, `PAGADO SIN RENDIR` 7, `A PAGAR/DEBE RENDICIÓN`,
  `DOCUMENTACION EN ANALISIS` y `LISTO PARA PAGAR` (1 cada uno). 9 filas sin monto ni estado
  (relleno).
- `DESTINO` en las filas de funcionamiento es un número de fecha (error de carga, paraguas §4);
  solo las de obras traen texto (camión cisterna, minibús, motoniveladora, etc.).
- `Control`: crédito vigente por año, de funcionamiento y de obras (2024: 62,5 M; 2025: 2.300 M;
  2026: 2.530 M). Es el único dato de presupuesto.
  - Comprobado: con los estados de `Fondo`, el saldo que muestra `Control - Import` sale igual
    (crédito − cuotas no anuladas). Ejemplo: obras 2025 = 2.000 − 1.712 = 288 M.

### 1.4 Calidad de datos que el sync tolera

- Fechas como número serial y como texto (`30 de enero de 2025`, `28-dic-2023`).
- Mes de descuento escrito a mano (`marco`, celdas con el mes repetido en varias líneas).
- Filas repetidas idénticas (`FONDO PERM CUOTAS`): se guardan una vez.
- Filas sin código o con código fuera del nomenclador: se guardan y se informan como anomalía
  (mismo criterio de la Fase 2).

## 2. Modelo de datos

Migración Alembic **0007**. Igual que en la Fase 2, son espejos de solo lectura que se reemplazan
completos en cada corrida, así que **no tienen `deleted_at`** (excepción documentada). Todas
tienen `created_at` y `updated_at`. Montos en `Numeric(18, 2)`, en pesos.

| Tabla | Clave | Columnas |
|---|---|---|
| `ext_gob_fperm_localidad` | `codigo_indec` | `ultima_etapa`, `disponible`, `recibio_ultima`, `recibio_anteultima` |
| `ext_gob_fperm_etapa` | `codigo_indec, etapa` | `monto` (etapas disponibles sin otorgar) |
| `ext_gob_fperm_prestamo` | `id` | `codigo_indec` (nulo si falta), `etapas_texto`, `etapa_desde`, `etapa_hasta`, `fecha`, `mes_inicio_descuento`, `monto_total`, `cuota`, `plazo`, `ord_res`, `sticker`, `fecha_ingreso`, `destino` |
| `ext_gob_fondo_ambiental` | `id` | `codigo` (INDEC o ente), `es_ente`, `anio`, `expediente`, `convenio`, `fecha_convenio`, `resolucion`, `fecha_resolucion`, `objeto`, `monto`, `estado`, `fecha_estado` |
| `ext_gob_creg_cuota` | `id` | `codigo_ente`, `tipo`, `anio`, `expediente`, `destino` (nulo si vino como fecha), `monto`, `cuota`, `monto_cuota`, `estado`, `fecha_pago`, `resolucion`, `fecha_rendicion` |
| `ext_gob_fondo_valor` | `fondo, concepto, anio` | `monto`. Valores sueltos: saldo en cuenta y última etapa habilitada (Fondo Permanente), ingresos por año (Fondo Ambiental), crédito de funcionamiento y de obras por año (Comunidades Regionales) |

- Préstamo sin fecha = **listo para pagar**; con fecha = **otorgado**. Se calcula, no se guarda.
- `etapa_desde` y `etapa_hasta` salen del primer y del último número de `ETAPAS`; el texto
  original se guarda igual.
- `objeto` se guarda como viene. El backend agrupa sin el código de glosario y sin tildes.
- Log: el mismo `ext_gob_sync_log`, con fuentes `fondo_permanente`, `fondo_ambiental` y
  `comunidades_regionales`.

## 3. Sincronización

- Se suma al sync de la Fase 2: mismas fuentes en `POST /internal/datos-externos/gobierno/transferencias/sync?fuente=…`,
  mismo `sync-estado` y **mismo Cloud Scheduler diario** (`sync-gob-transferencias`, que ya corre
  `fuente=todas`). No hace falta otro job.
- IDs en variables de entorno: `GOOGLE_SHEET_GOB_FONDO_PERMANENTE_ID`,
  `GOOGLE_SHEET_GOB_FONDO_AMBIENTAL_ID`, `GOOGLE_SHEET_GOB_COMUNIDADES_REGIONALES_ID`, en
  `services/cloudbuild.yaml`. Las planillas ya están compartidas con `svc-datos-externos@`.
- **Columnas sensibles sin leer**: primero se lee solo la fila de encabezados; con eso se arman
  rangos de columnas que **saltean** `INTENDENTE`, `CUIT` y `OBSERVACIONES`, y se leen con
  `batchGet`. Así esos datos nunca llegan al servicio, ni siquiera en memoria. Si cambia el orden
  de columnas, el rango se recalcula en cada corrida.
- Columnas buscadas por encabezado; si falta una obligatoria, la fuente falla sin tocar datos.
- **Reemplazo completo por fuente, en una transacción**, como en la Fase 2. Las tres pestañas de
  Fondo Permanente forman una sola fuente: se aplican juntas o ninguna.
- **Freno de seguridad**, adaptado a cada fuente:
  - Fondo Permanente: `FONDO PERM` con al menos 400 localidades, y préstamos con al menos el 90 %
    de las filas anteriores. Las etapas disponibles **no** tienen freno por cantidad, porque bajan
    cuando se otorgan préstamos.
  - Fondo Ambiental y Comunidades Regionales: al menos el 90 % de las filas anteriores y al menos
    una fila.
- **Control interno** registrado en el log de cada corrida, sin frenar: localidades donde la suma
  de etapas disponibles no cierra con `FONDO PERM` (hoy 2); años de Comunidades Regionales donde
  las cuotas no anuladas superan el crédito.

## 4. Qué expone al tablero

La misma ruta que ya pasa por el gateway, `GET /api/v1/datos-externos/tablero/transferencias/{seccion}`,
con tres secciones nuevas: `fondo-permanente`, `fondo-ambiental` y `comunidades-regionales`. Así
**no hay cambio en el gateway**. El nombre de la ruta queda corto para estos fondos; se acepta para
no hacer un despliegue del gateway por una diferencia de nombre. Mismos filtros (año,
departamento, localidad) y mismos roles (`Admin` y `Autoridad`).

| Sección | KPIs | Gráficos y tablas |
|---|---|---|
| Fondo Permanente | Última etapa habilitada · Saldo en cuenta · Disponible en etapas sin otorgar · Otorgado en el año (monto, préstamos, localidades) · Listo para pagar | Velocímetros: % de localidades adheridas que recibieron la última y la anteúltima etapa. Préstamos por mes de otorgamiento. Tabla de préstamos (localidad, etapas, fecha, monto, cuota, plazo, inicio del descuento, destino). Tabla de etapas disponibles por localidad |
| Fondo Ambiental | Localidades con fondo · Entes con fondo · Ingresos del año · Afectado · Disponible (ingresos − afectado) | Grupos por estado (incluye los del Looker aunque estén en cero), torta por estado, objetos más financiados, tabla de convenios |
| Comunidades Regionales | Crédito vigente del año · Cuotas (monto y cantidad, sin anuladas) · Pagado · Saldo (crédito − cuotas no anuladas), separados en funcionamiento y obras | Cuotas por estado, tabla de cuotas, tabla de fondos para obras |

- **Filtro territorial**: Fondo Permanente y Fondo Ambiental filtran por localidad y
  departamento. Las comunidades regionales no son localidades: filtran por departamento, y con una
  localidad elegida la pantalla avisa que el dato es del departamento.
- **Año**: Fondo Ambiental y Comunidades Regionales por `AÑO`; préstamos por año de `FECHA`. Las
  etapas y el saldo en cuenta son una foto de hoy y no dependen del año.
- **Saldo en cuenta**: la planilla trae el monto pero no la fecha de corte en una celda propia.
  Se muestra con la leyenda "según planilla".
- **Resumen general**: se suma la columna **Fondo Ambiental**, solo lo `PAGADO` y solo a
  localidades. Las transferencias a comunidades regionales no entran, igual que en el Looker.
  Los préstamos del Fondo Permanente **no** se suman al total hasta que el área confirme si el
  "Fondo Federal" del Looker es este fondo (§8.1), porque son préstamos que vuelven.

## 5. Privacidad

- El sync no lee nombres de intendentes, CUIT ni observaciones (§3).
- Préstamos, saldos y estados son dato financiero de nivel medio (paraguas §5). Hoy solo los ve
  el tablero (`Admin` y `Autoridad`).
- `DESTINO DEL PRÉSTAMO` es texto de la administración (p. ej. "pago de haberes"). Se muestra
  completo. Si aparece un nombre de persona, se le pide al área que lo corrija en la planilla.

## 6. Pantallas (Etapa 3 del panel)

Fondos › **Fondo Permanente** (nueva), **Fondo Ambiental** y **Comunidades Regionales**, con el
mismo esqueleto que las pantallas de la Etapa 2 (KPIs, estados, tabla con descarga a Excel).
FOCOM, Fondo Federal y adelantos siguen "en preparación" con el motivo a la vista.

## 7. Tests y verificación

- Parseo puro con filas sintéticas: etapas `49 a 50`, `49 y 50`, `48 (b) a 50`, `51 (a)`; mes
  `marco` y mes repetido; fecha serial y en texto; filas repetidas; destino que vino como fecha;
  filas de relleno; código de ente; código faltante.
- Rangos que saltean columnas sensibles, con las columnas en otro orden.
- Freno de seguridad por fuente; las etapas disponibles pueden bajar sin frenar.
- Cálculos puros: velocímetros, disponible = ingresos − afectado, saldo de comunidades sin
  anuladas, Fondo Ambiental en el Resumen sin entes.
- Endpoints: 401 sin token, 403 para otros roles, secciones nuevas con filtros combinados.
- Verificación en producción, con autorización: los totales de §1 cierran contra la planilla y
  el usuario compara las tres pantallas con el Looker.

## 8. Preguntas para el área (no bloquean la implementación)

1. **¿El "Fondo Federal" del Looker es el Fondo Permanente?** El 2026-10-13 el área dijo que son
   distintos (paraguas §3.5), pero lo que muestra la página "Fondo Federal" del Looker sale de esta
   planilla: última etapa habilitada, saldo en cuenta, disponible por etapas, y los velocímetros
   de última y anteúltima etapa coinciden con `ULT_C_SOC` y `AULT_C_S`. Si es el mismo fondo, la
   pantalla Fondo Federal se reemplaza por Fondo Permanente y se define qué suma al total. Si es
   otro, sigue esperando su planilla.
2. **Estados de las etapas**: el Looker agrupa las etapas en Pedidas, Disponibles, Listo para pagar
   y Pagado. La planilla deja ver disponibles, listas para pagar y otorgadas, pero no "pedidas".
   ¿De dónde sale ese estado?
3. **Aportes COPA y repago de préstamos del año** (KPIs del Looker): ¿salen de algún descuento de
   la planilla de coparticipación? Si es así, se calculan desde la Fase 2.
4. **Fondo Ambiental, ingresos**: ¿la celda `INGRESOS` es el ingreso total del fondo en el año?
5. **Comunidades Regionales, destino**: en las cuotas de funcionamiento la columna `DESTINO` trae
   fechas. ¿Se puede corregir o se deja vacía?

## 9. Criterios de aceptación

1. Las tres fuentes sincronizan a diario con el job existente, con log por fuente y sin leer las
   columnas sensibles de §0.
2. Los totales de §1 cierran con la planilla.
3. Fondo Permanente, Fondo Ambiental y Comunidades Regionales se ven en el tablero con los filtros
   de año, departamento y localidad.
4. El Resumen general suma Fondo Ambiental pagado a localidades.
5. Ningún cambio en el gateway.
