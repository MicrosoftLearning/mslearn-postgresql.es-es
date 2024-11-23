---
lab:
  title: Enumeración de cambios de tabla con descodificación lógica
  module: Understand write-ahead logging
---

# Enumeración de cambios de tabla con descodificación lógica

En este ejercicio, configurará la replicación lógica, que es nativa de PostgreSQL. Creará dos servidores, que actúan como publicador y suscriptor. Los datos de zoodb se replicarán entre ellos.

## Antes de comenzar

Debe tener una suscripción a Azure propia para completar este ejercicio. Si no tiene una suscripción a Azure, puede obtener una [evaluación gratuita de Azure](https://azure.microsoft.com/free).

## Creación de un grupo de recursos

1. Inicie sesión en Azure Portal. La cuenta de usuario debe tener el rol Propietario o Colaborador para la suscripción de Azure.
1. Seleccione **Grupos de recursos** y, después, Seleccione **+ Crear**.
1. Seleccione su suscripción.
1. En Grupo de recursos, escribe **rg-PostgreSQL_Replication**.
1. Seleccione una región cercana a su ubicación.
1. Seleccione **Revisar + crear**.
1. Seleccione **Crear**.

## Creación de un servidor de publicador

1. En Servicios de Azure, seleccione **Crear un recurso**. En **Categorías**, seleccione **Bases de datos**. En **Azure Database for PostgreSQL**, seleccione **Crear**.
1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Suscripción: su suscripción.
    1. Grupo de recursos: selecciona **rg-PostgreSQL_Replication**.
    1. Nombre del servidor: **psql-postgresql-pub9999** (el nombre debe ser único globalmente, por tanto, reemplaza 9999 por números aleatorios).
    1. Región: seleccione la misma región que el grupo de recursos.
    1. Versión de PostgreSQL: selecciona 16.
    1. Tipo de carga de trabajo: **Desarrollo**.
    1. Proceso y almacenamiento: **Ampliable**. Seleccione **Configurar servidor** y examine las opciones de configuración. No realice ningún cambio ni cierre la sección.
    1. Zona de disponibilidad: 1. Si no se admiten zonas de disponibilidad, deje la opción Sin preferencia.
    1. Alta disponibilidad: deshabilitada.
    1. Método de autenticación: solo autenticación de PostgreSQL.
    1. En **nombre de usuario de administrador**, escribe **`demo`**.
    1. En **contraseña**, escribe una contraseña compleja adecuada.
    1. Seleccione **Siguiente: Redes >**.
1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Método de conectividad: (o) Acceso público (direcciones IP permitidas).
    1. **Permitir acceso público a este servidor desde cualquier servicio de Azure dentro de Azure**: activado. Debe estar activado para que las bases de datos de publicador y suscriptor puedan comunicarse entre sí.
    1. En Reglas de firewall, seleccione **+Agregar dirección IP del cliente actual**. Esto agrega la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall.
1. Seleccione **Revisar + crear**. Seleccione **Crear**.
1. Dado que la creación de una instancia de Azure Database for PostgreSQL puede tardar unos minutos, comienza con el paso siguiente en cuanto esta implementación esté en curso. Recuerda abrir una nueva ventana o pestaña del explorador para continuar.

## Creación de un servidor de suscriptor

1. En Servicios de Azure, seleccione **Crear un recurso**. En **Categorías**, seleccione **Bases de datos**. En **Azure Database for PostgreSQL**, seleccione **Crear**.
1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Suscripción: su suscripción.
    1. Grupo de recursos: selecciona **rg-PostgreSQL_Replication**.
    1. Nombre del servidor: **psql-postgresql-sub9999** (el nombre debe ser único globalmente, por tanto, reemplaza 9999 por números aleatorios).
    1. Región: seleccione la misma región que el grupo de recursos.
    1. Versión de PostgreSQL: selecciona 16.
    1. Tipo de carga de trabajo: **Desarrollo**.
    1. Proceso y almacenamiento: **Ampliable**. Seleccione **Configurar servidor** y examine las opciones de configuración. No realice ningún cambio ni cierre la sección.
    1. Zona de disponibilidad: 2. Si no se admiten zonas de disponibilidad, deje la opción Sin preferencia.
    1. Alta disponibilidad: deshabilitada.
    1. Método de autenticación: solo autenticación de PostgreSQL.
    1. En **nombre de usuario de administrador**, escribe **`demo`**.
    1. En **contraseña**, escribe una contraseña compleja adecuada.
    1. Seleccione **Siguiente: Redes >**.
1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Método de conectividad: (o) Acceso público (direcciones IP permitidas)
    1. **Permitir acceso público a este servidor desde cualquier servicio de Azure dentro de Azure**: activado. Debe estar activado para que las bases de datos de publicador y suscriptor puedan comunicarse entre sí.
    1. En Reglas de firewall, seleccione **+Agregar dirección IP del cliente actual**. Esto agrega la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall.
1. Seleccione **Revisar + crear**. Seleccione **Crear**.
1. Espera a que se implementen ambos servidores de Azure Database for PostgreSQL.

## Configuración de la replicación

Para *ambos* servidores de publicador y suscriptor:

1. En Azure Portal, vaya al servidor y, en Configuración, seleccione **Parámetros del servidor**.
1. Con la barra de búsqueda, busque cada parámetro y realice los cambios siguientes:
    1. `wal_level` = LOGICAL
    1. `max_worker_processes` = 24
1. Seleccione **Guardar**. A continuación, seleccione **Guardar y reiniciar**.
1. Espera a que ambos servidores se reinicien.

    > Nota:
    >
    > Después de volver a implementar los servidores, es posible que tengas que actualizar las ventanas del explorador para observar que los servidores se han reiniciado.

## Antes de continuar:

Asegúrese de lo siguiente:

1. Has instalado e iniciado ambos servidores flexibles de Azure Database for PostgreSQL.
1. Ya has clonado los scripts de laboratorio de [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git). Si todavía no lo has hecho, clona el repositorio localmente:
    1. Abre una línea de comandos o un terminal.
    1. Ejecute el comando:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > NOTA
       > 
       > Si **git** no está instalado, [descarga e instala la aplicación ***git*** ](https://git-scm.com/download)e intenta volver a ejecutar los comandos anteriores.
1. Ha instalado Azure Data Studio. Si no lo has hecho, [descarga e instala ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Instala la extensión **PostgreSQL** en Azure Data Studio.

## Configuración del publicador

1. Abra Azure Data Studio y conéctese al servidor del publicador. (Copie el nombre del servidor de la sección Información general).
1. Selecciona **Archivo**, **Abrir archivo** y ve a la carpeta donde has guardado los scripts.
1. Abre el script **../Allfiles/Labs/06/Lab6_Replication.sql** y conéctate al servidor.
1. Resalte y ejecute la sección **Concesión del permiso de replicación del usuario administrador**.
1. Resalte y ejecute la sección **Creación de una base de datos zoodb**.
1. Seleccione zoodb como base de datos actual mediante la lista desplegable de la barra de herramientas. Compruebe que zoodb es la base de datos actual mediante la ejecución de la instrucción **SELECT**.
1. Resalta y ejecuta la sección **Creación de tablas** y **Restricciones de clave externa** en zoodb.
1. Resalte y ejecute la sección **Relleno de las tablas en zoodb**.
1. Resalte y ejecute la sección **Creación de una publicación**. Al ejecutar la instrucción SELECT, no aparecerá nada, ya que la replicación aún no está activa.

## Configuración del suscriptor

1. Abre una segunda instancia de Azure Data Studio y conéctate al servidor de suscriptor.
1. Selecciona **Archivo**, **Abrir archivo** y ve a la carpeta donde has guardado los scripts.
1. Abre el script **../Allfiles/Labs/06/Lab6_Replication.sql** y conéctate al servidor del suscriptor. (Copie el nombre del servidor de la sección Información general).
1. Resalte y ejecute la sección **Concesión del permiso de replicación del usuario administrador**.
1. Resalte y ejecute la sección **Creación de una base de datos zoodb**.
1. Seleccione zoodb como base de datos actual mediante la lista desplegable de la barra de herramientas. Compruebe que zoodb es la base de datos actual mediante la ejecución de la instrucción **SELECT**.
1. Resalta y ejecuta la sección **Creación de tablas** y **Restricciones de clave externa** en zoodb.
1. Desplácese hacia abajo hasta la sección **Crear una suscripción**.
    1. Edita la instrucción **CREATE SUBSCRIPTION** para que tenga el nombre correcto del servidor del publicador y la contraseña segura del publicador. Resalte y ejecute la instrucción.
    1. Resalte y ejecute la instrucción **SELECT**. Esto muestra la suscripción "sub" que ha creado.
1. En la sección **Mostrar las tablas**, resalte y ejecute cada instrucción **SELECT**. La replicación del publicador ha rellenado las tablas.

## Cambios en la base de datos del publicador

- En la primera instancia de Azure Data Studio (*tu instancia de editor*), en **Insertar más animales**, resalta y ejecuta la instrucción **INSERT**. *Asegúrate de **no** ejecutar esta instrucción INSERT en el suscriptor*.

## Visualización de los cambios en la base de datos del suscriptor

- En la segunda instancia de Azure Data Studio (suscriptor), en **Mostrar las tablas de animales**, resalte y ejecute la instrucción **SELECT**.

Ahora ha creado dos servidores flexibles de Azure Database for PostgreSQL y ha configurado uno como publicador y el otro como suscriptor. En la base de datos del publicador ha creado y rellenado la base de datos del zoo. En la base de datos del suscriptor, ha creado una base de datos vacía, que luego se ha rellenado mediante la replicación de streaming.

## Limpieza

1. Una vez finalizado el ejercicio, elimina el grupo de recursos que contiene ambos servidores. Se le cobrará por los servidores a menos que los detenga o elimine.
1. Si es necesario, elimina la carpeta .\DP3021Lab.
