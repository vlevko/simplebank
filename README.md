# simplebank
My implementation of the Tech School's [backend master class](https://www.youtube.com/playlist?list=PLy_6D98if3ULEtXtNSY_2qN21VCKgoQAE) about designing, developing and deploying a complete backend system from scratch using PostgreSQL, Golang and Docker.

## 1. Design DB schema and generate SQL code with dbdiagram.io

Check the dbdiagram.io [schema](https://dbdiagram.io/d/644e30eadca9fb07c4452f97) described in DBML.

## 2. Install & use Docker + Postgres + TablePlus to create DB schema

Download and install [Docker](https://www.docker.com/).

Launch it and run the following commands using CLI:

```bash
docker pull postgres:15-alpine  # pull the PostgreSQL 15 with Alpine Linux from Docker Hub
```

```bash
docker images  # list all available docker images
```

```bash
docker run --name postgres15 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:15-alpine  # start a PostgreSQL server container
```

```bash
docker ps  # list all running containers
```

```bash
docker exec -it postgres15 psql -U root  # connect to PostgreSQL server and access its console
```

Inside the PostgreSQL console run simple queries:

```sql
select now();  -- get the current time
```

```sql
\q  -- quit the console
```

In the CLI run the command:

```bash
docker logs postgres15  # display the logs of the container
```

Download and install [TablePlus](https://tableplus.com/).

Open the app and create a new PostgreSQL connection entering its name e.g. `postgres15`, host `127.0.0.1`, port `5432`, user `root`, password `secret`, database `root`.

Click `Test` to check the connection and `Connect` to get access to the PostgreSQL server.

In TablePlus, click `SQL` icon, enter a simple query `select now();` and run it by clicking `Run Current` button.

Via the Main menu, open the PostgreSQL schema file from the previous lesson, select all queries and run them.

Reload the view by clicking `Circle arrow` icon, explore created tables and ensure via `Structure` tab of the selected table that all foreign keys are nullable.

Fix the above issue by adding the `not null` constraint to all foreign key fields in the dbdiagram.io schema and export the new SQL code.

Delete the recently created tables and repeat their creation by using the updated PostgreSQL schema.
