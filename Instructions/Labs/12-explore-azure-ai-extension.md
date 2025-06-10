---
lab:
  title: Exploración de la extensión de Azure AI
  module: Explore Generative AI with Azure Database for PostgreSQL
---

# Exploración de la extensión de Azure AI

Como desarrollador principal de Margie's Travel, se te ha encargado la creación de una aplicación con tecnología de IA para proporcionar a tus clientes recomendaciones inteligentes sobre las propiedades de alquiler. Quieres obtener más información sobre la extensión `azure_ai` para Azure Database for PostgreSQL y cómo puede ayudarte a integrar la potencia de la IA generativa (GenAI) en la aplicación. En este ejercicio, instalará la extensión `azure_ai` en un servidor flexible de Azure Database for PostgreSQL y explorarás sus funcionalidades para integrar los servicios de ML y Azure AI.

## Antes de comenzar

Necesitarás una [suscripción a Azure](https://azure.microsoft.com/free) en la que tengas derechos administrativos

### Implementación de recursos en tu suscripción a Azure

Este paso te guiará por el uso de los comandos de la CLI de Azure desde Azure Cloud Shell para crear un grupo de recursos y ejecutar un script de Bicep para implementar los servicios de Azure necesarios para completar este ejercicio en la suscripción a Azure.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/).

