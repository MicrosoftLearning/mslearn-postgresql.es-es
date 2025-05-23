---
lab:
  title: Descripción del bloqueo
  module: Understand concurrency in PostgreSQL
---

# Descripción del bloqueo

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
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados son un servidor flexible de Azure Database for PostgreSQL. El script de Bicep también crea la base de datos adventureworks.

    La implementación suele tarda varios minutos en completarse. Puedes supervisarlo desde el terminal de Bash o ir a la página **Implementaciones** del grupo de recursos que has creado anteriormente y observar el progreso de la implementación allí.

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

### Instala psql

Puesto que es necesario copiar los datos del archivo CSV en la base de datos de PostgreSQL, es necesario instalar `psql` en el equipo local. *Si ya has instalado `psql`, omite esta sección.*

1. Para comprobar si **psql** ya está instalado en tu entorno, abre la línea de comandos o el terminal y ejecuta el comando ***psql***. Si devuelve un mensaje como "*psql: error: connection to server on socket...*", significa que la herramienta **psql** ya está instalada en tu entorno y no es necesario volver a instalarla. Puedes omitir esta sección.

1. Instala [psql](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893).

1. En el asistente para instalación, sigue las indicaciones hasta llegar al cuadro de diálogo **Seleccionar componentes**, selecciona **Herramientas de línea de comandos**. Puedes desactivar los demás componentes si no planeas usarlos. Sigue las indicaciones que aparecen en pantalla para completar la instalación.

1. Para comprobar si la instalación se realizó correctamente, abre una nueva ventana de terminal o símbolo del sistema y ejecuta **psql**. Si ves un mensaje como "*psql: error: connection to server on socket...*", significa que la herramienta **psql** se ha instalado correctamente. De lo contrario, es posible que tengas que agregar el directorio bin de PostgreSQL a la variable PATH del sistema.

    1. Si usas Windows, asegúrate de agregar el directorio bin de PostgreSQL a la variable PATH del sistema. El directorio bin suele estar ubicado en `C:\Program Files\PostgreSQL\<version>\bin`.
        1. Puedes comprobar si el directorio bin está en la variable PATH. Para ello, ejecuta el comando `echo %PATH%` en un símbolo del sistema y comprueba si aparece el directorio bin de PostgreSQL. Si no es así, puedes agregarlo manualmente:
        1. Para agregarlo manualmente, haz clic con el botón derecho en el botón **Inicio**.
        1. Selecciona **Sistema** y, a continuación, selecciona **Configuración avanzada del sistema**.
        1. Selecciona el botón **Variables de entorno**.
        1. Haz doble clic en la variable **Path** de la sección **Variables del sistema**.
        1. Selecciona **Nuevo** y agrega la ruta de acceso al directorio bin de PostgreSQL.
        1. Después de agregarla, cierra y vuelve a abrir el símbolo del sistema para que los cambios surtan efecto.

    1. Si usas macOS o Linux, el directorio `bin` de PostgreSQL se encuentra normalmente en `/usr/local/pgsql/bin`.  
        1. Puedes comprobar si este directorio está en la variable de entorno `PATH` mediante la ejecución de `echo $PATH` en un terminal.  
        1. Si no es así, puedes agregarlo editando el archivo de configuración del Shell, normalmente `.bash_profile`, `.bashrc` o `.zshrc`, en función del Shell.

### Rellenado de la base de datos con datos

Una vez hayas comprobado que **psql** está instalado, puedes conectarte al servidor de PostgreSQL mediante la línea de comandos; para ello, abre una ventana de símbolo del sistema o terminal.

> &#128221; Si usas Windows, puedes usar **Windows PowerShell** o el **símbolo del sistema**. Si usas macOS o Linux, puedes usar la aplicación **Terminal**.

La sintaxis para conectarse al servidor es la siguiente:

```sql
psql -h <servername> -p <port> -U <username> <dbname>
```

1. En el símbolo del sistema o terminal, escribe **`--host=<servername>.postgres.database.azure.com`** dónde `<servername>` es el nombre de Azure Database for PostgreSQL que has creado anteriormente.

    Puedes encontrar el nombre del servidor en **Información general** en Azure Portal o como salida del script de Bicep o en Azure Portal.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin postgres
    ```

    Se te pedirá la contraseña de la cuenta de administrador que copiaste anteriormente.

1. Debes crear una tabla dentro de la base de datos y rellenarla con datos de ejemplo para que tengas información con la que trabajar mientras revisas el bloqueo en este ejercicio.

1. Ejecuta el siguiente comando para crear la tabla `production.workorder` para cargar los datos:

    ```sql
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
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

