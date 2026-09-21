# Contexto detallado — Secretaría General de Gobierno (programa ATP)

> **⚠️ BORRADOR SIN VALIDAR CON EL ÁREA.** Según `docs/context/areas/README.md`,
> este archivo debería surgir de una reunión con el responsable del área — esa
> reunión **todavía no se hizo**. Todo lo que sigue es inferido únicamente del
> análisis del archivo `ATP - Compromiso Gobernador.xlsx` que el usuario
> proveyó (copia local de un Google Sheet real, id
> `1x3E73hlGwDPBiLc9fbQuWxWUxjaibs3glModHsxCMZM`), sin confirmar con nadie del
> área. Está pensado como punto de partida concreto para esa reunión, no como
> sustituto de ella. No se debe avanzar `docs/files/spec-svc-gralgob.md` a
> estado `approved` en base a este documento solo.

## 1. Responsable del área

**Desconocido.** No se relevó en esta sesión — pendiente de la reunión real.

## 2. Procesos actuales

El área lleva el seguimiento de compromisos de **ATP (Aporte del Tesoro
Provincial)** — anuncios de fondos comprometidos por el Gobernador a
localidades — en un Google Sheet ("ATP - Compromiso Gobernador"), uno de al
menos 16 pestañas mezcladas en el mismo libro (mismo patrón ya visto en
`SEC. GAS PIT.xlsx` de Gasifera). Se observan varios procesos/programas
distintos conviviendo en el mismo archivo:

1. **ATP - Compromiso Gobernador** (hoja `BD`, la única fuente de esta
   entrega): compromiso anunciado, a qué ministerio/secretaría se derivó su
   ejecución, monto, destino (obra/uso) y cronograma de pago mensual.
