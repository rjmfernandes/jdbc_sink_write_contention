# JDBC Sink Write Contention

- [JDBC Sink Write Contention](#jdbc-sink-write-contention)
  - [Setup](#setup)
  - [Install connectors plugins](#install-connectors-plugins)
  - [Create and populate topic](#create-and-populate-topic)
  - [Create the Database table](#create-the-database-table)
  - [Topic key should be part of table primary key](#topic-key-should-be-part-of-table-primary-key)
  - [Pay attention to concurrent sink connectors](#pay-attention-to-concurrent-sink-connectors)
  - [Reduce Concurrency](#reduce-concurrency)
  - [Max Retries](#max-retries)
  - [Database overload - Table level locking](#database-overload---table-level-locking)
  - [DB triggers](#db-triggers)
  - [General Rules](#general-rules)
  - [Cleanup](#cleanup)


## Setup

Start services:

```shell
docker compose up -d
```

Check logs:

```shell
docker compose logs -f
```

You should have: 
- Control Center 1 available at: http://localhost:9021/clusters
- Control Center 2 available at: http://localhost:9022/clusters

## Install connectors plugins

Now let's install our connector plugins:

Go into connect instance 1:

```shell
docker compose exec connect bash
```

Install plugins:

```shell
confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:latest
confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:latest
```

After restart both Connect instances since they share plugins folder:

```shell
docker compose restart connect
docker compose restart connect2
```

Check connector plugins for both:

```shell
curl localhost:8083/connector-plugins | jq
curl localhost:8084/connector-plugins | jq
```

## Create and populate topic

First lets create our topic with 8 partitions for each cluster:

```shell
kafka-topics --bootstrap-server localhost:9092 --topic customers --create --partitions 8 --replication-factor 1
kafka-topics --bootstrap-server localhost:9093 --topic customers --create --partitions 8 --replication-factor 1
```

After we create our Datagen connectors to populate it.

```shell
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/my-datagen-source/config -d '{
    "name" : "my-datagen-source",
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic" : "customers",
    "output.data.format" : "AVRO",
    "quickstart" : "SHOE_CUSTOMERS",
    "tasks.max" : "1"
}'
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8084/connectors/my-datagen-source/config -d '{
    "name" : "my-datagen-source",
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic" : "customers",
    "output.data.format" : "AVRO",
    "quickstart" : "SHOE_CUSTOMERS",
    "tasks.max" : "1"
}'
```

Check on each cluster a topic `customers` was created and is being populated by corresponding connector.

## Create the Database table

In Postgres we create the table:

```sql
create table customers (id text NOT NULL,first_name text NOT NULL, last_name text NOT NULL, 
    email text NOT NULL,phone text NOT NULL, street_address text NOT NULL, "state" text NOT NULL,
    zip_code text NOT NULL, country text NOT NULL, country_code text NOT NULL,
    PRIMARY KEY (first_name,last_name,"state", country, country_code) );
```

## Topic key should be part of table primary key

Now let's create our sink connectors:

```shell
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-si1/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "first_name,last_name,state,country,country_code",
          "tasks.max"          : "8",
          "batch.size"          : "50",
          "auto.create"        : "false",
          "auto.evolve"        : "false",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8084/connectors/my-si1/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "first_name,last_name,state,country,country_code",
          "tasks.max"          : "8",
          "batch.size"          : "50",
          "auto.create"        : "true",
          "auto.evolve"        : "true",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
```

Check the logs for postgres:

```shell
docker compose logs -f postgres
```

You should see some deadlocks detected looking like this:

```
postgres  | 2024-09-28 14:21:31.817 GMT [632] ERROR:  deadlock detected
postgres  | 2024-09-28 14:21:31.817 GMT [632] DETAIL:  Process 632 waits for ShareLock on transaction 15020; blocked by process 633.
postgres  | 	Process 633 waits for ShareLock on transaction 15024; blocked by process 632.
postgres  | 	Process 632: INSERT INTO "postgres"."public"."customers" ("first_name","last_name","state","country","country_code","id","email","phone","street_address","zip_code") VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10) ON CONFLICT ("first_name","last_name","state","country","country_code") DO UPDATE SET "id"=EXCLUDED."id","email"=EXCLUDED."email","phone"=EXCLUDED."phone","street_address"=EXCLUDED."street_address","zip_code"=EXCLUDED."zip_code"
postgres  | 	Process 633: INSERT INTO "postgres"."public"."customers" ("first_name","last_name","state","country","country_code","id","email","phone","street_address","zip_code") VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10) ON CONFLICT ("first_name","last_name","state","country","country_code") DO UPDATE SET "id"=EXCLUDED."id","email"=EXCLUDED."email","phone"=EXCLUDED."phone","street_address"=EXCLUDED."street_address","zip_code"=EXCLUDED."zip_code"
postgres  | 2024-09-28 14:21:31.817 GMT [632] HINT:  See server log for query details.
postgres  | 2024-09-28 14:21:31.817 GMT [632] CONTEXT:  while inserting index tuple (2,19) in relation "customers"
postgres  | 2024-09-28 14:21:31.817 GMT [632] STATEMENT:  INSERT INTO "postgres"."public"."customers" ("first_name","last_name","state","country","country_code","id","email","phone","street_address","zip_code") VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10) ON CONFLICT ("first_name","last_name","state","country","country_code") DO UPDATE SET "id"=EXCLUDED."id","email"=EXCLUDED."email","phone"=EXCLUDED."phone","street_address"=EXCLUDED."street_address","zip_code"=EXCLUDED."zip_code"
```

The first obvious source of the deadlocks is the fact that our topic key is not part of the primary key of the table. This means that different connector tasks will try to update the same rows on the database table.

## Pay attention to concurrent sink connectors

Let's delete our connectors and our table in postgres and recreate the table with an example that matches the key of the topic (in reality you would probably do the other way around and start with a topic with a key that it was part of the primary key of the table if possible):

```shell
curl -X DELETE http://localhost:8083/connectors/my-si1
curl -X DELETE http://localhost:8084/connectors/my-si1
```

```sql
DROP TABLE customers;
create table customers (id text NOT NULL,first_name text NOT NULL, last_name text NOT NULL, 
    email text NOT NULL,phone text NOT NULL, street_address text NOT NULL, "state" text NOT NULL,
    zip_code text NOT NULL, country text NOT NULL, country_code text NOT NULL,
    PRIMARY KEY (id) );
```

Now we create our new connectors:

```shell
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-sink1/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "id",
          "tasks.max"          : "8",
          "batch.size"          : "50",
          "auto.create"        : "false",
          "auto.evolve"        : "false",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8084/connectors/my-sink1/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "id",
          "tasks.max"          : "8",
          "batch.size"          : "50",
          "auto.create"        : "true",
          "auto.evolve"        : "true",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
```

Still some deadlocks should show up in the postgres log. That's because there are still concurrent independent updates coming from the fact we have two streams/sinks going to the database independently and they may be trying to update the same rows.

If it was the same data you should definitely try to avoid having both running at same time.

Just for the sake of confirming this, let's delete our connectors and recreate the table as before:

```shell
curl -X DELETE http://localhost:8083/connectors/my-sink1
curl -X DELETE http://localhost:8084/connectors/my-sink1
```

But now we create only one connector:

```shell
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-sink2/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "id",
          "tasks.max"          : "8",
          "batch.size"          : "50",
          "auto.create"        : "false",
          "auto.evolve"        : "false",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
```

You should see no deadlocks because we have only one sink happening and with the right match between topic key and table primary key. (Basically following the rule that any field on the key is part of the primary key of the table.)

## Reduce Concurrency

If you needed for some reason to have both sinks running but want to reduce the chance of possible collisions originating deadlocks you need to consider reducing concurrency. So for example we could switch to less tasks on each connector.

So first we delete again our connector and our table and recreate the table as before.

```shell
curl -X DELETE http://localhost:8083/connectors/my-sink2
```

Now we create our connectors with just 1 task each:

```shell
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/my-sink-con1/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "id",
          "tasks.max"          : "1",
          "batch.size"          : "50",
          "auto.create"        : "false",
          "auto.evolve"        : "false",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8084/connectors/my-sink-con1/config \
    -d '{
          "connector.class"    : "io.confluent.connect.jdbc.JdbcSinkConnector",
          "connection.url"     : "jdbc:postgresql://host.docker.internal:5432/postgres",
          "connection.user"    : "postgres",
          "connection.password": "password",
          "topics"             : "customers",
          "insert.mode": "upsert",
          "pk.mode": "record_value",
          "pk.fields": "id",
          "tasks.max"          : "1",
          "batch.size"          : "50",
          "auto.create"        : "true",
          "auto.evolve"        : "true",
          "value.converter.schema.registry.url": "http://schema-registry:8081",
          "value.converter.schemas.enable":"false",
          "key.converter"       : "org.apache.kafka.connect.storage.StringConverter",
          "value.converter"     : "io.confluent.connect.avro.AvroConverter"}'
```

You may still get some deadlock although for sure less than before but since we can't guarantee there are no concurrent updates since both sinks happen independently there's not much we can do to improve although we should be able to handle it to a good extent with `max.retries`. (see next)

## Max Retries

If you really can't avoid concurrent updates on same rows: 
- Maybe because (like just before) you have two independent sinks happening which will always eventually compete for same rows.
- Maybe because you can't change the fact that your topic key is not part of the primary key of the table and so different tasks will try to concurrently update sometimes the same rows. (**In this case reducing the number of tasks to 1 would also avoid the problem but with corresponding perfomance cost.**)

You can leverage `max.retries` so that errors are retried before giving up. (**Also consider configuring a dead letter queue for those ones!**)

By default `max.retries` is 10 and you can increase although the default may be sufficient for you. You should use it in conjunction with `retry.backoff.ms` (default to 3000, 3 seconds) which you can also change as per your needs.

## Database overload - Table level locking

Although not as easy to reproduce in a toy example as this one, sometimes your database can get overloaded by too many connector tasks hitting the same table. This can lead to the database switching from row level locking to table locking and deadlocks will explode in those circunstances and it will be hard to manage only with retries...

Different databases will have different conditions under which this can happen so involve your DBA team to monitor the database and investigate the cause of deadlocks.

But in any case as a general rule for large topics (and large tables) with many topic partitions you probably don't want to assign a number of sink connector tasks as large your number of partitions and will want to use a much smaller number. Remember: In general the level of parallelization that Kafka and Kafka Connect can achieve is way above what your database can handle.

## DB triggers

Database triggers associated with your database can also lead to even more chances of deadlocks, by some of the following reasons:
- they may compete for common resources as autoincrementing fields
- they may lead to more database load and make it easier for the database to switch to table level locking and so cause deadlock explosion

Pay attention and review any database triggers you may have defined to see if they compete for common resources. And again involve your DBA team to confirm if the level of load you are imposing cause of your DB triggers are leading to deadlock explosion.

## General Rules

* Confirm your topic key (all fields) are part of the primary key. If any field of your topic key is not part of the primary key, or if you have no key at all, this will mean that with multiple sink tasks they will compete for same rows and so you will have deadlocks.
  - If you can't change either the topic key or the database table primary key consider using for your sink connector consider using just 1 task.
  - If the performance burden of using just 1 task is too much leverage `max.retries` and dead letter topic.
* Confirm if there are multiple independent sinks happening against the same database table. If there are then most likely you will hit deadlocks.
  - If you can't change that again leverage `max.retries` and dead letter topic.
* Review your possible database triggers that may be competing for shared resources.
  - If you can't change the db trigger leading to deadlocks because of common resources competition consider using just 1 task for your sink connector and/or leverage `max.retries` and dead letter topic.
* Confirm if the cause of the deadlocks is table level locking lead by excessive load on your database and table.
  - Make sure you work with your DBA team to monitor the sink from the database side and check for any possible switch to table level locking happening and so leading to deadlock explosion.
  - If that's the case work with the database team to tune the database and/or from Kafka Connect side to reduce the number of tasks and so reducing the load on database. In any case don't try to use generally the number of partitions of your topic as the number of tasks for your sink connector.
  - Also review any possible db triggers that may be leading to excessive load.

## Cleanup

```shell
docker compose down -v
```
