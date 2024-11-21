---
lab:
  title: Análisis de opinión
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# Análisis de opinión

Como parte de la aplicación con tecnología de inteligencia artificial que estás creando para Margie's Travel, quieres proporcionar a los usuarios información sobre la opinión de las revisiones individuales y el sentimiento general de todas las opiniones de una propiedad de alquiler determinada. Para lograr esto, usa la extensión `azure_ai` en un servidor flexible Azure Database for PostgreSQL para integrar la funcionalidad de análisis de sentimiento en tu base de datos.

## Antes de comenzar

Necesitas una [suscripción a Azure](https://azure.microsoft.com/free) con derechos administrativos y debes tener aprobación para el acceso a Azure OpenAI en esa suscripción. Si necesita acceso a Azure OpenAI, solicítelo en la página [Acceso limitado de Azure OpenAI](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access).

### Implementación de recursos en tu suscripción a Azure

Este paso te guía por el uso de comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en tu suscripción a Azure.

1. Abra un explorador web y vaya a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure con el icono de Cloud Shell resaltado en un cuadro rojo.](media/11-portal-toolbar-cloud-shell.png)

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

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados incluyen un servidor flexible de Azure Database for PostgreSQL, Azure OpenAI y un servicio de lenguaje de Azure AI. El script de Bicep también realiza algunos pasos de configuración, como agregar las extensiones `azure_ai` y `vector` a la _lista de permitidos_ del servidor PostgreSQL (a través del parámetro de servidor azure.extensions), crear una base de datos denominada `rentals` en el servidor y agregar una implementación denominada `embedding` con el modelo `text-embedding-ada-002` a Azure OpenAI Service. Ten en cuenta que todos los módulos de esta ruta de aprendizaje comparten el archivo Bicep, por lo que solo puedes usar algunos de los recursos implementados en algunos ejercicios.

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

## Conéctate a tu base de datos mediante psql en Azure Cloud Shell

