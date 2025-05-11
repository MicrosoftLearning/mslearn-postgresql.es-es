---
lab:
  title: Configuración de permisos en Azure Database for PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Configuración de permisos en Azure Database for PostgreSQL

En estos ejercicios de laboratorio, asignará roles de RBAC para controlar el acceso a los recursos de Azure Database for PostgreSQL y concesiones de PostgreSQL para controlar el acceso a las operaciones de base de datos.

## Antes de comenzar

Debe tener una suscripción a Azure propia para completar este ejercicio. Si no tiene una suscripción a Azure, cree una [evaluación gratuita de Azure](https://azure.microsoft.com/free).

Para completar estos ejercicios, debes instalar un servidor PostgreSQL que esté conectado a Microsoft Entra ID (anteriormente Azure Active Directory).

### Creación de un grupo de recursos

1. En un explorador web, vaya a [Azure Portal](https://portal.azure.com). Inicie sesión con una cuenta de propietario o colaborador.
2. En Servicios de Azure, seleccione **Grupos de recursos** y, a continuación, seleccione **+ Crear**.
3. Comprueba que se muestra la suscripción correcta y, a continuación, especifica el nombre del grupo de recursos como **rg-PostgreSQL_Entra**. Seleccione una **región**.
4. Selecciona **Revisar + crear.** Seleccione **Crear**.

## Creación de servidor flexible de Azure Database for PostgreSQL

1. En Servicios de Azure, seleccione **+ Crear un recurso**.
    1. En **Buscar en Marketplace**, escribe **`azure database for postgresql flexible server`**, elige **Servidor flexible de Azure Database for PostgreSQL** y haz clic en **Crear**.
1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Suscripción: su suscripción.
    1. Grupo de recursos: **rg-PostgreSQL_Entra**.
    1. Nombre del servidor: **psql-postgresql-fx7777** (el nombre del servidor debe ser único globalmente, por lo que reemplaza 7777 por números aleatorios).
    1. Región: seleccione la misma región que el grupo de recursos.
    1. Versión de PostgreSQL: selecciona 16.
    1. Tipo de carga de trabajo: **Desarrollo**.
    1. Proceso y almacenamiento: **Ampliable, B1ms**.
    1. Zona de disponibilidad: sin preferencia.
    1. Alta disponibilidad: déjelo desactivado.
    1. Método de autenticación: selecciona **Autenticación de PostgreSQL y Microsoft Entra**
    1. En administrador de Microsoft Entra, selecciona **Establecer administrador**
        1. Busca tu cuenta en **Seleccionar administradores de Microsoft Entra** y (o) tu cuenta y haz clic en **Seleccionar**
    1. En **nombre de usuario de administrador**, escribe **`demo`**.
    1. En **contraseña**, escribe una contraseña adecuadamente compleja.
    1. Seleccione **Siguiente: Redes >**.
1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Método de conectividad: (o) Acceso público (direcciones IP permitidas) y Punto de conexión privado
    1. Acceso público, selecciona **Permitir el acceso público a este recurso a través de Internet mediante una dirección IP pública**
    1. En Reglas de firewall, seleccione **+ Agregar dirección IP del cliente actual** para agregar la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall. También, agrega **Agregar 0.0.0.0 - 255.255.255.255** y haz clic en **Continuar**
1. Selecciona **Revisar + crear.** Revisa la configuración, y, después, selecciona **Crear** para crear el servidor flexible de Azure Database for PostgreSQL. Una vez completada la implementación, seleccione **Ir al recurso** para ir el paso siguiente.

## Instalación de Azure Data Studio

Para instalar Azure Data Studio para su uso con Azure Database for PostgreSQL:

1. En un explorador, vaya a [Download and install Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio) (Descargar e instalar Azure Data Studio) y, en la plataforma de Windows, seleccione **User installer (recommended)** (Instalador de usuario [recomendado]). El archivo ejecutable se descargará en la carpeta Descargas.
1. Seleccione **Abrir archivo**.
1. Se muestra el contrato de licencia. Lea y **acepte el contrato** y, después, seleccione **Siguiente**.
1. En **Seleccionar tareas adicionales**, seleccione **Agregar a PATH** y cualquier otra adición que necesite. Seleccione **Siguiente**.
1. Se muestra el cuadro de diálogo **Listo para instalar**. Revise la configuración. Seleccione **Atrás** para realizar cambios o **Instalar**.
1. Se muestra el cuadro de diálogo **Completing the Azure Data Studio Setup Wizard** (Finalización del Asistente para instalar Azure Data Studio). Seleccione **Finalizar**. Azure Data Studio se inicia.

### Instalación de la extensión de PostgreSQL

1. Abra Azure Data Studio si aún no lo ha hecho.
1. En el menú izquierdo, seleccione **Extensiones** para mostrar el panel Extensiones.
1. En la barra de búsqueda, escriba **PostgreSQL**. Se muestra el icono de la extensión de PostgreSQL para Azure Data Studio.
1. Seleccione **Instalar**. La extensión se instala.

### Conexión con el servidor flexible de Azure Database for PostrgreSQL

1. Abra Azure Data Studio si aún no lo ha hecho.
1. En el menú de la izquierda, seleccione **Conexiones**.
1. Seleccione **Nueva conexión**.
1. En **Detalles de conexión**, en **Tipo de conexión**, seleccione **PostgreSQL** en la lista desplegable.
1. En **Nombre del servidor**, escriba el nombre completo del servidor tal y como aparece en Azure Portal.
1. En **Tipo de autenticación**, deje Contraseña.
1. En Nombre de usuario y Contraseña, escribe la **demostración** del nombre de usuario y la contraseña compleja que creaste anteriormente.
1. Seleccione [ x ] Recordar contraseña.
1. Los demás campos son opcionales.
1. Seleccione **Conectar**. Se ha conectado con el servidor de Azure Database for PostgreSQL.
1. Se muestra una lista de las bases de datos de servidor. Esto incluye las bases de datos del sistema y del usuario.

### Creación de la base de datos ZooDb

1. Ve a la carpeta con los archivos de script del ejercicio o descarga **Lab2_ZooDb.sql** de los [Laboratorios de PostgreSQL de MSLearn](https://github.com/MicrosoftLearning/mslearn-postgresql/blob/main/Allfiles/Labs/02).
1. Abra Azure Data Studio si aún no lo ha hecho.
1. Seleccione **Archivo**, **Abrir archivo** y vaya a la carpeta donde guardó el script. Selecciona **../Allfiles/Labs/02/Lab2_ZooDb.sql** y **Abrir**. Si se muestra una advertencia de confianza, seleccione **Abrir**.
1. Ejecute el script. Se crea la base de datos ZooDb.

## Creación de una cuenta de usuario en Microsoft Entra ID

> [!NOTE]
> En la mayoría de los entornos de producción o desarrollo, es muy posible que no tengas los privilegios de la cuenta de suscripción para crear cuentas en tu servicio Microsoft Entra ID.  En ese caso, si la organización lo permite, intenta pedir al administrador de Microsoft Entra ID que cree una cuenta de prueba automáticamente. Si no puedes obtener la cuenta Entra de prueba, omite esta sección y continúa con la sección **CONCEDER acceso a Azure Database for PostgreSQL**. 

1. En [Azure Portal](https://portal.azure.com), inicia sesión con una cuenta de Propietario y ve a Microsoft Entra ID.
1. En **Administrar**, seleccione **Usuarios**.
1. En la parte superior izquierda, seleccione **Nuevo usuario** y, a continuación, seleccione **Crear nuevo usuario**.
1. En la página **Nuevo usuario**, escriba estos detalles y seleccione **Crear**:
    - **Nombre principal de usuario:** elige un nombre de Principio.
    - **Nombre para mostrar:** elige un nombre para mostrar
    - **Contraseña:** desactiva **Generar automáticamente la contraseñas** y, a continuación, escribe una contraseña segura. Registra el nombre principal y la contraseña.
    - Haga clic en **Revisar y crear**

    > [!TIP]
    > Cuando se cree el usuario, anote el **nombre principal de usuario** completo para usarlo más adelante para iniciar sesión.

### Asignación del rol Lector

1. En el Azure Portal, seleccione **Todos los recursos** y, a continuación, seleccione el recurso de Azure Database for PostgreSQL.
1. Seleccione **Control de acceso (IAM)** y, luego, seleccione **Asignaciones de roles**. La nueva cuenta no aparece en la lista.
1. Seleccione **+ Agregar** y, luego, **Agregar asignación de roles**.
1. Seleccione el rol **Lector** y, a continuación, seleccione **Siguiente**.
1. Elige **+ Seleccionar miembros**, agrega la nueva cuenta que agregaste en el paso anterior a la lista de miembros y, a continuación, selecciona **Siguiente**.
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

1. Abre Azure Data Studio y conéctate al servidor de Azure Database for PostgreSQL con el usuario de **demostración** que estableciste como administrador anteriormente.
1. En el panel de consulta, ejecuta este código en la base de datos postgres. Se deben devolver doce roles de usuario, incluido el rol de **demostración** que usa para conectarse:

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Para crear un nuevo rol, ejecute este código.

    ```SQL
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```
    > [!NOTE]
    > Asegúrate de reemplazar la contraseña en el script anterior por una contraseña compleja.

1. Para incluir en la lista el nuevo rol, vuelve a ejecutar la consulta SELECT anterior en **pg_catalog.pg_roles**. Debería ver el rol **dbuser** en la lista.
1. Para habilitar el nuevo rol para consultar y modificar datos en la tabla **animal** de la base de datos **zoodb**, ejecuta este código en la base de datos zoodb:

    ```SQL
    GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
    ```

## Prueba de la aplicación nueva

1. En Azure Data Studio, en la lista de **CONEXIONES**, seleccione el botón de nueva conexión.
1. En **Tipo de conexión**, seleccione **PostgreSQL**.
1. En el cuadro de texto **Nombre del servidor**, escriba el nombre completo del servidor para el recurso de Azure Database for PostgreSQL. Puede hacerlo desde Azure Portal.
1. En la lista **Tipo de autenticación**, seleccione **Contraseña**.
1. En el cuadro de texto **Nombre de usuario**, escribe **dbuser** y, en el cuadro de texto **Contraseña**, escribe la contraseña compleja con la que creaste la cuenta.
1. Active la casilla **Recordar contraseña** y seleccione **Conectar**.
1. Seleccione **Nueva consulta** y ejecute este código:

    ```SQL
    SELECT * FROM animal;
    ```

1. Para probar si tiene el privilegio UPDATE, ejecute este código:

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

Estas pruebas muestran que el nuevo usuario puede ejecutar comandos del lenguaje de manipulación de datos (DML) para consultar y modificar datos, pero no puede usar comandos del lenguaje de definición de datos (DDL) para cambiar el esquema. Además, el nuevo usuario no puede conceder privilegios nuevos para eludir los permisos.

## Limpieza

No usarás este servidor PostgreSQL de nuevo, por tanto, elimina el grupo de recursos que creaste que quitará el servidor.
