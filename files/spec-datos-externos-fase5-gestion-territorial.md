# Spec: Fase 5 — Gestión territorial desde las planillas de Gobierno

**Estado**: implemented
**Versión**: 1.0.1
**Responsable de spec**: Pedro Bonafe
**Última actualización**: 2026-10-21
**Servicio**: `svc-datos-externos`
**Spec padre**: `spec-datos-externos-tablero-gobierno.md` (approved 1.4.1), Fase 5
**Pantallas que habilita**: `spec-tablero-gobierno-panel.md`, Etapa 5

## Changelog

- **1.0.1** (2026-10-21): implementado y en producción. Backend `7567e2d` (migración `0008`
  aplicada), gateway `ministerio-config-v20261021b` (infra `b3e4bb3`), frontend `04a69fc`. Sin job
  nuevo: las cinco fuentes entran al `sync-gob-transferencias` diario. Verificado con las
  planillas reales contra Postgres 15 descartable y en producción: obras 1.054 filas (883 obras;
  9 descartadas), recorridos 429 + 427 de Visitas Gober, ministro 347 (18 descartadas: 6 fechas
  futuras, 1 inválida, 11 sin código), reuniones 578, convenios 427 localidades × 15 columnas.
  Cobertura: 274 de 427 localidades visitadas por funcionarios; 321 por el Gobernador. Ningún
  rango pedido a Sheets incluyó columnas excluidas y la respuesta de obras no trae montos.
  Visitas del Gobernador no tiene filtro de año (es una foto de hoy).
- **1.0.0** (2026-10-21): aprobado por el usuario sin cambios, con autorización para implementar
  y desplegar.
- **0.9.0** (2026-10-21): primera versión, a partir de la lectura de estructura y agregados de
  cinco planillas (solo lectura, con la cuenta `svc-datos-externos@`, autorizada por el usuario).
  No se imprimieron textos libres, nombres de personas ni partidos; los funcionarios se
  anonimizaron con letras.

## 0. Alcance

Sincronizar en solo lectura cinco planillas de Gobierno y exponerlas en la sección **Gestión
territorial** del Tablero de Gobierno, más los bloques de la portada que salen de ellas
("Reuniones en el Panal", "Visitas en Territorio", convenios).

