# Informe grupal — Infraestructura de Redes

## Etapa 6: Monitoreo con Docker Compose, Prometheus y Grafana

## Introducción

El objetivo de este informe es documentar y explicar la infraestructura de monitoreo utilizada en la Etapa 6, tomando como referencia la configuración real del archivo `docker-compose.yml` y los archivos auxiliares del proyecto.

La infraestructura está compuesta por cinco servicios principales ejecutados mediante Docker Compose: MySQL, MySQL Exporter (`dbexporter`), cAdvisor, Prometheus y Grafana. Además, la aplicación CRUD desarrollada en Flask se ejecuta directamente en el host y expone métricas que Prometheus puede recolectar.

Cada componente cumple una función específica. MySQL administra los datos de la aplicación; `dbexporter` expone métricas relacionadas con la base de datos; cAdvisor recopila información sobre los recursos utilizados por los contenedores; Prometheus recolecta y almacena las métricas; y Grafana permite consultarlas y representarlas mediante paneles y gráficos.

El informe se divide en tres partes. La primera explica MySQL y `dbexporter`; la segunda aborda cAdvisor, Prometheus y la configuración de recolección; y la tercera describe Grafana y la red compartida `mynetwork`. En cada sección se analizan las funciones de los servicios y los elementos de sintaxis YAML utilizados.
