# Spec: Fase 3b — Fondo Federal desde la planilla de administración del área

**Estado**: approved
**Versión**: 1.0.0
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-11-26
**Servicio**: `svc-datos-externos`
**Spec padre**: `spec-datos-externos-fase3-fondos.md` (implemented 1.0.1), §8.1
**Pantallas que habilita**: `spec-tablero-gobierno-panel.md`, Fondos › Fondo Federal (Etapa 3)

## Changelog

- **1.0.0** (2026-11-26): aprobado por el usuario. Decisiones: el texto de las observaciones solo lo
  ven `Admin` y `Autoridad` (§4.2); la pantalla, "todos los de gobierno" (ver §8.1: hoy el tablero
  entero es solo `Admin`/`Autoridad`, falta confirmar si se abre); los préstamos **no** suman al
  total del Resumen general hasta que el área responda (§8.2); el Fondo Permanente sigue en su
  pantalla aparte (§8.3).
- **0.9.0** (2026-11-26): primera versión, a partir de la lectura de la planilla (solo lectura, con la
  cuenta `svc-datos-externos@`, que el área agregó como lectora).

## 0. Alcance

El área compartió la planilla **"0. Fondo Federal"**
(`1Rtio-7evxppbGSbR5otfUdw5R63ojKUo1MEILqaC4ak`) con `svc-datos-externos@`. Es la planilla de
administración que estaba faltando para la pantalla Fondo Federal del Looker (pregunta §8.1 de la
Fase 3).

**Qué fondo es.** La planilla habla de Fondo Permanente en todos lados («DATOS RESUMEN DEL FONDO
PERMANENTE», «SALDO EN CUENTA FP»), pero no es la misma que la «7. FONDO PERMANENTE» que ya
sincronizamos:

| | «7. FONDO PERMANENTE» (Fase 3) | «0. Fondo Federal» (esta spec) |
|---|---|---|
| Etapas | 1 a 55 | 1 a 4 (resoluciones 2/25, 04/2025, 05/2025, 01/2026) + extras |
| Plazo de los préstamos | 36 meses (2023–2025) | 60 meses |
| Monto por etapa (Achiras) | $ 5 M | $ 20 M |

Lectura: el Fondo Federal es la **ronda nueva de etapas (desde 2025) que se paga desde la cuenta
del Fondo Permanente**. La «7. FONDO PERMANENTE» queda como el esquema anterior. Esto es
coherente con lo que dijo el área el 2026-10-13 (son distintos). Falta que el área lo confirme,
pero no bloquea nada.

Se leen tres pestañas:

| Pestaña | Qué trae |
|---|---|
| `1. Etapas habilitadas` | Una fila por localidad y etapa: estado, préstamo, devolución, rendición, observaciones |
| `0. Resumen` | Bloque general: saldo en cuenta, disponible, en proceso, déficit, amortizaciones restantes |
| `6. Copa` | Aporte al fondo y devolución de préstamos descontados de la COPA, por localidad y año |

**Pestañas que no se leen**:
- `2. Construcción`, `4.1 Disponibles` y `4.2 Rendición y Amortización`: son vistas que la propia planilla arma para una localidad elegida.
- `3. Etapas aprobadas`: el catálogo de montos por categoría; cada fila de etapas ya trae su monto.
- `5. Perfil de amortización` y `7. Libro Banco`: se pueden sumar más adelante si el área los pide.
- `Hoja 47` y `Localidades`: borradores y nomenclador propio.

## 1. Estructura

### 1.1 `1. Etapas habilitadas`

- La fila 1 trae títulos de grupo y la fila 2, los encabezados. La grilla tiene ~24.000 filas, pero
  solo ~1.743 traen datos (427 localidades: 271 municipios y 156 comunas). El resto es relleno
  con un `-` en `CUOTAS DEVUELTA`.
- Columnas: `ETAPA`, `PARTE`, `RESO`, `FECHA RESO`, `ID_LOC` (código INDEC, el mismo del
  nomenclador), `LOCALIDAD`, `ID_DPTO`, `DEPARTAMENTO`, `TIPO` (MU/CO), `CATEGORÍA POBLACIÓN`,
  `MONTO ETAPA`, `DISPONIBLE ETAPA`, `ESTADO`, **`INTENDENTE`**, **`PARTIDO`**,
  `ORDENANZA/RESOLUCIÓN`, `FECHA INGRESO`, `STICKER PEDIDO`, `OTORGADO`, `CUOTA` (cuota mensual de
  devolución), `PLAZO (MESES)`, `FECHA DE ENTREGA`, `INICIO DESCUENTO COPA`, `RENDICIÓN` (sticker
  de la rendición), `DESTINO`, `DESTINO DEL PRÉSTAMO`, `SALDO` (lo que falta devolver), `DEVUELTO`,
  `CUOTAS DEVUELTA`, `AMORTIZACIÓN` (SI/NO), **`OBSERVACIONES`**, **`FECHA DE AVISO`**,
  `STICKER RENDICION`, `TRÁMITE`.
