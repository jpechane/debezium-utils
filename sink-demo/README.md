# Debezium Sink Demo

This setup is going to demonstrate how to receive events from MySQL database and stream them down to PostreSQL database and into an ElasticSearch instance to keep all three datastores synchronized.


## Topology

```
                   +-------------+
                   |             |
                   |    MySQL    |
                   |             |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |  (Debezium, JDBC, ElasticSearch  |
          |           connectors)            |
          |                                  |
          +---+-----------------------+------+
              |                       |
              |                       |
              |                       |
              |                       |
+-------------v--+                +---v---------------+
|                |                |                   |
|   PostgreSQL   |                |   ElasticSearch   |
|                |                |                   |
+----------------+                +-------------------+

```
We are using a Docker Compose to deploy following components
* MySQL
* Kafka
  * Zookeeper
  * Kafka Broker
  * Kafka Connect with [Debezium](http://debezium.io/), [JDBC](https://github.com/confluentinc/kafka-connect-jdbc) and [ElasticSearch](https://github.com/confluentinc/kafka-connect-elasticsearch) Connectors
* PostgreSQL
* ElasticSearch

## Pre-requisities
To get this setup running you need
* https://github.com/debezium/debezium/pull/270
* https://github.com/debezium/docker-images/pull/44
* You need to locally build a 0.6 Debezium Connect image with
  * Debezium 0.6.0-SNAPSHOT
  * JDBC and ElasticSearch Connectors developed by Confluent

## Usage
How to run:

```shell
# Start the application
env DEBEZIUM_VERSION=0.6 docker-compose up

# Start PostgreSQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @jdbc-sink.json

# Start ElasticSearch connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @elastic-sink.json

# Start MySQL connector
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source.json
```

Check content of MySQL database
```shell
docker exec -it sinkdemo_mysql_1 bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory -e "select * from customers"'
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne       | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
```

Verify that PostgreSQL database has the same content
```shell
docker exec -it sinkdemo_postgres_1 bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
 Thomas    | 1001 | Sally      | sally.thomas@acme.com
 Bailey    | 1002 | George     | gbailey@foobar.com
 Walker    | 1003 | Edward     | ed@walker.com
 Kretchmar | 1004 | Anne       | annek@noanswer.org
(4 rows)
```

Verify that ElasticSearch has the same content
```shell
curl 'http://localhost:9200/customers/_search?pretty'
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 4,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "customers",
        "_type" : "kafka-connect",
        "_id" : "1001",
        "_score" : 1.0,
        "_source" : {
          "id" : 1001,
          "first_name" : "Sally",
          "last_name" : "Thomas",
          "email" : "sally.thomas@acme.com"
        }
      },
      {
        "_index" : "customers",
        "_type" : "kafka-connect",
        "_id" : "1004",
        "_score" : 1.0,
        "_source" : {
          "id" : 1004,
          "first_name" : "Anne",
          "last_name" : "Kretchmar",
          "email" : "annek@noanswer.org"
        }
      },
      {
        "_index" : "customers",
        "_type" : "kafka-connect",
        "_id" : "1002",
        "_score" : 1.0,
        "_source" : {
          "id" : 1002,
          "first_name" : "George",
          "last_name" : "Bailey",
          "email" : "gbailey@foobar.com"
        }
      },
      {
        "_index" : "customers",
        "_type" : "kafka-connect",
        "_id" : "1003",
        "_score" : 1.0,
        "_source" : {
          "id" : 1003,
          "first_name" : "Edward",
          "last_name" : "Walker",
          "email" : "ed@walker.com"
        }
      }
    ]
  }
}
```

Insert a new record into MySQL
```shell
docker exec -it sinkdemo_mysql_1 bash -c 'mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD inventory'
mysql> insert into customers values(null, 'Reinhard', 'Scheer', 'cic@hsf.de');
Query OK, 1 row affected (0.02 sec)
```

PostgreSQL contains a new record
```shell
docker exec -it sinkdemo_postgres_1 bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
...
Scheer    | 1005 | Reinhard   | cic@hsf.de
(5 rows)

```

ElasticSearch contains a new record
```shell
curl 'http://localhost:9200/customers/_search?pretty'
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 1.0,
    "hits" : [
...
      {
        "_index" : "customers",
        "_type" : "kafka-connect",
        "_id" : "1005",
        "_score" : 1.0,
        "_source" : {
          "id" : 1005,
          "first_name" : "Reinhard",
          "last_name" : "Scheer",
          "email" : "cic@hsf.de"
        }
      },

...
```

Update a record in MySQL
Insert a new record into MySQL
```shell
mysql> update customers set first_name='Franz', last_name='von Hipper' where last_name='Scheer';
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

Record in PostgreSQL is updated
```shell
docker exec -it sinkdemo_postgres_1 bash -c 'psql -U $POSTGRES_USER $POSTGRES_DB -c "select * from customers"'
 last_name |  id  | first_name |         email         
-----------+------+------------+-----------------------
...
von Hipper | 1005 | Franz      | cic@hsf.de
(5 rows)

```

ElasticSearch contains a new record
```shell
curl 'http://localhost:9200/customers/_search?pretty'
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 1.0,
    "hits" : [
...
      {
        "_index" : "customers",
        "_type" : "kafka-connect",
        "_id" : "1005",
        "_score" : 1.0,
        "_source" : {
          "id" : 1005,
          "first_name" : "Franz",
          "last_name" : "von Hipper",
          "email" : "cic@hsf.de"
        }
      },

...
```

End application
```shell
# Shut down the cluster
docker-compose down
```

