---
lab:
  title: Configuración de permisos en Azure Database for PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Configuración de permisos en Azure Database for PostgreSQL

En estos ejercicios de laboratorio, asignarás control de acceso basado en rol (RBAC) para controlar el acceso a los recursos de Azure Database for PostgreSQL y concesiones de PostgreSQL para controlar el acceso a las operaciones de base de datos.

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

## Creación de una cuenta de usuario en Microsoft Entra ID

> &#128221; En la mayoría de los entornos de producción o desarrollo, es muy posible que no tengas los privilegios de la cuenta de suscripción para crear cuentas en tu servicio Microsoft Entra ID. En ese caso, si la organización lo permite, intenta pedir al administrador de Microsoft Entra ID que cree una cuenta de prueba automáticamente. *Si no puedes obtener la cuenta de prueba Microsoft Entra, omite esta sección y continúa con la sección **Concesión de acceso a Azure Database for PostgreSQL***.

1. En [Azure Portal](https://portal.azure.com), inicia sesión con una cuenta de Propietario y ve a Microsoft Entra ID.

1. En **Administrar**, seleccione **Usuarios**.

1. En la parte superior izquierda, seleccione **Nuevo usuario** y, a continuación, seleccione **Crear nuevo usuario**.

1. En la página **Nuevo usuario**, escriba estos detalles y seleccione **Crear**:
    - **Nombre principal de usuario:** elige un nombre de Principio.
    - **Nombre para mostrar:** elige un nombre para mostrar
    - **Contraseña:** desactiva **Generar automáticamente la contraseñas** y, a continuación, escribe una contraseña segura. Registra el nombre principal y la contraseña.
    - Seleccionar **Revisar y crear**.

    > &#128161; Cuando se cree el usuario, anota el **nombre principal de usuario** completo para usarlo más adelante para iniciar sesión.

### Asignación del rol Lector

1. En el Azure Portal, seleccione **Todos los recursos** y, a continuación, seleccione el recurso de Azure Database for PostgreSQL.

1. Seleccione **Control de acceso (IAM)** y, luego, seleccione **Asignaciones de roles**. La nueva cuenta no aparece en la lista.

1. Seleccione **+ Agregar** y, luego, **Agregar asignación de roles**.

1. Seleccione el rol **Lector** y, a continuación, seleccione **Siguiente**.

1. Elige **+ Seleccionar miembros**, agrega la nueva cuenta que agregaste en el paso anterior a la lista de miembros y, después, selecciona **Siguiente**.

1. Seleccione **Revisar y asignar**.

### Prueba del rol Lector

1. En la esquina superior derecha de Azure Portal, seleccione su cuenta de usuario y, luego, **Cerrar sesión**.

1. Inicia sesión como el nuevo usuario, con el nombre principal de usuario y la contraseña que anotaste. Reemplace la contraseña predeterminada si se le solicita y tome nota de la nueva.

1. Elige **Preguntarme más adelante** si se te solicita autenticación multifactor

1. En la página principal del portal, seleccione **Todos los recursos** y, después, seleccione el recurso Azure Database for PostgreSQL.

1. Seleccione **Detener**. Se muestra un error porque el rol Lector permite ver el recurso, pero no cambiarlo.

### Asignación del rol Colaborador

1. En la esquina superior derecha de Azure Portal, selecciona la cuenta de usuario de la nueva cuenta y, luego, selecciona **Cerrar sesión**.

1. Inicie sesión con su cuenta de propietario original.

1. Vaya al recurso de Azure Database for PostgreSQL y seleccione **Control de acceso (IAM)**.

1. Seleccione **+ Agregar** y, luego, **Agregar asignación de roles**.

1. Selecciona **Roles de administrador con privilegios**

1. Seleccione el rol **Colaborador** y, a continuación, seleccione **Siguiente**.

1. Agrega la nueva cuenta que agregaste anteriormente a la lista de miembros y, a continuación, selecciona **Siguiente**.

1. Seleccione **Revisar y asignar**.

1. Seleccione **Asignaciones de roles**. La nueva cuenta ahora tiene asignaciones para los roles Lector y Colaborador.

## Prueba del rol Colaborador

1. En la esquina superior derecha de Azure Portal, seleccione su cuenta de usuario y, luego, **Cerrar sesión**.

1. Inicia sesión como la nueva cuenta, con el nombre principal de usuario y la contraseña que anotaste.

1. En la página principal del portal, seleccione **Todos los recursos** y, después, seleccione el recurso Azure Database for MySQL.

1. Seleccione **Detener** y, a continuación, seleccione **Sí**. Esta vez, el servidor se detiene sin errores porque la nueva cuenta tiene asignado el rol necesario.

1. Selecciona **Iniciar** para asegurarte de que el servidor PostgreSQL está listo para los pasos siguientes.

1. En la esquina superior derecha de Azure Portal, selecciona la cuenta de usuario de la nueva cuenta y, luego, selecciona **Cerrar sesión**.

1. Inicie sesión con su cuenta de propietario original.

## Concesión de acceso a Azure Database for PostgreSQL

En esta sección, crearás un nuevo rol en la base de datos PostgreSQL y le asignarás permisos para acceder a la base de datos. También probarás el nuevo rol para asegurarte de que tiene los permisos correctos.

1. Abre Visual Studio Code si aún no se ha abierto.

1. Abre la paleta de comandos (Ctrl+Mayús+P) y selecciona **PGSQL: New Query**.

1. Selecciona la conexión del servidor PostgreSQL en la lista de la paleta de comandos. Si solicita una contraseña, escribe la contraseña que has generado anteriormente.

1. En la ventana **New Query**, cambia la base de datos a **zoodb**. Para cambiar la base de datos, selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y al seleccionar **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos. Verifica que la base de datos está ahora en `zoodb` al ejecutar la instrucción **SELECT current_database();**.

1. En el panel de consulta, copia, resalta y ejecuta la siguiente instrucción SQL en la base de datos Postgres. Se deben devolver varios roles de usuario, incluido el rol de **pgAdmin** que usas para conectarte:

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Para crear un nuevo rol, ejecute este código.

    ```SQL
    -- Make sure to change the password to a complex one
    -- and replace the password in the script below
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```

    > &#128221; Asegúrate de reemplazar la contraseña en el script anterior por una contraseña compleja.

1. Para incluir en la lista el nuevo rol, vuelve a ejecutar la consulta SELECT anterior en **pg_catalog.pg_roles**. Debería ver el rol **dbuser** en la lista.

1. Para habilitar el nuevo rol para consultar y modificar datos en la tabla **animal** de la base de datos **zoodb**, 

    1. En la ventana **New Query**, cambia la base de datos a **zoodb**. Para cambiar la base de datos, selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y al seleccionar **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos. Verifica que la base de datos está ahora en `zoodb` al ejecutar la instrucción **SELECT current_database();**.

    1. ejecuta este código en la base de datos `zoodb`:

        ```SQL
        GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
        ```

## Prueba de la aplicación nueva

Vamos a probar el nuevo rol para asegurarnos de que tiene los permisos correctos.

1. En la extensión **PostgreSQL**, selecciona **+ Agregar conexión** para agregar una nueva conexión.

1. En el cuadro de diálogo **NUEVA CONEXIÓN**, escribe la siguiente información:

    - **Nombre del servidor**: <tu-nombre-servidor>.postgres.database.azure.com
    - **Tipo de autenticación**: contraseña
    - **Nombre de usuario**: dbuser
    - **Contraseña**: la contraseña que usaste al crear el nuevo rol.
    - Marca la casilla **Guardar contraseña**.
    - **Nombre de base de datos**: zoodb
    - **Nombre de conexión**: <tu-nombre-servidor> + "-dbuser"

1. Prueba la conexión al seleccionar **Probar conexión**. Si la conexión se realiza correctamente, selecciona **Guardar y conectar** para guardar la conexión; de lo contrario, revisa la información de conexión e inténtalo de nuevo.

1. Abre la paleta de comandos (Ctrl+Mayús+P) y selecciona **PGSQL: New Query**. Selecciona la nueva conexión que has creado de la lista de la paleta de comandos. Si solicitas una contraseña, escribe la contraseña que has creado para el nuevo rol.

1. La conexión debe mostrar la base de datos **zoodb** de forma predeterminada. Si no es así, puedes cambiar la base de datos a **zoodb**. Para cambiar la base de datos, selecciona los puntos suspensivos de la barra de menús con el icono *ejecutar* y al seleccionar **Cambiar base de datos de PostgreSQL**. Selecciona `zoodb` de la lista de bases de datos. Verifica que la base de datos está ahora en `zoodb` al ejecutar la instrucción **SELECT current_database();**.

1. En la ventana **New Query**, copia, resalta y ejecuta la siguiente instrucción SQL en la base de datos **zoodb**:

    ```SQL
    SELECT * FROM animal;
    ```

1. Para probar si tienes el privilegio UPDATE, copia, resalta y ejecuta este código:

    ```SQL
    UPDATE animal SET name = 'Linda Lioness' WHERE ani_id = 7;
    SELECT * FROM animal;
    ```

1. Para probar si tiene el privilegio DROP, ejecute este código. Si se produce un error, examine el código de error:

    ```SQL
    DROP TABLE animal;
    ```

1. Para probar si tiene el privilegio de concesión, ejecute este código:

    ```SQL
    GRANT ALL PRIVILEGES ON animal TO dbuser;
    ```

Estas pruebas muestran que el nuevo usuario puede ejecutar comandos del lenguaje de manipulación de datos (DML) para consultar y modificar datos. Pero el nuevo usuario no puede usar comandos del lenguaje de definición de datos (DDL) para cambiar el esquema. Además, el nuevo usuario no puede conceder privilegios nuevos para eludir los permisos.

## Limpieza

1. Si ya no necesitas este servidor PostgreSQL para otros ejercicios, para evitar incurrir en costes innecesarios de Azure, elimina el grupo de recursos creado en este ejercicio.

1. Si deseas mantener el servidor PostgreSQL en funcionamiento, puedes dejarlo en ejecución. Si no quiere dejar que se ejecute, puedes detener el servidor para evitar incurrir en costes innecesarios en el terminal de Bash. Para detener el servidor, ejecuta el siguiente comando:

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Reemplaza `<your-server-name>` por el nombre de tu servidor PostgreSQL.

    > &#128221; También puedes detener el servidor desde Azure Portal. En Azure Portal, ve a **Grupos de recursos** y selecciona el grupo de recursos que has creado anteriormente. Selecciona el servidor PostgreSQL y después **Detener** en el menú.

1. Si es necesario, elimina el repositorio Git que has clonado anteriormente.

Has completado correctamente este ejercicio. Has aprendido cómo asignar roles de RBAC para controlar el acceso a los recursos de Azure Database for PostgreSQL y concesiones de PostgreSQL para controlar el acceso a las operaciones de base de datos.

También has aprendido a crear una nueva cuenta de usuario en Microsoft Entra ID y asignarle los roles Lector y Colaborador. Por último, has creado un nuevo rol en PostgreSQL y le has asignado permisos para acceder a la base de datos.