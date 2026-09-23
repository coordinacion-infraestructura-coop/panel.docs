# Contexto detallado — Secretaría de Infraestructura Gasífera

> **⚠️ BORRADOR PARCIALMENTE VALIDADO.** La **primera reunión** con el área ya se
> hizo (2026-09-21) y confirmó 2 puntos puntuales (marcados `✅ confirmado en
> reunión` más abajo: alcance y catálogo geográfico). El resto de este documento
> (responsable del área, procesos, funcionalidades, estados reales, usuarios,
> prioridades — §1, §3, §5, §6, §8) **sigue sin relevar** — no hubo tiempo/agenda
> para cubrirlo en esa primera reunión. `SPIP` (§7.1) quedó sin respuesta, a
> re-consultar. **No se debe avanzar el spec de `svc-gasifera` a estado
> `approved` todavía** — falta demasiado para eso. Lo que sí se aprobó, como
> excepción puntual y documentada, es un panel de solo lectura sobre los datos ya
> sincronizados (`gas_pit_*`) — ver `docs/files/spec-sync-gasifera-pit.md §12`.

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
que comparten el mismo Sheet — **pregunta para la próxima reunión** (no cubierta en la primera).

**✅ confirmado en reunión (2026-09-21)**: el alcance del sistema, por ahora, es
**solo obras de gas** — coincide con lo ya sincronizado/construido. Las otras 4
categorías del tablero (vial, agua/cloaca, eléctrica, arquitectura) "probablemente"
se aborden en el futuro (palabras del usuario), pero no está comprometido ni tiene
fecha — no corresponde empezar a construirlas todavía.

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

1. **⏳ preguntado en la reunión, sin respuesta todavía** — ¿qué es `SPIP` y por qué
   se repite entre varias filas (obras multi-tramo)? ¿Existe un identificador de obra
   realmente único en algún sistema del área (SPIP en su sentido pleno — "Sistema
   Provincial de Inversión Pública" es una hipótesis, no confirmado)? El usuario va
   a re-consultarlo con el área.
2. **✅ confirmado en reunión**: el catálogo de Departamento/Localidad vigente **no**
   es ninguno de los dos que trae el propio Sheet (hoja `Desplegables`, 14 vs 27
   departamentos, ~437 vs ~450 localidades, no sincronizados entre sí) — el
   catálogo real y correcto es **`geo_localidades`/`info_localidades`** (los
   catálogos ya existentes en el sistema — `viv_geo_localidades` de `svc-vivienda`
   y `priv_localidades_info`/`priv_departamentos_info` de `svc-privada`, ADR-012).
   Implicancia para el diseño futuro: cualquier resolución de localidad/departamento
   debe apoyarse en esos catálogos canónicos (vía federación cross-service, mismo
   patrón que ADR-012/016), no en los catálogos internos del Sheet ni en uno nuevo
   propio de `svc-gasifera`. No implementado todavía (fuera de alcance del panel
   preliminar de solo lectura, que muestra el texto crudo del Sheet tal cual).
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
7. **✅ resuelto** — ver la confirmación de alcance en §2 (solo gas por ahora).

## 8. Prioridades

Desconocido — no relevado. A confirmar en la reunión.
