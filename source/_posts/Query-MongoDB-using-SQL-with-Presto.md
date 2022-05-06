---
title: Query MongoDB using SQL with Presto
date: 2019-11-17 17:00:00
tags: [sql, mongodb, presto, metabase, docker]
---

With an increasing number of specialized databases, each having their own query languages, data analysts have a hard time to combine data from multiples sources. To mitigate this issue, Facebook created [Presto](https://prestosql.io), a high performance, distributed SQL query engine for big data. I will be creating a small simulation using [Metabase](https://www.metabase.com), a web based open-source BI solution to visualize data from [MongoDB](https://www.mongodb.com).

## Docker

[Docker](https://www.docker.com) is a container platform. Containers package software applications and all their dependencies, which helps enable reproducible infrastructure. [Docker Hub](https://hub.docker.com) contains many container images so we have a starting point and don't have to package everything ourselves. This is extremely helpful, specially during development.

Each container should only have one function. For example, a simple scenario when doing back-end development is to use a container for the application and another for the database. To connect the containers in a local environment, Docker Compose is the easiest solution but for production, the recommended approach is to use an orchestration system like [Kubernetes](https://kubernetes.io).

## Presto

Following the rise of NoSQL databases, many specialized query languages exist today. This leads to huge data integration efforts and expensive ETL processes. In the Hadoop world, a number of solutions emerged to enable the usage of SQL to retrieve data, Presto being the most interesting in my opinion.

The main advantage of Presto is that it has many data connectors, such as Kafka, Cassandra, Elasticsearch, MongoDB, Postgres, etc... It is then able to infer the schema automatically and handle semi-structured data. Arrays, nested objects, multi-database joins and the regular SQL operations are all supported.

[Apache Drill](https://drill.apache.org) is a very similar alternative to Presto, however it appears to be less popular and doesn't support as many data sources. Performance comparisons are out of the scope for this post, but Presto is used by big players in data-intensive environments. Amazon created [Athena](https://aws.amazon.com/athena) which is based on Presto and heavily integrated in AWS. There is another recent popular Presto fork called [Trino](https://trino.io).

## Metabase

There is a lot of commercial Business Intelligence software out there. Metabase stands out for being an open-source BI technology that is easy to use even for people that don't know SQL, allowing them to explore the data and create web dashboards. Some advanced visualizations still require SQL knowledge.

Other alternatives include [Redash](https://redash.io) and [Superset](https://superset.incubator.apache.org). From my research, these support more chart types but are less user friendly and deployment is a bit more complicated.

## MongoDB

MongoDB is a document-oriented database. Documents are stored as JSON objects, making it a good choice for semi-structured data with a flexible schema. It does not support SQL out of the box, which makes it harder for analysts to extract data because they have to learn another query language.

## Simulation

I created 4 containers, 2 for different databases, 1 for Presto and the last for Metabase. Application data should be stored in volumes. Configuration files can either be copied and stored as part of the image or we can follow the volume approach. On Windows, named volumes must be used in some cases because of file permission issues.

```yaml docker-compose.yml
version: '3.4'

services:
  mongo:
    build: ./mongo
    container_name: mongo
    volumes:
      - mongo-data:/data/db
  postgres:
    build: ./postgres
    container_name: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret123
  presto:
    build: ./presto
    container_name: presto
    ports:
      - "8080:8080"
  metabase:
    image: metabase/metabase:v0.33.4
    container_name: metabase
    volumes:
      - ./data/metabase:/metabase-data
    ports:
      - "8000:3000"
    environment:
      - MB_DB_FILE=/metabase-data/metabase.db

  
volumes:
  mongo-data:
    name: mongo-data
  postgres-data:
    name: postgres-data
```

I'm including here some commands that I used for testing. [JSON Generator](https://www.json-generator.com) is a very nice tool to create datasets. For more details about how to seed the databases and configure Presto, check the repository.

```sql
docker exec -it mongo bash

# mongo
mongo
show dbs
use testdb
show collections
db.orders.count()
db.users.find().limit(2).pretty()

# postgres
su postgres
psql
\l  # list dbs
\c locationdb # connect to locationdb
select * from information_schema.schemata; # list all database schemas
select * from pg_tables where schemaname='userlocation'; # list all tables from schema
select * from userlocation.city;

# presto
presto
show catalogs;
show schemas from mongo;
show tables from mongo.testdb; 
describe mongo.testdb.orders;

# select all orders, including the mongo _id
select _id, * from mongo.testdb.orders limit 5;

# number of orders
select count(*) from mongo.testdb.orders;

# number of items from a certain order
select cardinality(items) from mongo.testdb.orders where id = 's1';

# average number of items from shooping cart
select avg(cardinality(items)) from mongo.testdb.orders;

# total number of items by user
select userId, sum(cardinality(items)) n from mongo.testdb.orders group by userId order by -n;

# orders between 2 dates, for some reason have to add an OR condition for it to work correctly, maybe a bug?
select _id, id, "when", year("when") y, month("when") m, day("when") d from mongo.testdb.orders where _id is null or "when" between timestamp '2019-10-28' and timestamp '2019-11-02';

# unnest objects in arrays
select id, userId, "when", name, price, quantity from mongo.testdb.orders CROSS JOIN UNNEST(items) AS t (name, price, quantity) where name like '%sic%' limit 5;

# total money spent by users
select sum(price * quantity) total from mongo.testdb.orders CROSS JOIN UNNEST(items) AS t (name, price, quantity);

# select unique users that had orders and join info with other collection and table in other database
select distinct(userId), name, age, location, country from mongo.testdb.orders o left join mongo.testdb.users u on o.userId = u.id left join postgres.userlocation.city l on u.location = l.city_name;

# location info for cities that have users - subqueries demo
select * from postgres.userlocation.city where city_name in (select location from mongo.testdb.users);

# metabase - number of orders by month
SELECT date_trunc('month', "testdb"."orders"."when") AS "when", sum(price * quantity) AS "total"
FROM "testdb"."orders"
CROSS JOIN UNNEST(items) AS t (name, price, quantity)
GROUP BY date_trunc('month', "testdb"."orders"."when")
ORDER BY date_trunc('month', "testdb"."orders"."when") ASC

# copy data from named volume to host
docker run -v postgres-data:/volumedata --name test --rm -it alpine sh
docker cp test:/volumedata ./mydata

# stop containers and remove a volume
docker container prune
docker volume rm postgres-data
```

### Metabase setup

Unlike the other tools, on Metabase there is no way to import/export dashboard configurations, other than copying the database it uses, which is H2 by default. For that reason I have included the data folder in the project repository.

{% asset_img "metabase-setup.png" "Metabase setup" %}

### Presto UI

It is possible to see the queries that are executed by the Presto cluster. A distributed system is required to handle large amounts of data.

{% asset_img "presto-ui.png" "Presto UI" %}

### Dashboard

This final dashboard can be obtained by navigating to *localhost:8000* after running *docker-compose up*. During the first manual setup I chose *test\[at\]example.com* as the username and *test1234* for the password. Ideally this should be a configuration of the image or automatically setup when the container starts.

{% asset_img "dashboard-example.png" "Simulation dashboard" %}

## Closing thoughts

I recently had to create a web dashboard that needed data from multiple databases and these technologies allowed me to quickly build a quality prototype. Feel free to check the [repository](https://github.com/ruial/presto-simulation).
