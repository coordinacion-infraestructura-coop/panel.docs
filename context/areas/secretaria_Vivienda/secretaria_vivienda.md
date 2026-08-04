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
- **Estado General — 100% manual (desde 2026-07-31)**: el campo Estado General se gestiona solo desde el cuadro de edición. No existe fórmula automática que lo calcule a partir de los sub-estados (Jurídico, Técnico, Presupuestario). Al crear un nuevo municipio, Estado General queda en `null` hasta que el usuario lo setee explícitamente.

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
- **Monto automático (alta y edición)**: tanto en el modal de alta como en el de edición, al ingresar o modificar `cantidad de casas` se auto-calcula `monto = cantidad_casas × monto_por_casa`. Muestra hint cyan "= X casas × $Y" cuando el monto coincide con el cálculo. Monto es editable para override manual. En el modal de alta el campo monto es solo lectura; en edición es editable.
- **Fecha del cambio de estado**: en el modal de edición, campo de fecha junto a "Estados por Dimensión" (default: hoy). Permite registrar la fecha real del cambio en el historial.
- **Estado General — 100% manual (desde 2026-07-31)**: igual que en CC, el Estado General se gestiona solo desde el cuadro de edición. No hay fórmula automática.
- **Comunicaciones multi-área (implementado 2026-07-13):** mismo comportamiento jerárquico que CC — supervision/Admin ven todo, infraestructura ve vivienda+infraestructura, vivienda solo las propias.
- **Edición**: desplegable cascada Depto→Localidad con catálogo oficial geo

**Parámetros configurables (botón ⚙ Parámetros, rol Supervisor/Admin):**
- Tab *Estados*: gestión del catálogo de estados (crear, editar, eliminar)
- Tab *Parámetros*: edición del valor `monto_por_casa` (actualmente $34.000.000). Se guarda en tabla `viv_ch_config` y afecta el cálculo automático en nuevas altas.

Badges: indigo "Infraestructura" / violeta "Supervisión". Nombre real del autor en cada entrada.

**Pendiente:** Reunión con el área para validar estados definitivos y flujo administrativo. La estructura puede ajustarse.

### Programa Mi Lugar
**Análisis completado 2026-08-04 a partir de `Panel_MI_LUGAR (69).html`.**

Programa de adquisición de tierras para vivienda. Tres tipos de gestiones bajo un panel unificado con tabs:
- **Expropiaciones** (🏗): tierras que pasarán a dominio provincial — 21 proyectos activos
- **Convenios con Municipios** (🤝): cesión de tierras municipales al programa — 15 proyectos
- **Lotes Provinciales** (🏛): tierras ya en dominio del Estado Provincial — 13 proyectos

**Estado de desarrollo:** análisis completo, pendiente spec + código. Decisiones de arquitectura acordadas con el usuario (2026-08-04).

**Funcionalidades del panel (UI — planificado):**
- Tabs por tipo, con color propio: Exp=#B03A2E / Muni=#1E8449 / Prov=#1A5276
- Columnas: #, Nombre/Obs. (clickeable), Localidad, Expediente, Geolocalización (links), Superficie, Lotes, E.Jurídico, E.Técnico, E.Presup., Monto, Avance, Acciones
- Columnas adicionales en Exp y Prov: Valor Fiscal, INFRA s/Nexos, Nexos, UNC, Costo Total INFRA
- **Click en nombre del proyecto**: abre panel lateral con historial de estados + comunicaciones multi-área (igual CC/CH)
- **Filtros**: búsqueda libre, localidad (select dinámico), estado
- **Exportar a Excel**: mismo patrón CC/CH
- **Estado General — 100% manual**: mismo comportamiento que CC/CH (campo nullable seteado manualmente en EditModal). Pendiente confirmación con área.
- **Avance**: calculado automáticamente como promedio de posición de las 3 dimensiones en el catálogo (≠ CC/CH donde Estado General es manual)
- **Botón ⚙ Parámetros** (Supervisor/Admin): Tab Estados (ABM catálogo) + Tab Parámetros (tipo_cambio, usd_por_lote)

