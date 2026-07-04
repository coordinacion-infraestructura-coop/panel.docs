## Orquestación del Flujo de Trabajo

### 1. Nodo de Planificación por Defecto
- Entra en modo de planificación para CUALQUIER tarea no trivial (más de 3 pasos o decisiones arquitectónicas).
- Si algo sale mal, DETENTE y vuelve a planificar de inmediato; no sigas forzando.
- Usa el modo de planificación para los pasos de verificación, no solo para la construcción.
- Escribe especificaciones detalladas de antemano para reducir la ambigüedad.

### 2. Estrategia de Subagentes
- Usa subagentes generosamente para mantener limpia la ventana de contexto principal.
- Delega la investigación, exploración y el análisis paralelo a los subagentes.
- Para problemas complejos, asigna más capacidad de cómputo mediante subagentes.
- Una tarea por subagente para una ejecución enfocada.

### 3. Bucle de Automejora
- Después de CUALQUIER corrección del usuario: actualiza `tasks/lessons.md` con el patrón detectado.
- Escribe reglas para ti mismo que eviten cometer el mismo error.
- Itera implacablemente sobre estas lecciones hasta que la tasa de errores disminuya.
- Revisa las lecciones al iniciar la sesión para el proyecto correspondiente.

### 4. Verificación antes de Finalizar
- Nunca marques una tarea como completada sin demostrar que funciona.
- Compara (diff) el comportamiento entre la rama principal y tus cambios cuando sea relevante.
- Pregúntate: "¿Aprobaría esto un ingeniero de nivel *staff*?"
- Ejecuta pruebas, revisa registros (logs) y demuestra la corrección del código.

### 5. Exigir Elegancia (Equilibrado)
- Para cambios no triviales: haz una pausa y pregunta "¿hay una forma más elegante?".
- Si una solución parece un "parche" (hacky): "Sabiendo todo lo que sé ahora, implementa la solución elegante".
- Omite esto para correcciones simples y obvias; no caigas en el exceso de ingeniería.
- Desafía tu propio trabajo antes de presentarlo.

### 6. Corrección Autónoma de Errores
- Cuando recibas un informe de error: simplemente arréglalo. No pidas que te lleven de la mano.
- Identifica logs, errores y pruebas fallidas; luego, resuélvelos.
- Cero necesidad de cambio de contexto por parte del usuario.
- Arregla las pruebas de CI fallidas sin que se te indique cómo hacerlo.

---

## Gestión de Tareas

1. **Planificar Primero**: Escribe el plan en `tasks/todo.md` con elementos verificables.
2. **Verificar Plan**: Confirma antes de comenzar la implementación.
3. **Seguimiento del Progreso**: Marca los elementos como completados a medida que avanzas.
4. **Explicar Cambios**: Resumen de alto nivel en cada paso.
5. **Documentar Resultados**: Añade una sección de revisión en `tasks/todo.md`.
6. **Capturar Lecciones**: Actualiza `tasks/lessons.md` tras las correcciones.

---

## Principios Fundamentales

- **Simplicidad Primero**: Haz que cada cambio sea lo más simple posible. Afecta al mínimo código necesario.
- **Sin Pereza**: Encuentra las causas raíz. Nada de soluciones temporales. Estándares de desarrollador sénior.
- **Impacto Mínimo**: Los cambios solo deben tocar lo estrictamente necesario. Evita introducir errores.