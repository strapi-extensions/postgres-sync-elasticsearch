# postgres-sync-elasticsearch


## Run with docker compose

```sh
$ docker compose up -d --build
```


### Postgres - grant permissions to current user

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


### Test 1 - insert new record to create new doc in es

```sql
INSERT INTO public.todos (title, completed, due)
VALUES ('Publish new release', false, '2024-02-15 16:00:00.000');
```


### Test 2 - update record to upate fields of doc in es


```sql
UPDATE public.todos SET completed = true 
WHERE id = 1;
```