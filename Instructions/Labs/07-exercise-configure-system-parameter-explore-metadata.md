---
lab:
  title: Configuración de parámetros del sistema y exploración de metadatos con catálogos y vistas del sistema
  module: Configure and manage Azure Database for PostgreSQL
---

# Configuración de parámetros del sistema y exploración de metadatos con catálogos y vistas del sistema

En este ejercicio, verá los parámetros del sistema y los metadatos en PostgreSQL.

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

### Implementación de recursos en tu suscripción a Azure

Si los recursos de Azure ya están instalados, *omite la implementación de recursos*.

Este paso te guiará por el uso de los comandos de la CLI de Azure desde Visual Studio Code para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en la suscripción a Azure.

> &#128221; Si vas a realizar varios módulos en esta ruta de aprendizaje, puedes compartir el entorno de Azure entre ellos. En ese caso, solo deberás completar este paso de implementación de recursos una vez.

1. Abre Visual Studio Code si no está ya abierto, y abre la carpeta donde has clonado el repositorio de GitHub.

1. Expande la carpeta **mslearn-postgresql** en el panel Explorador.

1. Expande la carpeta **Allfiles/Labs/Shared**.

1. Haz clic con el botón derecho del ratón en la carpeta **Allfiles/Labs/Shared** y selecciona **Open in Integrated Terminal**. Esta selección abre una ventana de terminal en la ventana de Visual Studio Code.

1. El terminal puede abrir una ventana de **PowerShell** de forma predeterminada. Para esta sección del laboratorio, vas a usar el **shell de Bash**. Además del icono **+**, hay una flecha desplegable. Selecciónalo y elige **Git Bash** o **Bash** en la lista de perfiles disponibles. Esta selección abre una nueva ventana de terminal con el **shell de Bash**.

    > &#128221; Puedes cerrar la ventana del terminal de **PowerShell** si quieres, pero no es necesario. Puedes tener varias ventanas de terminal abiertas al mismo tiempo.

1. En la ventana del terminal, ejecuta el siguiente comando para iniciar sesión en tu cuenta de Azure:

    ```bash
    az login
    ```

    Este comando abre una nueva ventana del explorador que te solicita que inicies sesión en tu cuenta de Azure. Después de iniciar sesión, vuelve a la ventana del terminal.

1. A continuación, ejecutarás tres comandos para definir variables para reducir la escritura redundante al usar comandos de la CLI de Azure para crear recursos de Azure. Las variables representan el nombre que se asigna al grupo de recursos (`RG_NAME`), la región de Azure (`REGION`) en la que se implementan los recursos y una contraseña generada aleatoriamente para el inicio de sesión de administrador de PostgreSQL (`ADMIN_PASSWORD`).

    En el primer comando, la región asignada a la variable correspondiente es `eastus`, pero también puedes reemplazarla por una ubicación de tu preferencia.

    ```bash
    REGION=eastus
    ```

    El siguiente comando asigna el nombre que se usará para el grupo de recursos que hospeda todos los recursos usados en este ejercicio. El nombre del grupo de recursos asignado a la variable correspondiente es `rg-learn-work-with-postgresql-$REGION`, donde `$REGION` es la ubicación especificada anteriormente. *Sin embargo, puedes cambiarlo a cualquier otro nombre de grupo de recursos que se adapte a tu preferencia o que ya puedas tener*.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    El comando final genera aleatoriamente una contraseña para el inicio de sesión de administrador de PostgreSQL. Asegúrate de copiarlo en un lugar seguro para poder usarlo más adelante para conectarte al servidor flexible de PostgreSQL.

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. (Omite este paso si usas una suscripción predeterminada). Si tienes acceso a más de una suscripción a Azure, y tu suscripción predeterminada *no es* en la que deseas crear el grupo de recursos y otros recursos para este ejercicio, ejecuta este comando para establecer la suscripción adecuada al reemplazar el token `<subscriptionName|subscriptionId>` por el nombre o el id. de la suscripción que quieras usar:

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (Omitir si vas a usar un grupo de recursos existente) Ejecuta el siguiente comando de la CLI de Azure para crear tu grupo de recursos:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Por último, usa la CLI de Azure para ejecutar un script de implementación de Bicep para aprovisionar recursos de Azure en tu grupo de recursos:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados son un servidor flexible de Azure Database for PostgreSQL. El script de bicep también crea una base de datos, que se puede configurar en la línea de comandos como parámetro.

    La implementación tarda normalmente varios minutos en completarse. Puedes supervisarlo desde el terminal de Bash o ir a la página **Implementaciones** del grupo de recursos que has creado anteriormente y observar el progreso de la implementación allí.

