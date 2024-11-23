---
lab:
  title: Migración de base de datos de PostgreSQL en línea
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## Migración de base de datos de PostgreSQL en línea

En este ejercicio configurarás la replicación lógica entre un servidor PostgreSQL de origen y un servidor flexible de Azure Database for PostgreSQL para permitir que se realice una actividad de migración en línea.

## Antes de comenzar

Debe tener una suscripción a Azure propia para completar este ejercicio. Si no tiene una suscripción a Azure, puede obtener una [evaluación gratuita de Azure](https://azure.microsoft.com/free).

> **Nota:** este ejercicio requerirá que el servidor que uses como origen para la migración sea accesible para el servidor flexible de Azure Database for PostgreSQL para que pueda conectarse y migrar bases de datos. Esto requerirá que el servidor de origen sea accesible a través de una dirección IP pública y un puerto. > Se puede descargar una lista de direcciones IP de la región de Azure desde [Intervalos IP y etiquetas de servicio de Azure: nube pública](https://www.microsoft.com/en-gb/download/details.aspx?id=56519) para ayudar a minimizar los intervalos permitidos de direcciones IP en tus reglas de firewall en función de la región de Azure usada.

Abre el firewall de tu servidor para permitir que la característica de migración del servidor de flexible de Azure Database for PostgreSQL acceda al servidor PostgreSQL de origen que, de manera predeterminada, es el puerto TCP 5432.
>
Cuando se usa un dispositivo de firewall frente a la base de datos de origen, puede que tengas que agregar reglas de firewall para permitir que la característica de migración del servidor flexible de Azure Database for PostgreSQL acceda a las bases de datos de origen para realizar la migración.
>
> La versión máxima admitida de PostgreSQL para la migración es la versión 16.

### Requisitos previos

> **Nota**: antes de iniciar este ejercicio, deberás haber completado el ejercicio anterior para que las bases de datos de origen y de destino se hayan implementado para configurar la replicación lógica, ya que este ejercicio se basa en la actividad de ese ejercicio.

## Crear publicación: servidor de origen

1. Abre PGAdmin y conéctate al servidor de origen que contiene la base de datos que va a actuar como origen para la sincronización de datos con el servidor flexible de Azure Database for PostgreSQL.
1. Abre una nueva ventana Consulta conectada a la base de datos de origen con los datos que queremos sincronizar.
1. Configura el servidor de origen wal_level en **lógico** para permitir la publicación de datos.
    1. Busca y abre el archivo **postgresql.conf** en el directorio bin dentro del directorio de instalación de PostgreSQL.
    1. Busca la línea con el valor de configuración **wal_level**.
    1. Asegúrate de que la línea no esté comentada y establece el valor en **lógico**.
    1. Guarde y cierre el archivo.
    1. Reinicia el servicio PostgreSQL.
1. Ahora configura una publicación que contendrá todas las tablas de la base de datos.

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## Crear suscripción: servidor de destino

1. Abre PGAdmin y conéctate al servidor flexible de Azure Database for PostgreSQL que contiene la base de datos que va a actuar como destino para la sincronización de datos desde el servidor de origen.
1. Abre una nueva ventana Consulta conectada a la base de datos de origen con los datos que queremos sincronizar.
1. Crea la suscripción al servidor de origen.

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. Comprueba el estado de replicación de la tabla.

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## Prueba de la replicación de datos

1. En el servidor de origen, comprueba el recuento de filas de la tabla workorder.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. En el servidor de destino, comprueba el recuento de filas de la tabla workorder.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. Comprueba que los valores de recuento de filas coinciden.
1. Ahora descarga el archivo Lab11_workorder.csv desde el repositorio [aquí](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11) en C:\
1. Carga nuevos datos en la tabla workorder en el servidor de origen desde el CSV mediante el comando siguiente.

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

La salida del comando debe ser `COPY 490`, que indica que se han escrito 490 filas adicionales en la tabla desde el archivo CSV.

1. Comprueba que los recuentos de filas de la tabla workorder en el origen (72591) y destino coinciden para comprobar que la replicación de datos funciona.

## Limpieza del ejercicio

La instancia de Azure Database for PostgreSQL que hemos implementado en este ejercicio incurrirá en cargos que puedes eliminar del servidor después de este ejercicio. Como alternativa, puedes eliminar el grupo de recursos **rg-learn-work-with-postgresql-eastus** para quitar todos los recursos implementados como parte de este ejercicio.
