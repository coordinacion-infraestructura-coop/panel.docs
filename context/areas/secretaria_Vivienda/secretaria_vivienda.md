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

**Estado en producción (2026-07-05):**
- 54 municipios activos (46 originales + 8 nuevos incorporados en Panel #25: ETRURIA, MANFREDI, LA LAGUNA, CHAZON, AUSONIA, TIO PUJIO, SAN ANTONIO DE LITIN, CRUZ ALTA Marcos Juárez)
- Panel de referencia más reciente: `Panel_Cordon_Cuneta (25).html`
- Historial de cambios de estado operativo — tabla `viv_cc_estado_historial`
- Columnas en UI: orden, municipio, departamento, expediente, monto, CC ml, m² adoquinado, ok_gob, **Estado General**, Estado Jurídico, Estado Técnico, Estado Presup.

**Catálogo de estados (15 estados, workflow unificado CC y CH, migración 0009):**
1. Sin Iniciar — sin contacto ni documentación
2. Para Notificar — listo para enviar notificación
3. Notificado — se envió documentación de requerimientos por SUAC/CIDI
4. Sin Expediente de Gobierno — nota presentada pero sin ok de gobierno
5. A la espera de Documentación — contacto hecho, municipio trabajando en docs
6. En Revisión Técnica — documentación recibida, en análisis
7. En Corrección — se hicieron observaciones, municipio corrigiendo
8. Documentación Completa — todo ok
9. Administración para NP — va al área financiera interna para reserva del gasto
10. Para Firma de Convenio — en camino a firma
11. Convenio Firmado — firmado en coop antes de fin doc técnica
12. Legales para Proyecto de Dictamen y Resolución — legales interna de vivienda
13. Legales del MCyM — analiza legales del ministerio de coop
14. Administración OC — vuelve a legales de vivienda para orden de compra
15. TC — enviado al Tribunal de Cuentas

La unidad de análisis es el municipio, cada alta baja o modificación se hace sobre el municipio como unidad de análisis.
Al hacer click en un municipio se accede al detalle con dos pestañas: Comunicaciones (pedidos) e Historial de estados.
La pestaña de historial muestra cada transición de estado (campo, estado anterior, estado nuevo, fecha, actor).


### Programa Córdoba Hogar
Se firmó convenio recientemente. **ABM provisorio implementado y en producción (2026-07-02).**

**Estado actual:**
- Panel operativo en `https://gestorcooperativo.web.app/vivienda/cordoba-hogar`
- Backend: módulo `app/cordoba_hogar/` en svc-vivienda (migración 0006)
- 43 localidades seeded en producción
- Unidad de análisis: localidad (igual que Cordón Cuneta con municipios)
- Permite: ver localidades, registrar ok_gob, cantidad de casas, monto, expediente, pedidos

**Pendiente:** Reunión con el área para definir procedimientos administrativos, estados definitivos y campos. La estructura actual es provisoria y puede cambiar.

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