**Localidad (campo dual):**
- `localidad_id` FK nullable a `viv_geo_localidades` — vincula con CC/CH cuando coincide con catálogo
- `localidad_nombre` texto libre siempre requerido — para proyectos en barrios o sectores sin catálogo (ej: "Barrio Chingolo", "Capital - Villa Retiro Sur")
- Modal usa desplegable cascada Depto→Localidad + opción "Otra localidad / Barrio" con campo libre

**Datos geográficos:**
- **NO son centroides de `viv_geo_localidades`** — son vértices/puntos de referencia del predio específico (hasta 12 puntos por proyecto en casos complejos como Cruz del Eje)
- Almacenados en tabla separada `viv_ml_geo_puntos` (1:N con proyectos)
- Input en modal: URL de Google Maps o líneas de `lat,lng` (una por punto)

**Catálogo de estados (17 estados, workflow adquisición tierras — diferente de CC/CH):**
1. Sin Iniciar | 2. Nota del Municipio al MCyM | 3. Pase Sec. Gestión Infra | 4. Intervención Min. Infra | 5. Informe Prefactibilidad | 6. Dir. Reg. Obras VF+30% | 7. Para Evaluación Tasación | 8. Adm. DGV NP | 9. Legales Proy. Resolución | 10. Legales MCyM Dictamen | 11. Adm. DGV OC | 12. TC | 13. Min. Economía | 14. Sec. Gral. Gobernación | 15. Consejo Gral. Tasaciones | 16. COMPLETADO | 17. PROCURACION

**Superficie y lotes — auto-cálculo (confirmado por área 2026-08-04):**
- `superficie` = `NUMERIC(10,4)` en Ha (no texto libre — el área confirmó normalizar a Ha)
- `lotes_por_ha = 25` nuevo parámetro configurable en ⚙ Parámetros
- Al ingresar `superficie`, auto-calcula `lotes = superficie × lotes_por_ha` (overrideable manualmente — mismo patrón UX que monto en CH)
- Hint cyan "= X Ha × 25 lotes/Ha" cuando el valor coincide con el auto-calculado
- **Pendiente confirmar con área**: si 1 Ha = siempre 25 lotes o hay excepciones por tipo de terreno

**Modelo financiero — cadena de auto-cálculos (constantes configurables en ⚙ Parámetros):**
- `TIPO_CAMBIO = $1.450/US$` → `infra_sin_nexos = lotes × 10.000 × tipo_cambio` (Exp + Prov)
- `USD_POR_LOTE = US$10.000`
- `LOTES_POR_HA = 25`
- `valor_expropiacion = valor_fiscal × 1.30` (solo Exp, auto readonly)
- `costo_total_infra = valor_expropiacion + infra_sin_nexos + costo_nexos + convenio_unc`

**4to tipo de proyecto (confirmado por área 2026-08-04):**
- Existe (tierras cooperativas u otro) pero aún indefinido en su estructura
- Columna `tipo` sin CHECK constraint fijo para permitir extensión sin DDL complejo
- Documentado para implementar en próxima versión cuando el área lo defina

**Migración de datos históricos:**
- Los 49 registros del boceto se migran en script de seed (parte de migración 0020 o script aparte)
- Seed data extraída del HTML parseando los `<tr>` de cada `<tbody>`

**Pendiente confirmar con área:**
- Si 1 Ha = siempre 25 lotes o hay excepciones por tipo de terreno
- Las expropiaciones de Capital (ej: Villa Retiro Sur) — ¿van en sub-tipo `exp` o necesitan un tipo nuevo?

**Comunicaciones multi-área:** mismo comportamiento que CC/CH — supervisión/Admin ven todo, infraestructura ve vivienda+infraestructura, vivienda solo las propias.

## Unidad Ejecutora de Loteos
### Programa Loteos 
Tenemos distintos  tipos de loteos 
	por expropiaciones
	por tierras provinciales -fiscales
	tierras municipales
	tierras de cooperativas y gremios -> caso AGEC en rio cuarto. El min ayuda a construir la infraestructura
El servicio será un ABM por tipo de loteo donde se irán registrando los avances en cada área.  