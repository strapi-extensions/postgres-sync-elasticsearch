# postgres-sync-elasticsearch


## Run with docker compose

```sh
$ docker compose up -d --build
```


### Postgres - grant permissions to current user


```sh
docker ps

CONTAINER ID   IMAGE                                             COMMAND                  CREATED             STATUS                         PORTS                                                                                            NAMES
9b6a9a89fa34   quay.io/debezium/connect:latest                   "/docker-entrypoint.…"   About an hour ago   Up About an hour               8778/tcp, 9092/tcp, 0.0.0.0:8082->8083/tcp, :::8082->8083/tcp                                    pg-connect
2f6b5f9c352b   confluentinc-es-connect:latest                    "/etc/confluent/dock…"   About an hour ago   Up About an hour (unhealthy)   9092/tcp, 0.0.0.0:8084->8083/tcp, :::8084->8083/tcp                                              es-connect
ec9246f8832f   docker.redpanda.com/redpandadata/console:latest   "./console"              About an hour ago   Up About an hour               0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                                                        kafka-console
1dc12a3d8d45   bitnami/postgresql:16                             "/opt/bitnami/script…"   About an hour ago   Up About an hour               0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                                        postgresql
130c19dda245   bitnami/kafka:3.6                                 "/opt/bitnami/script…"   About an hour ago   Up About an hour               0.0.0.0:9092->9092/tcp, :::9092->9092/tcp                                                        kafka
a9cb8eba8f03   bitnami/elasticsearch:8                           "/opt/bitnami/script…"   About an hour ago   Up About an hour               0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp             elasticsearch

docker exec -i -t 1dc12a3d8d45 /bin/bash
```

try to use root user run the following sql -> 

> PGPASSWORD=admin123abc psql -U postgres demo

```sql
ALTER ROLE "pg-connector" SUPERUSER CREATEDB NOCREATEROLE INHERIT LOGIN REPLICATION NOBYPASSRLS;
```

### Postgres - create demo table 

```sql
CREATE TABLE public.todos (
	id serial4 PRIMARY KEY NOT NULL,
	title varchar(255) NULL,
	completed bool NULL,
	due timestamp(6) NULL
);
```

### pg-connect - install pg-sink-connector

```sh
curl --location 'http://localhost:8082/connectors' \
--header 'Content-Type: application/json' \
--data '{
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.dbname": "demo",
        "database.hostname": "postgresql",
        "database.password": "pg123abc",
        "database.port": "5432",
        "database.user": "pg-connector",
        "name": "pg-sink-connector",
        "table.include.list": "public.todos",
        "message.key.columns": "my_database.users:id",
        "schema.include.list": "public",
        "tasks.max": "1",
        "topic.prefix": "demo",
        "plugin.name": "pgoutput",
        "transforms":"unwrap",
        "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
        "transforms.unwrap.drop.tombstones": "false",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "transforms.unwrap.add.fields": "table,ts_ms"
    },
    "name": "pg-sink-connector"
}'
```

### Postgres - insert a todo 

```sql
INSERT INTO public.todos (title, completed, due)
VALUES ('Fix 502 bug', false, '2024-02-15 16:00:00.000');
```

### Kafka - check the topic and messages

open url with your browser -> http://localhost:8080/topics/demo.public.todos?p=-1&s=50&o=-1#messages

![image](https://github.com/strapi-extensions/postgres-sync-elasticsearch/assets/5119542/47899738-ddbe-4e73-ad37-dbf52c4205b8)

the first step is done.

### es-connect - install es-sink-connector

```sh
curl --location 'http://localhost:8084/connectors' \
--header 'Content-Type: application/json' \
--data '{
    "name": "es-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "tasks.max": "1",
        "topics": "demo.public.todos",
        "connection.url": "http://elasticsearch:9200",
        "key.ignore": "false",
        "transforms": "extractKey",
        "transforms.extractKey.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
        "transforms.extractKey.field": "id"
    }
}'
```

we can check elasticsearch with https://github.com/cars10/elasticvue.

![image](https://github.com/strapi-extensions/postgres-sync-elasticsearch/assets/5119542/6b57e212-7d30-4072-b5dc-b25e439fbafb)


### Test 1 - insert new record to create new doc in es

```sql
INSERT INTO public.todos (title, completed, due)
VALUES ('Publish new release', false, '2024-02-15 16:00:00.000');
```


### Test 2 - update record to upate fields of doc in es

```sql
UPDATE public.todos SET completed = false, title = 'Create bug issue edited'
WHERE id = 2;
```

there we should see the doc is upserted and should not add a new record with id is 2.

![image](https://github.com/strapi-extensions/postgres-sync-elasticsearch/assets/5119542/96d66f71-8511-4ef4-9a94-950a06622fa3)


