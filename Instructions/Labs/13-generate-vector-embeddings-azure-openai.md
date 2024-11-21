---
lab:
  title: Generación de inserciones de vectores con Azure OpenAI
  module: Enable Semantic Search with Azure Database for PostgreSQL
---

# Generación de inserciones de vectores con Azure OpenAI

Para realizar búsquedas semánticas, primero debes generar vectores de inserción a partir de un modelo, almacenarlos en una base de datos vectorial y, a continuación, consultar las incrustaciones. Crearás una base de datos, la rellenarás con datos de ejemplo y ejecutarás búsquedas semánticas en esas listas.

Al final de este ejercicio, tendrás una instancia de servidor flexible de Azure Database for PostgreSQL con las extensiones `vector` y `azure_ai` habilitadas. Generarás incrustaciones para la tabla `listings` del conjunto de datos [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle?select=listings.csv). También ejecutarás búsquedas semánticas en estas listas mediante la generación del vector de inserción de una consulta y la realización de una búsqueda de distancia de coseno de vectores.

## Antes de comenzar

Necesitas una [suscripción a Azure](https://azure.microsoft.com/free) con derechos administrativos y debes tener aprobación para el acceso a Azure OpenAI en esa suscripción. Si necesita acceso a Azure OpenAI, solicítelo en la página [Acceso limitado de Azure OpenAI](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implementación de recursos en tu suscripción a Azure

Este paso te guía por el uso de comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en tu suscripción a Azure.

> **Nota**: si vas a realizar varios módulos en esta ruta de aprendizaje, puedes compartir el entorno de Azure entre ellos. En ese caso, solo debes completar este paso de implementación de recursos una vez.

1. Abra un explorador web y vaya a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure con el icono de Cloud Shell resaltado en un cuadro rojo.](media/13-portal-toolbar-cloud-shell.png)

    Si se te solicita, selecciona las opciones necesarias para abrir un shell de *Bash*. Si anteriormente has usado una consola de *PowerShell*, cámbiala a un shell de *Bash*.

3. En el símbolo del sistema de Cloud Shell, escribe lo siguiente para clonar el repositorio de GitHub que contiene recursos del ejercicio:

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. A continuación, ejecutarás tres comandos para definir variables para reducir la escritura redundante al usar comandos de la CLI de Azure para crear recursos de Azure. Las variables representan el nombre que se va a asignar a tu grupo de recursos (`RG_NAME`), la región de Azure (`REGION`) en la que se implementarán los recursos y una contraseña generada aleatoriamente para el inicio de sesión de administrador de PostgreSQL (`ADMIN_PASSWORD`).

    En el primer comando, la región asignada a la variable correspondiente es `eastus`, pero también puedes reemplazarla por una ubicación de tu preferencia. Sin embargo, si reemplazas el valor predeterminado, debes seleccionar otra [región de Azure compatible con el resumen abstracto](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) para asegurarte de que puedes completar todas las tareas de los módulos de esta ruta de aprendizaje.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados incluyen un servidor flexible de Azure Database for PostgreSQL, Azure OpenAI y un servicio de lenguaje de Azure AI. El script de Bicep también realiza algunos pasos de configuración, como agregar las extensiones `azure_ai` y `vector` a la _lista de permitidos_ del servidor PostgreSQL (a través del parámetro de servidor `azure.extensions`), crear una base de datos denominada `rentals` en el servidor y agregar una implementación denominada `embedding` mediante el modelo `text-embedding-ada-002` a Azure OpenAI Service. Ten en cuenta que todos los módulos de esta ruta de aprendizaje comparten el archivo Bicep, por lo que solo puedes usar algunos de los recursos implementados en algunos ejercicios.

    La implementación tarda normalmente varios minutos en completarse. Puedes supervisarla desde Cloud Shell o ir a la página **Implementaciones** del grupo de recursos que creaste anteriormente y observar allí el progreso de la implementación.

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

## Conéctate a tu base de datos mediante psql en Azure Cloud Shell

