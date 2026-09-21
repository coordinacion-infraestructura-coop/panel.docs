# Contexto detallado — Secretaría de Infraestructura Gasífera

> **⚠️ BORRADOR SIN VALIDAR CON EL ÁREA.** Según `docs/context/areas/README.md`, este
> archivo debería surgir de una reunión con el responsable del área — esa reunión
> **todavía no se hizo** (estado "⏳ pendiente"). Todo lo que sigue es inferido
> únicamente del análisis del archivo `SEC. GAS PIT.xlsx` que el usuario proveyó,
> sin confirmar con nadie del área. Está pensado como punto de partida concreto
> para esa reunión, no como sustituto de ella. No se debe avanzar el spec de
> `svc-gasifera` a estado `approved` en base a este documento solo.

## 1. Responsable del área

**Desconocido.** No se relevó en esta sesión — pendiente de la reunión real.

## 2. Procesos actuales

El área (o al menos quien mantiene la planilla "SEC. GAS PIT") lleva el seguimiento de
obras de infraestructura provincial en un Google Sheet compartido, con al menos 11
pestañas. El Sheet no es exclusivo de gas: es un tablero de toda la Secretaría de
Infraestructura con 5 categorías de obra (vial, agua/cloaca, eléctrica, arquitectura y
gas); quien nos compartió el archivo trabaja específicamente con la porción de gas,
filtrando/consultando ese tablero general (de ahí el nombre "SEC. GAS PIT" — hipótesis:
PIT = Plan de Inversión/Infraestructura Territorial).

Se observan al menos tres procesos distintos mezclados en el mismo libro:
1. **Seguimiento de obras** (`MATRIZ (NO TOMAR)`): expediente, contratista, montos,
   plazos, avance — más orientado a control de gestión/presupuesto.
2. **Seguimiento territorial** (`ACCIONES TERRITORIO`): hitos por localidad ("Segunda
   etapa red de gas", "Inaugurado"), con estado Cumplido/Pendiente/En ejecución —
   más orientado a agenda política/territorial.
3. **Comunicación institucional** (`Anuncios`, `Inauguraciones`): registro de anuncios
   de prensa y eventos de agenda de funcionarios, con links a notas de prensa.

No sabemos si estos tres procesos los lleva la misma persona/equipo o distintas áreas
que comparten el mismo Sheet — **pregunta para la reunión**.

## 3. Funcionalidades requeridas

No relevado directamente. Por analogía con lo que terminó necesitando Checklist Técnico
DGV (que también arrancó como un Excel), es razonable esperar que el área quiera, a
futuro: ABM de obras, filtros/búsqueda, reportes por departamento/región/estado,
y quizás un tablero tipo Resumen Territorial. **No implementar nada de esto sin
confirmarlo** — es una hipótesis para orientar la conversación, no un requerimiento.

## 4. Campos y datos

Ver el detalle completo de columnas, tipos y valores enum en
`docs/files/spec-sync-gasifera-pit.md` (sync) y `docs/files/spec-svc-gasifera.md`
(dominio completo). Resumen de lo que el Sheet real gestiona hoy para obras de gas:

- Identificación: `SPIP` (no único, con datos corruptos — ver §5), `EXPEDIENTE`.
- Clasificación: división (ej. "HYG"), tipo de obra ("GASODUCTOS"), sub-tipo
  ("E- OBRAS DE GAS").
- Ejecución: contratista, estado de obra, avance (0-1), plazos (licitación,
  vencimiento, plazo original/vigente).
- Montos: contrato base, ampliación, enmienda, importe actualizado (ARS y USD).
- Gestión: prioridad presupuestaria, categoría (1-4), región, si está "autorizada
  2025", si está en el "PIT".
- Territorial: localidad (frecuentemente multivalor), departamento.
- Seguimiento de hitos (`ACCIONES TERRITORIO`): fecha, acción, detalle, estado,
  monto solicitado, comentarios.

**No aparece en el Sheet ninguna entidad "cooperativa ejecutora"** con datos propios
(CUIT, contacto, etc.) — solo el valor fijo `"Cooperativas y mutuales"` en la columna
`Ministerio` de `ACCIONES TERRITORIO`. Si el sistema necesita gestionar cooperativas
como entidad (como Cordón Cuneta gestiona municipios), ese dato no existe hoy en esta
fuente y hay que relevarlo aparte.

## 5. Estados y flujos

- `ESTADO DE OBRA` (obra): 9 valores, con 2 casi-duplicados por formato
  (`"PROC. DE ADJUDICACION"` / `"EN PROCESO DE ADJUDICACIÓN"`) que el sync colapsa.
- `ESTADO-RESUMEN` (obra): 3 valores agregados, aparentemente derivado de
  `ESTADO DE OBRA` para reporting simplificado.
- `PRIORIDAD` (obra): 12 valores, es un estado de gestión presupuestaria
  ("PENDIENTE", "PEDIDO 1", "SIN FINANCIAMIENTO"...), no un ranking numérico.
- `Estado` (acción territorial): 3 valores limpios (`Cumplido`/`Pendiente`/`En
  ejecución`).
- No hay ninguna máquina de estados formal documentada en el Sheet — los valores
  parecen cargarse a criterio de quien edita cada fila. **Pregunta para la reunión**:
  ¿hay un flujo real detrás de `ESTADO DE OBRA` (qué transiciones son válidas), o es
  informativo libre?

## 6. Usuarios del sistema

Desconocido — no relevado. El Sheet en sí es editado manualmente, sin roles
diferenciados visibles (no hay protección de rangos ni historial de quién cambió qué,
más allá del historial nativo de Google Sheets).

## 7. Integraciones — y preguntas abiertas detectadas en el análisis

Ninguna integración automática hoy (100% carga manual). Preguntas concretas que
conviene llevar a la reunión, surgidas de problemas de calidad de datos reales
detectados en el Excel:

1. ¿Qué es `SPIP` y por qué se repite entre varias filas (obras multi-tramo)? ¿Existe
   un identificador de obra realmente único en algún sistema del área (SPIP en su
   sentido pleno — "Sistema Provincial de Inversión Pública" es una hipótesis, no
   confirmado)?
2. Hay **dos catálogos distintos de Departamento** y **dos de Localidad** dentro de la
   misma hoja `Desplegables` (14 vs 27 departamentos; ~437 vs ~450 localidades), cada
   uno usado por una pestaña distinta y no sincronizados entre sí. ¿Cuál es el
   vigente/correcto?
3. La columna `ALERTA_LOCALIDAD` (validación cruzada de localidad contra catálogo)
   está calculada solo en 277 de 1077 filas — parece haberse corrido una vez y no
   recalculado más. ¿Sigue siendo relevante?
4. ¿El área reordena manualmente filas en `ACCIONES TERRITORIO`? Afecta directamente
   la estabilidad del sync (ver `spec-sync-gasifera-pit.md §5.3`).
5. La hoja `Hoja 11` (catálogo geográfico con latitud/longitud, sin encabezados) —
   ¿de dónde sale, se mantiene actualizada, quién es dueño de esos datos?
6. ¿Las otras ~14 planillas que mantiene el área tienen una estructura similar (misma
   familia de columnas) o son heterogéneas? Esto determina si el patrón de esta
   primera sync es reutilizable directamente o si cada una necesita su propio análisis.
7. Confirmar el alcance: ¿el objetivo es un sistema solo para obras de **gas**, o el
   área espera que esto termine cubriendo las 5 categorías de obra del tablero
   general (lo cual correspondería más a una futura `svc-infraestructura`)?

## 8. Prioridades

Desconocido — no relevado. A confirmar en la reunión.
