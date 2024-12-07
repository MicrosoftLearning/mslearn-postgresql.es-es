---
lab:
  title: Extracción de información con Lenguaje de Azure AI
  module: Extract insights using the Azure AI Language service with Azure Database for PostgreSQL
---

# Extracción de información mediante el servicio Lenguaje de Azure AI con Azure Database for PostgreSQL

Recuerda que la empresa inmobiliaria quiere analizar las tendencias de mercado, como las frases o lugares más populares. El equipo también pretende mejorar las protecciones para la información de identificación personal (PII). Los datos actuales se almacenan en un servidor flexible de Azure Database for PostgreSQL. El presupuesto del proyecto es pequeño, por lo que es esencial minimizar los costes iniciales y los costes continuos en el mantenimiento de palabras clave y etiquetas. Los desarrolladores desconfían de las múltiples formas que DCP puede adoptar y prefieren una solución rentable y contrastada a un buscador interno de coincidencias de expresiones regulares.

Integrarás la base de datos con los servicios de Lenguaje Azure AI mediante la extensión `azure_ai`. La extensión proporciona API de función SQL definidas por el usuario a varias API de Azure Cognitive Service, entre las que se incluyen:

- la extracción de frases clave
- reconocimiento de entidades
- Reconocimiento de DCP

Este enfoque permitirá que el equipo de ciencia de datos se una rápidamente a los datos de popularidad de ofertas inmobiliarias para determinar las tendencias del mercado. También proporcionará a los desarrolladores de aplicaciones un texto seguro de DCP para que lo presenten en situaciones que no requieren acceso. El almacenamiento de entidades identificadas permite la revisión humana en caso de investigación o reconocimiento de DCP falso positivo (pensar que algo es DCP y que no sea).

Al final, tendrás cuatro nuevas columnas en la tabla `listings` con información extraída:

- `key_phrases`
- `recognized_entities`
- `pii_safe_description`
- `pii_entities`

## Antes de comenzar

Necesitarás una [suscripción a Azure](https://azure.microsoft.com/free) en la que tengas derechos administrativos

### Implementación de recursos en tu suscripción a Azure

Este paso te guiará por el uso de los comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en la suscripción a Azure.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure Portal, con el icono de Cloud Shell resaltado por un cuadro rojo.](media/17-portal-toolbar-cloud-shell.png)

    Si se te solicita, selecciona las opciones necesarias para abrir un shell de *Bash* . Si anteriormente has usado una consola de *PowerShell*, cámbiala a un shell de *Bash*.

3. En el símbolo del sistema de Cloud Shell, escribe lo siguiente para clonar el repositorio de GitHub que contiene recursos del ejercicio:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. A continuación, ejecutarás tres comandos para definir variables para reducir la escritura redundante al usar comandos de la CLI de Azure para crear recursos de Azure. Las variables representan el nombre que se asignará al grupo de recursos (`RG_NAME`), la región de Azure (`REGION`) en la que se implementarán los recursos y una contraseña generada aleatoriamente para el inicio de sesión de administrador de PostgreSQL (`ADMIN_PASSWORD`).

    En el primer comando, la región asignada a la variable correspondiente es `eastus`, pero también puedes reemplazarla por una ubicación de tu preferencia. Sin embargo, si reemplazas el valor predeterminado, deberás seleccionar otra [región de Azure que admita el resumen abstracto](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) para asegurarte de que puedes completar todas las tareas de los módulos de esta ruta de aprendizaje.

    ```bash
    REGION=eastus
    ```

    El siguiente comando asigna el nombre que se usará para el grupo de recursos que hospedará todos los recursos usados en este ejercicio. El nombre del grupo de recursos asignado a la variable correspondiente es `rg-learn-postgresql-ai-$REGION`, donde `$REGION` es la ubicación especificada anteriormente. Sin embargo, puedes cambiarlo a cualquier otro nombre de grupo de recursos que se adapte a tu preferencia.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    El comando final genera aleatoriamente una contraseña para el inicio de sesión de administrador de PostgreSQL. **Asegúrate de copiarlo** en un lugar seguro para usarlo más adelante para conectarte al servidor flexible de PostgreSQL.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados incluyen un servidor flexible de Azure Database for PostgreSQL, Azure OpenAI y un servicio de Lenguaje de Azure AI. El script de Bicep también realiza algunos pasos de configuración, como agregar las extensiones `azure_ai` y `vector` a la _lista de permitidos_ del servidor PostgreSQL (a través del parámetro de servidor `azure.extensions`), crear una base de datos denominada `rentals` en el servidor y agregar una implementación denominada `embedding` mediante el modelo `text-embedding-ada-002` a Azure OpenAI Service. Ten en cuenta que todos los módulos de esta ruta de aprendizaje comparten el archivo Bicep, por lo que solo podrás usar algunos de los recursos implementados en algunos ejercicios.

    La implementación suele tarda varios minutos en completarse. Puedes supervisarla desde Cloud Shell o ir a la página **Implementaciones** del grupo de recursos que creaste anteriormente y observar el progreso de la implementación allí.

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