1. Dado que el script crea un nombre aleatorio para el servidor PostgreSQL, puedes encontrar el nombre del servidor al ejecutar el siguiente comando:

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    Anota el nombre del servidor, ya que lo necesitas para conectarte al servidor más adelante en este ejercicio.

    > &#128221; También puedes encontrar el nombre del servidor en Azure Portal. En Azure Portal, ve a **Grupos de recursos** y selecciona el grupo de recursos que has creado anteriormente. El servidor PostgreSQL aparece en el grupo de recursos.

### Solución de errores de implementación

Es posible que encuentres algunos errores al ejecutar el script de implementación de Bicep. Los mensajes y los pasos más comunes para resolverlos son:

- Si anteriormente has ejecutado el script de implementación de Bicep para esta ruta de aprendizaje y, posteriormente, has eliminado los recursos, puedes recibir un mensaje de error similar al siguiente si intentas volver a ejecutar el script en un plazo de 48 horas después de eliminar los recursos:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Si recibes este mensaje, modifica el comando `azure deployment group create` anterior para que el parámetro `restore` sea igual a `true` y vuelve a ejecutarlo.

- Si la región seleccionada está restringida al aprovisionamiento de recursos específicos, deberás establecer la variable `REGION` en otra ubicación y volver a ejecutar los comandos para crear el grupo de recursos y ejecutar el script de implementación de Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si el laboratorio requiere recursos de IA, es posible que recibas el siguiente error. Este error se produce cuando el script no puede crear un recurso de IA debido al requisito de aceptar el contrato de IA responsable. Si es así, usa la interfaz de usuario de Azure Portal para crear un recurso de Servicios de Azure AI y después vuelve a ejecutar el script de implementación.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Conexión a la extensión de PostgreSQL en Visual Studio Code

En esta sección, te conectarás al servidor PostgreSQL mediante la extensión PostgreSQL en Visual Studio Code. Usa la extensión PostgreSQL para ejecutar scripts de SQL en el servidor PostgreSQL.

1. Abre Visual Studio Code si no está ya abierto, y abre la carpeta donde has clonado el repositorio de GitHub.

1. Selecciona el icono de **PostgreSQL** en el menú izquierdo.

    > &#128221; Si no ves el icono de PostgreSQL, selecciona el icono **Extensiones** y busca **PostgreSQL**. Selecciona la extensión **PostgreSQL** de Microsoft y selecciona **Instalar**.

1. Si ya has creado una conexión con el servidor PostgreSQL, ve al paso siguiente. Para crear una nueva conexión:

    1. En la extensión **PostgreSQL**, selecciona **+ Agregar conexión** para agregar una nueva conexión.

    1. En el cuadro de diálogo **NUEVA CONEXIÓN**, escribe la siguiente información:

        - **Nombre del servidor**: `<your-server-name>`.postgres.database.azure.com
        - **Tipo de autenticación**: contraseña
        - **Nombre de usuario**: pgAdmin
        - **Contraseña**: la contraseña aleatoria que has generado anteriormente.
        - Marca la casilla **Guardar contraseña**.
        - **Nombre de conexión**: `<your-server-name>`

    1. Prueba la conexión al seleccionar **Probar conexión**. Si la conexión se realiza correctamente, selecciona **Guardar y conectar** para guardar la conexión; de lo contrario, revisa la información de conexión e inténtalo de nuevo.

1. Si aún no estás conectado, selecciona **Conectar** para el servidor PostgreSQL. Se ha conectado con el servidor de Azure Database for PostgreSQL.

1. Expande el nodo de servidor y sus bases de datos. Se muestran las bases de datos existentes.

