---
lab:
  title: Evaluación del rendimiento de las consultas mediante el almacén de consultas
  module: Tune queries in Azure Database for PostgreSQL
---

# Evaluación del rendimiento de las consultas mediante el almacén de consultas

En este ejercicio, aprenderá a consultar las métricas de rendimiento mediante Almacén de consultas en Azure Database for PostgreSQL.

## Antes de comenzar

Para completar los ejercicios de este módulo se necesita una suscripción de Azure propia. Si no tiene una suscripción a Azure, puede crear una cuenta de evaluación gratuita en [Cree soluciones en la nube con una cuenta gratuita de Azure](https://azure.microsoft.com/free/).

## Creación del entorno de ejercicio

### Implementación de recursos en tu suscripción a Azure

Este paso te guía por el uso de comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en tu suscripción a Azure.

1. Abra un explorador web y vaya a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure con el icono de Cloud Shell resaltado en un cuadro rojo.](media/09-portal-toolbar-cloud-shell.png)

    Si se te solicita, selecciona las opciones necesarias para abrir un shell de *Bash*. Si anteriormente has usado una consola de *PowerShell*, cámbiala a un shell de *Bash*.

3. En el símbolo del sistema de Cloud Shell, escribe lo siguiente para clonar el repositorio de GitHub que contiene recursos del ejercicio:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. A continuación, ejecutarás tres comandos para definir variables para reducir la escritura redundante al usar comandos de la CLI de Azure para crear recursos de Azure. Las variables representan el nombre que se va a asignar a tu grupo de recursos (`RG_NAME`), la región de Azure (`REGION`) en la que se implementarán los recursos y una contraseña generada aleatoriamente para el inicio de sesión de administrador de PostgreSQL (`ADMIN_PASSWORD`).

    En el primer comando, la región asignada a la variable correspondiente es `eastus`, pero también puedes reemplazarla por una ubicación de tu preferencia.

    ```bash
    REGION=eastus
    ```

    El siguiente comando asigna el nombre que se usará para el grupo de recursos que hospedará todos los recursos usados en este ejercicio. El nombre del grupo de recursos asignado a la variable correspondiente es `rg-learn-work-with-postgresql-$REGION`, donde `$REGION` es la ubicación que especificaste anteriormente. Sin embargo, puedes cambiarlo a cualquier otro nombre de grupo de recursos que se adapte a tu preferencia.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    El comando final genera aleatoriamente una contraseña para el inicio de sesión de administrador de PostgreSQL. Asegúrate de copiarlo en un lugar seguro para poder usarlo más adelante para conectarte al servidor flexible de PostgreSQL.

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
       do
       a[$RANDOM]=$i
    done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

5. Si tienes acceso a más de una suscripción a Azure y tu suscripción predeterminada no es aquella en la que quieres crear el grupo de recursos y otros recursos para este ejercicio, ejecuta este comando para establecer la suscripción adecuada. Para ello, reemplaza el token `<subscriptionName|subscriptionId>` por el nombre o el identificador de la suscripción que quieres usar:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. Ejecuta el siguiente comando de la CLI de Azure para crear tu grupo de recursos:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Por último, usa la CLI de Azure para ejecutar un script de implementación de Bicep para aprovisionar recursos de Azure en tu grupo de recursos:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados son un servidor flexible de Azure Database for PostgreSQL. El script de bicep también crea una base de datos, que se puede configurar en la línea de comandos como parámetro.

    La implementación tarda normalmente varios minutos en completarse. Puedes supervisarla desde Cloud Shell o ir a la página **Implementaciones** del grupo de recursos que creaste anteriormente y observar allí el progreso de la implementación.

8. Cierra el panel de Cloud Shell una vez completada la implementación de recursos.

### Solución de errores de implementación

Es posible que encuentres algunos errores al ejecutar el script de implementación de Bicep. Los mensajes más comunes y los pasos para resolverlos son:

- Si anteriormente ejecutaste el script de implementación de Bicep para esta ruta de aprendizaje y, posteriormente, eliminaste los recursos, puedes recibir un mensaje de error similar al siguiente si intentas volver a ejecutar el script en un plazo de 48 horas después de eliminar los recursos:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Si recibes este mensaje, modifica el comando `azure deployment group create` anterior para establecer el parámetro `restore` igual a `true` y vuelve a ejecutarlo.