- **Etapas**: las comunes son 1 a 4. Las extras vienen escritas de muchas formas (`EXTRA`,
  `EXTRA 1`, `Extra 1`, `EXTRA 2 (PARCIAL)`, `EXTRA 2 (SALDO)`, `EXTRA 3 PARCIAL 1`…), y una extra
  pagada en partes ocupa varias filas. Hay una fila con etapa `R` (La Carlota, resolución `S/R`).
- **Palabras**: lo que cobra una localidad es una **etapa**; la **cuota** es lo que devuelve por
  mes. La pantalla usa esas mismas palabras.

### 1.2 Estados

Al 2026-11-26: `PAGADO` (1.106), `PAGADO SIN RENDIR` (376), `DISPONIBLE` (194),
`LISTO PARA PAGAR` (54), `PEDIDO OBSERVADO` (5), `NO ADHERIDO` (4), `REMANENTE DISPONIBLE` (3) y
`PAGADO RENDICIÓN OBSERVADA` (1).

- **Cobrada**: el estado empieza con `PAGADO`.
- **Pendiente**: `DISPONIBLE`, `LISTO PARA PAGAR`, `PEDIDO OBSERVADO` y `REMANENTE DISPONIBLE`.
- **Debe rendición**: `PAGADO SIN RENDIR` y `PAGADO RENDICIÓN OBSERVADA`.
- Si aparece un estado nuevo, se guarda tal cual y se muestra al final de la franja de estados.

### 1.3 `0. Resumen` y `6. Copa`

- `0. Resumen`: rótulo en la columna B y valor en la E. Solo se lee el primer bloque, hasta el
  rótulo «… POR LOCALIDAD».
- Comprobado: el disponible del Resumen (16.890 M) es la suma de lo disponible en las etapas
  `DISPONIBLE` + `REMANENTE DISPONIBLE`.
- `6. Copa`: tiene `ID_LOC`, `AÑO` (2024–2026), `APORTE FONDO PERMANENTE` y
  `DEVOLUCIÓN FONDO PERMANENTE`. El encabezado de la columna G dice hasta qué COPA está
  actualizada («Actualizado hasta la Copa de AGOSTO /2026»).

## 2. Modelo

Son espejos de solo lectura con reemplazo completo en cada corrida. Igual que en la Fase 3, no
llevan `deleted_at` (excepción documentada) y no tienen FK al nomenclador.

| Tabla | Clave | Contenido |
|---|---|---|
| `ext_gob_ffed_etapa` | `id` (+ `fila` de la planilla) | Todas las columnas de §1.1 salvo intendente y partido. `etapa_texto` tal cual, `etapa_numero` (la etapa común, o el número de extra) y `es_extra` |
| `ext_gob_ffed_copa` | `codigo_indec, anio` | `aporte`, `devolucion` |
| `ext_gob_fondo_valor` (existente) | `fondo = fondo_federal` | Los 6 valores del Resumen. `fondo = fondo_federal_copa`, concepto `actualizado_hasta` (`anio` = año, `monto` = mes) |

`ext_gob_ffed_etapa` no tiene clave natural: una etapa extra puede repetirse en varias filas.
Migración `0013`. Cuelga de la `0011` porque la `0012` (acciones en territorio) está en otra rama
sin mergear; la que se mergee segunda se encadena a la otra.

## 3. Sync

- Es una fuente más del sync de Gobierno (`fondo_federal`), con la variable de entorno
  `GOOGLE_SHEET_GOB_FONDO_FEDERAL_ID`. Corre en la misma corrida diaria (`fuente=todas`) y con el
  botón «Actualizar datos».
- `INTENDENTE` y `PARTIDO` **nunca se piden a Sheets**: los rangos de lectura los saltean.
- `OBSERVACIONES` y `FECHA DE AVISO` sí se leen, porque el área anota ahí qué le falta a cada
  localidad. Traen mails de municipios y nombres de personas, así que **nunca van a los logs ni a
  las anomalías**.
- `FECHA DE AVISO` puede ser una fecha o texto libre. Si es texto, se guarda en `aviso_texto` y
  la fecha se toma del primer `d/m/aaaa` que aparezca.
- Freno de seguridad: las etapas y la COPA usan el freno de la Fase 2 (al menos 400 localidades y
  el 90 % de las filas previas). Las cuatro pestañas se aplican juntas o ninguna.
- Anomalías: `etapa_no_reconocida` (p. ej. `R`), `valor_invalido` (un `#REF!` en AMORTIZACIÓN, un
  número fuera de rango en una fecha), `filas_de_relleno` (solo se cuentan), y
  `control_disponible_vs_resumen` cuando el disponible del Resumen no cierra con las filas.

