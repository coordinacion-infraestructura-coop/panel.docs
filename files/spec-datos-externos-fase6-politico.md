# Spec: Fase 6 — Político: padrón de intendentes y resultados electorales

**Estado**: implemented
**Versión**: 1.0.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-21
**Servicio**: `svc-datos-externos`
**Spec padre**: `spec-datos-externos-tablero-gobierno.md` (approved 1.4.1), Fase 6
**Pantallas que habilita**: `spec-tablero-gobierno-panel.md`, Etapa 6 (Político 🔒)
**ADR**: ADR-027 (sin cambio de mecanismo en esta fase, ver §6)

## Changelog

- **1.0.1** (2026-10-21): implementado y en producción. Backend `cd5bd0b` (migración `0009`
  aplicada), gateway `ministerio-config-v20261021c` (infra `3f15ebd`), frontend `9874d09`. Sin
  job nuevo: `intendentes` y `elecciones` entran al `sync-gob-transferencias` diario. Verificado
  con las planillas reales contra Postgres 15 descartable y en producción: padrón 427 de 427 (3
  sin celular); elecciones 4.166 filas (634 circuitos por elección, 557 con localidad, 427
  municipales, 3 × 427 de las pestañas pegadas). Ningún rango pedido a Sheets incluyó intendente,
  partido ni habitantes de la planilla 18. Diferencias con la planilla en producción: 97 (64 de
  margen, 27 de % por fuerza en `RESULTADOS CALVO`, 4 ganadores en `GANADORES`, 2 en
  `RESUMEN_VOTOS`). Dos ajustes de implementación respecto de la 1.0.0: (1) las diferencias se
  calculan al leer en vez de guardarse en `ext_gob_eleccion_control` (tabla que no se creó): así
  siempre comparan contra el último sync sin un paso más; (2) los 77 circuitos rurales sin
  gobierno local (parajes; la planilla deja la localidad vacía a propósito) no son un descarte:
  cuentan en la provincia y en su departamento y se informan aparte (§1.3).
- **1.0.0** (2026-10-21): aprobado por el usuario, con autorización para implementar y desplegar.
  Suma como **paso siguiente** (fuera de esta fase) validar el padrón de Gobierno contra el de
  Privada y completar en cada uno lo que falte y esté en el otro (§6).
- **0.9.0** (2026-10-21): primera versión, a partir de la lectura de estructura de la planilla
  "18. Resultados elecciones 21-25" y de la pestaña `Intendentes` de "1. REGISTRO REUNIONES"
  (solo lectura, con la cuenta `svc-datos-externos@`, autorizada por el usuario). El usuario
  levantó ese día la regla que impedía leer la pestaña `Intendentes`. No se imprimieron nombres,
  teléfonos ni partidos por localidad: el cruce de §1.4 solo cuenta diferencias. Definiciones del
  usuario: el panel lo ven solo `Admin` y `Autoridad`; resultados municipales desde
  `Municipales - 2023`; padrón y contacto desde `Intendentes`; Privada convive sin cambios (§6);
  se copian también las pestañas calculadas, corrigiendo los errores que se detecten (§3.3).

## 0. Alcance

Sincronizar en solo lectura el padrón de intendentes y los resultados electorales, y exponerlos
en la sección **Político** del Tablero de Gobierno.