- Si la región seleccionada está restringida al aprovisionamiento de recursos específicos, debes establecer la variable `REGION` en otra ubicación y volver a ejecutar los comandos para crear el grupo de recursos y ejecutar el script de implementación de Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si el script no puede crear un recurso de IA debido al requisito de aceptar el contrato de IA responsable, puedes experimentar el siguiente error; en cuyo caso, usa la interfaz de usuario de Azure Portal para crear un recurso de Servicios de Azure AI y, a continuación, vuelve a ejecutar el script de implementación.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Conéctate a tu base de datos mediante psql en Azure Cloud Shell

En esta tarea, te conectas a la base de datos `adventureworks`de tu servidor de Azure Database for PostgreSQL mediante la utilidad de línea de comandos [psql](https://www.postgresql.org/docs/current/app-psql.html) de [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. En [Azure Portal](https://portal.azure.com/), ve al servidor flexible de Azure Database for PostgreSQL que acabas de crear.

2. En el menú de recursos, en **Configuración**, selecciona **Bases de datos**, selecciona **Conectar** para la base de datos `adventureworks`.

    ![Captura de pantalla de la página de bases de datos de Azure Database for PostgreSQL. Las bases de datos y Conectar para la base de datos adventureworks están resaltadas en cuadros rojos.](media/08-postgresql-adventureworks-database-connect.png)

3. En el símbolo del sistema "Contraseña para el usuario pgAdmin" de Cloud Shell, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin**.

    Una vez iniciada la sesión, se muestra la solicitud `psql` de la base de datos `adventureworks`.

4. Durante el resto de este ejercicio, seguirás trabajando en Cloud Shell, por lo que puede ser útil expandir el panel dentro de la ventana del explorador al seleccionar el botón **Maximizar** en la parte superior derecha del panel.

    ![Captura de pantalla del panel Azure Cloud Shell con el botón Maximizar resaltado con un cuadro rojo.](media/08-azure-cloud-shell-pane-maximize.png)

### Rellenado de la base de datos con datos

1. Debes crear una tabla dentro de la base de datos y rellenarla con datos de ejemplo para que tengas información con la que trabajar mientras revisas el bloqueo en este ejercicio.
1. Ejecuta el siguiente comando para crear la tabla `production.workorder` para cargar los datos:

    ```sql
    /*********************************************************************************
    Create Schema: production
    *********************************************************************************/
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
    /*********************************************************************************
    Create Table: production.workorder
    *********************************************************************************/
    
    DROP TABLE IF EXISTS production.workorder;
    CREATE TABLE production.workorder
    (
        workorderid integer NOT NULL,
        productid integer NOT NULL,
        orderqty integer NOT NULL,
        scrappedqty smallint NOT NULL,
        startdate timestamp without time zone NOT NULL,
        enddate timestamp without time zone,
        duedate timestamp without time zone NOT NULL,
        scrapreasonid smallint,
        modifieddate timestamp without time zone NOT NULL DEFAULT now()
    )
    WITH (
        OIDS = FALSE
    )
    TABLESPACE pg_default;
    ```

1. Después, usa el comando `COPY` para cargar datos de archivos CSV en la tabla que has creado anteriormente. Comienza por ejecutar el siguiente comando para rellenar la tabla `production.workorder`:

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 72591`, que indica que 72591 filas se has escrito en la tabla desde el archivo CSV.

1. Cierra el panel de Cloud Shell una vez cargados los datos

### Conectarse a la base de datos mediante Azure Data Studio

1. Si aún no has instalado Azure Data Studio, [descarga e instala ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Inicie Azure Data Studio.
1. Si no has instalado la extensión **PostgreSQL** en Azure Data Studio, instálala ahora.
1. Seleccione **Conexiones**.
1. Seleccione **Servidores** y **Nueva conexión**.
1. En **Tipo de conexión**, seleccione **PostgreSQL**.
1. En **Nombre del servidor**, escriba el valor que especificó al implementar el servidor.
1. En **Nombre de usuario**, escribe **pgAdmin**.
1. En **Contraseña**, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin** que generaste.
1. Seleccione **Remember password** (Recordar contraseña).
1. Haga clic en **Conectar**

### Crea tablas en la base de datos

1. Expanda **Bases de datos**, haga clic con el botón derecho en **adventureworks** y seleccione **Nueva consulta**.
   
    ![Captura de pantalla de la base de datos de Adventureworks que muestra la opción de menú contextual Nueva consulta](media/09-new-query.png)

1. Seleccione la pestaña **SQLQuery_1**, escriba la consulta siguiente y seleccione **Ejecutar**.

    ```sql
    SELECT * FROM production.workorder;
    ```

## Tarea 1: Activación del modo de captura de consultas

1. Vaya a Azure Portal e inicie sesión.
1. Seleccione el servidor de Azure Database for PostgreSQL para este ejercicio.
1. En **Configuración**, seleccione **Parámetros del servidor**.
1. Ve a la configuración **`pg_qs.query_capture_mode`**.
1. Seleccione **TOP**.

   ![Captura de pantalla de la configuración para activar Almacén de consultas](media/09-settings-turn-query-store-on.png)

1. Ve a **`pgms_wait_sampling.query_capture_mode`**, selecciona **TODOS** y **Guardar**.
   
    ![Captura de pantalla de la configuración para activar pgms_wait_sampling.query_capture_mode](media/09-query-capture-mode.png)
   
1. Espere a que se actualicen los parámetros del servidor.

## Visualización de los datos de pg_stat

1. Inicie Azure Data Studio.
1. Seleccione **Conectar**.
   
    ![Captura de pantalla en la que se muestra el icono Conectar](media/09-connect.png)
   
1. Seleccione el servidor de PostgreSQL y seleccione **Conectar**.
1. Escribe la siguiente consulta y selecciona **Ejecutar**.

    ```sql
    SELECT 
        pid,                    -- Process ID of the server process
        datid,                  -- OID of the database
        datname,                -- Name of the database
        usename,                -- Name of the user
        application_name,       -- Name of the application connected to the database
        client_addr,            -- IP address of the client
        client_hostname,        -- Hostname of the client (if available)
        client_port,            -- TCP port number that the client is using for the connection
        backend_start,          -- Timestamp when the backend process started
        xact_start,             -- Timestamp of the current transaction start, if any
        query_start,            -- Timestamp when the current query started, if any
        state_change,           -- Timestamp when the state was last changed
        wait_event_type,        -- Type of event the backend is waiting for, if any
        wait_event,             -- Event that the backend is waiting for, if any
        state,                  -- Current state of the session (e.g., active, idle, etc.)
        backend_xid,            -- Transaction ID, if active
        backend_xmin,           -- Transaction ID that the process is working with
        query,                  -- Text of the query being executed
        encode(backend_type::bytea, 'escape') AS backend_type,           -- Type of backend (e.g., client backend, autovacuum worker). We use encode(…, 'escape') to safely display raw data with invalid characters by converting it into a readable format, doing this prevents a UTF-8 conversion error in Azure Data Studio.
        leader_pid,             -- PID of the leader process, if this is a parallel worker
        query_id               -- Query ID (added in more recent PostgreSQL versions)
    FROM pg_stat_activity;
    ```

1. Revise las métricas disponibles.
1. Deje abierto Azure Data Studio para la tarea siguiente.

## Tarea 2: Examen de las estadísticas de consulta

> [!NOTE]
> En el caso de una base de datos recién creada, puede haber estadísticas limitadas, si existen. Si espera 30 minutos, habrá estadísticas sobre los procesos en segundo plano.

1. Seleccione la base de datos **azure_sys**.

    ![Captura de pantalla del selector de bases de datos](media/09-database-selector.png)

1. Escriba las siguientes consultas y seleccione **Ejecutar**.

    ```sql
    SELECT * FROM query_store.query_texts_view;
    ```

    ```sql
    SELECT * FROM query_store.qs_view;
    ```

    ```sql
    SELECT * FROM query_store.runtime_stats_view;
    ```

    ```sql
    SELECT * FROM query_store.pgms_wait_sampling_view;
    ```

1. Revise las métricas disponibles.

## Limpieza del ejercicio

La instancia de Azure Database for PostgreSQL que hemos implementado en este ejercicio incurrirá en cargos que puedes eliminar del servidor después de este ejercicio. Como alternativa, puedes eliminar el grupo de recursos **rg-learn-work-with-postgresql-eastus** para quitar todos los recursos implementados como parte de este ejercicio.