## 4. Endpoint y pantalla

### 4.1 Endpoint

`GET /api/v1/datos-externos/tablero/transferencias/fondo-federal?anio=&codigo_indec=`. Es una
sección más del path que ya existe, así que **no hace falta una config nueva del gateway**.

Devuelve:

- **`resumen`**: los 6 valores del `0. Resumen`. Viene nulo con filtro territorial, porque es del
  fondo entero.
- **Totales**:
  - última etapa habilitada (la más alta de la provincia, aun con filtro);
  - localidades adheridas;
  - cuántas cobraron la última y la anteúltima etapa (velocímetros);
  - localidades que deben rendición y localidades con pendientes;
  - otorgado, saldo a devolver y devuelto.
- **Franja de estados**: por cada estado, cantidad de etapas, localidades, monto de etapa y
  otorgado.
- **Por año de entrega**: otorgado, etapas y localidades, con el detalle mensual.
- **COPA**: aporte y devolución por año, y hasta qué COPA está actualizada.
- **Por localidad**:
  - última etapa y su estado;
  - última etapa cobrada y etapas cobradas;
  - pendientes, con su estado y monto;
  - etapas sin rendir;
  - otorgado, saldo y devuelto;
  - cantidad de observaciones;
  - detalle de cada etapa.

### 4.2 Observaciones

- `observaciones` y `aviso_texto` se devuelven **solo a `Admin` y `Autoridad`** (decisión del
  usuario 2026-11-26).
- Cualquier otro rol recibe esos dos campos nulos y `ver_observaciones = false`. Igual ve que la
  etapa tiene una observación (`tiene_observacion`) y la fecha del aviso.
- Hoy el router del tablero ya frena con 403 a los otros roles. El enmascarado está igual en el
  servicio para cuando se abra la pantalla (§8.1).

### 4.3 Pantalla

Fondos › Fondo Federal reemplaza el «En preparación». Tiene, en este orden:

1. **KPIs del Resumen** (solo sin filtro de localidad): saldo en cuenta, disponible, en proceso,
   déficit y amortizaciones restantes con su tiempo medio.
2. **KPIs de seguimiento**: última etapa habilitada, otorgado y saldo a devolver, y dos tarjetas
   que filtran la tabla: *Deben rendición* y *Con etapas pendientes*.
3. **Franja de estados**: un clic en un estado filtra la tabla a las localidades que tienen alguna
   etapa en ese estado.
4. **Velocímetros** de la última y la anteúltima etapa, y **otorgado por fecha de entrega**, con
   su propio selector de año.
5. **Aportes y devoluciones por COPA**.
6. **Tabla por localidad**. «Ver detalle» abre las etapas de esa localidad con su préstamo,
   devolución, rendición y observación.

## 5. Permisos

Los mismos del tablero: `Admin` y `Autoridad` (`ROLES_TABLERO`). Ver §8.1.

## 6. Fuera de alcance

- Sumar el Fondo Federal al total del Resumen general (§8.2).
- Unificar con la pantalla Fondo Permanente (§8.3).
- Pestañas 3, 5 y 7 (catálogo de etapas, perfil de amortización, libro banco).
- Escribir en la planilla.

## 7. Criterios de aceptación

1. El sync lee las tres pestañas y nunca pide `INTENDENTE` ni `PARTIDO`. Ni el log ni las
   anomalías llevan el texto de las observaciones.
2. Una planilla vaciada o cortada no borra los datos anteriores.
3. Por localidad se ve la última etapa y su estado, la última cobrada, las pendientes y si debe
   rendición.
4. `Admin` y `Autoridad` ven el texto de las observaciones; otro rol, solo que existen.
5. Con filtro territorial no se muestran los valores del fondo entero.
6. Tests en `tests/test_gob_fondo_federal.py`. La verificación en producción la hace el usuario,
   comparando la pantalla con la planilla.

## 8. Preguntas abiertas

1. **¿Quiénes son «todos los de gobierno»?** El usuario pidió que por ahora vean la pantalla
   «todos los de gobierno». Hoy todo el Tablero de Gobierno es solo `Admin` y `Autoridad`
   (decisión del 2026-10-19, `spec-tablero-gobierno-panel.md` §1). Si significa abrir la pantalla
   a otros roles (p. ej. usuarios con la secretaría `gralgob`), eso cambia una decisión ya
   registrada y queda pendiente de confirmación. Hasta entonces, la pantalla mantiene el acceso
   del tablero.
2. **¿Los préstamos suman al total del Resumen general?** El usuario se lo pregunta al área. Hasta
   que responda, no suman.
3. **¿El Fondo Permanente sigue como pantalla aparte?** El usuario se lo pregunta al área. Por
   ahora sigue aparte.
4. **¿Es la misma cuenta?** Confirmar con el área que el Fondo Federal es la ronda nueva de etapas
   del Fondo Permanente (§0).
