---
lab:
  title: Ejecución de la instrucción EXPLAIN
  module: Understand PostgreSQL query processing
---

# Ejecución de la instrucción EXPLAIN

En este ejercicio, verá la función EXPLAIN y cómo puede mostrar el plan de ejecución que el planificador de PostgreSQL genera para una instrucción proporcionada.

## Antes de comenzar

Debe tener una suscripción a Azure propia para completar este ejercicio. Si no tiene una suscripción a Azure, puede obtener una [evaluación gratuita de Azure](https://azure.microsoft.com/free).

## Creación del entorno de ejercicio

En este ejercicio y todos los ejercicios posteriores usarás Bicep en Azure Cloud Shell para implementar el servidor PostgreSQL.
Omite la implementación de recursos y la instalación de Azure Data Studio si ya los tienes instalados.

### Implementación de recursos en tu suscripción a Azure

Este paso te guía por el uso de comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en tu suscripción a Azure.

> Nota:
>
> Si vas a realizar varios módulos en esta ruta de aprendizaje, puedes compartir el entorno de Azure entre ellos. En ese caso, solo debes completar este paso de implementación de recursos una vez.

1. Abra un explorador web y vaya a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure con el icono de Cloud Shell resaltado en un cuadro rojo.](media/03-portal-toolbar-cloud-shell.png)

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si el script no puede crear un recurso de IA debido al requisito de aceptar el contrato de IA responsable, puedes experimentar el siguiente error; en cuyo caso, usa la interfaz de usuario de Azure Portal para crear un recurso de Servicios de Azure AI y, a continuación, vuelve a ejecutar el script de implementación.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Antes de continuar:

Asegúrese de lo siguiente:

1. Ha instalado e iniciado un servidor flexible de Azure Database for PostgreSQL. El script de Bicep anterior debería haber instalado esto.
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
1. Abre Azure Data Studio y conéctate al servidor flexible de Azure Database for PostgreSQL creado por el script de Bicep. Escribe el nombre de usuario **pgAdmin** y **la contraseña de administrador aleatoria** que creaste anteriormente.
1. Si aún no has creado la base de datos zoodb, selecciona **Archivo**, **Abrir archivo** y ve a la carpeta donde guardaste los scripts. Selecciona **../Allfiles/Labs/02/Lab2_ZooDb.sql** y **Abrir**.
   1. Resalta las instrucciones **DROP** y **CREATE** y ejecútalas.
   1. En la parte superior de la pantalla, use la flecha desplegable para mostrar las bases de datos del servidor, incluidas ZooDb y las bases de datos del sistema. Selecciona la base de datos **zoodb**.
   1. Resalta las secciones **Crear tablas**, **Crear claves externas** y **Rellenar tablas** y ejecútalas.
   1. Resalta las 3 instrucciones **SELECT** al final del script y ejecútalos para comprobar que las tablas se crearon y rellenaron.

## Práctica de EXPLAIN ANALYZE

1. En [Azure Portal](https://portal.azure.com), vaya al servidor flexible de Azure Database for PostgreSQL. Compruebe que el servidor se ha iniciado o reinícielo si es necesario.
1. Abra Azure Data Studio y conéctese al servidor flexible de Azure Database for PostgreSQL.
1. Seleccione **Archivo**, **Abrir archivo** y vaya a la carpeta donde guardó los scripts. Abre **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**. Vuelva a conectarse al servidor si es necesario.
1. Seleccione **Ejecutar** para ejecutar la consulta. De esta manera, se rellena la base de datos ZooDb.
1. Selecciona Archivo, **Abrir archivo** y **../Allfiles/Labs/03/Lab3_explain.sql**
1. En el archivo de Laboratorio, en la sección **1. Investigate EXPLAIN ANALYZE**, resalta y ejecuta la Instrucción A y la Instrucción B por separado.
    1. ¿Qué instrucción actualizó la base de datos y por qué?
    1. ¿Cuántos milisegundos se tardó en planear la instrucción A?
    1. ¿Cuál fue el tiempo de ejecución de la instrucción B?

## Práctica de EXPLAIN

1. En el archivo de Laboratorio, en la sección **2. Investigate EXPLAIN**, resalta y ejecuta esa instrucción.
    1. ¿Qué clave de ordenación se usó y por qué?
1. En el archivo de Laboratorio, en la sección **3. Investigate EXPLAIN options**, resalta y ejecuta cada instrucción por separado. Compare las estadísticas del plan de consulta de cada opción.

## Limpieza

1. Elimina el grupo de recursos creado en este ejercicio para evitar incurrir en costos innecesarios de Azure.
1. Si es necesario, elimina la carpeta **.\DP3021Lab**.

