---
lab:
  title: Exploración de Azure Database for PostgreSQL
  module: Explore PostgreSQL architecture
---

# Exploración de Azure Database for PostgreSQL

En este ejercicio, creará un servidor flexible de Azure Database for PostgreSQL y configurará el período de retención de copia de seguridad.

## Antes de comenzar

Debe tener una suscripción a Azure propia para completar este ejercicio. Si no tiene una suscripción a Azure, puede obtener una [evaluación gratuita de Azure](https://azure.microsoft.com/free).

### Creación de un grupo de recursos

1. En un explorador web, vaya a [Azure Portal](https://portal.azure.com). Inicie sesión con una cuenta de propietario o colaborador.
2. En Servicios de Azure, seleccione **Grupos de recursos** y, a continuación, seleccione **+ Crear**.
3. Comprueba que se muestra la suscripción correcta y después pon al grupo de recursos el nombre **rg-PostgreSQL_Flexi**. Seleccione una **región**.
4. Seleccione **Revisar + crear**. Seleccione **Crear**.

### Creación de servidor flexible de Azure Database for PostgreSQL

1. En Servicios de Azure, seleccione **+ Crear un recurso**.
    1. En **Buscar en Marketplace**, escribe **`azure database for postgresql flexible server`**, elige **Servidor flexible de Azure Database for PostgreSQL** y haz clic en **Crear**.
1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Suscripción: su suscripción.
    1. Grupo de recursos: **rg-PostgreSQL_Flexi**.
    1. Nombre del servidor: **psql-postgresql-fx9999** (el nombre del servidor debe ser único globalmente, por lo que reemplaza 9999 por cuatro números aleatorios).
    1. Región: seleccione la misma región que el grupo de recursos.
    1. Versión de PostgreSQL: selecciona 16.
    1. Tipo de carga de trabajo: **Desarrollo**.
    1. Proceso y almacenamiento: **Ampliable, B1ms**. Seleccione **Configurar servidor** y examine las opciones de configuración. No realices ningún cambio y cierra la hoja de la esquina derecha.
    1. Zona de disponibilidad: sin preferencia.
    1. Alta disponibilidad: deshabilitada.
    1. Método de autenticación: **solo autenticación de PostgreSQL**
    1. En **nombre de usuario de administrador**, escribe **`demo`**.
    1. En **contraseña**, escribe una contraseña adecuadamente compleja.
    1. Seleccione **Siguiente: Redes >**.
1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Método de conectividad: (o) Acceso público (direcciones IP permitidas) y Punto de conexión privado
    1. Acceso público, selecciona **Permitir el acceso público a este recurso a través de Internet mediante una dirección IP pública**
    1. En Reglas de firewall, seleccione **+ Agregar dirección IP del cliente actual** para agregar la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall.
1. Seleccione **Revisar + crear**. Revisa la configuración, y, después, selecciona **Crear** para crear el servidor flexible de Azure Database for PostgreSQL. Una vez completada la implementación, seleccione **Ir al recurso** para ir el paso siguiente.

## Examen de los parámetros del servidor

1. En **Configuración**, seleccione **Parámetros del servidor**.
1. En el cuadro **Buscar para filtrar elementos...**, escribe **`connections`**. Se muestran los parámetros del servidor relacionados con las conexiones. Anote el valor de **max_connections**. No realice ningún cambio.
1. En el menú de la izquierda, seleccione **Información general** para salir de **Parámetros del servidor**.

## Cambio del período de retención de la copia de seguridad

1. Vaya a la hoja **Información general**, en **Configuración**, y seleccione **Proceso y almacenamiento**. En esta sección se muestra el nivel de proceso actual y la opción de actualizarlo. También se muestra la cantidad de almacenamiento que ha aprovisionado y la opción para aumentar el almacenamiento.
1. En **copias de seguridad**, se muestra el período de retención de copia de seguridad en días. Con la barra deslizante, cambie el período de retención de copia de seguridad a 14 días. Seleccione **Guardar** para conservar los cambios.
1. Cuando hayas completado este ejercicio, ve a la hoja **Información general** y haz clic en **DETENER**, lo que detendrá el servidor.
    1. No se te cobrará mientras el servidor esté en estado detenido, pero ten en cuenta que en un plazo de siete días se reiniciará el servidor si no lo has eliminado.

## Ejercicio opcional: configuración de un servidor de alta disponibilidad

1. En Servicios de Azure, seleccione **+ Crear un recurso**.
    1. En **Buscar en Marketplace**, escribe **`azure database for postgresql flexible server`**, elige **Servidor flexible de Azure Database for PostgreSQL** y haz clic en **Crear**.
1. En la pestaña **Aspectos básicos** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Suscripción: su suscripción.
    1. Grupo de recursos: **rg-PostgreSQL_Flexi**.
    1. Nombre del servidor: **psql-postgresql-fx8888** (el nombre del servidor debe ser único globalmente, por lo que reemplace 8888 por números aleatorios).
    1. Región: seleccione la misma región que el grupo de recursos.
    1. Versión de PostgreSQL: selecciona 16.
    1. Tipo de carga de trabajo: **Producción (pequeña o mediana)**
    1. Proceso y almacenamiento: dejar como **Uso general**.
    1. Zona de disponibilidad: puedes dejar esto en "Sin preferencia" y Azure elegirá automáticamente las zonas de disponibilidad para tus servidores principal y secundario. Como alternativa, especifique una zona de disponibilidad para ubicar conjuntamente con la aplicación.
    1. Habilitar alta disponibilidad: active esta opción. Tenga en cuenta los costos estimados cuando se selecciona esta opción.
    1. Modo de alta disponibilidad: elige **Redundancia de zona: un servidor en espera siempre está disponible dentro de otra zona en la misma región que el servidor principal**.
    1. Método de autenticación: **solo autenticación de PostgreSQL**
    1. En **Nombre de usuario de administrador**, escriba **demo**.
    1. En **contraseña**, escribe una contraseña adecuadamente compleja.
    1. Seleccione **Siguiente: Redes >**.
1. En la pestaña **Redes** del servidor flexible, escriba cada campo de la siguiente manera:
    1. Método de conectividad: (o) Acceso público (direcciones IP permitidas) y Punto de conexión privado
    1. Acceso público, selecciona **Permitir el acceso público a este recurso a través de Internet mediante una dirección IP pública**
    1. En Reglas de firewall, seleccione **+ Agregar dirección IP del cliente actual** para agregar la dirección IP actual como regla de firewall. Opcionalmente, puede asignar un nombre significativo a esta regla de firewall.
1. Seleccione **Revisar + crear**. Revisa tu configuración y después selecciona **Crear** para crear tu servidor flexible Azure Database for PostgreSQL. Una vez completada la implementación, seleccione **Ir al recurso** para ir el paso siguiente.

### Inspección del nuevo servidor

1. En **Configuración**, seleccione **Alta disponibilidad**. La alta disponibilidad está habilitada y la zona de disponibilidad principal debe ser la zona 1.
    1. La zona de disponibilidad en espera se ha asignado automáticamente y será diferente a la zona de disponibilidad principal y normalmente es la zona 2.

### Forzado de una conmutación por error

1. En la hoja **Alta disponibilidad**, en el menú superior, seleccione **Conmutación por error forzada**. Se muestra un mensaje, seleccione **Aceptar**.
1. Se inicia el proceso de conmutación por error. Se muestra un mensaje cuando la operación de conmutación por error se ha completado correctamente.
1. En la hoja **Alta disponibilidad**, puede ver que la zona principal es ahora 2 y que la zona de disponibilidad en espera es 1. Puede que necesite actualizar el explorador para ver la información más reciente.
1. Cuando haya completado este ejercicio, elimine el servidor.

## Limpieza

El servidor de este ejercicio de laboratorio incurrirá en cargos. Elimina el grupo de recursos **rg-PostgreSQL_Flexi** una vez que hayas completado el ejercicio. Esto quitará el servidor y cualquier otro recurso que hayas implementado en este ejercicio de laboratorio.