2. **Seguimiento de expedientes** (`Estado de exp`): expediente, fecha de
   ingreso/último estado, área que lo tiene ("MINISTERIO DE COOPERATIVAS Y
   MUTUALES", "MINISTERIO JEFATURA DE GABINETE") — parece un tracking de
   trámite administrativo, entidad distinta de `BD` (vinculable por
   `Nro. Expediente` cuando existe, pero `BD` casi nunca lo tiene cargado).
3. **Otros programas de fondos** (`ATP ACUMULADO`, `COPA+FOFINDES`, `SALDO`,
   `BD NATALIO`): nombres sugieren programas o fuentes de financiamiento
   *distintos* de ATP-Compromiso-Gobernador (COPA, FOFINDES, FOCOM aparecen
   como columnas/hojas propias) — no está claro si son predecesores
   históricos, programas paralelos vigentes, o llevados por otra persona
   ("BD NATALIO" sugiere autoría individual). **No se tocan en esta entrega.**
4. **Pivots/copias derivadas de `BD`** (`BD ORDENADA - Gobierno/Por
   ministerio/POR MES`, `Copia de BD`, `Hoja 31`): reordenamientos o filtros
   de la tabla maestra, algunos con errores visibles (`#N/A` en
   `BD ORDENADA - POR MES`) — no son fuente, son vistas derivadas dentro del
   propio Sheet.
5. **Catálogos de soporte** (`Validadores`, `Datos Localidad`, `DPTO`):
   listas para los desplegables de carga del propio Sheet. `Datos Localidad`
   en particular es un padrón (población, electores, intendente, partido) —
   **ya existe una fuente canónica equivalente** en `priv_localidades_info`/
   `priv_departamentos_info` (ADR-012, propiedad de `svc-privada`); no
   conviene duplicarlo acá.

No sabemos si estos procesos los lleva la misma persona o distintas áreas que
comparten el archivo — **pregunta para la reunión**.

## 3. Funcionalidades requeridas

El usuario indicó el flujo esperado (mismo patrón que Checklist Técnico DGV /
sync de Gasifera): primero espejo de solo lectura del Sheet (esta entrega),
después ABM propio donde el área cargue directo en el sistema en lugar del
Sheet. No se relevó con el área todavía qué campos serían editables, quién
carga hoy los compromisos, ni si hay un flujo de aprobación (anuncio →
derivación → pago) con roles distintos en cada paso. **No implementar el ABM
sin confirmar esto.**

## 4. Campos y datos

Ver el detalle completo de columnas, tipos y valores en
`docs/files/spec-sync-atp-compromiso-gobernador.md`. Resumen de lo que la
hoja `BD` gestiona hoy (~1110 compromisos, marzo 2026 en adelante):

- Identificación: sin clave de negocio única — el índice manual (columna
  `#`) y el `Nro. Expediente` (mayormente vacío) no sirven como clave.
- Territorial: `DEPARTAMENTO` (24 valores) / `LOCALIDAD` (246 valores).
- Ejecución: `Ministerio` — a qué área se derivó el compromiso para
  ejecutarlo (`Gobierno` en el 69% de los casos = queda dentro de la propia
  Secretaría; el resto se reparte entre Cooperativas y Mutuales, Vinculación
  y Gestión Institucional, Salud, Infraestructura y Servicios Públicos,
  Vivienda, Educación, y dos valores que parecen typos de área real
  — "Biagroindustria", "Habitat y desarrollo emprendendor").
- Gestión: `Fecha de anuncio`, `Nro. Expediente`, `Derivado` (booleano),
  `Monto`, `Destino` (texto libre: obra o uso del fondo).
- Financiero: `SALDO ATP` (calculado por el Sheet con una fórmula
  `FILTER` de Google Sheets — no se recalcula, se sincroniza el valor ya
  calculado), y un **cronograma de pago mensual** (34 columnas hoy, Marzo
  2025 → Diciembre 2027, y sigue creciendo hacia la derecha con el tiempo).

**No aparece en el Sheet ninguna entidad "cooperativa/municipio ejecutor"**
con datos propios — solo `LOCALIDAD`/`DEPARTAMENTO` como texto. Si el sistema
necesita gestionar el ejecutor como entidad propia, no está en esta fuente.

## 5. Estados y flujos

- `Ministerio` funciona como un semi-estado: `"Gobierno"` = el compromiso no
  se derivó, se ejecuta/trackea dentro de la propia Secretaría; cualquier
  otro valor = se derivó a otra área para su ejecución (y el `SALDO ATP` de
  esa fila se fuerza a 0 en el Sheet — el saldo de lo derivado, si se
  trackea, vive en otro lado).
- `Derivado` (booleano) parece corresponder 1:1 con "¿`Ministerio` != Gobierno?"
  pero no se verificó la correlación exacta fila por fila — **pregunta para
  la reunión**: ¿son dos campos redundantes o hay casos donde difieren
  (ej. derivado pero temporalmente vuelto a "Gobierno")?
- No hay una máquina de estados formal del compromiso en sí (anunciado →
  derivado → pagado): se infiere del `Monto`, `SALDO ATP` y las columnas
  mensuales, no de un campo de estado explícito.

## 6. Usuarios del sistema

Desconocido — no relevado. El Sheet se edita manualmente, sin roles
diferenciados visibles.

## 7. Integraciones — y preguntas abiertas detectadas en el análisis

Ninguna integración automática hoy (100% carga manual). Preguntas concretas
para la reunión, surgidas de problemas de calidad de datos reales:

1. **Alcance real del archivo**: ¿"ATP - Compromiso Gobernador" es solo la
   hoja `BD`, o el área espera que el sistema termine cubriendo también
   `Estado de exp` (expedientes) y/o los otros programas de fondos
   (`ATP ACUMULADO`, `COPA+FOFINDES`, `SALDO`, `BD NATALIO`)? Esta entrega
   asume que son alcances separados (posibles specs hijos futuros), no se
   asume nada de ellos todavía.
2. **`Ministerio` vs `Derivado`**: ¿son redundantes o hay reglas donde
   divergen? ¿Cuál es la lista cerrada real de ministerios/secretarías
   destino (para poder tratar `"Biagroindustria"`/`"Habitat y desarrollo
   emprendendor"` como typos de valores conocidos, en vez de adivinarlo)?
3. **`SALDO ATP` forzado a 0 cuando se deriva**: ¿el saldo de un compromiso
   derivado se trackea en otro sistema (el del área destino), o simplemente
   no se sigue más una vez derivado?
4. **Cronograma creciente**: el sync de esta entrega parsea cualquier columna
   con forma "NombreMes AA" de forma genérica (no hardcodea los 34 meses
   actuales) para no requerir migración cada vez que el área agregue una
   columna — ¿confirma el área que ese es el único patrón de columna nueva
   que agregan (nunca insertan una columna en el medio, ni cambian el
   formato del encabezado)?
5. **Vínculo con `Estado de exp`**: ¿tiene sentido cruzar `BD` con
   `Estado de exp` por `Nro. Expediente` a futuro, o son necesariamente
   independientes (dado que `BD` casi nunca tiene expediente cargado)?
6. **`Datos Localidad`**: ¿el área usa activamente esos datos (población,
   intendente, partido) para decidir montos/prioridad, o es un catálogo de
   apoyo heredado que se puede reemplazar por el padrón canónico de
   `svc-privada` (ADR-012) sin pérdida de información?

## 8. Prioridades

El usuario confirmó la prioridad inmediata: sincronizar `BD` (ATP) para que
sea consultable por localidad desde Resumen Territorial — la federación
concreta hacia ese panel es una fase posterior (spec y ADR propios, mismo
patrón que ADR-016 hizo con Privada), no parte de esta entrega. El resto
(ABM propio, expedientes, otros programas de fondos) queda sin priorizar
hasta la reunión real con el área.
