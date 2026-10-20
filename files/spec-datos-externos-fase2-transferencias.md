# Spec: Fase 2 — Transferencias automáticas desde las planillas de Gobierno

**Estado**: approved
**Versión**: 1.0.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-20
**Servicio**: `svc-datos-externos`
**Spec padre**: `spec-datos-externos-tablero-gobierno.md` (approved 1.4.1), Fase 2
**Pantallas que habilita**: `spec-tablero-gobierno-panel.md`, Etapa 2

## Changelog

- **1.0.1** (2026-10-20): implementado (svc-datos-externos `ffd3fef`, `ec4c15d`; frontend
  `9d97570`). Verificado con las planillas reales contra Postgres 15 descartable: ~60.000 filas en
  18 s sin descartes; % a sueldos 63,4 % / 129,6 %, igual que el Looker. Hallazgo en §8.2.
  FOMMEP devuelve todos los años más la gestión, y las retenciones vienen para los 12 meses, para que
  la pantalla elija sin volver a pedir (maqueta §2.5).
- **1.0.0** (2026-10-20): aprobado por el usuario sin cambios.
- **0.9.0** (2026-10-20): primera versión, a partir de la lectura de estructura de las cinco
  planillas (solo lectura, con la cuenta `svc-datos-externos@`).

## 0. Alcance

Sincronizar en solo lectura cinco planillas de Gobierno y exponerlas al Tablero de Gobierno:

| Planilla | Pestaña que se lee | Qué trae |
|---|---|---|
| 11.1 COPARTICIPACION | `COPA` | Coparticipación por localidad y quincena: bruto, ajuste, neto y ~40 descuentos |
| 11.1 FOFINDES | `FOFINDES` | FOFINDES por localidad y pago: bruto, excedentes, descuentos, neto |
| 11.1 FODEMEP | `FODEMEP`, `ESTADO` | Presupuestado, acumulado retenido y neto por localidad y mes; estado de adhesión |
| 11.1 FOMMEP | `FROMMEP` | Móviles y edificios (categorías A, B y C) por localidad y año |
| 13. EMPLEADOS | `Activos MyC` | Empleados, salario promedio y masa salarial por localidad y año |

"13. EMPLEADOS" estaba en la Fase 5 del paraguas; se adelanta porque el % de la coparticipación que
va a sueldos la necesita.

**Fuera de alcance**:
- Las demás pestañas de esas planillas. En particular, **nunca se leen** `INFORME AUGUSTO` ni
  `Copia de RESULTADOS CALVO` (FODEMEP), que tienen partido, intendente y resultados electorales,
  ni `Búsqueda` (EMPLEADOS). Tampoco se leen las pestañas auxiliares `SUMA` ni `Validadores`.
- Cualquier escritura hacia las planillas.
- Retirar el scraping de Transparencia: queda como respaldo permanente y control (paraguas §7).

## 1. Estructura relevada (2026-10-20)

Todas usan `ID_LOC` = código INDEC del gobierno local, la misma llave que `ext_gob_localidades`
(Fase 1). Las cinco cubren las 427 localidades del nomenclador.

### 1.1 COPA

- 28.182 filas: 2024 (24 quincenas), 2025 (24) y 2026 (18 hasta hoy). Una fila por localidad, año,
  mes y quincena, **sin repetidos**.
- Columnas de identificación: `ID_LOC`, `LOCALIDAD`, `DEPARTAMENTO`, `Quincena` (`1°`/`2°`), `Mes`,
  `Año`, `Fecha de Pago`, `CODIGO` (= `ID_COPA`), `INDICE`.
- `Mes` es una fecha con formato de mes: **se toma solo el mes**; el año sale de `Año` (la fecha
  de la celda no siempre es del mismo año).
- Montos: `Total COP. MUNICIPIOS` (bruto), `AJUSTE`, `TOTAL AJUSTADO`, `NETO RESULTANTE`,
  `NETO A TRANSFERIR`, y 39 columnas de descuento entre `Caja de Jubilaciones` y `Varios`.
