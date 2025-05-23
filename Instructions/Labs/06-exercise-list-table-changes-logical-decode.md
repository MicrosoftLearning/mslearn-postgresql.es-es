---
lab:
  title: Enumeración de cambios de tabla con descodificación lógica
  module: Understand write-ahead logging
---

# Enumeración de cambios de tabla con descodificación lógica

En este ejercicio, configurará la replicación lógica, que es nativa de PostgreSQL. Cree dos servidores, que actúan como publicador y suscriptor. Los datos de zoodb se replican entre ellos.

## Antes de comenzar

Debe tener una suscripción a Azure propia para completar este ejercicio. Si no tiene una suscripción a Azure, puede obtener una [evaluación gratuita de Azure](https://azure.microsoft.com/free).

Además, debes tener lo siguiente instalado en el equipo:

- Visual Studio Code.
- Extensión Postgres Visual Studio Code de Microsoft
- Azure CLI.
- Git.

## Creación del entorno de ejercicio

En estos ejercicios y los ejercicios posteriores, se usa un script de Bicep para implementar el servidor flexible de Azure Database for PostgreSQL, así como otros recursos en la suscripción a Azure. Los scripts de Bicep se encuentran en la carpeta `/Allfiles/Labs/Shared` del repositorio de GitHub que clonaste anteriormente.

### Descarga e instala Visual Studio Code y la extensión de PostgreSQL.

Si no tienes instalado Visual Studio Code:

1. En un explorador, ve a [Descargar Visual Studio Code](https://code.visualstudio.com/download) y selecciona la versión adecuada para tu sistema operativo.

1. Sigue las instrucciones de instalación para tu sistema operativo.

1. Abra Visual Studio Code.

1. En el menú izquierdo, seleccione **Extensiones** para mostrar el panel Extensiones.

1. En la barra de búsqueda, escriba **PostgreSQL**. Se muestra el icono de la extensión de PostgreSQL para Visual Studio Code. Asegúrate de seleccionar la de Microsoft.

1. Seleccione **Instalar**. La extensión se instala.

### Descarga e instalación de la CLI de Azure y Git

Si no tienes la CLI de Azure o Git, debes instalarla:

1. En el explorador, ve a [Instalar la CLI de Azure](https://learn.microsoft.com/cli/azure/install-azure-cli) y sigue las instrucciones para el sistema operativo.

1. En un explorador, ve a [Descargar e instalar Git](https://git-scm.com/downloads) y sigue las instrucciones del sistema operativo.

### Descarga de los archivos de los ejercicios

Si ya has clonado el repositorio de GitHub que contiene los archivos del ejercicio, *omite descargar de los archivos del ejercicio*.

Para descargar los archivos del ejercicio, clona el repositorio de GitHub que contiene los archivos del ejercicio en la máquina local. El repositorio contiene todos los scripts y recursos que necesitas para completar este ejercicio.

1. Abre Visual Studio Code si aún no se ha abierto.

1. Selecciona **Mostrar todos los comandos** (Ctrl+Mayús+P) para abrir la paleta de comandos.

1. En la paleta de comandos, busca y selecciona **Git: Clone**.

1. En la paleta de comandos, escribe lo siguiente para clonar el repositorio de GitHub que contiene los recursos del ejercicio y presiona **Entrar**:

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Sigue las indicaciones para seleccionar una carpeta para clonar el repositorio. El repositorio se clona en una carpeta denominada `mslearn-postgresql` en la ubicación seleccionada.

1. Cuando se le pregunte si desea abrir el repositorio clonado, seleccione**Abrir**. El repositorio se abre en Visual Studio Code.

## Creación de un grupo de recursos

En esta sección, crearás un grupo de recursos para que contenga los servidores de Azure Database for PostgreSQL. El grupo de recursos es un contenedor lógico que incluye los recursos relacionados para una solución de Azure.

1. Inicie sesión en Azure Portal. La cuenta de usuario debe tener el rol Propietario o Colaborador para la suscripción de Azure.

1. Seleccione **Grupos de recursos** y, después, Seleccione **+ Crear**.

1. Seleccione su suscripción.

1. En Grupo de recursos, escribe **rg-PostgreSQL_Replication**.

1. Seleccione una región cercana a su ubicación.

1. Selecciona **Revisar + crear.**

1. Seleccione **Crear**.

## Creación de un servidor de publicador

En esta sección, crearás el servidor de publicador. El servidor de publicador es el origen de los datos que se van a replicar en el servidor de suscriptor.

1. En Servicios de Azure, seleccione **Crear un recurso**. En **Categorías**, seleccione **Bases de datos**. En **Azure Database for PostgreSQL**, seleccione **Crear**.

1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    - **Suscripción**: su suscripción.
    - **Grupo de recursos**: selecciona **rg-PostgreSQL_Replication**.
    - **Nombre del servidor** - *psql-postgresql-pub9999* (el nombre debe ser único globalmente, por tanto, reemplaza 9999 por cuatro números aleatorios).
    - **Región**: seleccione la misma región que el grupo de recursos.
    - **Versión de PostgreSQL**: selecciona 16.
    - **Tipo de carga de trabajo** - *Desarrollo.*.
    - **Proceso y almacenamiento** - *ampliable*. Seleccione **Configurar servidor** y examine las opciones de configuración. No realice ningún cambio ni cierre la sección.
    - **Zona de disponibilidad**: 1. Si no se admiten zonas de disponibilidad, deje la opción Sin preferencia.
    - **Alta disponibilidad**: deshabilitada.
    - **Método de autenticación**: solo autenticación de PostgreSQL.
    - En **nombre de usuario de administrador**, escribe **`pgAdmin`**.
    - En **contraseña**, escribe una contraseña compleja adecuada.

1. Seleccione **Siguiente: Redes >**.

1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    - **Método de conectividad**: (o) Acceso público (direcciones IP permitidas).
    - **Permitir acceso público a este servidor desde cualquier servicio de Azure dentro de Azure**: activado. Esta casilla debe estar activada para que las bases de datos de publicador y suscriptor puedan comunicarse entre sí.
    - **Reglas de firewall**, selecciona **+Agregar dirección IP del cliente actual**. Esta opción agrega la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall.

1. Selecciona **Revisar + crear.** Seleccione **Crear**.

1. Dado que la creación de una instancia de Azure Database for PostgreSQL puede tardar unos minutos, comienza con el paso siguiente en cuanto esta implementación esté en curso. Recuerda abrir una nueva ventana o pestaña del explorador para continuar.

## Creación de un servidor de suscriptor

1. En Servicios de Azure, seleccione **Crear un recurso**. En **Categorías**, seleccione **Bases de datos**. En **Azure Database for PostgreSQL**, seleccione **Crear**.

1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    - **Suscripción**: su suscripción.
    - **Grupo de recursos**: selecciona **rg-PostgreSQL_Replication**.
    - **Nombre del servidor** - *psql-postgresql-pub9999* (el nombre debe ser único globalmente, por tanto, reemplaza 9999 por cuatro números aleatorios).
    - **Región**: seleccione la misma región que el grupo de recursos.
    - **Versión de PostgreSQL**: selecciona 16.
    - **Tipo de carga de trabajo** - *Desarrollo.*.
    - **Proceso y almacenamiento** - *ampliable*. Seleccione **Configurar servidor** y examine las opciones de configuración. No realice ningún cambio ni cierre la sección.
    - **Zona de disponibilidad**: 2. Si no se admiten zonas de disponibilidad, deje la opción Sin preferencia.
    - **Alta disponibilidad**: deshabilitada.
    - **Método de autenticación**: solo autenticación de PostgreSQL.
    - En **nombre de usuario de administrador**, escribe **`pgAdmin`**.
    - En **contraseña**, escribe una contraseña compleja adecuada.

1. Seleccione **Siguiente: Redes >**.

1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    - **Método de conectividad**: (o) Acceso público (direcciones IP permitidas).
    - **Permitir acceso público a este servidor desde cualquier servicio de Azure dentro de Azure**: activado. Esta casilla debe estar activada para que las bases de datos de publicador y suscriptor puedan comunicarse entre sí.
    - **Reglas de firewall**, selecciona **+Agregar dirección IP del cliente actual**. Esta opción agrega la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall.

1. Selecciona **Revisar + crear.** Seleccione **Crear**.

1. Espera a que se implementen ambos servidores de Azure Database for PostgreSQL.

## Configuración de la replicación

Para *ambos* servidores de publicador y suscriptor:

1. En Azure Portal, vaya al servidor y, en Configuración, seleccione **Parámetros del servidor**.

1. Con la barra de búsqueda, busque cada parámetro y realice los cambios siguientes:
    - `wal_level` = LOGICAL
    - `max_worker_processes` = 24

1. Seleccione **Guardar**. A continuación, seleccione **Guardar y reiniciar**.

1. Espera a que ambos servidores se reinicien.

    > &#128221; Después de volver a implementar los servidores, es posible que tengas que actualizar las ventanas del explorador para observar que los servidores se han reiniciado.

## Configuración del publicador

En esta sección, configurarás el servidor de publicador. El servidor de publicador es el origen de los datos que se van a replicar en el servidor de suscriptor.

1. Abre la primera instancia de Visual Studio Code para conectarte al servidor de publicador.

1. Abre la carpeta en la que has clonado el repositorio de GitHub.

1. Selecciona el icono de **PostgreSQL** en el menú izquierdo.

    > &#128221; Si no ves el icono de PostgreSQL, selecciona el icono **Extensiones** y busca **PostgreSQL**. Selecciona la extensión **PostgreSQL** de Microsoft y selecciona **Instalar**.

1. Si ya has creado una conexión con el servidor del *publicador* de PostgreSQL, ve al paso siguiente. Para crear una nueva conexión:

    1. En la extensión **PostgreSQL**, selecciona **+ Agregar conexión** para agregar una nueva conexión.

    1. En el cuadro de diálogo **NUEVA CONEXIÓN**, escribe la siguiente información:

        - **Nombre del servidor**: `<your-publisher-server-name>`.postgres.database.azure.com
        - **Tipo de autenticación**: contraseña
        - **Nombre de usuario**: pgAdmin
        - **Contraseña**: la contraseña aleatoria que has generado anteriormente.
        - Marca la casilla **Guardar contraseña**.
        - **Nombre de conexión**: `<your-publisher-server-name>`

    1. Prueba la conexión al seleccionar **Probar conexión**. Si la conexión se realiza correctamente, selecciona **Guardar y conectar** para guardar la conexión; de lo contrario, revisa la información de conexión e inténtalo de nuevo.

1. Si aún no estás conectado, selecciona **Conectar** para el servidor PostgreSQL. Se ha conectado con el servidor de Azure Database for PostgreSQL.

1. Expande el nodo de servidor y sus bases de datos. Se muestran las bases de datos existentes.

1. En Visual Studio Code, selecciona **File**, **Open File** y ve a la carpeta donde guardaste los scripts. Selecciona **../Allfiles/Labs/06/Lab6_Replication.sql** y **Open**.

1. En la parte inferior derecha de Visual Studio Code, asegúrate de que la conexión sea verde. Si no es así, debería decir **PGSQL desconectado**. Selecciona el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. Es la hora de configurar el *publicador*.

    1. Resalte y ejecute la sección **Concesión del permiso de replicación del usuario administrador**.

    1. Resalte y ejecute la sección **Creación de una base de datos zoodb**.

    1. Si resaltas solo la instrucción **SELECT current_database()** y la ejecutas, observas que la base de datos está establecida actualmente en `postgres`. Debes cambiarla a `zoodb`.

    1. Selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y selecciona **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos.

        > &#128221; También puedes cambiar la base de datos en el panel de consulta. Puedes anotar el nombre del servidor y el nombre de la base de datos en la propia pestaña de consulta. Al seleccionar el nombre de la base de datos, aparecerá una lista de bases de datos. Selecciona la base de datos `zoodb` de la lista.

    1. Resalta y ejecuta la sección **Creación de tablas** y **Restricciones de clave externa** en zoodb.

    1. Resalte y ejecute la sección **Relleno de las tablas en zoodb**.

    1. Resalte y ejecute la sección **Creación de una publicación**. Al ejecutar la instrucción SELECT, no aparecerá nada, ya que la replicación aún no está activa.

    1. NO ejecutes la sección **CREATE SUBSCRIPTION**. Este script se ejecuta en el servidor de suscriptor.

    1. NO cierres esta instancia de Visual Studio Code, solo tienes que minimizarla. Volverás a ella después de configurar el servidor de suscriptor.

Has creado el servidor de publicador y la base de datos zoodb. La base de datos contiene las tablas y los datos que se replican en el servidor de suscriptor.

## Configuración del suscriptor

En esta sección, configurarás el servidor de suscriptor. El servidor de suscriptor es el destino de los datos que se replican desde el servidor de publicador. Crea una nueva base de datos en el servidor de suscriptor, que se rellena con los datos del servidor de publicador.

1. Abre una *segunda instancia* de Visual Studio Code y conéctate al servidor de suscriptor.

1. Abre la carpeta en la que has clonado el repositorio de GitHub.

1. Selecciona el icono de **PostgreSQL** en el menú izquierdo.

    > &#128221; Si no ves el icono de PostgreSQL, selecciona el icono **Extensiones** y busca **PostgreSQL**. Selecciona la extensión **PostgreSQL** de Microsoft y selecciona **Instalar**.

1. Si ya has creado una conexión al servidor de *suscriptor* de PostgreSQL, ve al paso siguiente. Para crear una nueva conexión:

    1. En la extensión **PostgreSQL**, selecciona **+ Agregar conexión** para agregar una nueva conexión.

    1. En el cuadro de diálogo **NUEVA CONEXIÓN**, escribe la siguiente información:

        - **Nombre de servidor**: `<your-subscriber-server-name>`.postgres.database.azure.com
        - **Tipo de autenticación**: contraseña
        - **Nombre de usuario**: pgAdmin
        - **Contraseña**: la contraseña aleatoria que has generado anteriormente.
        - Marca la casilla **Guardar contraseña**.
        - **Nombre de conexión**: `<your-subscriber-server-name>`

    1. Prueba la conexión al seleccionar **Probar conexión**. Si la conexión se realiza correctamente, selecciona **Guardar y conectar** para guardar la conexión; de lo contrario, revisa la información de conexión e inténtalo de nuevo.

1. Si aún no estás conectado, selecciona **Conectar** para el servidor PostgreSQL. Se ha conectado con el servidor de Azure Database for PostgreSQL.

1. Expande el nodo de servidor y sus bases de datos. Se muestran las bases de datos existentes.

1. En Visual Studio Code, selecciona **File**, **Open File** y ve a la carpeta donde guardaste los scripts. Selecciona **../Allfiles/Labs/06/Lab6_Replication.sql** y **Open**.

1. En la parte inferior derecha de Visual Studio Code, asegúrate de que la conexión sea verde. Si no es así, debería decir **PGSQL desconectado**. Selecciona el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. Es hora de configurar el *suscriptor*.

    1. Resalte y ejecute la sección **Concesión del permiso de replicación del usuario administrador**.

    1. Resalte y ejecute la sección **Creación de una base de datos zoodb**.

    1. Si resaltas solo la instrucción **SELECT current_database()** y la ejecutas, observas que la base de datos está establecida actualmente en `postgres`. Debes cambiarla a `zoodb`.

    1. Selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y selecciona **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos.

        > &#128221; También puedes cambiar la base de datos en el panel de consulta. Puedes anotar el nombre del servidor y el nombre de la base de datos en la propia pestaña de consulta. Al seleccionar el nombre de la base de datos, aparecerá una lista de bases de datos. Selecciona la base de datos `zoodb` de la lista.

    1. Resalta y ejecuta la sección **Crear tablas** y **Restricciones de clave externa** en `zoodb`.

    1. NO ejecutes la sección **Crear una publicación**, esa instrucción ya se ejecutó en el servidor de publicador.

    1. Desplácese hacia abajo hasta la sección **Crear una suscripción**.

        1. Edita la instrucción **CREATE SUBSCRIPTION** para que tenga el nombre correcto del servidor del publicador y la contraseña segura del publicador. Resalte y ejecute la instrucción.

        1. Resalte y ejecute la instrucción **SELECT**. Esto muestra la suscripción "sub" que creaste anteriormente.

    1. En la sección **Mostrar las tablas**, resalta y ejecuta cada instrucción **SELECT**. El servidor de publicador ha rellenado estas tablas a través de la replicación.

Has creado el servidor de suscriptor y la base de datos zoodb. La base de datos contiene las tablas y los datos que se replicaron desde el servidor de publicador.

## Cambios en la base de datos del publicador

- En la primera instancia de Visual Studio Code (*tu instancia de publicador*), en **Insert more animals**, resalta y ejecuta la instrucción **INSERT**. *Asegúrate de **no** ejecutar esta instrucción INSERT en el suscriptor*.

## Visualización de los cambios en la base de datos del suscriptor

- En la segunda instancia de Visual Studio Code (suscriptor), en **Display the animal tables**, resalta y ejecuta la instrucción **SELECT**.

Ahora has creado dos servidores flexibles de Azure Database for PostgreSQL y has configurado uno como publicador y el otro como suscriptor. En la base de datos del publicador has creado y rellenado la base de datos del zoo. En la base de datos del suscriptor, ha creado una base de datos vacía, que luego se ha rellenado mediante la replicación de streaming.

## Limpieza

1. Si ya no necesitas estos servidores PostgreSQL para otros ejercicios, para evitar incurrir en costes innecesarios de Azure, elimina el grupo de recursos que creaste en este ejercicio.

1. Si deseas mantener los servidores PostgreSQL en funcionamiento, puedes dejarlos en ejecución. Si no quiere dejar que se ejecuten, puedes detener el servidor para evitar incurrir en costes innecesarios en el terminal de Bash. Para detener los servidores, ejecuta el siguiente comando para cada servidor:

    ```bash

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Reemplaza `<your-server-name>` por el nombre de los servidores PostgreSQL.

    > &#128221; También puedes detener el servidor desde Azure Portal. En Azure Portal, ve a **Grupos de recursos** y selecciona el grupo de recursos que has creado anteriormente. Selecciona el servidor PostgreSQL y después **Detener** en el menú. Haz esto para ambos servidores de publicador y suscriptor.

1. Si es necesario, elimina el repositorio Git que has clonado anteriormente.

Has creado correctamente un servidor PostgreSQL y lo has configurado para la replicación lógica. Has creado un servidor de publicador y un servidor de suscriptor, y has configurado la replicación entre ellos. También has realizado cambios en la base de datos del publicador y has visto los cambios en la base de datos del suscriptor. Ahora tienes una buena comprensión de cómo configurar la replicación lógica en PostgreSQL.