1. Si aún no has creado la base de datos zoodb, selecciona **Archivo**, **Abrir archivo** y ve a la carpeta donde has guardado los scripts. Selecciona **../Allfiles/Labs/02/Lab2_ZooDb.sql** y **Abrir**.

1. En la parte inferior derecha de Visual Studio Code, asegúrate de que la conexión sea verde. Si no es así, debería decir **PGSQL desconectado**. Selecciona el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. Es hora de crear la base de datos.

    1. Resalta las instrucciones **DROP** y **CREATE** y ejecútalas.

    1. Si resaltas solo la instrucción **SELECT current_database()** y la ejecutas, observas que la base de datos está establecida actualmente en `postgres`. Debes cambiarla a `zoodb`.

    1. Selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y selecciona **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos.

        > &#128221; También puedes cambiar la base de datos en el panel de consulta. Puedes anotar el nombre del servidor y el nombre de la base de datos en la propia pestaña de consulta. Al seleccionar el nombre de la base de datos, aparecerá una lista de bases de datos. Selecciona la base de datos `zoodb` de la lista.

    1. Ejecute de nuevo la instrucción **SELECT current_database()** para confirmar que la base de datos está ahora establecida en `zoodb`.

    1. Resalta las secciones **Crear tablas**, **Crear claves externas** y **Rellenar tablas** y ejecútalas.

    1. Resalta las 3 instrucciones **SELECT** al final del script y ejecútalos para comprobar que las tablas se crearon y rellenaron.

## Tarea 1: Exploración del proceso de vaciado en PostgreSQL

En esta sección, explorarás el proceso de vaciado en PostgreSQL. El proceso de vaciado se usa para recuperar espacio de almacenamiento y optimizar el rendimiento de la base de datos. Puedes ejecutar el proceso de vaciado manualmente o configurarlo para que se ejecute automáticamente.

1. En la ventana de Visual Studio Code, selecciona **File**, **Open File** y después ve a los scripts de laboratorio. Selecciona **../Allfiles/Labs/07/Lab7_vacuum.sql** y, después, selecciona **Abrir**. Si es necesario, vuelve a conectarte al servidor al seleccionar el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. En la parte inferior derecha de Visual Studio Code, asegúrate de que la conexión sea verde. Si no es así, debería decir **PGSQL desconectado**. Selecciona el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. Ejecuta la instrucción **SELECT current_database()** para comprobar la base de datos actual. Comprueba si la conexión está establecida actualmente en la base de datos **zoodb**. Si no es así, puedes cambiar la base de datos a **zoodb**. Para cambiar la base de datos, selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y al seleccionar **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos. Verifica que la base de datos está ahora en `zoodb` al ejecutar la instrucción **SELECT current_database();**.

1. Resalte y ejecute la sección **Display dead tuples**. Esta consulta muestra el número de tuplas inactivas y activas de la base de datos. Anote el número de tuplas inactivas.

1. Resalta y ejecuta la sección **Change weight** 10 veces seguidas. Esta consulta actualiza la columna de peso de todos los animales.

1. Vuelva a ejecutar la sección **Display dead tuples**. Anota el número de tuplas inactivas después de que se hayan completado las actualizaciones.

1. Ejecute la sección de **Manually run VACUUM** para ejecutar el proceso de vacado.

1. Vuelva a ejecutar la sección **Display dead tuples**. Anota el número de tuplas inactivas después de que se complete el proceso de vaciado.

## Tarea 2: Configuración de parámetros del servidor de vaciado automático

En esta sección, configurarás los parámetros del servidor de vaciado automático. El proceso de vaciado automático se usa para recuperar automáticamente el espacio de almacenamiento y optimizar el rendimiento de la base de datos. Puedes configurar el proceso de vaciado automático para que se ejecute automáticamente en función de parámetros específicos.

