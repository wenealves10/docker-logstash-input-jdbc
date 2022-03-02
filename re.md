version: "2.2"

services:
  setup:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: "7.2.1"
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
    user: "0"
    command: >
      bash -c '
        yum install -y -q -e 0 unzip;
        if [ ! -f certs/ca.zip ]; then
          echo "Creating CA";
          bin/elasticsearch-certutil ca --silent --pem -out config/certs/ca.zip;
          unzip config/certs/ca.zip -d config/certs;
        fi;
        if [ ! -f certs/certs.zip ]; then
          echo "Creating certs";
          echo -ne \
          "instances:\n"\
          "  - name: es01\n"\
          "    dns:\n"\
          "      - es01\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: es02\n"\
          "    dns:\n"\
          "      - es02\n"\
          "      - localhost\n"\
          "    ip:\n"\
          "      - 127.0.0.1\n"\
          "  - name: es03\n"\
          "    dns:\n"\
          "      - es03\n"\
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
        until curl -s -X POST --cacert config/certs/ca/ca.crt -u elastic:79ae62e534e5bb81889839deed90a3c8 -H "Content-Type: application/json" https://es01:9200/_security/user/kibana_system/_password -d "{\"password\":\"79ae62e534e5bb81889839deed90a3c8\"}" | grep -q "^{}"; do sleep 10; done;
        echo "All done!";
      '
    healthcheck:
      test: ["CMD-SHELL", "[ -f config/certs/es01/es01.crt ]"]
      interval: 1s
      timeout: 5s
      retries: 120
    networks:
      - bonitas-mensagens-network


  es01:
    depends_on:
      setup:
        condition: service_healthy
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: "7.2.1"
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - esdata01:/usr/share/elasticsearch/data
    environment:
      - node.name=es01
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es02,es03
      - ELASTIC_PASSWORD=79ae62e534e5bb81889839deed90a3c8
      - ES_JAVA_OPTS=-Xmx512m -Xms512m
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es01/es01.key
      - xpack.security.http.ssl.certificate=certs/es01/es01.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es01/es01.key
      - xpack.security.transport.ssl.certificate=certs/es01/es01.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
    networks:
      - bonitas-mensagens-network
    restart: always

  es02:
    depends_on:
      - es01
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: "7.2.1"
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - esdata02:/usr/share/elasticsearch/data
    environment:
      - node.name=es02
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es03
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=79ae62e534e5bb81889839deed90a3c8
      - ES_JAVA_OPTS=-Xmx512m -Xms512m
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es02/es02.key
      - xpack.security.http.ssl.certificate=certs/es02/es02.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es02/es02.key
      - xpack.security.transport.ssl.certificate=certs/es02/es02.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
    networks:
      - bonitas-mensagens-network
    restart: always

  es03:
    depends_on:
      - es02
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: "7.2.1"
    volumes:
      - certs:/usr/share/elasticsearch/config/certs
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - esdata03:/usr/share/elasticsearch/data
    environment:
      - node.name=es03
      - cluster.initial_master_nodes=es01,es02,es03
      - discovery.seed_hosts=es01,es02
      - xpack.security.enabled=true
      - ELASTIC_PASSWORD=79ae62e534e5bb81889839deed90a3c8
      - ES_JAVA_OPTS=-Xmx512m -Xms512m
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.key=certs/es03/es03.key
      - xpack.security.http.ssl.certificate=certs/es03/es03.crt
      - xpack.security.http.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.key=certs/es03/es03.key
      - xpack.security.transport.ssl.certificate=certs/es03/es03.crt
      - xpack.security.transport.ssl.certificate_authorities=certs/ca/ca.crt
      - xpack.security.transport.ssl.verification_mode=certificate
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --cacert config/certs/ca/ca.crt https://localhost:9200 | grep -q 'missing authentication credentials'",
        ]
      interval: 10s
      timeout: 10s
      retries: 120
    networks:
      - bonitas-mensagens-network
    restart: always

  bonitas-postgres:
    image: postgres
    container_name: bonitas-postgres
    user: postgres
    environment:
      - POSTGRES_PASSWORD=xyKwxy4Ja6adKnV9xsyKwadAsAxy4Ja6KnV9
      - POSTGRES_USER=postgres
      - POSTGRES_DB=bonitasmensagens
    volumes:
      - postgres-bonitasmensagens:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - bonitas-mensagens-network
    restart: always

  # pgadmin-compose:
  #   image: dpage/pgadmin4
  #   environment:
  #     PGADMIN_DEFAULT_EMAIL: "weneplay5@gmail.com"
  #     PGADMIN_DEFAULT_PASSWORD: "wenealves1601"
  #   ports:
  #     - "4001:80"
  #   depends_on:
  #     bonitas-postgres:
  #       condition: service_healthy
  #   networks:
  #     - bonitas-mensagens-network

  bonitas-back:
    container_name: bonitas-back
    build: .
    entrypoint: ./.docker/entrypoint.sh
    environment:
      - DATABASE_URL=postgresql://postgres:xyKwxy4Ja6adKnV9xsyKwadAsAxy4Ja6KnV9@bonitas-postgres:5432/bonitasmensagens?schema=public
      - SECRET_JWT=64286c1d4759312bb66110b1fbdb1c
      - SESSION_SECRET=6cb58bb8016223cf23e2680152426503
      - AWS_ACCESS_KEY=VTQ1RGAUEAOZI7XNXAZ5
      - AWS_SECRET_ACCESS_KEY=Cv5O6bwO5FU6JAPzpZ6ReOdn3sogzjqGVeBc61Pe
      - AWS_END_POINT=s3.eu-central-1.wasabisys.com
      - AWS_BUCKET_NAME=cdn.bonitasmensagens.com.br
      - URL_SITE=https://www.bonitasmensagens.com.br
      - URL_API=https://api.bonitasmensagens.com.br
      - HASH_AUTH=cc69efe6ed6dba4229d0f87ea735687a
      - BUCKET_URL=https://cdn.bonitasmensagens.com.br
      - URL_CRAWLER=https://scraper.bonitasmensagens.com.br
      - SECRET_CRAWLER=c1dd5757e6eef656fa3b8dfe3edadd8c
      - MESSAGE_SM_SIZE_WIDTH=387
      - MESSAGE_SM_SIZE_HEIGHT=387
      - MESSAGE_MD_SIZE_WIDTH=595
      - MESSAGE_MD_SIZE_HEIGHT=595
      - AUTHOR_SIZE_WIDTH=80
      - AUTHOR_SIZE_HEIGHT=80
      - ELASTICSEARCH_NODE=https://es01:9200
      - ELASTICSEARCH_NODE1=https://es02:9200
      - ELASTICSEARCH_NODE2=https://es03:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=79ae62e534e5bb81889839deed90a3c8
    volumes:
      - ./:/home/back-bonitasmensagens
    depends_on:
      bonitas-postgres:
        condition: service_healthy

    networks:
      - bonitas-mensagens-network
    restart: always

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: "7.2.1"
    user: "0"
    volumes:
      - certs:/usr/share/logstash/config/certs
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/config/pipelines.yml:/usr/share/logstash/config/pipelines.yml:ro
      - ./logstash/queries:/usr/queries:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - bonitas-mensagens-network
    depends_on:
      bonitas-postgres:
        condition: service_healthy
    restart: always

  # kibana:
  #   depends_on:
  #     - es01
  #     - es02
  #     - es03
  #   build:
  #     context: kibana/
  #     args:
  #       ELK_VERSION: "7.2.1"
  #   user: "0"
  #   volumes:
  #     - certs:/usr/share/kibana/config/certs
  #     - kibanadata:/usr/share/kibana/data
  #     - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro
  #   networks:
  #     - bonitas-mensagens-network
  #   restart: always
  #   healthcheck:
  #     test:
  #       [
  #         "CMD-SHELL",
  #         "curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'",
  #       ]
  #     interval: 10s
  #     timeout: 10s
  #     retries: 120


  # bonitas-front:
  #   container_name: bonitas-front
  #   build:
  #     context: ../front-bonitasmensagens/
  #     args:
  #       - BASE_API=https://api.bonitasmensagens.com.br
  #       - BASE_SITE=https://bonitasmensagens.com.br
  #       - BASE_SCRAPER=https://scraper.bonitasmensagens.com.br/crawler
  #       - BASE_CDN=https://cdn.bonitasmensagens.com.br
  #   depends_on:
  #     - bonitas-back
  #   networks:
  #     - bonitas-mensagens-network
  #   restart: always

  # web_server:
  #   build:
  #     context: ./nginx
  #   restart: always
  #   ports:
  #     - 80:80
  #   volumes:
  #     - ./nginx/config/nginx.conf:/etc/nginx/nginx.conf:ro
  #   networks:
  #     - bonitas-mensagens-network


volumes:
  certs:
    driver: local
  postgres-bonitasmensagens:
    driver: local
  esdata01:
    driver: local
  esdata02:
    driver: local
  esdata03:
    driver: local
  bonitas-elasticsearch:
    driver: local
  bonitas-back-sitemap:
    driver: local
  # kibanadata:
  #   driver: local

networks:
  bonitas-mensagens-network:
    driver: bridge

    # external:
    #   name: nginx
