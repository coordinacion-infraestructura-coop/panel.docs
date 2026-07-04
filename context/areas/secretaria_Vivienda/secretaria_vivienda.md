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
El panel está en docs\context\areas\secretaria_Vivienda\Panel_Cordon_Cuneta.html 

El significado de cada  valor de Estado es el siguiente:
Sin iniciar -> No han tenido ningun contacto desde el área, o bien está iniciado pero el área no se contactó o bien ni siquiera presentó nada el municipio
Sin expediente de gobierno -> hay una nota pero no el ok de gobierno, es necesario definir si lo tendremos o no
Notificado -> se le envió la documentación de los requerimientos pulidos, por suac. Fecha en la que se notificó x cidi
A la espera de documentación -> en algún momento se hizo contacto, hay certeza de que recibieron los instructivos nuevos y están trabajando en ello
en revición técnina -> volvió esa documentación y lo estamos analizando (puede ser alguna de las veces q volvió la documentación
En correción -> se le hicieron observaciones y lo tiene el municipio corrigiendolo
Documentación completa -> está todo ok
Legales para convenio -> una vez que la documentación está todo ok se envía a min coop para firma de convenio
Administración para NP -> una vez que está firmado el convenio va al área financiera interna para reserva del gasto
Convenio firmado -> en coop se firmó el convenio antes de finalizar la documentación técnica
Legales para proyecto de dictamen y reso -> legales interna de vivienda
legales del MCyM -> lo analiza legales del ministerio de coop
Administración DGV OC -> vuelve a legales de vivienda para hacer la orden de compra
TC -> Se envía al tribunal de cuentas
Visado TC -> si vuelve visado por el tribunal de cuentas (aprobado) ahi pasa al director de infraestructura para el inicio de obra

La unidad de análisis es el municipio, cada alta baja o modificación se hace sobre el municipio como unidad de análisis.
Además debemos permitir registrar una actualización de estados por pedidos realizados, al hacer click en un municipio se debe acceder a todo el detalle de las comunicaciones 
que se han tenido con este municipio. Allí debe haber un botón de actualización de pedidos, en donde se registre en una caja de texto qué fue lo que se le pidió por última vez y la fecha
la fecha por deffault debe cargarse la fecha de registro de esta actualización pero se debe poder modificar, ya que es probable que hoy cargue un pedido que hice ayer. 


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