1. Si todavía no lo has hecho, ve a [Azure Portal](https://portal.azure.com) e inicia sesión.

1. En Azure Portal, vaya al servidor flexible de Azure Database for PostgreSQL.

1. En **Configuración**, seleccione **Parámetros del servidor**.

1. En la barra de búsqueda, escribe **`vacuum`**. Busque los parámetros siguientes y cambie los valores como se indica a continuación:

    - autovacuum = ON (debe estar activado de forma predeterminada)
    - autovacuum_vacuum_scale_factor = 0.1
    - autovacuum_vacuum_threshold = 50

    Estos cambios son como ejecutar el proceso de vaciado automático cuando el 10 % de una tabla tiene filas marcadas para su eliminación, o cuando hay 50 filas actualizadas o eliminadas en cualquier tabla.

1. Seleccione **Guardar**. El servidor se reinicia.

## Tarea 3: Visualización de metadatos de PostgreSQL en Azure Portal

En esta sección, verás los metadatos de PostgreSQL en Azure Portal. Azure Portal proporciona una interfaz gráfica para administrar y supervisar el servidor PostgreSQL.

1. Si todavía no lo has hecho, ve a [Azure Portal](https://portal.azure.com) e inicia sesión.

1. Busca **Azure Database for PostgreSQL** y selecciónalo.

1. Seleccione el servidor flexible de Azure Database for PostgreSQL que creó para este ejercicio.

1. En **Supervisión**, seleccione **Métricas**.

1. Seleccione **Métrica** y **Porcentaje de CPU**.

1. Tenga en cuenta que puede ver varias métricas sobre las bases de datos.

## Tarea 4: Visualización de datos en tablas del catálogo del sistema

En esta sección, verás los datos en las tablas del catálogo del sistema. Las tablas del catálogo del sistema se usan para almacenar metadatos sobre los objetos de base de datos en PostgreSQL. Puedes consultar estas tablas para recuperar información sobre los objetos de la base de datos.

1. Abre Visual Studio Code si aún no se ha abierto.

1. Abre la paleta de comandos (Ctrl+Mayús+P) y selecciona **PGSQL: New Query**. Selecciona la nueva conexión que has creado de la lista de la paleta de comandos. Si solicitas una contraseña, escribe la contraseña que has creado para el nuevo rol.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL:

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. Tenga en cuenta que puede ver confirmaciones y reversiones para cada base de datos.

## Visualización de una consulta de metadatos compleja mediante una vista del sistema

En esta sección, verás una consulta de metadatos compleja mediante una vista del sistema Las vistas del sistema se usan para proporcionar una interfaz simplificada para consultar metadatos sobre los objetos de base de datos en PostgreSQL. 

1. Abre Visual Studio Code si aún no se ha abierto.

1. Abre la paleta de comandos (Ctrl+Mayús+P) y selecciona **PGSQL: New Query**. Selecciona la nueva conexión que has creado de la lista de la paleta de comandos. Si solicitas una contraseña, escribe la contraseña que has creado para el nuevo rol.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL:

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. Tenga en cuenta que puede ver una gran cantidad de información estadística.

1. Al utilizar las vistas del sistema, puede reducir la complejidad del SQL que necesita escribir. La consulta anterior necesitaría el código siguiente, si no estabas usando la vista **pg_stats**. Ahora vamos a ejecutar el código siguiente para ver cómo funciona: Copia, resalta y ejecuta la siguiente instrucción SQL en la ventana **New Query**:

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## Limpieza

1. Si ya no necesitas este servidor PostgreSQL para otros ejercicios, para evitar incurrir en costes innecesarios de Azure, elimina el grupo de recursos creado en este ejercicio.

1. Si deseas mantener el servidor PostgreSQL en funcionamiento, puedes dejarlo en ejecución. Si no quiere dejar que se ejecute, puedes detener el servidor para evitar incurrir en costes innecesarios en el terminal de Bash. Para detener el servidor, ejecuta el siguiente comando:

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Reemplaza `<your-server-name>` por el nombre de tu servidor PostgreSQL.

    > &#128221; También puedes detener el servidor desde Azure Portal. En Azure Portal, ve a **Grupos de recursos** y selecciona el grupo de recursos que has creado anteriormente. Selecciona el servidor PostgreSQL y después **Detener** en el menú.

1. Si es necesario, elimina el repositorio Git que has clonado anteriormente.

En este ejercicio, has aprendido cómo configurar los parámetros del sistema y explorado los metadatos en PostgreSQL. También has aprendido a ver los metadatos de PostgreSQL en Azure Portal y a ver los datos en las tablas del catálogo del sistema. Además, has aprendido a ver una consulta de metadatos compleja mediante una vista del sistema.
