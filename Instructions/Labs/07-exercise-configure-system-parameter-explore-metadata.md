---
lab:
  title: Configuración de parámetros del sistema y exploración de metadatos con catálogos y vistas del sistema
  module: Configure and manage Azure Database for PostgreSQL
---

# Configuración de parámetros del sistema y exploración de metadatos con catálogos y vistas del sistema

En este ejercicio, verás los parámetros del sistema y los metadatos en PostgreSQL.

## Antes de comenzar

> [!IMPORTANT]
> Para completar los ejercicios de este módulo se necesita una suscripción de Azure propia. Si no tiene una suscripción a Azure, puede crear una cuenta de evaluación gratuita en [Cree soluciones en la nube con una cuenta gratuita de Azure](https://azure.microsoft.com/free/).

## Creación del entorno de ejercicio

### Implementación de recursos en tu suscripción a Azure

Este paso te guía por el uso de comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en tu suscripción a Azure.

> Nota:
>
> Si vas a realizar varios módulos en esta ruta de aprendizaje, puedes compartir el entorno de Azure entre ellos. En ese caso, solo debes completar este paso de implementación de recursos una vez.

1. Abra un explorador web y vaya a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure con el icono de Cloud Shell resaltado en un cuadro rojo.](media/07-portal-toolbar-cloud-shell.png)

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
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

### Conectarse a la base de datos mediante Azure Data Studio

1. Si aún no lo has hecho, clona los scripts del laboratorio desde el repositorio de GitHub de [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) localmente:
    1. Abre una línea de comandos o un terminal.
    1. Ejecute el comando:
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > NOTA
       > 
       > Si **git** no está instalado, [descarga e instala la aplicación ***git*** ](https://git-scm.com/download)e intenta volver a ejecutar los comandos anteriores.
1. Si aún no has instalado Azure Data Studio, [descarga e instala ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Si no has instalado la extensión **PostgreSQL** en Azure Data Studio, instálala ahora.
1. Abra Azure Data Studio.
1. Seleccione **Conexiones**.
1. Seleccione **Servidores** y **Nueva conexión**.
1. En **Tipo de conexión**, seleccione **PostgreSQL**.
1. En **Nombre del servidor**, escriba el valor que especificó al implementar el servidor.
1. En **Nombre de usuario**, escribe **pgAdmin**.
1. En **Contraseña**, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin** que generaste.
1. Seleccione **Remember password** (Recordar contraseña).
1. Haga clic en **Conectar**
1. Si aún no has creado la base de datos zoodb, selecciona **Archivo**, **Abrir archivo** y ve a la carpeta donde guardaste los scripts. Selecciona **../Allfiles/Labs/02/Lab2_ZooDb.sql** y **Abrir**.
   1. Resalta las instrucciones **DROP** y **CREATE** y ejecútalas.
   1. En la parte superior de la pantalla, use la flecha desplegable para mostrar las bases de datos del servidor, incluidas ZooDb y las bases de datos del sistema. Selecciona la base de datos **zoodb**.
   1. Resalta las secciones **Crear tablas**, **Crear claves externas** y **Rellenar tablas** y ejecútalas.
   1. Resalta las 3 instrucciones **SELECT** al final del script y ejecútalos para comprobar que las tablas se crearon y rellenaron.

## Tarea 1: Exploración del proceso de vaciado en PostgreSQL

1. Si no está abierto, abre Azure Data Studio.
1. En Azure Data Studio, seleccione **Archivo**, **Abrir archivo** y, a continuación, vaya a los scripts de laboratorio. Selecciona **../Allfiles/Labs/07/Lab7_vacuum.sql** y, después, selecciona **Abrir**. Si es necesario, vuelva a conectarse al servidor.
1. Selecciona la base de datos **zoodb** en la lista desplegable de bases de datos.
1. Resalte y ejecute la sección **Check zoodb database is selected**. Si es necesario, convierta ZooDb en la base de datos actual mediante la lista desplegable.
1. Resalte y ejecute la sección **Display dead tuples**. Esta consulta muestra el número de tuplas inactivas y activas de la base de datos. Anote el número de tuplas inactivas.
1. Resalta y ejecuta la sección **Change weight** 10 veces seguidas. Esta consulta actualiza la columna de peso de todos los animales.
1. Vuelva a ejecutar la sección **Display dead tuples**. Anote el número de tuplas inactivas después de que se hayan realizado las actualizaciones.
1. Ejecute la sección de **Manually run VACUUM** para ejecutar el proceso de vacado.
1. Vuelva a ejecutar la sección **Display dead tuples**. Anote el número de tuplas inactivas después de ejecutar el proceso de vaciado.

## Tarea 2: Configuración de parámetros del servidor de vaciado automático

1. En Azure Portal, vaya al servidor flexible de Azure Database for PostgreSQL.
1. En **Configuración**, seleccione **Parámetros del servidor**.
1. En la barra de búsqueda, escribe **`vacuum`**. Busque los parámetros siguientes y cambie los valores como se indica a continuación:
    1. autovacuum = ON (debe estar activado de forma predeterminada)
    1. autovacuum_vacuum_scale_factor = 0.1
    1. autovacuum_vacuum_threshold = 50

    Esto es como ejecutar el proceso de vaciado automático cuando el 10 % de una tabla tiene filas marcadas para su eliminación, o cuando hay 50 filas actualizadas o eliminadas en cualquier tabla.

1. Seleccione **Guardar**. El servidor se reinicia.

## Tarea 3: Visualización de metadatos de PostgreSQL en Azure Portal

1. Vaya a [Azure Portal](https://portal.azure.com) e inicie sesión.
1. Busca **Azure Database for PostgreSQL** y selecciónalo.
1. Seleccione el servidor flexible de Azure Database for PostgreSQL que creó para este ejercicio.
1. En **Supervisión**, seleccione **Métricas**.
1. Seleccione **Métrica** y **Porcentaje de CPU**.
1. Tenga en cuenta que puede ver varias métricas sobre las bases de datos.

## Tarea 4: Visualización de datos en tablas del catálogo del sistema

1. Cambie a Azure Data Studio.
1. En **SERVIDORES**, seleccione el servidor de PostgreSQL y espere hasta que se realice una conexión y se muestre un círculo verde en el servidor.
1. Haga clic con el botón derecho en el servidor y seleccione **Nueva consulta**.
1. Escriba el siguiente SQL y seleccione **Ejecutar**:

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. Tenga en cuenta que puede ver confirmaciones y reversiones para cada base de datos.

## Visualización de una consulta de metadatos compleja mediante una vista del sistema

1. Haga clic con el botón derecho en el servidor y seleccione **Nueva consulta**.
1. Escriba el siguiente SQL y seleccione **Ejecutar**:

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. Tenga en cuenta que puede ver una gran cantidad de información estadística.
1. Al utilizar las vistas del sistema, puede reducir la complejidad del SQL que necesita escribir. La consulta anterior necesitaría el código siguiente si no estaba usando la vista **pg_stats**:

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

## Limpieza del ejercicio

1. La instancia de Azure Database for PostgreSQL que hemos implementado en este ejercicio incurrirá en cargos que puedes eliminar del servidor después de este ejercicio. Como alternativa, puedes eliminar el grupo de recursos **rg-learn-work-with-postgresql-eastus** para quitar todos los recursos implementados como parte de este ejercicio.
1. Si es necesario, elimina la carpeta .\DP3021Lab.
