---
lab:
  title: Traducción de texto con Traductor de Azure AI
  module: Translate Text using the Azure AI Translator and Azure Database for PostgreSQL
---

# Traducción de texto con Traductor de Azure AI

Como desarrollador principal de Margie's Travel, se te ha pedido que ayudes en un trabajo de internacionalización. Hoy en día, todas las ofertas inmobiliaria de alquiler del servicio de alquiler a corto plazo de la empresa están en inglés. Quieres traducir estas ofertas inmobiliarias a una variedad de idiomas sin un trabajo de desarrollo extenso. Todos tus datos están hospedados en un servidor flexible de Azure Database for PostgreSQL, y deseas usar Servicios de Azure AI para realizar la traducción.

En este ejercicio, traducirás texto en inglés a varios idiomas mediante el servicio Traductor de Azure AI a través de una base de datos de servidor flexible de Azure Database for PostgreSQL.

## Antes de comenzar

Necesitarás una [suscripción a Azure](https://azure.microsoft.com/free) en la que tengas derechos administrativos

### Implementación de recursos en tu suscripción a Azure

Este paso te guiará por el uso de comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en tu suscripción a Azure.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure Portal, con el icono de Cloud Shell resaltado por un cuadro rojo.](media/11-portal-toolbar-cloud-shell.png)

    Si se te solicita, selecciona las opciones necesarias para abrir un shell de *Bash* . Si anteriormente has usado una consola de *PowerShell*, cámbiala a un shell de *Bash*.

3. En el símbolo del sistema de Cloud Shell, escribe lo siguiente para clonar el repositorio de GitHub que contiene recursos del ejercicio:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

    Si ya has clonado este repositorio de GitHub en un módulo anterior, seguirá estando disponible para ti y puedes recibir el siguiente mensaje de error:

    ```bash
    fatal: destination path 'mslearn-postgresql' already exists and is not an empty directory.
    ```

    Si recibes este mensaje, puedes continuar con el paso siguiente de forma segura.

4. A continuación, ejecutarás tres comandos para definir variables para reducir la escritura redundante al usar comandos de la CLI de Azure para crear recursos de Azure. Las variables representan el nombre que se asignará al grupo de recursos (`RG_NAME`), la región de Azure (`REGION`) en la que se implementarán los recursos y una contraseña generada aleatoriamente para el inicio de sesión de administrador de PostgreSQL (`ADMIN_PASSWORD`).

    En el primer comando, la región asignada a la variable correspondiente es `eastus`, pero también puedes reemplazarla por una ubicación de tu preferencia. Sin embargo, si reemplazas el valor predeterminado, deberás seleccionar otra [región de Azure que admita el resumen abstracto](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) para asegurarte de que puedes completar todas las tareas de los módulos de esta ruta de aprendizaje.

    ```bash
    REGION=eastus
    ```

    El siguiente comando asigna el nombre que se usará para el grupo de recursos que hospedará todos los recursos usados en este ejercicio. El nombre del grupo de recursos asignado a la variable correspondiente es `rg-learn-postgresql-ai-$REGION`, donde `$REGION` es la ubicación especificada anteriormente. Sin embargo, puedes cambiarlo a cualquier otro nombre de grupo de recursos que se adapte a tu preferencia.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    El comando final genera aleatoriamente una contraseña para el inicio de sesión de administrador de PostgreSQL. Asegúrate de copiarlo en un lugar seguro para usarlo más adelante para conectarte al servidor flexible de PostgreSQL.

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

5. Si tienes acceso a más de una suscripción a Azure y la suscripción predeterminada no es en la que deseas crear el grupo de recursos y otros recursos para este ejercicio, ejecuta este comando para establecer la suscripción adecuada, reemplazando el token `<subscriptionName|subscriptionId>` por el nombre o el identificador de la suscripción que deseas usar:

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. Ejecuta el siguiente comando de la CLI de Azure para crear tu grupo de recursos:

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Por último, usa la CLI de Azure para ejecutar un script de implementación de Bicep para aprovisionar recursos de Azure en tu grupo de recursos:

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-translate.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados incluyen un servidor flexible de Azure Database for PostgreSQL y un servicio Traductor de Azure AI. El script Bicep también realiza algunos pasos de configuración, como agregar las extensiones `azure_ai` y `vector` a la _lista de permitidos_ del servidor PostgreSQL (a través del parámetro de servidor azure.extensions) y crear una base de datos denominada `rentals` en el servidor. **Ten en cuenta que el archivo bicep difiere de los demás módulos de esta ruta de aprendizaje.**

    La implementación tarda normalmente varios minutos en completarse. Puedes supervisarla desde Cloud Shell o ir a la página **Implementaciones** del grupo de recursos que creaste anteriormente y observar el progreso de la implementación allí.