| Planilla | Pestañas que se leen | Pantalla |
|---|---|---|
| [1. REGISTRO REUNIONES](https://docs.google.com/spreadsheets/d/1VWNyjcHy0rnyldCI1LU50sb92nD3oWpMBZasI6oC7rQ/edit) | `Intendentes` | Político › Intendentes |
| [18. Resultados elecciones 21-25](https://docs.google.com/spreadsheets/d/1n14N3Ht4v_qbp7XP9nzye0Iqzm0CZZ2X0Oy0jps6Cks/edit) | `2025`, `2023`, `2021`, `RESUMEN_VOTOS_CIRCUITOS` (2 columnas), `Municipales - 2023`, `RESULTADOS CALVO` (gobernador 2023), `GANADORES` (alineación) y, solo como control, `RESUMEN_VOTOS`, `RESUMEN_%`, `INFOMRE`, `DPTOS` | Político › Elecciones |

**Fuera de alcance**:
- **Legisladores**: no están en ninguna planilla compartida (estaban en `DATOS LOC` de "0. Resumen
  General", que el área ya no comparte). Se piden al área.
- La unificación con Privada (ADR-027): ver §6.
- Las pestañas `INFOMRE`, `VARIACION`, `Copia de RANKING`, `Resultado electoral-diferencias
  porcentuales` y `RESULTADOS CALVO - TOTAL DPTO` no se copian tal cual: todo lo que muestran se
  recalcula desde la base (§3.3).

## 1. Estructura relevada (2026-10-21)

### 1.1 Padrón — `Intendentes` (planilla 1)

Encabezado en fila 1; 452 filas: 427 gobiernos locales (`ID_LOC` de 6 dígitos, el código INDEC)
y 25 comunidades regionales (código de texto, sin intendente).

| Columna | Tipo | Uso |
|---|---|---|
| `ID_LOC` | entero | Vínculo con el nomenclador (Fase 1) |
| `Localidades`, `DEPARTAMENTO` | texto | Solo para control; el nombre que se muestra es el oficial |
| `INTENDENTE` | texto (427) | Nombre del intendente o jefe comunal |
| `REELECTO` | texto, 2 valores | Booleano |
| `PARTIDO` | texto, 43 valores | Partido del intendente, tal como lo carga el área |
| `CONTINUIDAD` | texto, 2 valores | Booleano |
| `CELULAR` | número (419) o texto (5) | Contacto. **Dato personal** (§5) |
| `FOTO` | enlace a Google Drive (419) | Se guarda el enlace; no se descarga la imagen |

### 1.2 Votos por circuito — `2025`, `2023`, `2021` (planilla 18)

Encabezado en fila 1; 634 circuitos (633 en 2021). Las tres pestañas comparten `circuito_id`
(1 diferencia en 2021). Columnas fijas: `circuito_id`, `circuito_nombre`, `seccion_nombre`
(departamento), `padrón` y `mesas` (solo 2025), `nulos`, `blancos`, `comando`, `impugnados`,
`totalVotos`, `afirmativos`, `validos`. Después, **una columna por fuerza** (18 en 2025; en 2023 y
2021 con prefijo `POSITIVO_`, más `RECURRIDO(S)` y `OTROS`). Son las elecciones de **diputados
nacionales** de cada año. Las fuerzas cambian de un año a otro, así que se guardan como pares
fuerza → votos y no como columnas fijas.

### 1.3 Circuito → localidad — `RESUMEN_VOTOS_CIRCUITOS`

634 filas, la única pestaña que asigna cada circuito a una localidad (`LOCALIDAD` por nombre,
`circuito_id`). 557 circuitos nombran una de las 427 localidades (Córdoba Capital tiene 119);
los otros **77 son circuitos rurales sin gobierno local** (parajes, 4.323 votos afirmativos en
2025) y vienen sin localidad a propósito: cuentan en la provincia y en su departamento, no en una
localidad. Se leen **solo** esas dos columnas; las de intendente y partido de esa pestaña no se piden (§3.1). El nombre se resuelve
a código INDEC contra el padrón de §1.1 (los 427 nombres de las pestañas pegadas cruzan 427 de 427
por nombre normalizado); lo que no resuelva queda en anomalías.

### 1.4 Municipales 2023 — `Municipales - 2023`

`IMPORTRANGE` de otra planilla, 427 filas con `ID_LOC`. Período (425 son 2023-2027; 1 es
2022-2026 y 1 es 2024-2028), tipo de gobierno, fecha, participación, votos afirmativos, padrón,
partido ganador (texto y normalizado, 5 valores), votos y % del ganador, sexo y nombre del
gobernante electo, ídem segundo, otras fuerzas. **58 filas no tienen votos** (texto en las
columnas numéricas): se muestran como "sin datos de votos".

### 1.5 Pestañas calculadas y control (2026-10-21)

No son fórmulas: son valores pegados. Recalculadas desde la base:

| Pestaña | Resultado del control |
|---|---|
| `RESUMEN_VOTOS_CIRCUITOS` vs `2025` | 1 circuito con afirmativos y votos de una fuerza mal transcriptos |
| `RESUMEN_VOTOS` vs suma de circuitos | Coincide salvo esa misma localidad (6 campos) |
| `RESUMEN_%` vs `RESUMEN_VOTOS` | Coincide (0 diferencias sobre 427) |
| `RESULTADOS CALVO`, diputados 2025 | % por fuerza: 4–11 localidades fuera de tolerancia. Ganador: 426 de 427 correctos. Segundo: 419 de 427. Diferencia ganador–segundo: 362 de 427 |
| `RESULTADOS CALVO`, gobernador 2023 | **No hay base** para recalcularlo: se copia y se controla que sume 100 % |
| `RESULTADOS CALVO` / `RESUMEN_VOTOS` / `GANADORES`, intendente y partido | 106 localidades con otro intendente y 71 con otro partido que el padrón `Intendentes`: las pestañas pegadas están desactualizadas |
| `Intendentes` vs electo en `Municipales - 2023` | Distintos en muchas localidades por formato (orden apellido/nombre) y por cambios posteriores: son fuentes distintas, no un error |

## 2. Modelo de datos (`db_datos_externos`, migración `0009`)

Todas con `id`, `sincronizado_en`, `deleted_at`; reemplazo completo por fuente en cada corrida,
con freno al 50 % (mismo patrón que Fase 5).

| Tabla | Clave | Columnas |
|---|---|---|
| `ext_gob_intendente` | `codigo_indec` | `intendente`, `reelecto` (bool), `partido`, `continuidad` (bool), `celular` (texto, restringido), `foto_url` |
| `ext_gob_eleccion_circuito` | (`eleccion`, `circuito_id`) | `eleccion` (`diputados_2025`, `diputados_2023`, `diputados_2021`), `circuito_nombre`, `departamento`, `codigo_indec` (nulo si no resolvió), `padron`, `mesas`, `nulos`, `blancos`, `impugnados`, `recurridos`, `total_votos`, `afirmativos`, `validos`, `votos` (JSON fuerza → votos) |
| `ext_gob_eleccion_municipal` | `codigo_indec` | `periodo`, `tipo_gobierno`, `fecha`, `participacion`, `afirmativos`, `padron`, `ganador_partido`, `ganador_partido_norm`, `ganador_votos`, `ganador_pct`, `ganador_nombre`, `ganador_sexo`, `segundo_partido`, `segundo_partido_norm`, `segundo_votos`, `segundo_pct`, `segundo_nombre`, `otras_votos`, `otras_pct` |
| `ext_gob_eleccion_copia` | (`pestana`, `codigo_indec`) | Valores que **no se pueden recalcular** y se copian: gobernador 2023 por fuerza (`RESULTADOS CALVO`) y alineación del intendente (`GANADORES.ALIADO`), como JSON |

## 3. Sincronización

### 3.1 Fuentes

Entran al job diario existente `sync-gob-transferencias` como `_Pestana` nuevas: `intendentes`,
`elecciones_circuitos` (las tres pestañas de año + el cruce de circuitos), `elecciones_municipales`
y `elecciones_control` (pestañas pegadas). Se lee primero el encabezado y después solo los
rangos permitidos (`rangos_sin_sensibles`). Columnas que **no se piden**: `Intendente` y `Partido`
de `RESUMEN_VOTOS_CIRCUITOS`, `INTENDENTE`, `PARTIDO` y `HABITANTES` de `RESULTADOS CALVO`,
`Intendente` y `Partido` de `GANADORES`, `RESUMEN_VOTOS` y `RESUMEN_%`: el intendente y el partido
salen solo del padrón.

### 3.2 Reglas de parseo

- Booleanos `REELECTO`/`CONTINUIDAD`: `SI`/`NO` (cualquier otra cosa → nulo + anomalía).
- `CELULAR`: se guarda como texto de dígitos; nunca entra en logs ni en anomalías.
- Columnas de fuerza: todo encabezado que no es una columna fija es una fuerza; se quita el
  prefijo `POSITIVO_`. `OTROS` se guarda como fuerza "Otros".
- `Fecha_elec`: serial de Sheets → fecha.
- Filas de `Municipales - 2023` con texto en votos: se guardan con votos nulos.
- Un circuito cuya localidad no resuelve se guarda con `codigo_indec` nulo y cuenta en el total
  provincial y de su departamento, no en ninguna localidad.

### 3.3 Pestañas calculadas: se copian y se corrigen

Pedido del usuario: copiar también lo calculado, corrigiendo los errores. Como la planilla es
de solo lectura, "corregir" significa:

1. Lo que se puede recalcular desde la base (votos y % por localidad y departamento, ganador,
   segundo, diferencias, variación entre elecciones, conteo de intendentes por partido y
   departamento) **se calcula en nuestro backend** y es lo que se muestra.
2. Al leer se compara contra la última copia de la pestaña pegada (tolerancia: 0,05 puntos
   porcentuales; exacto en votos y etiquetas). La pantalla las lista en un bloque **"Diferencias con la planilla"** y se informan
   al área en el documento de pendientes.
3. Lo que no tiene base (gobernador 2023 y la alineación del intendente) se copia tal cual, con
   el control de que el gobernador sume 100 % por localidad.
4. Ganador y segundo se calculan sobre **todas** las fuerzas, no solo las cuatro columnas
   resumidas (con eso coinciden 426 de 427 ganadores de la planilla).

## 4. Qué expone al tablero

`GET /api/v1/datos-externos/tablero/politico/{seccion}` con `codigo_indec` opcional (filtro
territorial común). Roles: **solo `Admin` y `Autoridad`** (`ROLES_TABLERO`; cualquier otro, 403).
Gateway: config nueva con el path y su `options:`.

- `intendentes`: KPIs (gobiernos locales con dato, reelectos, continuidad, por partido y por
  alineación), conteo por departamento × partido (reemplaza `INFOMRE`), tabla por localidad:
  departamento, localidad, intendente, reelecto, partido, continuidad, celular, enlace a la foto.
- `elecciones`: para cada elección (diputados 2025, 2023 y 2021; gobernador 2023 copiado;
  municipales 2023): totales provinciales, por departamento y por localidad (padrón, participación,
  % por fuerza, ganador, segundo, margen), variación 2023→2025 de las fuerzas comparables, la
  tabla municipal (ganador, segundo, margen) y el bloque de diferencias con la planilla.

Nada de esto entra en `/internal/datos-externos/rollup-territorial`, en `resumen_territorial` ni
en la ficha de localidad de otros roles.

## 5. Privacidad

| Dato | Tratamiento |
|---|---|
| Celular del intendente | Se sincroniza y se muestra solo en Político (Admin/Autoridad). No va a logs, anomalías ni a la exportación general; sí a la exportación Excel de la tabla de intendentes, que tiene el mismo control de rol |
| Foto | Solo el enlace; abre Drive con los permisos propios de quien mira |
| Nombre del intendente, partido, alineación, resultados | Visibles solo para Admin/Autoridad |
| Logs del sync | Solo conteos; nunca nombres, partidos por localidad ni teléfonos |

## 6. ADR-027: convivencia con Privada

Decisión del usuario (2026-10-21): **convivir por ahora**. Privada sigue con su
`intendente_jefe_comunal` y `partido_politico` editables y sin cambios de contrato. El padrón de
Gobierno vive solo en `svc-datos-externos` y solo se ve en Político. La unificación (espejo o
federación) queda para un spec aparte, que puede apoyarse en `ext_gob_intendente`, ahora que
tiene `ID_LOC`, reelecto y contacto. Se actualiza la consecuencia de ADR-027 en
`arquitectura.md`.

**Paso siguiente, pedido por el usuario al aprobar (2026-10-21)**: validar un padrón contra el
otro, localidad por localidad, y completar en cada fuente lo que falte y esté en la otra
(intendente y partido en ambos sentidos). Va en su propio spec, junto con la decisión de espejo o
federación, y define antes quién gana cuando los dos tienen dato y no coinciden.

## 7. Pantallas (Etapa 6 del panel)

Sección **Político 🔒** con dos subsecciones:

- **Intendentes**: KPIs, conteo por departamento y partido, y tabla buscable y exportable.
  Aviso visible: "Legisladores: falta la fuente, pedida al área".
- **Elecciones**: selector de elección; KPIs (padrón, participación, fuerza ganadora y cantidad
  de localidades ganadas); % por fuerza a nivel provincia; tabla por departamento; tabla por
  localidad (ganador, segundo, margen); tabla municipal 2023; variación 2023→2025; bloque
  "Diferencias con la planilla".

La sección no aparece en el menú de otros roles (de todos modos, el tablero entero ya es solo
Admin/Autoridad).

## 8. Tests y verificación

- Parsers con filas sintéticas (sin datos reales): booleanos, celular, fuerzas variables,
  prefijo `POSITIVO_`, municipales sin votos, circuito sin localidad.
- Recálculo: ganador/segundo sobre todas las fuerzas, margen, variación; el control registra
  diferencias por encima de la tolerancia y no por debajo.
- Que los rangos pedidos a Sheets no incluyan las columnas excluidas de §3.1.
- 403 para roles distintos de Admin/Autoridad.
- Prueba con las planillas reales contra Postgres 15 descartable y, después del deploy, sync en
  producción.

## 9. Preguntas para el área (se suman al documento de pendientes)

1. **Legisladores** departamentales y su partido: ¿en qué planilla están?
2. Las pestañas pegadas de la planilla 18 tienen 106 localidades con un intendente distinto al
   padrón: ¿se dejan de usar?
3. Lista de diferencias de `RESULTADOS CALVO` (segundo, margen, % por fuerza) y el circuito mal
   transcripto en `RESUMEN_VOTOS_CIRCUITOS`.
4. ¿De dónde salen los datos de gobernador 2023 por localidad? Si hay una base por circuito, la
   recalculamos.
5. ¿Cómo se define la alineación (puro / aliado / oposición) y quién la mantiene?

## 10. Criterios de aceptación

- Las 427 localidades con intendente aparecen vinculadas por código INDEC.
- Los totales provinciales por elección coinciden con la suma de los circuitos.
- Las diferencias de §1.5 aparecen en el bloque de control.
- Un usuario que no es Admin ni Autoridad recibe 403 y no ve la sección.
- Ningún dato político ni teléfono aparece en logs, anomalías, rollup ni Resumen Territorial.
