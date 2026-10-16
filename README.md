# Informe grupal — Infraestructura de Redes

## Etapa 6: Monitoreo con Docker Compose, Prometheus y Grafana

## Introducción

El objetivo de este informe es documentar y explicar la infraestructura de monitoreo utilizada en la Etapa 6, tomando como referencia la configuración real del archivo `docker-compose.yml` y los archivos auxiliares del proyecto.

La infraestructura está compuesta por cinco servicios principales ejecutados mediante Docker Compose: MySQL, MySQL Exporter (`dbexporter`), cAdvisor, Prometheus y Grafana. Además, la aplicación CRUD desarrollada en Flask se ejecuta directamente en el host y expone métricas que Prometheus puede recolectar.

Cada componente cumple una función específica. MySQL administra los datos de la aplicación; `dbexporter` expone métricas relacionadas con la base de datos; cAdvisor recopila información sobre los recursos utilizados por los contenedores; Prometheus recolecta y almacena las métricas; y Grafana permite consultarlas y representarlas mediante paneles y gráficos.

El informe se divide en tres partes. La primera explica MySQL y `dbexporter`; la segunda aborda cAdvisor, Prometheus y la configuración de recolección; y la tercera describe Grafana y la red compartida `mynetwork`. En cada sección se analizan las funciones de los servicios y los elementos de sintaxis YAML utilizados.

## Parte 1 — Persistencia de datos: MySQL y dbexporter

### 1.1. Servicio MySQL

El servicio `mysql` utiliza la imagen `mysql:latest` y tiene configurada la política `restart: always`, que indica a Docker que intente reiniciar el contenedor cuando se detiene.

La variable de entorno `MYSQL_ROOT_PASSWORD` establece la contraseña del usuario administrador de MySQL. En la configuración analizada, el valor utilizado es `mysecretpassword`.

El servicio está conectado a la red `mynetwork`, lo que permite que los demás contenedores conectados a esa red se comuniquen con MySQL utilizando el nombre del servicio como destino.

MySQL no tiene un puerto publicado en el host dentro del archivo `docker-compose.yml` analizado. Por lo tanto, su acceso está destinado a los servicios que se comunican a través de la red interna de Docker.

### 1.2. Base de datos y persistencia

El script de instalación prepara una base de datos llamada `alumnos` y crea una tabla del mismo nombre con los campos `id`, `nombre`, `apellido` y `fecha_nacimiento`. También incorpora registros iniciales para probar la aplicación.

En el archivo `docker-compose.yml` analizado no se declara un volumen específico para almacenar los datos de MySQL fuera del contenedor. Por eso, no corresponde afirmar que la configuración garantiza la persistencia de los datos mediante volúmenes.

### 1.3. Servicio dbexporter

El servicio `dbexporter` utiliza la imagen `prom/mysqld-exporter:latest`. Su función es consultar información de MySQL y exponer métricas que Prometheus puede recolectar.

La directiva `depends_on` establece que MySQL debe iniciarse antes que `dbexporter`. Sin embargo, esto determina el orden de inicio y no garantiza que la base de datos esté preparada para aceptar conexiones.

El exportador utiliza el archivo de configuración `my.cnf`, montado mediante:

`./my.cnf:/.my.cnf:ro`

La ruta de la izquierda corresponde al archivo del proyecto en el host y la de la derecha indica dónde estará disponible dentro del contenedor. El sufijo `ro` significa *read-only* o solo lectura, lo que permite leer el archivo sin modificarlo mediante ese montaje.

El archivo `my.cnf` contiene los parámetros de conexión: usuario, contraseña, host y puerto. En esta práctica, el host configurado es `mysql`, que coincide con el nombre del servicio en la red compartida.

La opción `command` indica al exportador que utilice ese archivo:

`--config.my-cnf=/.my.cnf`

Así, `dbexporter` dispone de la configuración necesaria para conectarse a MySQL y exponer sus métricas.

### 1.4. Sintaxis YAML aplicada

- **`services`** agrupa los servicios de la infraestructura.
- **`image`** indica la imagen utilizada para crear un contenedor.
- **`environment`** define variables de entorno del servicio.
- **`volumes`** permite montar archivos o directorios del host dentro del contenedor.
- **`depends_on`** establece dependencias y orden de inicio entre servicios.
- **`command`** especifica comandos o argumentos para el contenedor.
- **`networks`** indica a qué red se conecta cada servicio.
- **`restart: always`** configura una política de reinicio.

En YAML, la indentación determina la relación entre las claves y sus valores. Mantener una estructura correcta permite que el archivo sea interpretado adecuadamente.