1. En [Azure Portal](https://portal.azure.com/), ve al servidor flexible recién creado de Azure Database for PostgreSQL.

2. En el menú de recursos, en **Configuración**, selecciona **Bases de datos** selecciona **Conectar** para la base de datos `rentals`.

    ![Captura de pantalla de la página Base de datos de Azure Database for PostgreSQL. Bases de datos y Conectar la base de datos de alquileres están resaltadas por cuadros rojos.](media/17-postgresql-rentals-database-connect.png)

3. En el símbolo del sistema "Contraseña para el usuario pgAdmin" de Cloud Shell, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin**.

    Una vez que hayas iniciado sesión, se muestra la solicitud `psql` de la base de datos `rentals`.

4. Durante el resto de este ejercicio, seguirás trabajando en Cloud Shell, por lo que puede resultar útil expandir el panel dentro de la ventana del explorador seleccionando el botón **Maximizar** en la parte superior derecha del panel.

    ![Captura de pantalla del panel de Azure Cloud Shell con el botón Maximizar resaltado por un cuadro rojo.](media/17-azure-cloud-shell-pane-maximize.png)

## Configuración: configuración de extensiones

Para almacenar y consultar vectores, y para generar incrustaciones, deberás habilitar dos extensiones para el servidor flexible de Azure Database for PostgreSQL: `vector` y `azure_ai`.

1. Para enumerar ambas extensiones, agrega `vector` y `azure_ai` al parámetro de servidor `azure.extensions`, según las instrucciones proporcionadas en [Uso de extensiones de PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Para habilitar la extensión `vector`, ejecuta el siguiente comando de SQL. Para obtener instrucciones detalladas, consulta [Habilitación y uso de `pgvector` en el servidor flexible de Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Para habilitar la extensión `azure_ai`, ejecuta el siguiente comando de SQL. Necesitarás el punto de conexión y la clave de API para el recurso de ***Azure OpenAI***. Para obtener instrucciones detalladas, lee [Habilitar la extensión `azure_ai`](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

4. Para realizar correctamente llamadas a tus servicios de ***Lenguaje de Azure AI*** mediante la extensión `azure_ai`, debes proporcionar su punto de conexión y clave a la extensión. Con la misma pestaña del explorador donde está abierto Cloud Shell, ve al recurso de servicio de lenguaje en [Azure Portal](https://portal.azure.com/) y selecciona el elemento **Claves y punto de conexión** en **Administración de recursos** en el menú de navegación izquierdo.

5. Copia los valores de punto de conexión y clave de acceso y, a continuación, en los comandos siguientes, reemplaza los tokens `{endpoint}` y `{api-key}` por los valores que copiaste de Azure Portal. Ejecuta los comandos desde el símbolo del sistema `psql` en Cloud Shell para agregar los valores a la tabla `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Rellenado de la base de datos con datos de ejemplo

Antes de explorar la extensión `azure_ai`, agrega un par de tablas a la base de datos `rentals` y rellénalas con datos de ejemplo para que tengas información con la que trabajar mientras revisas la funcionalidad de la extensión.

1. Ejecuta los siguientes comandos para crear las tablas `listings` y `reviews` para almacenar los datos de listados de propiedades de alquiler y de reseñas de clientes:

    ```sql
    DROP TABLE IF EXISTS listings;
    
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

    ```sql
    DROP TABLE IF EXISTS reviews;
    
    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. A continuación, usa el comando `COPY` para cargar datos de archivos CSV en cada tabla que creaste anteriormente. Empieza por ejecutar el siguiente comando para rellenar la tabla `listings`:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 50`, que indica que se han escrito 50 filas en la tabla desde el archivo CSV.

3. Por último, ejecuta el comando siguiente para cargar las reseñas de clientes en la tabla `reviews`:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 354`, que indica que se han escrito 354 filas en la tabla desde el archivo CSV.

Para restablecer los datos de ejemplo, puedes ejecutar `DROP TABLE listings` y repetir estos pasos.

## Extracción de frases clave

1. Las frases clave se extraen como `text[]` según revela la función `pg_typeof`:

    ```sql
    SELECT pg_typeof(azure_cognitive.extract_key_phrases('The food was delicious and the staff were wonderful.', 'en-us'));
    ```

    Crea una columna para que contenga los resultados clave.

    ```sql
    ALTER TABLE listings ADD COLUMN key_phrases text[];
    ```

1. Rellena la columna en lotes. Dependiendo de la cuota, es posible que desees ajustar el valor `LIMIT`. *No dudes en ejecutar el comando tantas veces como quieras*; no necesitas que todas las filas estén rellenadas para este ejercicio.

    ```sql
    UPDATE listings
    SET key_phrases = azure_cognitive.extract_key_phrases(description)
    FROM (SELECT id FROM listings WHERE key_phrases IS NULL ORDER BY id LIMIT 100) subset
    WHERE listings.id = subset.id;
    ```

1. Consulta de listados por frases clave:

    ```sql
    SELECT id, name FROM listings WHERE 'closet' = ANY(key_phrases);
    ```

    Obtendrás resultados como este, dependiendo de qué listados tengan frases clave rellenadas:

    ```sql
       id    |                name                
    ---------+-------------------------------------
      931154 | Met Tower in Belltown! MT2
      931758 | Hottest Downtown Address, Pool! MT2
     1084046 | Near Pike Place & Space Needle! MT2
     1084084 | The Best of the Best, Seattle! MT2
    ```

## Reconocimiento de entidades con nombre

1. Las entidades se extraen como `azure_cognitive.entity[]` según revela la función `pg_typeof`:

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Crea una columna para que contenga los resultados clave.

    ```sql
    ALTER TABLE listings ADD COLUMN entities azure_cognitive.entity[];
    ```

2. Rellena la columna en lotes. Este proceso puede tardar varios minutos. Puede que desees ajustar el valor `LIMIT` en función de la cuota o devolver más rápidamente resultados parciales. *No dudes en ejecutar el comando tantas veces como quieras*; no necesitas que todas las filas estén rellenadas para este ejercicio.

    ```sql
    UPDATE listings
    SET entities = azure_cognitive.recognize_entities(description, 'en-us')
    FROM (SELECT id FROM listings WHERE entities IS NULL ORDER BY id LIMIT 500) subset
    WHERE listings.id = subset.id;
    ```

3. Ahora puedes consultar todas las entidades de la lista para buscar propiedades con sótanos:

    ```sql
    SELECT id, name
    FROM listings, unnest(listings.entities) AS e
    WHERE e.text LIKE '%roof%deck%'
    LIMIT 10;
    ```

    Que devuelve algo como esto:

    ```sql
       id    |                name                
    ---------+-------------------------------------
      430610 | 3br/3ba. modern, roof deck, garage
      430610 | 3br/3ba. modern, roof deck, garage
     1214306 | Private Bed/bath in Home: green (A)
       74328 | Spacious Designer Condo
      938785 | Best Ocean Views By Pike Place! PA1
       23430 | 1 Bedroom Modern Water View Condo
      828298 | 2 Bedroom Sparkling City Oasis
      338043 | large modern unit & fab location
      872152 | Luxurious Local Lifestyle 2Bd/2+Bth
      116221 | Modern, Light-Filled Fremont Flat
    ```

## Reconocimiento de DCP

1. Las entidades se extraen como `azure_cognitive.pii_entity_recognition_result` según revela la función `pg_typeof`:

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_pii_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Este valor es un tipo compuesto que contiene texto redactado y una matriz de entidades DCP, tal como se comprueba mediante:

    ```sql
    \d azure_cognitive.pii_entity_recognition_result
    ```

    Que imprime:

    ```sql
         Composite type "azure_cognitive.pii_entity_recognition_result"
         Column    |           Type           | Collation | Nullable | Default 
    ---------------+--------------------------+-----------+----------+---------
     redacted_text | text                     |           |          | 
     entities      | azure_cognitive.entity[] |           |          | 
    ```

    Crea una columna para que contenga el texto redactado y otra para las entidades reconocidas:

    ```sql
    ALTER TABLE listings ADD COLUMN description_pii_safe text;
    ALTER TABLE listings ADD COLUMN pii_entities azure_cognitive.entity[];
    ```

2. Rellena la columna en lotes. Este proceso puede tardar varios minutos. Puede que desees ajustar el valor `LIMIT` en función de la cuota o devolver más rápidamente resultados parciales. *No dudes en ejecutar el comando tantas veces como quieras*; no necesitas que todas las filas estén rellenadas para este ejercicio.

    ```sql
    UPDATE listings
    SET
        description_pii_safe = pii.redacted_text,
        pii_entities = pii.entities
    FROM (SELECT id, description FROM listings WHERE description_pii_safe IS NULL OR pii_entities IS NULL ORDER BY id LIMIT 100) subset,
    LATERAL azure_cognitive.recognize_pii_entities(subset.description, 'en-us') as pii
    WHERE listings.id = subset.id;
    ```

3. Ahora puedes mostrar descripciones de las listas con cualquier DCP potencial redactada:

    ```sql
    SELECT description_pii_safe
    FROM listings
    WHERE description_pii_safe IS NOT NULL
    LIMIT 1;
    ```

    Que muestra:

    ```sql
    A lovely stone-tiled room with kitchenette. New full mattress futon bed. Fridge, microwave, kettle for coffee and tea. Separate entrance into book-lined mudroom. Large bathroom with Jacuzzi (shared occasionally with ***** to do laundry). Stone-tiled, radiant heated floor, 300 sq ft room with 3 large windows. The bed is queen-sized futon and has a full-sized mattress with topper. Bedside tables and reading lights on both sides. Also large leather couch with cushions. Kitchenette is off the side wing of the main room and has a microwave, and fridge, and an electric kettle for making coffee or tea. Kitchen table with two chairs to use for meals or as desk. Extra high-speed WiFi is also provided. Access to English Garden. The Ballard Neighborhood is a great place to visit: *10 minute walk to downtown Ballard with fabulous bars and restaurants, great ****** farmers market, nice three-screen cinema, and much more. *5 minute walk to the Ballard Locks, where ships enter and exit Puget Sound
    ```

4. También puedes identificar las entidades reconocidas en DCP; por ejemplo, con la lista idéntica como se ha indicado anteriormente:

    ```sql
    SELECT entities
    FROM listings
    WHERE entities IS NOT NULL
    LIMIT 1;
    ```

    Que muestra:

    ```sql
                            pii_entities                        
    -------------------------------------------------------------
    {"(hosts,PersonType,\"\",0.93)","(Sunday,DateTime,Date,1)"}
    ```

## Comprobar el trabajo

Vamos a asegurarnos de que se han rellenado las frases clave extraídas, las entidades reconocidas y la DCP:

1. Comprobar frases clave:

    ```sql
    SELECT COUNT(*) FROM listings WHERE key_phrases IS NOT NULL;
    ```

    Deberías ver algo parecido a esto, en función del número de lotes que has ejecutado:

    ```sql
    count 
    -------
     100
    ```

2. Comprobar las entidades reconocidas:

    ```sql
    SELECT COUNT(*) FROM listings WHERE entities IS NOT NULL;
    ```

    Deberías ver algo parecido a lo siguiente:

    ```sql
    count 
    -------
     500
    ```

3. Comprobar la DCP redactada:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description_pii_safe IS NOT NULL;
    ```

    Si has cargado un único lote de 100, deberías ver lo siguiente:

    ```sql
    count 
    -------
     100
    ```

    Puedes comprobar cuántos listados han detectado DCP:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description != description_pii_safe;
    ```

    Deberías ver algo parecido a lo siguiente:

    ```sql
    count 
    -------
        87
    ```

4. Comprueba las entidades DCP detectadas: según lo anterior, deberíamos tener 13 sin una matriz DCP vacía.

    ```sql
    SELECT COUNT(*) FROM listings WHERE pii_entities IS NULL AND description_pii_safe IS NOT NULL;
    ```

    Resultado:

    ```sql
    count 
    -------
        13
    ```

## Limpiar

Una vez completado este ejercicio, elimina los recursos de Azure que has creado. Se te cobrará por la capacidad configurada y no por cuanto se use la base de datos. Sigue estas instrucciones para eliminar el grupo de recursos y todos los recursos que creaste para este laboratorio.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/) y, en la página principal, selecciona **Grupos de recursos** en servicios de Azure.

    ![Captura de pantalla de los grupos de recursos resaltados por un cuadro rojo en servicios de Azure en Azure Portal.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. En el filtro de cualquier campo de búsqueda, escribe el nombre del grupo de recursos que creaste para este laboratorio y, después, selecciona el grupo de recursos de la lista.

3. En la página **Información general** del grupo de recursos, selecciona **Eliminar grupo de recursos**.

    ![Captura de pantalla de la hoja Información general del grupo de recursos con el botón Eliminar grupo de recursos resaltado por un cuadro rojo.](media/17-resource-group-delete.png)

4. En el cuadro de diálogo de confirmación, escribe el nombre del grupo de recursos que vas a eliminar y, después, selecciona **Eliminar**.