1. Después, usa el comando `COPY` para cargar datos de archivos CSV en la tabla que has creado anteriormente. Empieza por ejecutar el siguiente comando para rellenar la tabla `production.workorder`:

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 72591`, que indica que se han escrito 72 591 filas en la tabla desde el archivo CSV.

1. Cierra la ventana del símbolo del sistema o terminal.

### Conexión a la base de datos con Visual Studio Code

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

## Tarea 1: Investigación del comportamiento de bloqueo predeterminado

1. Abre Visual Studio Code si aún no se ha abierto.

1. Abre la paleta de comandos (Ctrl+Mayús+P) y selecciona **PGSQL: New Query**. Selecciona la nueva conexión que has creado de la lista de la paleta de comandos. Si solicitas una contraseña, escribe la contraseña que has creado para el nuevo rol.

1. En la parte inferior derecha de la ventana **New Query**, asegúrate de que la conexión está en verde. Si no es así, debería decir **PGSQL desconectado**. Selecciona el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL:

    ```sql
    SELECT current_database();
    ```

1. Si la base de datos actual no está establecida en **adventureworks**, debes cambiar la base de datos a **adventureworks**. Para cambiar la base de datos, selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y al seleccionar **Cambiar base de datos de PostgreSQL**. Selecciona `adventureworks` de la lista de bases de datos. Comprueba que la base de datos se ha establecido ahora en `adventureworks` mediante la ejecución de la instrucción **SELECT current_database();**.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL:

    ```sql
    SELECT * FROM production.workorder
    ORDER BY scrappedqty DESC;
    ```

1. Observa que el valor **scrappedqty** de la primera fila es **673**.

1. Es necesario abrir una segunda ventana de consulta para simular una transacción que actualice los datos en la primera ventana de consulta. Abre la paleta de comandos (Ctrl+Mayús+P) y selecciona **PGSQL: New Query**. Selecciona la nueva conexión que has creado de la lista de la paleta de comandos. Si solicitas una contraseña, escribe la contraseña que has creado para el nuevo rol.

1. En la parte inferior derecha de la ventana **New Query**, asegúrate de que la conexión está en verde. Si no es así, debería decir **PGSQL desconectado**. Selecciona el texto **PGSQL desconectado** y después selecciona tu conexión al servidor PostgreSQL de la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL:

    ```sql
    SELECT current_database();
    ```

1. Si la base de datos actual no está establecida en **adventureworks**, debes cambiar la base de datos a **adventureworks**. Para cambiar la base de datos, selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y al seleccionar **Cambiar base de datos de PostgreSQL**. Selecciona `adventureworks` de la lista de bases de datos. Comprueba que la base de datos se ha establecido ahora en `adventureworks` mediante la ejecución de la instrucción **SELECT current_database();**.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL:

    ```sql
    BEGIN TRANSACTION;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Observe que la segunda consulta inicia una transacción, pero no la confirma.

1. Vuelve a la *primera* ventana de consulta y vuelve a ejecutar la consulta en esa ventana.

1. Observa que el valor **stockedqty** de la primera fila sigue siendo **673**. La consulta usa una instantánea de los datos y no ve las actualizaciones de la otra transacción.

1. Selecciona la *segunda* pestaña de consulta, elimina la consulta existente, escribe la consulta siguiente y selecciona **Run**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

## Tarea 2: Aplicación de bloqueos de tabla a una transacción

1. Selecciona la *segunda* pestaña de consulta, escribe la consulta siguiente y selecciona **Ejecutar**.

    ```sql
    BEGIN TRANSACTION;
    LOCK TABLE production.workorder IN ACCESS EXCLUSIVE MODE;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Observe que la segunda consulta inicia una transacción, pero no la confirma.

1. Vuelve a la *primera* consulta y ejecuta la consulta de nuevo.

1. Ten en cuenta que la transacción está bloqueada y no se completará, por mucho que esperes.

1. Selecciona la *segunda* pestaña de consulta, elimina la consulta existente, escribe la consulta siguiente y selecciona **Run**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

1. Vuelve a la *primera* pestaña de consulta, espera unos segundos y observa que la consulta se ha completado y que el bloqueo se ha quitado.

## Limpieza

1. Si ya no necesitas este servidor PostgreSQL para otros ejercicios, para evitar incurrir en costes innecesarios de Azure, elimina el grupo de recursos creado en este ejercicio.

1. Si deseas mantener el servidor PostgreSQL en funcionamiento, puedes dejarlo en ejecución. Si no quiere dejar que se ejecute, puedes detener el servidor para evitar incurrir en costes innecesarios en el terminal de Bash. Para detener el servidor, ejecuta el siguiente comando:

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Reemplaza `<your-server-name>` por el nombre de tu servidor PostgreSQL.

    > &#128221; También puedes detener el servidor desde Azure Portal. En Azure Portal, ve a **Grupos de recursos** y selecciona el grupo de recursos que has creado anteriormente. Selecciona el servidor PostgreSQL y después **Detener** en el menú.

1. Si es necesario, elimina el repositorio Git que has clonado anteriormente.

En este ejercicio, hemos revisado el comportamiento de bloqueo predeterminado. Después, aplicamos bloqueos explícitamente y vimos que, aunque algunos bloqueos proporcionan niveles de protección muy altos, también pueden afectar al rendimiento.