| Planilla | Pestañas que se leen | Pantalla |
|---|---|---|
| [12. Obras](https://docs.google.com/spreadsheets/d/1-J9HDF7Km27JQ7d8gdkUVS9hXDOYbORzuyk6hYZ9pS8/edit) | `BD-TABLERO` | Gestión › Obras |
| [2. RECORRIDO Y VISITAS](https://docs.google.com/spreadsheets/d/1NA31sFzRNosJ0ZU0LWumdPB-5AGhd_byAMtUByF8ffI/edit) | `Recorrido`, `Visitas Gober` | Gestión › Visitas; Visitas del Gobernador |
| [2.1 RECORRIDO Y VISITAS - CALVO](https://docs.google.com/spreadsheets/d/1GSkmLb8DXv2CJvyDUdGeVuf81wllkqH2nkNjVts8oTM/edit) | `Recorrido` | Gestión › Visitas |
| [1. REGISTRO REUNIONES](https://docs.google.com/spreadsheets/d/1VWNyjcHy0rnyldCI1LU50sb92nD3oWpMBZasI6oC7rQ/edit) | `Reuniones` | Gestión › Reuniones |
| [6. CONVENIOS FIRMADOS Y RADIOS](https://docs.google.com/spreadsheets/d/1PgKPYtBzFWIUsNJIa_22pTm4rXDrrBfi3GoDcJWSkTs/edit) | `Convenios` | Gestión › Convenios |

**Fuera de alcance**:
- **Montos de obras**: se sincronizan pero **no se exponen** hasta que el área confirme la
  moneda (paraguas §3.7; pedido C8 al área). Tampoco se lee `BD-VARIAS` (prorrateo de montos):
  entra con los montos.
- **IPCAM**: sin planilla (pedido A5 al área). La pantalla sigue "en preparación".
- **Parques industriales** (pestaña oculta de Obras) y **Polideportivos** (planilla 6): la matriz
  de convenios ya trae la cantidad por localidad, que es lo que muestra el Looker. Su detalle
  queda para más adelante.
- **Columnas y pestañas que nunca se leen** (§5): `INTENDENTE` y `PARTIDO` de `Visitas Gober`;
  `TEMA` e `INSTITUCIÓN` de los recorridos; `DETALLE REUNIÓN` y `DETALLE RESOLUCIÓN` de
  reuniones; la pestaña `Compromisas` entera; la pestaña `Intendentes` (Etapa 6).

## 1. Estructura relevada (2026-10-21)

Todas tienen el encabezado en la fila 1 y el código INDEC en `ID_LOC`.

### 1.1 Obras (`BD-TABLERO`)

1.063 filas, 886 obras (`ID_OBRA`). Columnas: `ID_OBRA`, `MONTO`, `ID_LOC`, `TIPO`, `LOCALIDAD`,
`DEPARTAMENTO`, `NOMBRE DE OBRA`, `DIVISIÓN`, `ESTADO DE OBRA`, `AVANCE`, `FECHA INICIO`,
`FECHA DE FIN DE OBRA`, `CATEGORIA`.

- **Una fila por obra y localidad.** Las 61 obras con más de una fila repiten el monto total en
  cada fila (60 de 61); el prorrateo está en `BD-VARIAS`.
- `TIPO` (ámbito): `UNICA` 788 · `COMPARTIDA - DPTO UNICO` 156 · `COMPARTIDA` 69 ·
  `PROVINCIAL` 43. Las provinciales traen `ID_LOC` = `PROVINCIAL` (obras generales).
- `ID_LOC`: además de códigos, `nnZR` (zona rural de un departamento, 10 filas) y `1` (2 filas
  rotas con `#N/A` en localidad).
- `DIVISIÓN`: HYG 333 · EPEC 220 · VIALIDAD 213 · ARQUITECTURA 136 · CASISA 57 · URBANISMO 48 ·
  INF. ELÉCTRICA 46. Hay 3 filas con un número en lugar de la división.
- `ESTADO DE OBRA`: FINALIZADA 372 · OBRA EN EJECUCION 319 · EN PROYECTO 160 · EN LICITACIÓN 76 ·
  A LICITAR 73 · EN PROCESO DE ADJUDICACIÓN 46 · ADJUDICADA 6 · SUSPENDIDA 1; además `0` (2) y
  `-` (1). `CATEGORIA` los agrupa en FINALIZADA 372 · A EJECUTAR 365 · OBRA EN EJECUCION 319.
- **7 filas (3 obras) con `#N/A`** en tipo, estado, división y fechas (la búsqueda de la planilla
  no encontró `CAS-7`, `EPEC-169`, `EPEC-172`).
- `AVANCE`: proporción 0–1; `-` en 301 filas.
- Fechas: serial, `-`, texto `1 oct 2026` y `2026-04-07`. Inicio 2018–2027; fin 2022–2030.
- `MONTO`: si fuera en millones de dólares, como dice el Looker, las obras sumarían US$ 61.000
  millones, lo que es imposible. Por obra única suman 61.188 millones; las 43 provinciales son
  46.760 millones. La moneda sigue en duda (§0).

### 1.2 Recorridos de funcionarios (`Recorrido`, planillas 2 y 2.1)

Columnas: `FUNCIONARIO`, `FECHA`, `ID_LOC`, `LOCALIDAD`, `DEPARTAMENTO`, `TIPO DE VISITA`,
`INSTITUCIÓN`, `TEMA`.

| | Planilla 2 | Planilla 2.1 (ministro) |
|---|---|---|
| Filas | 429 | 365 |
| Funcionarios | 5 | 1 |
| Fechas | 2023-12-10 a **2026-03-03** | 2023-12-11 a 2027-06-27: **6 futuras** y 1 inválida |
| Por año | 2023: 4 · 2024: 325 · 2025: 80 · 2026: 20 | 2023: 6 · 2024: 190 · 2025: 88 · 2026: 74 · 2027: 6 |
| Sin código | 0 | 11 (5 vacías, 6 con `#N/A`) |
| Localidades | 195 | 199 |
| Repetidas (funcionario + fecha + localidad) | 8 | 13 |

`TIPO DE VISITA` (8 valores, con mayúsculas y espacios duros mezclados): REUNIÓN DE TRABAJO ·
INSTITUCIONAL · ENTREGA VIVIENDAS/CRÉDITOS/DINERO · LANZAMIENTO · VISITA OBRAS · FIRMA CONVENIO ·
CAPACITACIÓN · CONFERENCIA.

La pestaña oculta `Recorrido (F+Calvo)` (794 filas) es la unión de las dos; se leen las bases.

### 1.3 Visitas del Gobernador (`Visitas Gober`)

427 filas, una por gobierno local: `ID_LOC`, `LOCALIDAD`, `DEPARTAMENTO`, `VISITAS` (SI 321 / NO
106), `INTENDENTE`, `PARTIDO`, `HABITANTES` (5 rangos: menos de 2.000 · 2.000–5.000 ·
5.000–10.000 · 10.000–50.000 · más de 50.000) y `FECHA__ULT` (2024-01-10 a 2026-09-20).
**10 localidades dicen SI y no tienen fecha.**

### 1.4 Reuniones (`Reuniones`)

578 filas: `ID_REU`, `ID_LOC`, `LOCALIDAD`, `DEPARTAMENTO`, `FECHA`, `TEMAS`, `DETALLE REUNIÓN`,
`RESOLUCIÓN`, `DETALLE RESOLUCIÓN`.

- **Fechas de 2023-12-20 a 2024-12-27: la planilla no tiene reuniones de 2025 ni de 2026.**
- `TEMAS`: lista separada por comas (78 filas con más de un tema) sobre 14 temas: OBRAS ·
  DIFICULTADES FINANCIERAS · FOCOM · SERVICIOS PÚBLICOS · OTROS · EDUCACIÓN · FONDO PERMANENTE ·
  COPARTICIPACIÓN · FESTIVALES · SALUD · CATÁSTROFES NATURALES · SEGURIDAD · SUELDOS ·
  CONECTIVIDAD.
- `RESOLUCIÓN`: Resuelto 275 · Pendiente 186 · Parcial 64 · Derivado 33 · Sin resolución 1;
  vacía en 19.
- 2 `ID_REU` repetidos; 1 fila con zona rural (`7ZR`).

### 1.5 Convenios (`Convenios`)

427 filas, una por gobierno local. Columnas SI/NO: `EDUCACION`, `SEGURIDAD`, `CIDI`, `APROSS`,
`MONOTRIBUTO`, `AUTOMOTOR`, `URBANO`, `INFORMACIÓN` (con `Si`/`si`), `RADIO`, `OBRAS`,
`ACUERDO FEDERAL`; `BALANCE` (SI/NO/NC); `COMPROMISO VOLUNTARIO` (SI o vacío); cantidades:
`POLIDEPORTIVO` (0–4, y un 56) y `PARQUES INDUSTRIALES` (0–8); `AÑO RADIO` (año como texto, con
valores como `0891`, `5/22`); `FECHA` vacía en todas.

## 2. Modelo de datos (`db_datos_externos`, migración `0008`)

| Tabla | Una fila por | Campos |
|---|---|---|
| `ext_gob_obra` | fila de `BD-TABLERO` | `id_obra`, `codigo_indec` (NULL si no es localidad), `codigo_zona_rural` (`nnZR`), `ambito` (UNICA / COMPARTIDA - DPTO UNICO / COMPARTIDA / PROVINCIAL), `localidad`, `departamento`, `nombre`, `division`, `estado`, `categoria`, `avance` (0–1, NULL si `-`), `fecha_inicio`, `fecha_fin`, `monto` (**no se expone**, §0) |
| `ext_gob_visita` | visita | `origen` (`funcionarios` / `ministro`), `funcionario`, `fecha`, `codigo_indec`, `localidad`, `departamento`, `tipo` (normalizado: mayúsculas, sin espacios duros) |
| `ext_gob_visita_gobernador` | gobierno local | `codigo_indec`, `visitada` (bool), `fecha_ultima`, `rango_habitantes` |
| `ext_gob_reunion` | reunión | `id_reu`, `codigo_indec` (o `codigo_zona_rural`), `fecha`, `temas` (lista JSON de temas normalizados), `resolucion` |
| `ext_gob_convenio` | localidad × convenio | `codigo_indec`, `convenio` (nombre de la columna), `valor` (`SI` / `NO` / `NC` / NULL), `cantidad` (para polideportivos y parques), `detalle` (`AÑO RADIO`) |

`ext_gob_convenio` es en formato largo: una columna nueva de la planilla entra sin migración.

## 3. Sincronización

Mismo mecanismo que las Fases 2 y 3: `ext_gob_sync_log`, mismo endpoint IAM-only y mismo job
diario `sync-gob-transferencias` (`0 8 * * *`, `fuente=todas`). Fuentes nuevas:

| Fuente | Planilla | Pestañas | Freno |
|---|---|---|---|
| `obras` | 12 | `BD-TABLERO` | proporción (90 % de la carga anterior) |
| `recorridos` | 2 | `Recorrido`, `Visitas Gober` | proporción; `Visitas Gober` por localidades (≥ 400) |
| `recorridos_ministro` | 2.1 | `Recorrido` | proporción |
| `reuniones` | 1 | `Reuniones` | proporción |
| `convenios` | 6 | `Convenios` | localidades (≥ 400) |

- **Columnas excluidas por pestaña**: hoy `_Pestana.excluir` saltea una lista global de
  columnas sensibles. Pasa a ser una lista propia de cada pestaña (las de §0). El sync lee el
  encabezado y pide a Sheets solo los tramos de columnas permitidas, como en la Fase 3.
- Settings nuevos: `GOOGLE_SHEET_GOB_OBRAS_ID`, `..._RECORRIDOS_ID`, `..._RECORRIDOS_MINISTRO_ID`,
  `..._REUNIONES_ID`, `..._CONVENIOS_ID` en `cloudbuild.yaml`.

**Filas descartadas** (se informan como anomalía):
- Obras con `#N/A` en tipo o estado (7 filas, 3 obras), o con `ID_LOC` que no es código, zona
  rural ni `PROVINCIAL` (2 filas).
- Visitas con fecha futura o inválida, o sin código (el área confirmó que son errores de carga,
  paraguas §3.8). En la 2.1: 6 futuras, 1 inválida, 11 sin código.

**Anomalías informativas** (la fila se carga):
- Obras: estado `0` o `-` (se carga como "Sin dato"), división numérica (NULL).
- Visitas repetidas (funcionario + fecha + localidad): se cargan; la pantalla cuenta visitas.
- Visitas del Gobernador con SI y sin fecha.
- Reuniones con `ID_REU` repetido.
- Convenios: `POLIDEPORTIVO` mayor a 10 y `AÑO RADIO` que no es un año.

## 4. Qué expone al tablero

Ruta nueva **`GET /api/v1/datos-externos/tablero/gestion/{seccion}`** (config de gateway nueva),
secciones `obras`, `visitas`, `visitas-gobernador`, `reuniones` y `convenios`. Mismos roles
(`Admin` y `Autoridad`) y mismo filtro por código INDEC que el resto del tablero. Cada respuesta
trae todos los años y la pantalla elige (filtro de año propio, como en las Etapas 3 y 4).

- **Obras**: KPIs (obras, en ejecución, finalizadas, a ejecutar, avance promedio); por estado y
  por división; tablas como el Looker: obras de un solo departamento, de varios departamentos y
  obras generales (provinciales); con nivel localidad, obras de una localidad y de varias. Sin
  montos. Con filtro territorial, una obra compartida aparece en cada localidad que toca, y las
  provinciales no aparecen.
- **Visitas**: cobertura como `spec-tablero-gobierno-panel.md §2.5 #3`: localidades visitadas
  sobre el total, municipios y comunas (tipo MU/CO del nomenclador), "X de Y", y cobertura por
  departamento. También visitas por mes, por tipo y por funcionario (actividades, localidades,
  municipios, comunas), y la lista de visitas: fecha, localidad, departamento, funcionario y
  tipo. Las de la planilla 2.1 cuentan como del ministro.
- **Visitas del Gobernador**: visitadas y no visitadas, con cobertura total, por tipo, por
  departamento y por rango de habitantes; tablas de visitadas (con última visita) y de aún no
  visitadas.
- **Reuniones**: KPIs (reuniones, localidades), por mes, por tema (una reunión suma en cada uno
  de sus temas), por resolución, y tabla por localidad. Sin detalle. Aviso de que la planilla
  llega hasta diciembre de 2024 mientras el área no la actualice.
- **Convenios**: matriz localidad × convenio (SI/NO/NC y cantidades), porcentaje de "SI" por
  convenio, y lista de localidades que no firmaron un convenio elegido (el "listado de
  localidades por firmar" del Looker).
- **Inicio**: KPIs de localidades visitadas, visitas, reuniones y localidades visitadas por el
  Gobernador; bloque de convenios firmados. Se agregan a `/tablero/inicio`.

**Funcionarios**: la pantalla de visitas muestra el nombre del funcionario, como el Looker. Son
autoridades en ejercicio y el tablero lo ven solo `Admin` y `Autoridad`. Los documentos y el
chat lo siguen anonimizando.

## 5. Privacidad

| Dato | Tratamiento |
|---|---|
| `INTENDENTE`, `PARTIDO` (`Visitas Gober`) | No se leen (Etapa 6, ADR-027) |
| Pestaña `Compromisas` (nombre completo, firmante, partido, documentos) | No se lee |
| Pestaña `Intendentes` (celulares) | No se lee (Etapa 6) |
| `DETALLE REUNIÓN`, `DETALLE RESOLUCIÓN` | No se leen. Si la autoridad quiere el detalle, se agrega con otro spec |
| `TEMA`, `INSTITUCIÓN` de los recorridos | No se leen (texto libre que el Looker no muestra) |
| Funcionario de cada visita | Se guarda y se muestra (§4) |
| Montos de obras | Se guardan y no se exponen (§0) |

## 6. Pantallas (Etapa 5 del panel)

Las cinco subsecciones de **Gestión territorial** que hoy dicen "en preparación" (`secciones.ts`),
salvo IPCAM. Mismos bloques que el resto del tablero (KPIs, paneles, tablas con exportación a
Excel, aviso de fuente). Los mapas de cobertura de `spec-tablero-gobierno-panel.md §2.7` quedan
para la Etapa 7 (mejoras), igual que en las etapas anteriores.

## 7. Tests y verificación

- Parseo puro con filas sintéticas: `#N/A`, `nnZR`, `PROVINCIAL`, fechas en texto, `-`, temas
  múltiples, tipos de visita con espacios duros, `Si`/`si`, columnas excluidas que llegan vacías.
- Que ningún rango pedido a Sheets incluya columnas excluidas (como en la Fase 3).
- Agregados: cobertura por tipo y por departamento, obras por ámbito, una obra compartida
  contada una vez por departamento.
- Que la respuesta de obras no tenga montos.
- Verificación con las planillas reales contra Postgres descartable antes del deploy, con los
  conteos de §1.

## 8. Preguntas para el área (se suman al documento de pendientes)

1. **Reuniones**: la planilla termina en diciembre de 2024. ¿Se dejó de cargar o se carga en otro
   lado?
2. **Recorridos de funcionarios** (planilla 2): la última visita es del 3/3/2026. ¿Siguen
   cargándose?
3. **Recorridos del ministro** (planilla 2.1): 6 fechas futuras (hasta 2027) y 11 filas sin
   localidad reconocida.
4. **Obras**: 3 obras (`CAS-7`, `EPEC-169`, `EPEC-172`) con `#N/A`, y 2 filas con `ID_LOC` = 1.
5. **Visitas del Gobernador**: 10 localidades marcadas como visitadas sin fecha de última visita.
6. **Convenios**: una localidad con 56 polideportivos; `AÑO RADIO` con valores como `0891` o
   `5/22`.
7. Moneda de los montos de obras (ya pedido, C8).

## 9. Criterios de aceptación

1. El sync carga las cinco fuentes con los conteos de §1 menos los descartes de §3, sin leer
   ninguna columna excluida.
2. Las cinco pantallas responden a los filtros de departamento y localidad, y sus KPIs coinciden
   con el Looker con el mismo filtro, salvo las diferencias explicadas (descartes y montos).
3. Ninguna respuesta tiene montos de obras ni las columnas de §5.
4. Un usuario sin rol `Admin` ni `Autoridad` recibe 403.
