El siguiente sistema será un sistema integral para el manejo político y administrativo del Ministerio de Cooperativas y Mutuales. 

El ministerio tiene el siguiente organigrama, donde dentro de cada secretaría tenemos cada área de trabajo y programa:

* Secretaría de Vivienda
    Dirección de vivienda:
        - Programas de Vivienda
            * Córdoba Hogar
            * Mi Lugar
        - Cordón Cuneta

    Programa de Loteos

* Secretaría de Gestión y vinculación de Infraestructura - Luis Molinari 
    - Infraestructura Eléctrica
    - Agua y Saneamiento 

* Secretaría de Planificación y Articulación Territorial - Gabriel Fizza
    - Programa de Fortalecimientos para Cooperativas


* Secretaría de Desarrollo - Domingo Benso
    - Servicio de gestión de cooperativas (desarrollado por utn)

* Secretaría de Infraestructura Gasífera
    - Programa de conexión de Gas en Escuelas.
    - Asesoramientos legales y contables
    - Créditos para desarrollo de infraestructura hecho por cooperativas.  


* Secretaría Privada Ministro
    - Servicio de gestión de demandas. 

#Login
El login debería llevar a un panel central donde se puedan elegir por secretarías y dentro de cada secretaría los programas que tienen asociados
La idea es que de acuerdo a los diferentes usuarios les demos permisos a de acceso y a ver/editar/eliminar los diferentes paneles. En una primera instancia solo contamos con usuario admin que tiene todos los permisos.

Es necesario crear un panel de administración de usuarios al que solo tenga acceso el usuario admin. El panel debe permitir generar, editar y asignar permisos a usuarios, los usuarios deben ser creados asignándoles la secretaría y área a la que pertenecen. Luego solo deben poder ver los paneles que vinculados a la secretaría/área a la que pertenecen. 
El layout de componentes solo debe mostrar los componentes vinculados a su secretaría/área no todos. 
Cada secretaría debe tener estos usuarios roles:
- Supervisor (puede ver, editar, crear y eliminar casos)
- Operador (puede ver, editar y crear pero no eliminar)
- Visualizador (solo puede ver)
Creemos los usuarios con este tipo de nombre:
usuario: secretariaXXXXXX-{rol}@gmail.com 
contraseña: {rol}2026

#General
En general la página debe seguir la estética de la página docs\context\estetica gobierno\Ministerio de Cooperativas y Mutuales · Gobierno de Córdoba.html que son los colores de gobierno, además en la carpeta docs\context\estetica gobierno descargué todos los archivos de la página oficial de gobierno para que los revises y utilices si te son útiles. Prioricemos la simpleza y la velocidad de carga de la página, no son necesarias imágenes ni cosas que hagan lento la carga del frontend por ahora. Además todos los servicios deben ser responsivos y muy accesibles. 
La gente que lo utilizará son administrativos sin expertiz técnico, debe ser fácil para ellos. 