8. Cierra el panel de Cloud Shell una vez completada la implementación de recursos.

### Solución de errores de implementación

Es posible que encuentres algunos errores al ejecutar el script de implementación de Bicep.

- Si anteriormente ejecutaste el script de implementación de Bicep para esta ruta de aprendizaje y, posteriormente, eliminaste los recursos, puedes recibir un mensaje de error similar al siguiente si intentas volver a ejecutar el script en un plazo de 48 horas después de eliminar los recursos:

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Si recibes este mensaje, modifica el comando `azure deployment group create` anterior para establecer el parámetro `restore` igual a `true` y vuelve a ejecutarlo.

- Si la región seleccionada está restringida al aprovisionamiento de recursos específicos, deberás establecer la variable `REGION` en otra ubicación y volver a ejecutar los comandos para crear el grupo de recursos y ejecutar el script de implementación de Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si el script no puede crear un recurso de IA debido al requisito de aceptar el acuerdo de IA responsable, puedes experimentar el siguiente error; en cuyo caso, usa la interfaz de usuario de Azure Portal para crear un recurso de Servicios de Azure AI y, después, vuelve a ejecutar el script de implementación.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Conexión a la base de datos mediante psql en Azure Cloud Shell

En esta tarea, te conectarás a la base de datos `rentals` en el servidor de Azure Database for PostgreSQL mediante la [utilidad de línea de comandos psql](https://www.postgresql.org/docs/current/app-psql.html) de [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. En [Azure Portal](https://portal.azure.com/), ve al servidor flexible de Azure Database for PostgreSQL que acabas de crear.

2. En el menú de recursos, en **Configuración**, selecciona **Bases de datos** selecciona **Conectar** para la base de datos `rentals`.

    ![Captura de pantalla de la página Base de datos de Azure Database for PostgreSQL. Bases de datos y Conectar la base de datos de alquileres están resaltadas por cuadros rojos.](media/17-postgresql-rentals-database-connect.png)

3. En el símbolo del sistema "Contraseña para el usuario pgAdmin" de Cloud Shell, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin**.

    Una vez que hayas iniciado sesión, se muestra la solicitud `psql` de la base de datos `rentals`.

4. Durante el resto de este ejercicio, seguirás trabajando en Cloud Shell, por lo que puede resultar útil expandir el panel dentro de la ventana del explorador seleccionando el botón **Maximizar** en la parte superior derecha del panel.

    ![Captura de pantalla del panel Azure Cloud Shell con el botón Maximizar resaltado con un cuadro rojo.](media/17-azure-cloud-shell-pane-maximize.png)

## Rellenado de la base de datos con datos de listados

Debes tener los datos de listados en inglés disponibles para traducirlos. Si no has creado la tabla `listings` en la base de datos `rentals` en un módulo anterior, sigue estas instrucciones para crearla.

1. Ejecuta los siguientes comandos para crear la tabla `listings` para almacenar los datos de listados de propiedades de alquiler:

    ```sql
    CREATE TABLE listings (
        id int,
        name varchar(100),
        description text,
        property_type varchar(25),
        room_type varchar(30),
        price numeric,
        weekly_price numeric
    );
    ```

2. A continuación, usa el comando `COPY` para cargar datos de archivos CSV en cada tabla que creaste anteriormente. Empieza por ejecutar el siguiente comando para rellenar la tabla `listings`:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 50`, que indica que se han escrito 50 filas en la tabla desde el archivo CSV.

## Creación de tablas adicionales para traducción

Tienes los datos `listings` en su lugar, pero necesitas dos tablas adicionales para realizar la traducción.

1. Ejecuta los comandos siguientes para crear las tablas `languages` y `listing_translations`.

    ```sql
    CREATE TABLE languages (
        code VARCHAR(7) NOT NULL PRIMARY KEY
    );
    ```

    ```sql
    CREATE TABLE listing_translations(
        id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        listing_id INT,
        language_code VARCHAR(7),
        description TEXT
    );
    ```

2. A continuación, inserta una fila por idioma para la traducción. En este caso, crearás filas para cinco idiomas: alemán, chino simplificado, hindi, húngaro y swahili.

    ```sql
    INSERT INTO languages(code)
    VALUES
        ('de'),
        ('zh-Hans'),
        ('hi'),
        ('hu'),
        ('sw');
    ```

    La salida del comando debe ser `INSERT 0 5`, que indica que has insertado cinco filas nuevas en la tabla.

## Instalación y configuración de la extensión `azure_ai`

Antes de usar la extensión `azure_ai`, deberás instalarla en la base de datos y configurarla para conectarse a los recursos de Servicios de Azure AI. La extensión `azure_ai` permite integrar Azure OpenAI y los servicios de Lenguaje de Azure AI en la base de datos. Para habilitar la extensión en la base de datos, sigue estos pasos:

1. Ejecuta el siguiente comando en el símbolo del sistema `psql` para comprobar que las extensiones `azure_ai` y `vector` se agregaron correctamente a la _lista de permitidos_ del servidor mediante el script de implementación de Bicep que ejecutaste al configurar el entorno:

    ```sql
    SHOW azure.extensions;
    ```

    El comando muestra la lista de extensiones en la _lista de permitidos_ del servidor. Si todo se ha instalado correctamente, tu salida debe incluir `azure_ai` y `vector`, de la siguiente manera:

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Antes de que se pueda instalar y usar una extensión en el servidor flexible de Azure Database for PostgreSQL, se debe agregar a la _lista de permitidos_ del servidor, como se describe en [cómo usar extensiones de PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Ahora, estás listo para instalar la extensión `azure_ai` mediante el comando [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html).

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` carga una nueva extensión en la base de datos ejecutando su archivo de script. Este script normalmente crea nuevos objetos SQL, como funciones, tipos de datos y esquemas. Se produce un error si ya existe una extensión del mismo nombre. Agregar `IF NOT EXISTS` permite que el comando se ejecute sin producir un error si ya está instalado.

3. A continuación, debes usar la función `azure_ai.set_setting()` para configurar la conexión al servicio Traductor de Azure AI. Con la misma pestaña del explorador donde está abierto Cloud Shell, minimiza o restaura el panel de Cloud Shell y, a continuación, ve al recurso de Traductor de Azure AI en [Azure Portal](https://portal.azure.com/). Una vez que estés en la página de recursos de Traductor de Azure AI, en el menú de recursos, en la sección **Administración de recursos**, selecciona **Claves y punto de conexión** y, a continuación, copia una de las claves disponibles, tu región y el punto de conexión de traducción de documentos.

    ![Captura de pantalla de la página Claves y puntos de conexión del servicio Traductor de Azure AI que se muestra, con los botones de copia de CLAVE 1, Región y Punto de conexión de traducción de documentos resaltados en cuadros rojos.](media/18-azure-ai-translator-keys-and-endpoint.png)

    Puedes usar `KEY 1` o `KEY 2`. Tener siempre dos claves permite rotar y regenerar las claves de forma segura sin provocar una interrupción del servicio.

4. Configura los valores `azure_cognitive` para que apunten a tu punto de conexión de Traductor AI, la clave de suscripción y la región. El valor de `azure_cognitive.endpoint` será la dirección URL de traducción de documentos del servicio. El valor de `azure_cognitive.subscription_key` será Key 1 o Key 2. El valor de `azure_cognitive.region` será la región de la instancia de Traductor de Azure AI.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint','https://<YOUR_ENDPOINT>.cognitiveservices.azure.com/');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<YOUR_KEY>');
    SELECT azure_ai.set_setting('azure_cognitive.region', '<YOUR_REGION>');
    ```

## Creación de un procedimiento almacenado para traducir datos de listados

Para rellenar la tabla de traducción de idioma, crearás un procedimiento almacenado para cargar datos en lotes.

1. Ejecuta el siguiente comando en el símbolo del sistema `psql` para crear un nuevo procedimiento almacenado denominado `translate_listing_descriptions`.

    ```sql
    CREATE OR REPLACE PROCEDURE translate_listing_descriptions(max_num_listings INT DEFAULT 10)
    LANGUAGE plpgsql
    AS $$
    BEGIN
        WITH batch_to_load(id, description) AS
        (
            SELECT id, description
            FROM listings l
            WHERE NOT EXISTS (SELECT * FROM listing_translations ll WHERE ll.listing_id = l.id)
            LIMIT max_num_listings
        )
        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT b.id, l.code, (unnest(tr.translations)).TEXT
        FROM batch_to_load b
            CROSS JOIN languages l
            CROSS JOIN LATERAL azure_cognitive.translate(b.description, l.code) tr;
    END;
    $$;
    ```

    Este procedimiento almacenado cargará un lote de 5 registros, traducirá la descripción en cada idioma que selecciones e insertará las descripciones traducidas en la tabla `listing_translations`.

2. Ejecuta el procedimiento almacenado mediante el siguiente comando SQL:

    ```sql
    CALL translate_listing_descriptions(10);
    ```

    Esta llamada tardará aproximadamente un segundo por listado de alquiler en traducirse en cinco idiomas, por lo que cada ejecución debe tardar aproximadamente 10 segundos. La salida del comando debe ser `CALL`, lo que indica que la llamada al procedimiento almacenado se ha realizado correctamente.

3. Llama al procedimiento almacenado cuatro veces más, por cinco veces que hayas llamado a este procedimiento. Esto generará traducciones para cada listado de la tabla.

4. Ejecuta el siguiente script para obtener el recuento de traducciones de listados.

    ```sql
    SELECT COUNT(*) FROM listing_translations;
    ```

    La llamada debe devolver un valor de 250, lo que indica que cada oferta inmobiliaria se ha traducido a cinco idiomas. Puedes seguir analizando los datos consultando la tabla `listing_translations`.

## Creación de un procedimiento para agregar una nueva oferta inmobiliaria con traducciones

Tienes un procedimiento almacenado para traducir las descripciones existentes, pero los planes de internacionalización también requieren la traducción de nuevas descripciones a medida que se escriben. Para ello, crearás otro procedimiento almacenado.

1. Ejecuta el siguiente comando en el símbolo del sistema `psql` para crear un nuevo procedimiento almacenado denominado `add_listing`.

    ```sql
    CREATE OR REPLACE PROCEDURE add_listing(id INT, name VARCHAR(255), description TEXT)
    LANGUAGE plpgsql
    AS $$
    DECLARE
    listing_id INT;
    BEGIN
        INSERT INTO listings(id, name, description)
        VALUES(id, name, description);

        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT id, l.code, (unnest(tr.translations)).TEXT
        FROM languages l
            CROSS JOIN LATERAL azure_cognitive.translate(description, l.code) tr;
    END;
    $$;
    ```

    Este procedimiento almacenado insertará una fila en la tabla `listings`. Después, traducirás la descripción de cada idioma de la tabla `languages` e insertarás estas traducciones en la tabla `listing_translations`.

2. Ejecuta el procedimiento almacenado mediante el siguiente comando SQL:

    ```sql
    CALL add_listing(51, 'A Beautiful Home', 'This is a beautiful home in a great location.');
    ```

    La salida del comando debe ser `CALL`, lo que indica que la llamada al procedimiento almacenado se ha realizado correctamente.

3. Ejecuta el siguiente script para obtener las traducciones de la nueva descripción.

    ```sql
    SELECT l.id, l.name, l.description, lt.language_code, lt.description AS translated_description
    FROM listing_translations lt
        INNER JOIN listings l ON lt.listing_id = l.id
    WHERE l.name = 'A Beautiful Home';
    ```

    La llamada debe devolver cinco filas, con valores similares a la tabla siguiente.

    ```sql
     id  | listing_id | language_code |                    description                     
    -----+------------+---------------+------------------------------------------------------
     126 |          2 | de            | Dies ist ein schönes Haus in einer großartigen Lage.
     127 |          2 | zh-Hans       | 这是一个美丽的家，地理位置优越。
     128 |          2 | hi            | यह एक महान स्थान में एक सुंदर घर है।
     129 |          2 | hu            | Ez egy gyönyörű otthon egy nagyszerű helyen.
     130 |          2 | sw            | Hii ni nyumba nzuri katika eneo kubwa.
    ```

## Limpiar

Una vez completado este ejercicio, elimina los recursos de Azure que has creado. Se te cobrará por la capacidad configurada y no por cuanto se use la base de datos. Sigue estas instrucciones para eliminar el grupo de recursos y todos los recursos que creaste para este laboratorio.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/) y, en la página principal, selecciona **Grupos de recursos** en servicios de Azure.

    ![Captura de pantalla de los grupos de recursos resaltados con un cuadro rojo en Servicios de Azure en Azure Portal.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. En el cuadro de búsqueda Filtrar por cualquier campo, escribe el nombre del grupo de recursos que has creado para este laboratorio y después selecciona el grupo de recursos de la lista.

3. En la página **Información general** del grupo de recursos, selecciona **Eliminar grupo de recursos**.

    ![Captura de pantalla de la hoja Información general del grupo de recursos con el botón Eliminar grupo de recursos resaltado por un cuadro rojo.](media/17-resource-group-delete.png)

4. En el cuadro de diálogo de confirmación, escribe el nombre del grupo de recursos que vas a eliminar y, después, selecciona **Eliminar**.
