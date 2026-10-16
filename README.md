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

- ## Parte 2 — Recolección de métricas: cAdvisor y Prometheus

### 2.1. Servicio cAdvisor

El servicio `cadvisor` utiliza la imagen `gcr.io/cadvisor/cadvisor:v0.47.2`. Su función es recopilar información sobre el uso de recursos y el estado de los contenedores que se ejecutan en el host.

En el archivo `docker-compose.yml`, cAdvisor tiene varios montajes de directorios del sistema:

- `/:/rootfs:ro`
- `/var/run:/var/run:ro`
- `/sys:/sys:ro`
- `/var/lib/docker/:/var/lib/docker:ro`
- `/dev/disk/:/dev/disk:ro`

En cada montaje, la ruta de la izquierda corresponde al host y la de la derecha al destino dentro del contenedor. El sufijo `ro` indica que el montaje es de solo lectura. Esto permite que cAdvisor consulte información del sistema y de Docker sin tener permiso de escritura a través de esos montajes.

El servicio también está conectado a la red `mynetwork`, por lo que Prometheus puede acceder a sus métricas utilizando el nombre `cadvisor` como dirección de destino.

### 2.2. Servicio Prometheus

Prometheus utiliza la imagen `prom/prometheus:latest`. Su función es recolectar y almacenar las métricas expuestas por los servicios configurados como objetivos de monitoreo.

El archivo `docker-compose.yml` publica el puerto `9090` mediante la configuración:

`"9090:9090"`

El primer puerto corresponde al host y el segundo al contenedor. De esta manera, la interfaz web de Prometheus puede consultarse desde el host a través del puerto 9090.

El servicio monta el archivo de configuración `prometheus.yml` mediante:

`./prometheus.yml:/etc/prometheus/prometheus.yml`

Esto permite que Prometheus utilice el archivo de configuración generado por el proyecto.

### 2.3. Configuración de recolección

En `prometheus.yml` se define un intervalo de recolección de cinco segundos mediante `scrape_interval: 5s`.

La configuración incluye cuatro objetivos:

- `localhost:9090`, para recolectar métricas de Prometheus.
- `cadvisor:8080`, para obtener métricas de cAdvisor.
- `dbexporter:9104`, para obtener métricas relacionadas con MySQL.
- `host.docker.internal:8888`, para consultar las métricas de la aplicación CRUD que se ejecuta en el host.

Los nombres `cadvisor` y `dbexporter` permiten que Prometheus encuentre esos servicios dentro de la red compartida de Docker. En cambio, la aplicación CRUD se ejecuta directamente en el host y utiliza el puerto 8888.

### 2.4. Uso de extra_hosts

El servicio Prometheus contiene la configuración:

`host.docker.internal:host-gateway`

La directiva `extra_hosts` agrega una entrada al archivo de hosts del contenedor. En este caso, permite resolver el nombre `host.docker.internal` hacia la dirección del host proporcionada por Docker mediante `host-gateway`.

Esto es necesario para que Prometheus pueda consultar las métricas de la aplicación CRUD que se ejecuta fuera de los contenedores, en el host, en el puerto 8888.

### 2.5. Sintaxis YAML aplicada

- **`ports`** publica puertos del contenedor en el host.
- **`volumes`** monta archivos y directorios.
- **`extra_hosts`** agrega nombres y direcciones al archivo de hosts del contenedor.
- **`scrape_interval`** define cada cuánto tiempo Prometheus intenta recolectar métricas.
- **`scrape_configs`** agrupa la configuración de los objetivos que se van a monitorear.
- **`static_configs`** permite declarar objetivos de recolección de forma estática.
- **`targets`** contiene las direcciones y puertos de los objetivos.

La indentación es importante en YAML porque establece la relación entre las claves y sus valores. Una estructura incorrecta puede impedir que la configuración sea interpretada correctamente.

En YAML, la indentación determina la relación entre las claves y sus valores. Mantener una estructura correcta permite que el archivo sea interpretado adecuadamente.