En esta tarea, te conectarás a la base de datos `rentals` en el servidor de Azure Database for PostgreSQL mediante la [utilidad de línea de comandos psql](https://www.postgresql.org/docs/current/app-psql.html) de [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. En [Azure Portal](https://portal.azure.com/), ve a la instanacia del servidor flexible de Azure Database for PostgreSQL recién creada.

2. En el menú de recursos, en **Configuración**, selecciona **Bases de datos** selecciona **Conectar** en la base de datos `rentals`.

    ![Captura de pantalla de la página de bases de datos de Azure Database for PostgreSQL. Bases de datos y Conectar la base de datos de alquileres están resaltados con cuadros rojos.](media/17-postgresql-rentals-database-connect.png)

3. En el símbolo del sistema "Contraseña para el usuario pgAdmin" de Cloud Shell, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin**.

    Una vez iniciada la sesión, se muestra la solicitud `psql` de la base de datos `rentals`.

## Rellenar la base de datos con datos de ejemplo

Para poder analizar la opinión de las revisiones de propiedades de alquiler mediante la extensión `azure_ai`, debes agregar datos de ejemplo a tu base de datos. Agrega una tabla a la base de datos `rentals` y rellénela con revisiones de clientes para que tengas datos sobre los que realizar el análisis de sentimiento.

1. Ejecuta el siguiente comando para crear una tabla denominada `reviews` para almacenar las revisiones de propiedades enviadas por los clientes:

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. A continuación, usa el comando `COPY` para rellenar la tabla con datos de un archivo CSV. Ejecuta el comando siguiente para cargar las revisiones de clientes en la tabla `reviews`:

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    La salida del comando debe ser `COPY 354`, que indica que se han escrito 354 filas en la tabla desde el archivo CSV.

## Instalación y configuración de la extensión `azure_ai`

Antes de usar la extensión `azure_ai`, debes instalarla en tu base de datos y configurarla para que se conecte a tus recursos de Servicios de Azure AI. La extensión `azure_ai` te permite integrar los servicios Azure OpenAI y Lenguaje de Azure AI en tu base de datos. Para habilitar la extensión en tu base de datos, sigue estos pasos:

1. Ejecuta el siguiente comando en la solicitud `psql` para comprobar que las extensiones `azure_ai` y `vector` se han agregado correctamente a la _lista de permitidos_ de tu servidor mediante el script de implementación de Bicep que has ejecutado al configurar tu entorno:

    ```sql
    SHOW azure.extensions;
    ```

    El comando muestra la lista de extensiones de la _lista de permitidos_ del servidor. Si todo se ha instalado correctamente, tu salida debe incluir `azure_ai` y `vector`, de la siguiente manera:

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Para poder instalar y usar una extensión en una base de datos de servidor flexible de Azure Database for PostgreSQL, se debe agregar a la _lista de permitidos_ del servidor, como se describe en [cómo usar extensiones de PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Ahora, estás listo para instalar la extensión `azure_ai` mediante el comando [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html).

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` carga una nueva extensión en la base de datos al ejecutar su archivo de script. Este script normalmente crea nuevos objetos SQL, como funciones, tipos de datos y esquemas. Se produce un error si ya existe una extensión con el mismo nombre. Agregar `IF NOT EXISTS` permite que el comando se ejecute sin producir un error si ya está instalado.

## Conectar tu cuenta de Servicios de Azure AI

Las integraciones de servicios de Azure AI incluidas en el esquema `azure_cognitive` de la extensión `azure_ai` proporcionan un amplio conjunto de características de Lenguaje de IA accesibles directamente desde la base de datos. Las funcionalidades de análisis de sentimiento se habilitan a través del [servicio de Lenguaje de Azure AI](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Para realizar llamadas correctamente a tus servicios de Lenguaje de Azure AI mediante la extensión `azure_ai`, debes proporcionar su punto de conexión la clave para la extensión. Con la misma pestaña del explorador donde está abierto Cloud Shell, ve al recurso de servicio de lenguaje en [Azure Portal](https://portal.azure.com/) y selecciona el elemento **Claves y punto de conexión** en **Administración de recursos** en el menú de navegación izquierdo.

    ![Captura de pantalla de la página Claves y puntos de conexión del servicio de lenguaje de Azure que se muestra, con los botones de copia de Punto de conexión y CLAVE 1 resaltados en cuadros rojos.](media/16-azure-language-service-keys-endpoints.png)

    > [!Note]
    >
    > Si recibiste el mensaje `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` al instalar la extensión `azure_ai` anterior y has configurado previamente la extensión con tu punto de conexión y clave del servicio de lenguaje, puedes usar la función `azure_ai.get_setting()` para confirmar que la configuración es correcta y, a continuación, omitir el paso 2 si es correcta.

2. Copia los valores de punto de conexión y clave de acceso y, a continuación, en los comandos siguientes, reemplaza los tokens `{endpoint}` y `{api-key}` por los valores que copiaste de Azure Portal. Ejecuta los comandos desde el símbolo del sistema `psql` en Cloud Shell para agregar los valores a la tabla `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Revisar las funcionalidades de análisis de opinión de la extensión

En esta tarea, usarás la función `azure_cognitive.analyze_sentiment()` para evaluar las revisiones de los listados de propiedades de alquiler.

1. Para el resto de este ejercicio, trabajarás exclusivamente en Cloud Shell, por lo que puede resultar útil expandir el panel dentro de la ventana del explorador mediante la selección del botón **Maximizar** situado en la parte superior derecha del panel de Cloud Shell.

    ![Captura de pantalla del panel Azure Cloud Shell con el botón Maximizar resaltado con un cuadro rojo.](media/16-azure-cloud-shell-pane-maximize.png)

2. Al trabajar con `psql` en Cloud Shell, habilitar la presentación extendida para los resultados de la consulta puede resultar útil, ya que mejora la legibilidad de la salida para los comandos posteriores. Ejecuta el siguiente comando para permitir que se aplique automáticamente la pantalla extendida.

    ```sql
    \x auto
    ```

3. Las funcionalidades de análisis de sentimiento de la extensión `azure_ai` se encuentran en el esquema `azure_cognitive`. Se usa la función `analyze_sentiment()`. Usa el [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) para examinar la función mediante la ejecución de:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    La salida del metacomando muestra el esquema, el nombre, el tipo de datos de resultado y los argumentos de la función. Esta información te ayuda a comprender cómo interactuar con la función desde las consultas.

    La salida muestra tres sobrecargas de la función `analyze_sentiment()`, lo que te permite revisar sus diferencias. La propiedad `Argument data types` de la salida revela la lista de argumentos que esperan las tres sobrecargas de la función:

    | Argument | Tipo | Valor predeterminado | Descripción |
    | -------- | ---- | ------- | ----------- |
    | text | `text` o `text[]` || Los textos para los que debe analizarse la opinión. |
    | language_text | `text` o `text[]` || Código de idioma (o matriz de códigos de idioma) que representa el idioma del texto que se va a analizar para la opinión. Revise la [lista de idiomas admitidos](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support) para recuperar los códigos de idioma necesarios. |
    | batch_size | `integer` | 10 | Solo para las dos sobrecargas que esperan una entrada de `text[]`. Especifica el número de registros que se van a procesar a la vez. |
    | disable_service_logs | `boolean` | false | Marca que indica si se van a desactivar los registros de servicio. |
    | timeout_ms | `integer` | NULL | Tiempo de espera en milisegundos después del cual se detiene la operación. |
    | throw_on_error | `boolean` | true | Marca que indica si la función debe (en caso de error) producir una excepción, lo que da lugar a una reversión de la transacción de ajuste. |
    | max_attempts | `integer` | 1 | Número de veces que se reintenta la llamada a Servicios de Azure AI en caso de error. |
    | retry_delay_ms | `integer` | 1000 | Cantidad de tiempo, en milisegundos, que se debe esperar antes de intentar volver a llamar al punto de conexión de Servicios de Azure AI. |

4. También es imperativo comprender la estructura del tipo de datos que devuelve una función para poder controlar correctamente la salida en las consultas. Ejecuta el comando siguiente para inspeccionar el tipo `sentiment_analysis_result`:

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. La salida del comando anterior revela que el `sentiment_analysis_result` tipo es `tuple`. Puedes profundizar más en la estructura de `tuple` con la ejecución del siguiente comando para examinar las columnas contenidas en el tipo `sentiment_analysis_result`:

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    La salida de dicho comando debe ser similar a esta:

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    El `azure_cognitive.sentiment_analysis_result` es un tipo compuesto que contiene las predicciones de opinión del texto de entrada. Incluye la opinión, que puede ser positiva, negativa, neutra o mixta, y las puntuaciones para aspectos positivos, neutros y negativos encontrados en el texto. Las puntuaciones se representan como números reales entre 0 y 1. Por ejemplo, en (neutral, 0.26, 0.64, 0.09), la opinión es neutra, con una puntuación positiva de 0,26, neutra de 0,64 y negativa de 0,09.

## Analizar la opinión de reseñas

1. Ahora que has revisado la función `analyze_sentiment()` y `sentiment_analysis_result` la devuelve, pongamos la función en uso. Ejecuta la siguiente consulta básica, que realiza el análisis de sentimiento en una serie de comentarios de la tabla `reviews`:

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    A partir de los dos registros analizados, anota los valores `sentiment` de la salida `(mixed,0.71,0.09,0.2)` y `(positive,0.99,0.01,0)`. Estos representan el `sentiment_analysis_result` devuelto por la función `analyze_sentiment()` en la consulta anterior. El análisis se realizó sobre el campo `comments` de la tabla `reviews`.

    > [!Note]
    >
    > El uso de la función `analyze_sentiment()` insertada te permite analizar rápidamente la opinión del texto dentro de tus consultas. Aunque esto funciona bien para un pequeño número de registros, puede que no sea ideal para analizar la opinión de un gran número de registros o actualizar todos los registros de una tabla que puede contener decenas de miles de revisiones o más.

2. Otro enfoque que puede ser útil para las revisiones más largas es analizar la opinión de cada frase dentro de ella. Para ello, usa la sobrecarga de la función `analyze_sentiment()`, que acepta una matriz de texto.

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    En la consulta anterior, usaste la función `STRING_TO_ARRAY` de PostgreSQL. Además, se usó la función `ARRAY_REMOVE` para quitar los elementos de matriz que son cadenas vacías, ya que provocarán errores con la función `analyze_sentiment()`.

    La salida de la consulta te permite comprender mejor la opinión `mixed` asignada a la revisión general. Las oraciones son una mezcla de opiniones positivas, neutras y negativas.

3. Las dos consultas anteriores devolvieron directamente el `sentiment_analysis_result` de la consulta. Sin embargo, es probable que quieras recuperar los valores subyacentes dentro del `sentiment_analysis_result``tuple`. Ejecuta la consulta siguiente que busca revisiones abrumadoramente positivas y extrae los componentes de opinión en campos individuales:

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    La consulta anterior usa una expresión de tabla común o CTE para obtener las puntuaciones de opinión de todos los registros de la tabla `reviews`. A continuación, selecciona las columnas de tipo compuesto `sentiment` del `sentiment_analysis_result` devuelto por el CTE para extraer los valores individuales de `tuple.`.

## Almacenar opiniones en la tabla de opiniones

Para el sistema de recomendaciones de propiedades de alquiler que estás creando para Margie's Travel, quieres almacenar clasificaciones de opinión en la base de datos para que no tengas que hacer llamadas e incurrir en costos cada vez que se solicitan evaluaciones de opinión. Realizar análisis de sentimiento sobre la marcha puede ser excelente para un número reducido de registros o analizar datos casi en tiempo real. Sin embargo, agregar los datos de opinión a la base de datos para su uso en tu aplicación tiene sentido para las revisiones almacenadas. Para ello, quieres modificar la tabla `reviews` para agregar columnas para almacenar la valoración de opinión y las puntuaciones positivas, neutras y negativas.

1. Ejecuta la consulta siguiente para actualizar la tabla `reviews` para que pueda almacenar los detalles de opinión:

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. A continuación, quieres actualizar los registros existentes de la tabla `reviews` con su valor de la opinión y las puntuaciones asociadas.

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    La ejecución de esta consulta tarda mucho tiempo porque los comentarios de cada revisión de la tabla se envían individualmente al punto de conexión del servicio de lenguaje para su análisis. El envío de registros en lotes es más eficaz cuando se trabaja con muchos registros.

3. Vamos a ejecutar la consulta siguiente para realizar la misma acción de actualización, pero esta vez enviar comentarios de la tabla `reviews` en lotes de 10 (este es el tamaño máximo permitido del lote) y evaluar la diferencia en el rendimiento.

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    Aunque esta consulta es un poco más compleja, el uso de dos CTE es mucho más eficaz. En esta consulta, el primer CTE analiza la opinión de los lotes de comentarios de revisión y el segundo extrae la tabla resultante de `sentiment_analysis_results` en una nueva tabla que contiene un `id` basado en la posición ordinal y "sentiment_analysis_result" para cada fila. A continuación, el segundo CTE se puede usar en la instrucción de actualización para escribir los valores en la base de datos.

4. Después, ejecuta una consulta para observar los resultados de la actualización, buscando revisiones con una opinión **negativa**, empezando por la más negativa primero.

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## Limpiar

Una vez completado este ejercicio, elimina los recursos de Azure que has creado. Se te cobra por la capacidad configurada, no por cuánto se use la base de datos. Sigue estas instrucciones para eliminar el grupo de recursos y todos los recursos que has creado para este laboratorio.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/) y, en la página de inicio, selecciona **Grupos de recursos** en Servicios de Azure.

    ![Captura de pantalla de los grupos de recursos resaltados con un cuadro rojo en Servicios de Azure en Azure Portal.](media/16-azure-portal-home-azure-services-resource-groups.png)

2. En el filtro de cualquier campo de búsqueda, escribe el nombre del grupo de recursos que creaste para este laboratorio y, a continuación, selecciona el grupo de recursos de la lista.

3. En la página **Información general** del grupo de recursos, seleccione **Eliminar grupo de recursos**.

    ![Captura de pantalla de la hoja Información general del grupo de recursos con el botón Eliminar grupo de recursos resaltado con un cuadro rojo.](media/16-resource-group-delete.png)

4. En el cuadro de diálogo de confirmación, escribe el nombre del grupo de recursos que vas a eliminar para confirmar y después selecciona **Eliminar**.
