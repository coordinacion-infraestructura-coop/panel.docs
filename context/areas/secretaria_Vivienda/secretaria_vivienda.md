La Secretaría de Vivienda abarca las siguientes áreas y programas:
+ Dirección de vivienda:
    - Programas de Vivienda
        * Córdoba Hogar
        * Mi Lugar
    - Cordón Cuneta

+ Programa de Loteos

## Dirección de Vivienda

La dirección de vivienda es un área dentro del ministerio de Cooperativas y Mutuales cuyo funcionario a cargo es Garavaglia, funciona como unidad ejecutora independiente del ministerio por lo que tiene hacia adentro un área de:
- Presupuesto y contabilidad.
- Legal y técnica. 
- Área de análisis técnico.
- Área de gestión. 

En una primera instancia nuestro desarrollo estará dirigido al área de gestión. Este área lleva adelante la gestión y control del avance de los programas.

### Programa de Cordón Cuneta
Es el programa que está actualmente operativo. Es un ABM manejado desde un panel, que registra cambios de estado en cada una de las gestiones.

**Estado en producción (2026-07-13):**
- 54 municipios activos (46 originales + 8 nuevos incorporados en Panel #25; duplicados limpiados en julio 2026)
- Panel de referencia más reciente: `Panel_Cordon_Cuneta (25).html`
- Historial de cambios de estado operativo — tabla `viv_cc_estado_historial`
- Índice único activo: no pueden existir dos municipios con igual nombre y departamento (migración 0014)

**Funcionalidades del panel (UI):**
- Columnas: orden, municipio (clickeable), departamento, expediente, monto, CC ml, m² adoquinado, ok_gob, Última obs., **Últ. modif.**, Estado General, Estado Jurídico, Estado Técnico, Estado Presup., Avance, Acciones
- **Columnas ordenables**: click en cabecera ordena asc/desc. Estados se ordenan por posición en el workflow, no por ID.
- **Click en nombre del municipio**: abre directamente el panel lateral de historial y comunicaciones
- **Filtros**: por departamento (desde catálogo geo oficial), por OK Gobernación, por Estado General, búsqueda libre
- **Exportar a Excel**: botón "↓ Exportar (N)" descarga las filas filtradas/ordenadas como `.xlsx`
- **Alta de municipio**: valida nombre+departamento contra catálogo geo oficial (desplegable Depto→Municipio). Si el municipio ya existe, el backend devuelve 409 y el frontend muestra banner amber "Este municipio ya existe" con botón **Ir a editar** que abre el modal de edición directamente.
- **Edición**: mismo desplegable cascada para cambiar nombre/departamento
- **Fecha del cambio de estado**: en el modal de edición, campo de fecha junto al título "Estados por Dimensión" (default: hoy, max: hoy). Permite registrar la fecha real del cambio en el historial.

**Catálogo de estados (15 estados, workflow unificado CC y CH, migración 0009):**
1. Sin Iniciar | 2. Para Notificar | 3. Notificado | 4. Sin Expediente de Gobierno | 5. A la espera de Documentación | 6. En Revisión Técnica | 7. En Corrección | 8. Documentación Completa | 9. Administración para NP | 10. Para Firma de Convenio | 11. Convenio Firmado | 12. Legales para Proyecto de Dictamen y Resolución | 13. Legales del MCyM | 14. Administración OC | 15. TC

La unidad de análisis es el municipio. Al hacer click en el nombre del municipio (o en el ícono de historial) se accede al detalle con tres pestañas: Comunicaciones (pedidos), Historial de estados y Checklist Técnico.
La pestaña de historial muestra cada transición de estado (campo, estado anterior, estado nuevo, fecha, actor).

**Comunicaciones multi-área (implementado 2026-07-13, ampliado 2026-07-16):**

Jerarquía de visibilidad por secretaría:
- **vivienda**: ve solo las comunicaciones propias (secretaria='vivienda')
- **infraestructura**: ve vivienda + infraestructura (no supervision)
- **supervision** / **Admin**: ve todo (vivienda + infraestructura + supervision)
- Las comunicaciones de supervision son privadas al grupo — no las ve vivienda ni infraestructura

Usuarios del grupo supervision en producción:
- `infraestructura.coop@gmail.com` (secretarias: infraestructura + supervision)
- `lorena752aguilar@gmail.com` (secretaria: supervision, rol: Operador)
- `aguirrevictoriamariela@gmail.com` (secretaria: supervision, rol: Operador)

Badges: indigo "Infraestructura" / violeta "Supervisión". El botón "+ Nueva actualización" aparece para roles Admin/Supervisor/Operador o usuarios con secretaria infraestructura/supervision.

**En producción (2026-07-13): Checklist Técnico sincronizado desde Google Sheet.** El área técnica de Cordón Cuneta lleva su seguimiento documental en un Google Sheet propio (pestaña "Base TOTAL"), no en nuestro panel. Se sincroniza cada 15 min (Cloud Scheduler → endpoint interno IAM-protegido, solo lectura del Sheet) hacia `viv_cc_checklist_tecnico` + `viv_cc_checklist_items`, vinculada al municipio existente (100% de matching en la primera corrida: 46/46). Tercera pestaña "Checklist Técnico" en el detalle de cada municipio, solo lectura. Ver spec completo: `docs/files/spec-sync-cc-checklist-tecnico.md`.

**Resuelto (2026-07-16)**: las dos altas nuevas del Sheet ("Deán Funes", "La Laguna") que aparecían como error persistente por falta de Departamento ya fueron completadas por el área técnica — `filas_error=0` desde entonces.

**Incidente 2026-07-16 (resuelto)**: un valor de "REPARTICIÓN" más largo que la columna de la base (`VARCHAR(50)`) tumbó una corrida completa del sync — no solo esa fila, sino la corrida entera, sin dejar rastro en el log. Fix: columnas ensanchadas + aislamiento por fila con SAVEPOINT (migración `0018`, ver `docs/files/spec-sync-cc-checklist-tecnico.md` §13.5).


### Programa Córdoba Hogar
**ABM implementado y en producción (2026-07-02, mejoras julio 2026).**

**Estado actual (2026-07-13):**
- Panel operativo en `https://gestorcooperativo.web.app/vivienda/cordoba-hogar`
- Backend: módulo `app/cordoba_hogar/` en svc-vivienda
- 43 localidades en producción
- Unidad de análisis: localidad

**Funcionalidades del panel (UI):**
- Columnas: orden, localidad (clickeable), departamento, fecha anuncio, expediente, casas, monto, ok_gob, Última obs., **Últ. modif.**, Estado General, Estado Jurídico, Estado Técnico, Estado Presup., Avance, Acciones
- **Columnas ordenables**: click en cabecera ordena asc/desc
- **Click en nombre de localidad**: abre panel lateral de historial y comunicaciones
- **Filtros**: por departamento (catálogo geo oficial), por OK Gobernación, por Estado General, búsqueda libre
- **Exportar a Excel**: botón "↓ Exportar (N)" descarga filas filtradas/ordenadas como `.xlsx`
- **Alta con monto automático**: al ingresar `cantidad de casas` se calcula automáticamente `monto = cantidad_casas × monto_por_casa`. Muestra hint cyan con la fórmula. Campo monto es de sólo lectura.
- **Fecha del cambio de estado**: en el modal de edición, campo de fecha junto a "Estados por Dimensión" (default: hoy). Permite registrar la fecha real del cambio en el historial.
- **Edición**: desplegable cascada Depto→Localidad con catálogo oficial geo

**Parámetros configurables (botón ⚙ Parámetros, rol Supervisor/Admin):**
- Tab *Estados*: gestión del catálogo de estados (crear, editar, eliminar)
- Tab *Parámetros*: edición del valor `monto_por_casa` (actualmente $34.000.000). Se guarda en tabla `viv_ch_config` y afecta el cálculo automático en nuevas altas.

**Comunicaciones multi-área (implementado 2026-07-13):** Mismo comportamiento que CC — usuarios de coord infraestructura agregan y ven todas las comunicaciones; vivienda solo las propias. Badge indigo para comunicaciones de infraestructura, nombre real del autor en cada entrada.

**Pendiente:** Reunión con el área para validar estados definitivos y flujo administrativo. La estructura puede ajustarse.

### Programa Mi Lugar
Programa de expropiaciones, se creó una unidad ejecutora, hay un panel provisorio. 
Idem Córdoba Hogar, el servicio será un ABM manejado desde un panel, que registra cambios de estado en cada una de las gestiones. 
El panel provisorio esta en {pegar ruta.html}

## Unidad Ejecutora de Loteos
### Programa Loteos 
Tenemos distintos  tipos de loteos 
	por expropiaciones
	por tierras provinciales -fiscales
	tierras municipales
	tierras de cooperativas y gremios -> caso AGEC en rio cuarto. El min ayuda a construir la infraestructura
El servicio será un ABM por tipo de loteo donde se irán registrando los avances en cada área.  