- Comprobado: `NETO A TRANSFERIR` = máx(`NETO RESULTANTE`, 0) en todas las filas. `NETO RESULTANTE`
  = bruto − descuentos en 27.850 filas; en 332 no cierra (pregunta §8.5).
- Columnas auxiliares de la planilla (`ULT`, `MES_order`, `COMPLETO`, `ULT_COMP`) no se usan: el
  último pago y el último mes completo se calculan en el backend.
- Totales provinciales (control, en miles de millones): bruto 735,6 (2024), 1.060,3 (2025),
  983,1 (2026 hasta hoy); neto a transferir 442,7 / 530,7 / 441,4.

### 1.2 FOFINDES

- 14.518 filas, mensual desde enero de 2024. Clave: localidad, año, mes **y fecha de pago**:
  diciembre de 2024 tiene dos pagos (30/12/2024 y 10/01/2025), así que se suman, no se
  deduplican.
- Montos: `Total Parte VariableFOFINDES`, `Excedente 30%`, `Excedente 70%`, `FOFINDES` (bruto),
  tres descuentos (Bomberos Voluntarios Río Tercero, Aportes Reintegrables, Equipamientos
  Hospitalarios), `NETO RESULTANTE`, `NETO A TRANSFERIR`. Los tres descuentos aparecen **dos
  veces** con el mismo nombre; se toma la primera aparición (pregunta §8.3).
- Totales provinciales: bruto 103,0 / 144,0 / 133,9 mil M (2024 / 2025 / 2026).

### 1.3 FODEMEP

- `FODEMEP`: 14.785 filas, mensual desde octubre de 2023. Una fila por localidad y mes, sin
  repetidos. `MES` es una fecha (primer día del mes) con año correcto.
- Montos: `Total FODEEMEP` (presupuestado), `ACUMULADO`, `NETO A TRANSFERIR`. 1.281 filas no
  traen presupuestado (sobre todo los primeros meses); se guardan con `NULL`.
- `ESTADO`: 427 filas, `ESTADO` ∈ {NORMAL 323, SUSPENDIDO 89, CORTADO 9, NO ADHERIDO 6} y
  `SALDO DE GESTION` (puede ser negativo).

### 1.4 FOMMEP

- 854 filas: 2024 y 2025, una por localidad y año.
- `Cant_` y `Monto_` de móviles y de edificios A, B, C y total. Edificios total = A + B + C en
  todas las filas. **`Monto_total` es mayor que móviles + edificios en 803 filas** (pregunta
  §8.2).

### 1.5 EMPLEADOS

- `Activos MyC`: 1.125 filas, una por localidad y año (2024: 271 localidades; 2025 y 2026: 427).
  33 filas de 2025 traen `NA`. Columnas `EMPLEADOS`, `SALARIO PROMEDIO`, `MASA SALARIAL`.
- `MASA SALARIAL` ≈ salario promedio × empleados en 1.092 filas.
- Es **dato restringido** (paraguas §5, Fase 1 `CAMPOS_RESTRINGIDOS`): solo `Admin` y `Autoridad`.

### 1.6 Calidad de datos que el sync tolera

- Celdas con error de fórmula (`#####`, `#N/A`, `#DIV/0!`): se toman como vacías y se registra la
  anomalía con código de localidad y columna, nunca el valor.
- Fechas como número de serie: se leen con `dateTimeRenderOption=SERIAL_NUMBER`.
- `ID_LOC` con separador de miles o vacío: mismo `normalizar_codigo` de la Fase 1.
- Encabezados con espacios de más: se normalizan como en la Fase 1.

## 2. Modelo de datos

Migración Alembic **0006**, sobre la 0005 de la rama `datos-externos-fuentes-oficiales`. Si al
implementar esa rama todavía no está en `main`, se coordina antes de numerar. Todas las tablas
tienen `created_at` y `updated_at`. **Excepción a la regla de borrado lógico**: no tienen
`deleted_at`, porque son espejos de solo lectura que se reemplazan completos en cada corrida
(§3), no entidades que alguien edite. La historia de cada corrida queda en el log; la planilla
sigue siendo la fuente.

