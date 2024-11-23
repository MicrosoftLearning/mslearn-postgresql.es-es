---
lab:
  title: Creación de un procedimiento almacenado en Azure Database for PostgreSQL
  module: Procedures and functions in PostgreSQL
---

# Creación de un procedimiento almacenado en Azure Database for PostgreSQL

En este ejercicio, crearás un procedimiento almacenado.

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

    ![Captura de pantalla de la barra de herramientas de Azure con el icono de Cloud Shell resaltado en un cuadro rojo.](media/05-portal-toolbar-cloud-shell.png)

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

## Clonación del repositorio de GitHub localmente

Asegúrate de que ya has clonado los scripts de laboratorio de [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git). Si no lo has hecho, para clonar el repositorio localmente:

1. Abre una línea de comandos o un terminal.
1. Ejecute el comando:
    ```bash
    md .\DP3021Lab
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
    ```
    > NOTA
    > 
    > Si **git** no está instalado, [descarga e instala la aplicación ***git*** ](https://git-scm.com/download)e intenta volver a ejecutar los comandos anteriores.

## Instalación de Azure Data Studio

Si no tienes instalado Azure Data Studio:

1. En un explorador, vaya a [Download and install Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio) (Descargar e instalar Azure Data Studio) y, en la plataforma de Windows, seleccione **User installer (recommended)** (Instalador de usuario [recomendado]). El archivo ejecutable se descargará en la carpeta Descargas.
1. Seleccione **Abrir archivo**.
1. Se muestra el contrato de licencia. Lea y **acepte el contrato** y, después, seleccione **Siguiente**.
1. En **Seleccionar tareas adicionales**, seleccione **Agregar a PATH** y cualquier otra adición que necesite. Seleccione **Siguiente**.
1. Se muestra el cuadro de diálogo **Listo para instalar**. Revise la configuración. Seleccione **Atrás** para realizar cambios o **Instalar**.
1. Se muestra el cuadro de diálogo **Completing the Azure Data Studio Setup Wizard** (Finalización del Asistente para instalar Azure Data Studio). Seleccione **Finalizar**. Azure Data Studio se inicia.

## Instalación de la extensión de PostgreSQL

Si no tienes instalada la extensión PostgreSQL en Azure Data Studio:

1. Abra Azure Data Studio si aún no lo ha hecho.
1. En el menú izquierdo, seleccione **Extensiones** para mostrar el panel Extensiones.
1. En la barra de búsqueda, escriba **PostgreSQL**. Se muestra el icono de la extensión de PostgreSQL para Azure Data Studio.
1. Seleccione **Instalar**. La extensión se instala.

## Conexión con el servidor flexible de Azure Database for PostrgreSQL

1. Abra Azure Data Studio si aún no lo ha hecho.
1. En el menú de la izquierda, seleccione **Conexiones**.
1. Seleccione **Nueva conexión**.
1. En **Detalles de conexión**, en **Tipo de conexión**, seleccione **PostgreSQL** en la lista desplegable.
1. En **Nombre del servidor**, escriba el nombre completo del servidor tal y como aparece en Azure Portal.
1. En **Tipo de autenticación**, deje Contraseña.
1. En Nombre de usuario y Contraseña, escribe el nombre de usuario **pgAdmin** y **la contraseña de administrador aleatoria** que creaste anteriormente.
1. Seleccione [ x ] Recordar contraseña.
1. Los demás campos son opcionales.
1. Seleccione **Conectar**. Se ha conectado con el servidor de Azure Database for PostgreSQL.
1. Se muestra una lista de las bases de datos de servidor. Esto incluye las bases de datos del sistema y del usuario.
1. Si aún no has creado la base de datos zoodb, selecciona **Archivo**, **Abrir archivo** y ve a la carpeta donde guardaste los scripts. Selecciona **../Allfiles/Labs/02/Lab2_ZooDb.sql** y **Abrir**.
   1. Resalta las instrucciones **DROP** y **CREATE** y ejecútalas.
   1. En la parte superior de la pantalla, use la flecha desplegable para mostrar las bases de datos del servidor, incluidas ZooDb y las bases de datos del sistema. Selecciona la base de datos **zoodb**.
   1. Resalta las secciones **Crear tablas**, **Crear claves externas** y **Rellenar tablas** y ejecútalas.
   1. Resalta las 3 instrucciones **SELECT** al final del script y ejecútalos para comprobar que las tablas se crearon y rellenaron.

## Creación del procedimiento almacenado repopulate_zoo()

1. En la parte superior de la pantalla, use la flecha desplegable para convertir zoodb en la base de datos actual.
1. En Azure Data Studio, seleccione **Archivo**, **Abrir archivo** y, a continuación, vaya a los scripts de laboratorio. Selecciona **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** y, después, selecciona **Abrir**. Si es necesario, vuelva a conectarse al servidor.
1. Resalta la sección en **Crear procedimiento almacenado** de **DROP PROCEDURE** en **END $$.** Ejecuta el texto resaltado.
1. Mantenga abierto Azure Data Studio con el archivo abierto, listo para el ejercicio siguiente.

## Creación del procedimiento almacenado new_exhibit()

1. En la parte superior de la pantalla, use la flecha desplegable para convertir zoodb en la base de datos actual.
1. En Azure Data Studio, seleccione **Archivo**, **Abrir archivo** y, a continuación, vaya a los scripts de laboratorio. Selecciona **../Allfiles/Labs/05/Lab5_StoredProcedure.sql** y, después, selecciona **Abrir**. Si es necesario, vuelva a conectarse al servidor.
1. Resalta la sección en **Crear procedimiento almacenado** de **DROP PROCEDURE** en **END $$.** Ejecuta el texto resaltado. Lee el procedimiento. Verá que declara algunos parámetros de entrada y los usa para insertar filas en la tabla enclosure y la tabla animal.
1. Mantenga abierto Azure Data Studio con el archivo abierto, listo para el ejercicio siguiente.

## Llame al procedimiento almacenado

1. Resalte la sección bajo **Llamada al procedimiento almacenado**. Ejecuta el texto resaltado. De este modo, se llama al procedimiento almacenado pasando valores a los parámetros de entrada.
1. Resalte y ejecute las dos instrucciones **SELECT**. Ejecuta el texto resaltado. Puede ver que se insertó una fila nueva en enclosure y cinco filas nuevas insertadas en animal.

## Creación y llamada de una función con valores de tabla

1. En Azure Data Studio, seleccione **Archivo**, **Abrir archivo** y, a continuación, vaya a los scripts de laboratorio. Selecciona **../Allfiles/Labs/05/Lab5_Table_Function.sql** y, después, selecciona **Abrir**.
1. Resalte y ejecute la primera instrucción **SELECT** para comprobar que la base de datos zoodb esté seleccionada.
1. Resalte y ejecute el procedimiento almacenado **repopulate_zoo()** para empezar con datos limpios.
1. Resalte y ejecute la sección bajo **Creación de una función con valores de tabla**. Esta función devuelve una tabla denominada **enclosure_summary**. Lea el código de la función para comprender cómo se rellena la tabla.
1. Resalte y ejecute las dos instrucciones Select, pasando un identificador de contenedor diferente cada vez.
1. Resalte y ejecute la sección bajo **Uso de una función con valores de tabla con una combinación LATERAL**. Esto muestra la función con valores de tabla que se usa en lugar de un nombre de tabla en una combinación.

## Ejercicio opcional: Funciones integradas

1. En Azure Data Studio, seleccione **Archivo**, **Abrir archivo** y, a continuación, vaya a los scripts de laboratorio. Selecciona **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql** y, después, selecciona **Abrir**.
1. Resalte y ejecute cada función para ver cómo funciona. Consulte la [documentación en línea](https://www.postgresql.org/docs/current/functions.html) para más información sobre cada función.
1. Cierre Azure Data Studio sin guardar los scripts.
1. DETENGA el servidor de Azure Database for PostgreSQL para que no se le cobre cuando no use el servidor.

## Limpieza

1. Elimina el grupo de recursos creado en este ejercicio para evitar incurrir en costos innecesarios de Azure.
1. Si es necesario, elimina la carpeta .\DP3021Lab.

