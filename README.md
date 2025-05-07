# Reto Visualización

**Iñigo Murga, Mikel García y Jon Cañadas**

## Explicación

Este proyecto es un reto de visualización de datos mediante Kibana y Elastic. Por otro lado, hemos implementado seguridad con credenciales de acceso a los nodos. Para la realización hemos seguido los siguientes pasos:

1. Diseño del docker-compose sin seguridad

Para el diseño del docker-compose hemos tenido en cuenta las características más importantes que hemos considerado. Incluyendo dos contenedores.

2. Comprobar el funcionamiento del programa 

Tras construir y lanzar los contenedores, accedemos a los dos nodos comprobando que todo este correcto incluyendo la introduccion de un dataset de prueba.

3. Elección de CSV

Tras saber que todo funciona realizamos un analisis de los diferentes datasets que contine https://www.kaggle.com/ en busca de uno que nos encajase con nuestros requisitos.

5. Implementar visualizaciones

Continuamos analizando el CSV elegido para implementar las visualizaciones convenientes.

6. Implementación de seguridad

Tras lograr realizar todo lo anterior conseguimos implementar la seguridad de los nodos.
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

Sin seguridad:

    docker compose up -d
    
Con seguridad:

    docker-compose -f docker-composeSEC.yml up -d

## Uso

1. Acceder a Kibana:
    ```
    https://localhost:5601
    ```
2. Acceder a explorar por tu cuenta.

3. Acceder a Dashboard y seleccionar el existente.
   
## Configuración de Docker Compose

Aquí está la configuración de Docker Compose sin seguridad:

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
Este es el Docker Compose con seguridad:

```yaml
services:
  setup:
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
    user: "0"
    command: >
      bash -c '
        if [ x${ELASTIC_PASSWORD} == x ]; then
          echo "Set the ELASTIC_PASSWORD environment variable in the .env file";
          exit 1;
        elif [ x${KIBANA_PASSWORD} == x ]; then
          echo "Set the KIBANA_PASSWORD environment variable in the .env file";
          exit 1;
        fi;
        if [ ! -f config/certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
        if [ ! -f config/certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne \
          "instances:\n"\
          "  - name: es01\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: kibana\n"\
          "    dns:\n"\
          "      - kibana\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          > config/certs/instances.yml;
          bin/elasticsearch-certutil cert --silent --pem -out config/certs/certs.zip --in config/certs/instances.yml --ca-cert config/certs/ca/ca.crt --ca-key config/certs/ca/ca.key;
          unzip config/certs/certs.zip -d config/certs;
        fi;
        echo "Setting file permissions"
        chown -R root:root config/certs;
        find . -type d -exec chmod 750 \{\} \;;
        find . -type f -exec chmod 640 \{\} \;;
        echo "Waiting for Elasticsearch availability";
        until curl -s --cacert config/certs/ca/ca.crt https://es01:9200 | grep -q "missing authentication credentials"; do sleep 30; done;
        echo "Setting kibana_system password";
        until curl -s -X POST --cacert config/certs/ca/ca.crt -u "elastic:${ELASTIC_PASSWORD}" -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"${KIBANA_PASSWORD}\"}" | grep -q "^{}"; do sleep 10; done;
        echo "All done!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120

  es01:
    depends_on:
      setup:
        condition: service_healthy
    image: docker.elastic.co/elasticsearch/elasticsearch:${STACK_VERSION}
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - es01data:/usr/share/elasticsearch/data
    ports:
      - ${ES_PORT}:9200
    environment:
      - node.name=es01
      - cluster.name=${CLUSTER_NAME}
      - discovery.type=single-node
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.license.self_generated.type=${LICENSE}
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

  kibana:
    depends_on:
      es01:
        condition: service_healthy
    image: docker.elastic.co/kibana/kibana:${STACK_VERSION}
    volumes:
      - certs:/usr/share/kibana/config/certs
      - kibanadata:/usr/share/kibana/data
    ports:
      - ${KIBANA_PORT}:5601
    environment:
      - SERVERNAME=kibana
      - ELASTICSEARCH_HOSTS=https://es01:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -I -s --cacert config/certs/ca/ca.crt https://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120

volumes:
  certs:
    driver: local
  es01data:
    driver: local
  kibanadata:
    driver: local
```

## Posibles vías de mejora

✔️ Implementar más nodos de elastic para un mayor rendimiento y escalabilidad.

✔️ Se podría realizar un lavado del CSV haciendo diversas modificaciones como la eliminación de columnas no utilizadas.


## Problemas / Retos encontrados

🔴 En la integración de la seguridad en el sistema hemos tenido múltiples problemas como no poder acceder a elastic o tardía al lanzar el sistema para acceder a kibana.

🔴 Al analizar qué visualizaciones realizar, tuvimos problemas con el mapa geográfico por lo que tuvimos que buscar un CSV que se adecuara.


## Alternativas posibles

Se podría sustituir Kibana y utilizar Grafana puesto que es un proyecto más bien pequeño y temporal.

Se podría sustituir Elastic y utilizar Apache Solr analizando mejor el texto pero peor los números.

Sin embargo, la integración conjunta de Kibana y Elastic es más compatible que la de Grafana y Apache Solr.