| Tabla | Clave única | Columnas |
|---|---|---|
| `ext_gob_copa` | `codigo_indec, anio, mes, quincena` | `id_copa`, `fecha_pago`, `indice`, `bruto`, `ajuste`, `total_ajustado`, `neto_resultante`, `neto_a_transferir`, `caja_jubilaciones`, `apross`, `descuentos` (JSON `{encabezado: monto}`, todas las columnas de descuento, incluidas Caja y APROSS) |
| `ext_gob_fofindes` | `codigo_indec, anio, mes, fecha_pago` | `total_parte_variable`, `excedente_30`, `excedente_70`, `bruto` (`FOFINDES`), `descuentos` (JSON), `neto_resultante`, `neto_a_transferir` |
| `ext_gob_fodemep` | `codigo_indec, periodo` | `fecha_pago`, `presupuestado`, `acumulado`, `neto_a_transferir` |
| `ext_gob_fodemep_estado` | `codigo_indec` | `estado`, `saldo_gestion` |
| `ext_gob_fommep` | `codigo_indec, anio` | `cant_moviles`, `monto_moviles`, `cant_edificios_{a,b,c,total}`, `monto_edificios_{a,b,c,total}`, `monto_total` |
| `ext_gob_empleados` | `codigo_indec, anio` | `empleados`, `salario_promedio`, `masa_salarial` (restringidas) |

- Montos en `Numeric(18, 2)`, en pesos (las planillas vienen en pesos; la Vista en millones).
- `descuentos` en JSON por la misma razón que el cronograma de ATP: una columna de descuento
  nueva no necesita migración. Caja y APROSS se duplican en columnas propias porque las
  comparativas las consultan siempre.
- `codigo_indec` sin FK (puede venir un código que el nomenclador todavía no tiene). Esas filas se
  guardan igual y se informan como anomalía `codigo_fuera_del_nomenclador`; el tablero solo
  muestra lo que cruza con el nomenclador, salvo los totales sin filtro territorial, que suman
  todo (mismo criterio de la Etapa 1).
- `ext_gob_localidades` (Fase 1): se completan `empleados` y `masa_salarial` con el **último año
  con dato** de `ext_gob_empleados`. Así la ficha restringida que ya existe deja de estar vacía en
  esos dos campos. Se suman `copa_vista_m` y `fofindes_vista_m` (control, §4.2; nunca se exponen).
- Log: el mismo `ext_gob_sync_log` de la Fase 1, con fuentes `copa`, `fofindes`, `fodemep`,
  `fommep` y `empleados`.

## 3. Sincronización

- Endpoint IAM: `POST /internal/datos-externos/gobierno/transferencias/sync?fuente=todas|copa|…`
  y `GET .../gobierno/transferencias/sync-estado` (última corrida de cada fuente). Sin gateway.
- Cloud Scheduler **diario** (el área carga con hasta una semana de demora); se apaga con
  `GOB_TRANSFERENCIAS_SYNC_ENABLED`.
- IDs de planilla en variables de entorno (`GOOGLE_SHEET_GOB_COPA_ID`, etc.). Las planillas ya
  están compartidas con `svc-datos-externos@`: la lectura de estructura funcionó con esa cuenta.
- Rango **por nombre de pestaña** (`'COPA'!A1:BZ`), nunca la planilla entera.
- Columnas buscadas **por encabezado**, no por posición. Si falta una columna obligatoria, la
  fuente falla sin tocar datos. En COPA y FOFINDES, toda columna numérica entre el primer y el
  último descuento conocido entra a `descuentos` con su encabezado, para tolerar columnas nuevas.
- **Reemplazo completo por fuente, en una transacción** (borrar e insertar en bloque). Es distinto
  del SAVEPOINT por fila de la Fase 1, y es a propósito: son ~60.000 filas, validadas en Python
  antes de escribir. Una fila inválida se descarta y queda como anomalía antes de llegar a la
  base, sin necesidad de SAVEPOINT. Además, el reemplazo atómico evita mezclar dos versiones
  de la planilla.
- **Freno de seguridad**: si una fuente trae menos del 90 % de las filas de la corrida anterior,
  o menos de 400 localidades distintas, falla sin tocar datos. Cada fuente es independiente:
  si falla una, las demás se aplican y la corrida queda `parcial`.