En esta tarea, te conectarás a la base de datos `rentals` en el servidor de Azure Database for PostgreSQL mediante la [utilidad de línea de comandos psql](https://www.postgresql.org/docs/current/app-psql.html) de [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. En [Azure Portal](https://portal.azure.com/), ve al servidor flexible de Azure Database for PostgreSQL recién creado.

2. En el menú de recursos, en **Configuración**, selecciona **Bases de datos** selecciona **Conectar** en la base de datos `rentals`.

    ![Captura de pantalla de la página de bases de datos de Azure Database for PostgreSQL. Bases de datos y Conectar la base de datos de alquileres están resaltados con cuadros rojos.](media/13-postgresql-rentals-database-connect.png)

3. En el símbolo del sistema "Contraseña para el usuario pgAdmin" de Cloud Shell, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin**.

    Una vez iniciada la sesión, se muestra la solicitud `psql` de la base de datos `rentals`.

4. Durante el resto de este ejercicio, seguirás trabajando en Cloud Shell, por lo que puede ser útil expandir el panel dentro de la ventana del explorador al seleccionar el botón **Maximizar** en la parte superior derecha del panel.

    ![Captura de pantalla del panel Azure Cloud Shell con el botón Maximizar resaltado con un cuadro rojo.](media/13-azure-cloud-shell-pane-maximize.png)

## Configuración: Configuración de extensiones

Para almacenar y consultar vectores, y para generar incrustaciones, debes agregar a la lista de permitidos y habilitar dos extensiones para el servidor flexible de Azure Database for PostgreSQL: `vector` y `azure_ai`.

1. Para enumerar ambas extensiones, agrega `vector` y `azure_ai` al parámetro de servidor `azure.extensions`, según las instrucciones proporcionadas en [¿Cómo se utilizan las extensiones de PostgreSQL?](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Para habilitar la extensión `vector`, ejecuta el siguiente comando SQL. Para obtener instrucciones detalladas, consulta [Habilitación y uso de `pgvector` en un servidor flexible de Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Para habilitar la extensión `azure_ai`, ejecuta el siguiente comando SQL. Necesitarás el punto de conexión y la clave de API del recurso de Azure OpenAI. Para obtener instrucciones detalladas, lee [Habilitar la extensión `azure_ai`](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

## Rellenar la base de datos con datos de ejemplo

Antes de explorar la extensión `azure_ai`, agrega un par de tablas a la base de datos `rentals` y rellénalas con datos de ejemplo para que tengas información con la que trabajar mientras revisas la funcionalidad de la extensión.

1. Ejecuta los siguientes comandos para crear las tablas `listings` y `reviews` para almacenar la lista de propiedades de alquiler y los datos de revisión de clientes:

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

2. A continuación, usa el comando `COPY` para cargar datos de archivos CSV en cada tabla que creaste anteriormente. Comienza por ejecutar el siguiente comando para rellenar la tabla `listings`:

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 50`, que indica que se han escrito 50 filas en la tabla desde el archivo CSV.

3. Por último, ejecuta el comando siguiente para cargar las revisiones de clientes en la tabla `reviews`:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 354`, que indica que se han escrito 354 filas en la tabla desde el archivo CSV.

Para restablecer los datos de ejemplo, puedes ejecutar `DROP TABLE listings` y repetir estos pasos.

## Creación y almacenamiento de vectores de inserción

Ahora que tenemos algunos datos de ejemplo, es el momento de generar y almacenar los vectores de inserción. La extensión `azure_ai` facilita la llamada a la API de inserción de Azure OpenAI.

1. Agrega la columna de vector de inserción.

    El modelo `text-embedding-ada-002` está configurado para devolver 1536 dimensiones, por tanto, úsalo para el tamaño de columna vectorial.

    ```sql
    ALTER TABLE listings ADD COLUMN listing_vector vector(1536);
    ```

1. Genera un vector de inserción para la descripción de cada lista con una llamada a Azure OpenAI a través de la función definida por el usuario create_embeddings, que implementa la extensión azure_ai:

    ```sql
    UPDATE listings
    SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
    WHERE listing_vector IS NULL;
    ```

    Ten en cuenta que esto puede tardar varios minutos, según la cuota disponible.

1. Para ver un vector de ejemplo, ejecuta esta consulta:

    ```sql
    SELECT listing_vector FROM listings LIMIT 1;
    ```

    Obtendrás un resultado similar al siguiente, pero con 1536 columnas vectoriales:

    ```sql
    postgres=> SELECT listing_vector FROM listings LIMIT 1;
    -[ RECORD 1 ]--+------ ...
    listing_vector | [-0.0018742813,-0.04530062,0.055145424, ... ]
    ```

## Realización de una consulta de búsqueda semántica

Ahora que has listado los datos aumentados con vectores de inserción, es el momento de ejecutar una consulta de búsqueda semántica. Para ello, obtén el vector de inserción de la cadena de consulta y, a continuación, realiza una búsqueda de coseno para buscar las listas cuyas descripciones son más similares semánticamente a la consulta.

1. Genera la inserción para la cadena de consulta.

    ```sql
    SELECT azure_openai.create_embeddings('embedding', 'bright natural light');
    ```

    Obtendrás un resultado como este:

    ```sql
    -[ RECORD 1 ]-----+-- ...
    create_embeddings | {-0.0020871465,-0.002830255,0.030923981, ...}
    ```

1. Usa la inserción en una búsqueda de coseno (`<=>` representa la operación de distancia de coseno), para capturar las 10 listas más similares a la consulta.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Obtendrás un resultado similar a este. Los resultados pueden variar, ya que no se garantiza que los vectores de inserción sean deterministas:

    ```sql
        id    |                name                
    ----------+-------------------------------------
     6796336  | A duplex near U district!
     7635966  | Modern Capitol Hill Apartment
     7011200  | Bright 1 bd w deck. Great location
     8099917  | The Ravenna Apartment
     10211928 | Charming Ravenna Bungalow
     692671   | Sun Drenched Ballard Apartment
     7574864  | Modern Greenlake Getaway
     7807658  | Top Floor Corner Apt-Downtown View
     10265391 | Art filled, quiet, walkable Seattle
     5578943  | Madrona Studio w/Private Entrance
    ```

1. También puedes proyectar la columna `description` para poder leer el texto de las filas coincidentes cuyas descripciones eran semánticamente similares. Por ejemplo, esta consulta devuelve la mejor coincidencia:

    ```sql
    SELECT id, description FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 1;
    ```

    Lo que muestra algo parecido a: 

    ```sql
       id    | description
    ---------+------------
     6796336 | This is a great place to live for summer because you get a lot of sunlight at the living room. A huge living room space with comfy couch and one ceiling window and glass windows around the living room.
    ```

Para comprender intuitivamente la búsqueda semántica, observa que la descripción no contiene realmente los términos "bright" o "natural". Pero destaca "summer" y "sunlight", "windows" y "ceiling window".

## Comprobar el trabajo

Después de realizar los pasos anteriores, la tabla `listings` contiene datos de ejemplo de [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle/data?select=listings.csv) en Kaggle. Las listas se aumentaron con vectores de inserción para ejecutar búsquedas semánticas.

1. Confirma que la tabla de listados tiene cuatro columnas: `id`, `name`, `description` y `listing_vector`.

    ```sql
    \d listings
    ```

    Debe ser similar a la siguiente:

    ```sql
                            Table "public.listings"
          Column    |         Type           | Collation | Nullable | Default 
    ----------------+------------------------+-----------+----------+---------
      id            | integer                |           | not null | 
      name          | character varying(255) |           | not null | 
      description   | text                   |           | not null | 
     listing_vector | vector(1536)           |           |          | 
     Indexes:
        "listings_pkey" PRIMARY KEY, btree (id)
    ```

1. Confirma que al menos una fila tiene una columna listing_vector rellenada.

    ```sql
    SELECT COUNT(*) > 0 FROM listings WHERE listing_vector IS NOT NULL;
    ```

    El resultado debe mostrar un valor `t`, que significa verdadero. Indicación de que hay al menos una fila con incrustaciones de su columna de descripción correspondiente:

    ```sql
    ?column? 
    ----------
    t
    (1 row)
    ```

    Confirma que el vector de inserción tiene 1536 dimensiones:

    ```sql
    SELECT vector_dims(listing_vector) FROM listings WHERE listing_vector IS NOT NULL LIMIT 1;
    ```

    Rendimiento:

    ```sql
    vector_dims 
    -------------
            1536
    (1 row)
    ```

1. Confirma que las búsquedas semánticas devuelven resultados.

    Usa la inserción en una búsqueda de coseno, con la captura de las 10 listas más similares a la consulta.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Obtendrás un resultado similar al siguiente, en función de las filas a las que se asignaron vectores de inserción:

    ```sql
     id |                name                
    --------+-------------------------------------
     315120 | Large, comfy, light, garden studio
     429453 | Sunny Bedroom #2 w/View: Wallingfrd
     17951  | West Seattle, The Starlight Studio
     48848  | green suite seattle - dog friendly
     116221 | Modern, Light-Filled Fremont Flat
     206781 | Bright & Spacious Studio
     356566 | Sunny Bedroom w/View: Wallingford
     9419   | Golden Sun vintage warm/sunny
     136480 | Bright Cheery Room in Seattle House
     180939 | Central District Green GardenStudio
    ```

## Limpiar

Una vez completado este ejercicio, elimina los recursos de Azure que has creado. Se te cobra por la capacidad configurada, no por cuánto se use la base de datos. Sigue estas instrucciones para eliminar el grupo de recursos y todos los recursos que has creado para este laboratorio.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/) y, en la página de inicio, selecciona **Grupos de recursos** en Servicios de Azure.

    ![Captura de pantalla de los grupos de recursos resaltados con un cuadro rojo en Servicios de Azure en Azure Portal.](media/13-azure-portal-home-azure-services-resource-groups.png)

2. En el filtro de cualquier campo de búsqueda, escribe el nombre del grupo de recursos que creaste para este laboratorio y, a continuación, selecciona el grupo de recursos de la lista.

3. En la página **Información general** del grupo de recursos, seleccione **Eliminar grupo de recursos**.

    ![Captura de pantalla de la hoja Información general del grupo de recursos con el botón Eliminar grupo de recursos resaltado con un cuadro rojo.](media/13-resource-group-delete.png)

4. En el cuadro de diálogo de confirmación, escribe el nombre del grupo de recursos que vas a eliminar para confirmar y después selecciona **Eliminar**.
