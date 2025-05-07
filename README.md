# Reto Visualización

**Iñigo Murga, Mikel García y Jon Cañadas**

## Explicación

Este proyecto es un reto de visualización de datos mediante Kibana y Elastic. Por otro lado, hemos intentado implementar seguridad pero no hemos tenido exito. Para la realización hemos seguido los siguientes pasos:

1. Diseño del docker-compose sin seguridad

Para el diseño del docker-compose hemos tenido en cuenta las características más importantes que hemos considerado. Incluyendo dos contenedores.

2. Comprobar el funcionamiento del programa 

Tras construir y lanzar los contenedores, accedemos a los dos nodos comprobando que todo este correcto incluyendo la introduccion de un dataset de prueba.

3. Elección de CSV

Tras saber que todo funciona realizamos un analisis de los diferentes datasets que contine https://www.kaggle.com/ en busca de uno que nos encajase con nuestros requisitos.

5. Implementar visualizaciones

Por último, analizamos el CSV elegido para implementar las visualizaciones convenientes.
## Instalación

1. Clonar el repositorio:
    ```bash
    git clone https://github.com/inigomurga/visualizacion-G2.git
    ```
2. Navega al directorio del proyecto:
    ```bash
    cd visualizacion-G2
    ```
3. Construir y lanza los contenedores Docker:
    ```bash
    docker compose up -d
    ```

## Uso

1. Acceder a Kibana:
    ```
    http://localhost:5601
    ```
2. Acceder a explorar por tu cuenta.

3. Acceder a Dashboard y seleccionar el existente.
   
## Configuración de Docker Compose

Aquí está la configuración de Docker Compose:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION} # Imagen de Elastic para Docker
    container_name: elasticsearch # Nombre del contenedor
    environment: # Especificar la variables de entorno
      - discovery.type=single-node # Especificación de que va a ser un único nodo
      - bootstrap.memory_lock=true # Evita que Elastic se quede sin memoria bloqueando el uso
      - xpack.security.enabled=false # Deshabilitar seguridad
    ulimits: # Determina los límites de recursos que puedes aplicar a un contenedor en Docker
      memlock: # Límite de memoria bloqueada en RAM
        soft: -1 # Límite que puede ser superado temporalmente -1 es infinito
        hard: -1 # Límite que no puede ser superado -1 es infinito
    volumes: # Volumen para la persistencia de datos
      - esdata:/usr/share/elasticsearch/data #Ubicación del volumen
    ports: # Mapeo de puertos
      - "9200:9200" # Puertos específicos, el primero es en referencia al puerto local y el segundo al del contenedor de Docker

  kibana:
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION} # Imagen de Kibana para Docker
    container_name: kibana # Nombre del contenedor
    environment: # Especificar la variables de entorno
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200 # Conexión HTTP
      - xpack.security.enabled=false # Deshabilitar seguridad
    depends_on: # El servicio va a depender del contenedor que se especifique
      - elasticsearch # Contenedor del que depende
    ports: # Mapeo de puertos
      - "5601:5601" # Puertos específicos, el primero es en referencia al puerto local y el segundo al del contenedor de Docker

volumes: 
  esdata: # Define el volumen esdata usando el driver local de Docker
    driver: local

```

## Posibles vías de mejora

✔️ Para evitar acceso no deseados estaría bien la introducción de seguridad.

✔️ Se podría realizar un lavado del CSV haciendo diversas modificaciones como la eliminación de columnas no utilizadas.


## Problemas / Retos encontrados

🔴 Al probar a integrar el sistema con seguridad hemos tenido múltiples problemas como no poder acceder a elastic o tardía al lanzar el sistema para acceder a kibana.

🔴 Al analizar qué visualizaciones realizar, tuvimos problemas con el mapa geográfico por lo que tuvimos que buscar un CSV que se adecuara.


## Alternativas posibles

Se podría sustituir Kibana y utilizar Grafana puesto que es un proyecto más bien pequeño y temporal.

Se podría sustituir Elastic y utilizar Apache Solr analizando mejor el texto pero peor los números.

Sin embargo, la integración conjunta de Kibana y Elastic es más compatible que la de Grafana y Apache Solr.