- El log de la corrida falla se guarda con commit explícito antes de devolver el 502. Así no se
  repite el bug abierto de `_log_falla` en transferencias, que se pierde con el rollback de
  `get_db`.

## 4. Conciliación

### 4.1 Contra el scraping de Transparencia

En cada corrida de `copa` y `fofindes` se calcula, por mes, el bruto de la planilla (suma de
quincenas o de pagos) contra `ext_transferencias` (`coparticipacion_ley_8663` y `fofindes`),
vinculando por `id_geo` del nomenclador:

- total provincial de cada fuente y diferencia porcentual;
- localidades con diferencia mayor al 1 % (solo códigos).

Se guarda en `anomalias` del log y se expone en
`GET /internal/datos-externos/gobierno/transferencias/conciliacion?anio=`. Con esto se decide más
adelante si el scraping se retira (paraguas, Fase 2). Esta fase no lo decide.

### 4.2 Contra la Vista

La Fase 1 guarda `COPA` y `FOFINDES` de la Vista (en millones) en `copa_vista_m` y
`fofindes_vista_m`. La conciliación compara el total provincial contra tres períodos candidatos:
año en curso, desde el inicio de la gestión (diciembre de 2023) y todo lo que tiene la planilla.
Así se ve qué período usa la Vista, que es la pregunta pendiente al área. Nunca se muestran
(decisión del 2026-10-19, paraguas §3.9).

## 5. Qué expone al tablero

Todo bajo `GET /api/v1/datos-externos/tablero/transferencias/{seccion}`, que **ya está en el
gateway** (`ministerio-config-v20261019c`). Hay secciones nuevas, pero no hace falta una config
nueva. Solo `Admin` y `Autoridad`, igual que la Etapa 1. Filtros: `anio` y `codigo_indec`
(lista).

| `seccion` | Respuesta |
|---|---|
| `coparticipacion` | Lo de la Etapa 1, más: `fuente` (`planilla` o `transparencia`), serie mensual neta, último pago (mes, quincena, fecha, bruto, neto), último mes completo, pagos por quincena del año, descuentos por concepto, comparativas Caja/COPA bruta y APROSS/COPA bruta (mes a mes por año, acumulado al último mes y mismo mes de cada año) y % a sueldos (§5.1) |
| `fofindes` | Lo de la Etapa 1, más: `fuente`, serie neta, último mes completo, descuentos |
| `fodemeep` | KPIs (localidades, presupuestado y transferido en la gestión, último mes completo), serie mensual transferido / retenido (§8.1), tabla neto a transferir por localidad y mes del año, estados de adhesión (conteo y lista) |
| `fommep` | KPIs por año: localidades, total, móviles (cantidad y monto), edificios A, B y C; tabla por localidad |
| `resumen` | Por departamento y por localidad: COPA neta, FOFINDES neto, total y total por habitante (población del Censo) |

- **Fuente**: Coparticipación y FOFINDES usan la planilla cuando tiene datos del año pedido; si
  no, caen al scraping y lo informan en `fuente`, para que la pantalla lo diga.
- La respuesta de cada sección es un modelo propio; el endpoint declara la unión de esos modelos.
- `/tablero/meta` suma la frescura de las cinco fuentes.

### 5.1 % de la coparticipación que va a sueldos

Igual que el Looker:

- **Bruto**: masa salarial ÷ (COPA bruta + FOFINDES bruto) del último mes completo.
- **Neto**: masa salarial ÷ (COPA neta + FOFINDES neto) del mismo mes.

Detalles:

- Se usa la masa salarial del último año con dato y la respuesta dice qué año es.
- Numerador y denominador suman **solo las localidades que tienen masa salarial**. La respuesta
  informa cuántas son: "X de Y localidades con dato".
- Con filtro de una localidad, el porcentaje deja deducir la masa salarial. Es aceptable porque
  la pantalla es solo para `Admin` y `Autoridad`. El monto de la masa nunca viaja en la respuesta.

### 5.2 Resumen general en esta fase