2. Selecciona el icono de **Cloud Shell** en la barra de herramientas de Azure Portal para abrir un nuevo panel de [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en la parte inferior de la ventana del explorador.

    ![Captura de pantalla de la barra de herramientas de Azure Portal, con el icono de Cloud Shell resaltado por un cuadro rojo.](media/12-portal-toolbar-cloud-shell.png)

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

    El comando final genera aleatoriamente una contraseña para el inicio de sesión de administrador de PostgreSQL. Cópialo en un lugar seguro para usarlo más adelante al conectarte al servidor flexible de PostgreSQL.

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

    El script de implementación de Bicep aprovisiona los servicios de Azure necesarios para completar este ejercicio en tu grupo de recursos. Los recursos implementados incluyen un servidor flexible de Azure Database for PostgreSQL, Azure OpenAI y un servicio de Lenguaje de Azure AI. El script de Bicep también realiza algunos pasos de configuración, como agregar las extensiones `azure_ai` y `vector` a la _lista de permitidos_ del servidor PostgreSQL (a través del parámetro de servidor azure.extensions), crear una base de datos denominada `rentals` en el servidor y agregar una implementación denominada `embedding` con el modelo `text-embedding-ada-002` al Azure OpenAI Service. Ten en cuenta que todos los módulos de esta ruta de aprendizaje comparten el archivo Bicep, por lo que solo podrás usar algunos de los recursos implementados en algunos ejercicios.

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

En esta tarea, te conectarás a la base de datos `rentals` en el servidor flexible de Azure Database for PostgreSQL mediante la [utilidad de línea de comandos psql](https://www.postgresql.org/docs/current/app-psql.html) de [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. En [Azure Portal](https://portal.azure.com/), ve al servidor flexible de Azure Database for PostgreSQL recién creado.

1. En el menú de recursos, en **Configuración**, selecciona **Bases de datos** selecciona **Conectar** para la base de datos `rentals`. Ten en cuenta que al seleccionar **Conectar** no se conecta realmente a la base de datos; simplemente se proporcionan instrucciones para conectarse a la base de datos mediante varios métodos. Revisa las instrucciones para **conectarte desde el explorador o localmente** y úsalas para conectarte mediante Azure Cloud Shell.

    ![Captura de pantalla de la página Base de datos de Azure Database for PostgreSQL. Bases de datos y Conectar la base de datos de alquileres están resaltadas por cuadros rojos.](media/12-postgresql-rentals-database-connect.png)

1. En el símbolo del sistema "Contraseña para el usuario pgAdmin" de Cloud Shell, escribe la contraseña generada aleatoriamente para el inicio de sesión **pgAdmin**.

    Una vez que hayas iniciado sesión, se muestra la solicitud `psql` de la base de datos `rentals`.

1. Durante el resto de este ejercicio, seguirás trabajando en Cloud Shell, por lo que puede resultar útil expandir el panel dentro de la ventana del explorador seleccionando el botón **Maximizar** en la parte superior derecha del panel.

    ![Captura de pantalla del panel de Azure Cloud Shell con el botón Maximizar resaltado por un cuadro rojo.](media/12-azure-cloud-shell-pane-maximize.png)

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

## Instalación y configuración de la extensión `azure_ai`

Antes de usar la extensión `azure_ai`, deberás instalarla en la base de datos y configurarla para conectarse a los recursos de Servicios de Azure AI. La extensión `azure_ai` permite integrar Azure OpenAI y los servicios de Lenguaje de Azure AI en la base de datos. Para habilitar la extensión en la base de datos, sigue estos pasos:

1. Ejecuta el siguiente comando en el símbolo del sistema `psql` para comprobar que las extensiones `azure_ai` y `vector` se agregaron correctamente a la _lista de permitidos_ del servidor mediante el script de implementación de Bicep que ejecutaste al configurar el entorno:

    ```sql
    SHOW azure.extensions;
    ```

    El comando muestra la lista de extensiones en la _lista de permitidos_ del servidor. Si todo se instaló correctamente, la salida deberá incluir `azure_ai` y `vector`, de la siguiente manera:

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Para poder instalar y usar una extensión en una base de datos de servidor flexible de Azure Database for PostgreSQL, se debe agregar a la _lista de permitidos_ del servidor, como se describe en [cómo usar extensiones de PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Ahora estás preparado para instalar la extensión `azure_ai` mediante el comando [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html).

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` carga una nueva extensión en la base de datos ejecutando su archivo de script. Este script normalmente crea nuevos objetos SQL, como funciones, tipos de datos y esquemas. Se produce un error si ya existe una extensión del mismo nombre. La adición de `IF NOT EXISTS` permite que el comando se ejecute sin producir un error si ya está instalado.

## Revisión de los objetos que contiene la extensión `azure_ai`

La revisión de los objetos dentro de la extensión `azure_ai` puede ayudarte a comprender mejor sus funcionalidades. En esta tarea, inspeccionarás los distintos esquemas, funciones definidas por el usuario (UDF) y tipos compuestos agregados a la base de datos por la extensión.

1. Cuando trabajas con `psql` en Cloud Shell, puede resultar útil habilitar la pantalla extendida para los resultados de la consulta, ya que mejora la legibilidad de la salida para los comandos posteriores. Ejecuta el siguiente comando para permitir que se aplique automáticamente la pantalla extendida.

    ```sql
    \x auto
    ```

2. El [metacomando `\dx`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DX-LC) se usa para enumerar los objetos que se encuentran dentro de una extensión. Ejecuta lo siguiente desde el símbolo del sistema `psql` para ver los objetos de la extensión `azure_ai`. Es posible que tengas que presionar la barra espaciadora para ver la lista completa de objetos.

    ```psql
    \dx+ azure_ai
    ```

    La salida del metacomando muestra que la extensión `azure_ai` crea cuatro esquemas, varias funciones definidas por el usuario (UDF), varios tipos compuestos en la base de datos y la tabla `azure_ai.settings`. Aparte de los esquemas, todos los nombres de objeto van precedidos por el esquema al que pertenecen. Los esquemas se usan para agrupar funciones relacionadas y tipos que la extensión agrega a cubos. En la tabla siguiente se enumeran los esquemas agregados por la extensión y se proporciona una breve descripción de cada uno:

    | Esquema      | Descripción                                              |
    | ----------------- | ------------------------------------------------------------------------------------------------------ |
    | `azure_ai`    | Esquema principal donde residen la tabla de configuración y las UDF para interactuar con la extensión. |
    | `azure_openai`  | Contiene las UDF que habilitan la llamada a un punto de conexión de Azure OpenAI.                    |
    | `azure_cognitive` | Proporciona UDF y tipos compuestos relacionados con la integración de la base de datos con Servicios de Azure AI.     |
    | `azure_ml`    | Incluye las UDF para integrar los servicios de Azure Machine Learning (ML).                |

### Exploración del esquema de Azure AI

El esquema `azure_ai` proporciona el marco para interactuar directamente con los servicios de Azure AI y ML desde la base de datos. Contiene funciones para configurar conexiones a esos servicios y recuperarlas de la tabla `settings`, que también se hospeda en el mismo esquema. La tabla `settings` proporciona almacenamiento seguro en la base de datos para los puntos de conexión y las claves asociados a los servicios de Azure AI y ML.

1. Para revisar las funciones definidas en el esquema, puedes usar el [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC), especificando el esquema cuyas funciones deben mostrarse. Ejecuta lo siguiente para ver las funciones en el esquema `azure_ai`:

    ```sql
    \df azure_ai.*
    ```

    El resultado de este comando debe ser una tabla similar a la siguiente:

    ```sql
                  List of functions
     Schema |  Name  | Result data type | Argument data types | Type 
    ----------+-------------+------------------+----------------------+------
     azure_ai | get_setting | text      | key text      | func
     azure_ai | set_setting | void      | key text, value text | func
     azure_ai | version  | text      |           | func
    ```

    La función `set_setting()` te permite establecer el punto de conexión y la clave de los servicios de Azure AI y ML para que la extensión pueda conectarse a ellos. Acepta una **clave** y el **valor** para asignarla. La función `azure_ai.get_setting()` proporciona una manera de recuperar los valores establecidos con la función `set_setting()`. Acepta la **clave** de la configuración que deseas ver y devuelve el valor asignado. Para ambos métodos, la clave debe ser una de las siguientes:

    | Clave | Descripción |
    | --- | ----------- |
    | `azure_openai.endpoint` | Un punto de conexión de OpenAI compatible (por ejemplo, <https://example.openai.azure.com>). |
    | `azure_openai.subscription_key` | Una clave de suscripción para un recurso de Azure OpenAI. |
    | `azure_cognitive.endpoint` | Un punto de conexión Servicios de Azure AI compatible (por ejemplo, <https://example.cognitiveservices.azure.com>). |
    | `azure_cognitive.subscription_key` | Una clave de suscripción para un recurso de Servicios de Azure AI. |
    | `azure_ml.scoring_endpoint` | Un punto de conexión de puntuación de Azure ML compatible (por ejemplo, <https://example.eastus2.inference.ml.azure.com/score>) |
    | `azure_ml.endpoint_key` | Una clave de punto de conexión para una implementación de Azure ML. |

    > Importante
    >
    > Dado que la información de conexión de los Servicios de Azure AI, incluidas las claves de API, se almacena en una tabla de configuración de la base de datos, la extensión `azure_ai` define un rol denominado `azure_ai_settings_manager` para asegurar que esta información está protegida y es accesible solo para los usuarios asignados a ese rol. Este rol permite leer y escribir la configuración relacionada con la extensión. Solo los miembros del rol `azure_ai_settings_manager` pueden invocar las funciones `azure_ai.get_setting()` y `azure_ai.set_setting()`. En un servidor flexible de Azure Database for PostgreSQL, a todos los usuarios administradores (aquellos con el rol `azure_pg_admin` asignado) se les asigna el rol `azure_ai_settings_manager`.

2. Para demostrar cómo se usan las funciones `azure_ai.set_setting()` y `azure_ai.get_setting()`, configura la conexión a la cuenta de Azure OpenAI. Con la misma pestaña del explorador donde está abierto Cloud Shell, minimiza o restaura el panel de Cloud Shell y ve al recurso de Azure OpenAI en [Azure Portal](https://portal.azure.com/). Una vez que estés en la página de recursos de Azure OpenAI, en el menú de recursos, en la sección **Administración de recursos**, selecciona **Claves y punto de conexión** y, después, copia el punto de conexión y una de las claves disponibles.

    ![Captura de pantalla donde se muestra la página Claves y puntos de conexión de Azure OpenAI Service, con los botones CLAVE 1 y Copia de punto de conexión resaltados por cuadros rojos.](media/12-azure-openai-keys-and-endpoints.png)

    Puedes usar `KEY 1` o `KEY 2`. Tener siempre dos claves permite rotar y regenerar las claves de forma segura sin provocar una interrupción del servicio.

3. Una vez que tengas el punto de conexión y la clave, vuelve a maximizar el panel de Cloud Shell y usa los comandos siguientes para agregar los valores a la tabla de configuración. Asegúrate de reemplazar los tokens `{endpoint}` y `{api-key}` por los valores que copiaste de Azure Portal.

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '{api-key}');
    ```

4. Puedes comprobar la configuración escrita en la tabla `azure_ai.settings` mediante la función `azure_ai.get_setting()` en las siguientes consultas:

    ```sql
    SELECT azure_ai.get_setting('azure_openai.endpoint');
    SELECT azure_ai.get_setting('azure_openai.subscription_key');
    ```

    La extensión `azure_ai` está ahora conectada a la cuenta de Azure OpenAI.

### Revisión del esquema de Azure OpenAI

El esquema `azure_openai` proporciona la capacidad de integrar la creación de la incrustación de vectores de valores de texto en la base de datos mediante Azure OpenAI. Con este esquema, puedes [generar incrustaciones con Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/how-to/embeddings) directamente desde la base de datos para crear representaciones vectoriales del texto de entrada, que luego se pueden usar en búsquedas de similitud vectorial y que los modelos de aprendizaje automático pueden consumir. El esquema contiene una sola función, `create_embeddings()`, con dos sobrecargas. Una sobrecarga acepta una sola cadena de entrada y la otra espera una matriz de cadenas de entrada.

1. Como hiciste anteriormente, puedes usar el [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) para ver los detalles de las funciones en el esquema `azure_openai`:

    ```sql
    \df azure_openai.*
    ```

    La salida muestra las dos sobrecargas de la función `azure_openai.create_embeddings()`, lo que te permite revisar las diferencias entre las dos versiones de la función y los tipos que devuelven. La propiedad `Argument data types` de la salida revela la lista de argumentos que esperan las dos sobrecargas de la función:

    | Argumento    | Tipo       | Valor predeterminado | Descripción                                                          |
    | --------------- | ------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
    | deployment_name | `text`      |    | Nombre de la implementación en Azure OpenAI Studio que contiene el modelo `text-embedding-ada-002`.               |
    | input     | `text` o `text[]` |    | Texto de entrada (o matriz de texto) para el que se crean las incrustaciones.                                |
    | batch_size   | `integer`     | 100  | Solo para la sobrecarga que espera una entrada de `text[]`. Especifica el número de registros que se van a procesar a la vez.          |
    | timeout_ms   | `integer`     | 3600000 | Tiempo de espera en milisegundos después del cual se detiene la operación.                                 |
    | throw_on_error | `boolean`     | true  | Marca que indica si la función debe (en caso de error) producir una excepción, lo que da lugar a una reversión de la transacción de ajuste. |
    | max_attempts  | `integer`     | 1   | Número de veces que se reintenta la llamada a Azure OpenAI Service en caso de error.                     |
    | retry_delay_ms | `integer`     | 1 000  | Cantidad de tiempo, en milisegundos, que se debe esperar antes de intentar volver a llamar al punto de conexión de Azure OpenAI Service.        |

2. Para proporcionar un ejemplo simplificado de uso de la función, ejecuta la consulta siguiente, que crea una incrustación vectorial para el campo `description` de la tabla `listings`. El parámetro `deployment_name` de la función se establece en `embedding`, que es el nombre de la implementación del modelo `text-embedding-ada-002` en Azure OpenAI Service (el script de implementación de Bicep la creó con ese nombre):

    ```sql
    SELECT
        id,
        name,
        azure_openai.create_embeddings('embedding', description) AS vector
    FROM listings
    LIMIT 1;
    ```

    La salida es similar a esta:

    ```sql
     id |      name       |              vector
    ----+-------------------------------+------------------------------------------------------------
      1 | Stylish One-Bedroom Apartment | {0.020068742,0.00022734122,0.0018286322,-0.0064167166,...}
    ```

    Por motivos de brevedad, las incrustaciones vectoriales se abreviaron en la salida anterior.

    Las [incrustaciones](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#embeddings) son un concepto de aprendizaje automático y procesamiento del lenguaje natural (NLP) que implica representar objetos, como palabras, documentos o entidades, como [vectores](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#vectors) en un espacio multidimensional. Las incrustaciones permiten a los modelos de aprendizaje automático evaluar la estrecha relación entre dos piezas de información. Esta técnica identifica de manera eficaz las relaciones y similitudes entre los datos, lo que permite a los algoritmos identificar patrones y realizar predicciones precisas.

    La extensión `azure_ai` permite generar inserciones para texto de entrada. Para permitir que los vectores generados se almacenen junto con el resto de los datos de la base de datos, debes instalar la extensión `vector` siguiendo las instrucciones de la documentación [habilitación de la compatibilidad con vectores en la base de datos](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension). Sin embargo, esto está fuera del ámbito de este ejercicio.

### Examen del esquema de azure_cognitive

El esquema `azure_cognitive` proporciona el marco para interactuar directamente con Servicios de Azure AI desde la base de datos. Las integraciones de Servicios de Azure AI incluidas en el esquema proporcionan un amplio conjunto de características de Lenguaje de IA accesibles directamente desde la base de datos. Las funcionalidades incluyen análisis de sentimiento, detección de idioma y traducción, extracción de frases clave, reconocimiento de entidades y resumen de texto. Estas funcionalidades están habilitadas a través del [servicio de Lenguaje de Azure AI](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Para revisar todas las funciones definidas en un esquema, puedes usar el [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) tal y como has hecho anteriormente. Para ver las funciones en el esquema `azure_cognitive`, ejecuta:

    ```sql
    \df azure_cognitive.*
    ```

2. Hay numerosas funciones definidas en este esquema, por lo que la salida del [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) puede ser difícil de leer, por lo que es mejor dividirla en fragmentos más pequeños. Ejecuta lo siguiente para examinar solo la función `analyze_sentiment()`:

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    En la salida, observa que la función tiene tres sobrecargas, con una aceptación de una sola cadena de entrada y las otras dos que esperan matrices de texto. La salida muestra el esquema, el nombre, el tipo de datos de resultado y los tipos de datos de argumento de la función. Esta información puede ayudarte a comprender cómo usar la función.

3. Repite el comando anterior, reemplazando el nombre de la función `analyze_sentiment` por cada uno de los siguientes nombres de función, para inspeccionar todas las funciones disponibles en el esquema:

   - `detect_language`
   - `extract_key_phrases`
   - `linked_entities`
   - `recognize_entities`
   - `recognize_pii_entities`
   - `summarize_abstractive`
   - `summarize_extractive`
   - `translate`

    Para cada función, inspecciona las distintas formas de la función y sus entradas esperadas y los tipos de datos resultantes.

4. Además de las funciones, el esquema `azure_cognitive` también contiene varios tipos compuestos usados como tipos de datos devueltos de las distintas funciones. Es imperativo comprender la estructura del tipo de datos que devuelve una función para poder controlar correctamente la salida en las consultas. Por ejemplo, ejecuta el comando siguiente para inspeccionar el tipo `sentiment_analysis_result`:

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. La salida del comando anterior revela que el tipo `sentiment_analysis_result` es `tuple`. Puedes profundizar más en la estructura de ese `tuple` ejecutando el siguiente comando para examinar las columnas que contiene el tipo `sentiment_analysis_result`:

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    La salida del comando debe ser similar a la siguiente:

    ```sql
             Composite type "azure_cognitive.sentiment_analysis_result"
       Column  |   Type   | Collation | Nullable | Default | Storage | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment   | text      |     |     |    | extended | 
     positive_score | double precision |     |     |    | plain  | 
     neutral_score | double precision |     |     |    | plain  | 
     negative_score | double precision |     |     |    | plain  |
    ```

    `azure_cognitive.sentiment_analysis_result` es un tipo compuesto que contiene las predicciones de opinión del texto de entrada. Incluye la opinión, que puede ser positiva, negativa, neutra o mixta, y las puntuaciones para aspectos positivos, neutros y negativos encontrados en el texto. Las puntuaciones se representan como números reales entre 0 y 1. Por ejemplo, en (neutral, 0.26, 0.64, 0.09), la opinión es neutra, con una puntuación positiva de 0,26, neutra de 0,64 y negativa de 0,09.

6. Al igual que con las funciones `azure_openai`, para realizar llamadas correctamente en los Servicios de Azure AI mediante la extensión `azure_ai`, deberás proporcionar el punto de conexión y una clave para el servicio de Lenguaje de Azure AI. Con la misma pestaña del explorador donde Cloud Shell está abierto, minimiza o restaura el panel de Cloud Shell y, después, ve al recurso de servicio de lenguaje en [Azure Portal](https://portal.azure.com/). En el menú del recurso, en la sección **Administración de recursos**, selecciona **Claves y puntos de conexión**.

    ![Captura de pantalla donde se muestra la página Claves y puntos de conexión del servicio de Lenguaje de Azure, con los botones CLAVE 1 y Copia de punto de conexión resaltados por cuadros rojos.](media/12-azure-language-service-keys-and-endpoints.png)

7. Copia los valores de punto de conexión y clave de acceso y reemplaza los tokens `{endpoint}` y `{api-key}` por los valores que copiaste de Azure Portal. Vuelve a maximizar Cloud Shell y ejecuta los comandos desde el símbolo del sistema `psql` en Cloud Shell para agregar los valores a la tabla de configuración.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

8. Ahora, ejecuta la consulta siguiente para analizar la opinión de un par de reseñas:

    ```sql
    SELECT
        id,
        comments,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id IN (1, 3);
    ```

    Observa los valores `sentiment` de la salida, `(mixed,0.71,0.09,0.2)` y `(positive,0.99,0.01,0)`. Estos representan `sentiment_analysis_result` que devolvió la función `analyze_sentiment()` en la consulta anterior. El análisis se realizó sobre el campo `comments` de la tabla `reviews`.

## Inspección del esquema de Azure ML

El esquema `azure_ml` permite que las funciones se conecten a los servicios de Azure ML directamente desde la base de datos.

1. Para revisar las funciones definidas en un esquema, puedes usar el [metacomando `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC). Para ver las funciones en el esquema `azure_ml`, ejecuta:

    ```sql
    \df azure_ml.*
    ```

    En la salida, observa que hay dos funciones definidas en este esquema, `azure_ml.inference()` y `azure_ml.invoke()`, los detalles de los cuales se muestran a continuación:

    ```sql
                  List of functions
    -----------------------------------------------------------------------------------------------------------
    Schema       | azure_ml
    Name        | inference
    Result data type  | jsonb
    Argument data types | input_data jsonb, deployment_name text DEFAULT NULL::text, timeout_ms integer DEFAULT NULL::integer, throw_on_error boolean DEFAULT true, max_attempts integer DEFAULT 1, retry_delay_ms integer DEFAULT 1000
    Type        | func
    ```

    La función `inference()` usa un modelo de aprendizaje automático entrenado para realizar predicciones o generar salidas basadas en datos nuevos no vistos.

    Al proporcionar un punto de conexión y una clave, puedes conectarte a un punto de conexión implementado de Azure ML como si te hubieras conectado a los puntos de conexión de Azure OpenAI y Servicios de Azure AI. La interacción con Azure ML requiere tener un modelo entrenado e implementado, por lo que está fuera del ámbito de este ejercicio y no estás configurando esa conexión para probarla aquí.

## Limpieza

Una vez completado este ejercicio, elimina los recursos de Azure que has creado. Se te cobrará por la capacidad configurada y no por cuanto se use la base de datos. Sigue estas instrucciones para eliminar el grupo de recursos y todos los recursos que creaste para este laboratorio.

1. Abre un explorador web y ve a [Azure Portal](https://portal.azure.com/) y, en la página principal, selecciona **Grupos de recursos** en servicios de Azure.

    ![Captura de pantalla de los grupos de recursos resaltados por un cuadro rojo en servicios de Azure en Azure Portal.](media/12-azure-portal-home-azure-services-resource-groups.png)

2. En el filtro de cualquier campo de búsqueda, escribe el nombre del grupo de recursos que creaste para este laboratorio y, después, selecciona el grupo de recursos de la lista.

3. En la página **Información general** del grupo de recursos, selecciona **Eliminar grupo de recursos**.

    ![Captura de pantalla de la hoja Información general del grupo de recursos con el botón Eliminar grupo de recursos resaltado por un cuadro rojo.](media/12-resource-group-delete.png)

4. En el cuadro de diálogo de confirmación, escribe el nombre del grupo de recursos que vas a eliminar y, después, selecciona **Eliminar**.