El total del Looker suma ATP, FOCOM, COPA, FOFINDES, adelantos, Fondo Federal y Fondo Ambiental.
En esta fase solo hay COPA y FOFINDES. La pantalla muestra esas dos columnas y el total
**rotulado "COPA + FOFINDES"**. Las demás columnas aparecen como "llega con la Fase 3".

El semáforo ATP/COPA+FOFIN espera a la Fase 4 (regla de no duplicar ATP Interno y Compromisos).
Usar solo Compromisos daría otro indicador con el mismo nombre.

## 6. Privacidad

- `masa_salarial`, `salario_promedio` y `empleados` son restringidos. Nunca van a logs ni a
  `anomalias`; los endpoints internos de conciliación no los incluyen.
- Descuentos, saldos y estados son dato financiero de nivel medio (paraguas §5). Hoy solo los ve
  el tablero (`Admin` y `Autoridad`).
- El sync no lee las pestañas con datos políticos (§0).

## 7. Pantallas (Etapa 2 del panel)

Coparticipación completa, FOFINDES, FODEMEEP, FOMMEP y Resumen general, según la maqueta
aprobada (`spec-tablero-gobierno-panel.md` §2.5 y §2.6), con las limitaciones de §5.2 y §8.

## 8. Preguntas para el área (no bloquean la implementación)

1. **FODEMEP `ACUMULADO`**: ¿es lo retenido a una localidad suspendida o cortada, que se libera
   cuando vuelve a NORMAL? Hasta confirmarlo, la serie "transferido / pendiente" muestra
   transferido = `NETO A TRANSFERIR` y pendiente = `ACUMULADO` del mes, rotulado "retenido
   (según planilla)".
2. **FOMMEP `Monto_total`**: es mayor que móviles + edificios en 803 de 854 filas. ¿Qué más
   incluye? Mientras tanto se muestra el total de la planilla y la diferencia como "otros
   conceptos". **Hallazgo (2026-10-20)**: en 2025 la diferencia provincial es exactamente igual al
   monto de edificios (330,3 M), así que el total parece sumar los edificios dos veces. Se le
   pregunta al área si es un error de fórmula.
3. **FOFINDES**: los tres descuentos aparecen dos veces con el mismo nombre. ¿Son iguales o
   significan cosas distintas? Además, ¿el bruto es `FOFINDES` o `Total Parte Variable`? Coinciden
   solo en el 44 % de las filas; se usa `FOFINDES`.
4. **Masa salarial**: ¿es mensual? ¿Incluye aguinaldo? ¿De qué mes es cada año (2026 dice "ULT
   JUN-26")? ¿Por qué 2024 tiene 271 localidades?
5. **COPA**: en 332 filas el neto no cierra con bruto − descuentos. ¿Hay ajustes que no están en
   columnas?

## 9. Tests y verificación

- Parseo puro con filas sintéticas: mes desde fecha serial, quincena `1°`/`2°`, encabezado
  duplicado, celda `#####`, columna de descuento nueva, código fuera del nomenclador, `NA` en
  masa salarial.
- Freno de seguridad: menos del 90 % de las filas no toca datos; una fuente fallida no frena las
  demás.
- Cálculos puros: último pago, último mes completo (las dos quincenas), comparativas de
  retención, % a sueldos con cobertura parcial, suma de dos pagos en el mismo mes de FOFINDES.
- Endpoints: 401 sin token, 403 para roles distintos de `Admin`/`Autoridad`, 404 para sección
  inexistente, caída a scraping cuando la planilla no tiene el año.
- Verificación en producción, con autorización: los totales provinciales de §1 cierran contra
  la planilla y la conciliación de §4.1 corre.

## 10. Criterios de aceptación

1. Las cinco fuentes sincronizan a diario, con log por fuente y sin leer las pestañas de §0.
2. Los totales provinciales por año cierran con la planilla (§1).
3. La conciliación contra el scraping y contra la Vista queda registrada en cada corrida.
4. Coparticipación, FOFINDES, FODEMEEP, FOMMEP y Resumen general se ven en el tablero con los
   filtros de departamento, localidad y año.
5. La masa salarial no aparece en ninguna respuesta ni log; solo el porcentaje agregado.
6. Ningún cambio en el gateway.
