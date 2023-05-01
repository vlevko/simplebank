# simplebank
My implementation of the Tech School's [backend master class](https://www.youtube.com/playlist?list=PLy_6D98if3ULEtXtNSY_2qN21VCKgoQAE) about designing, developing and deploying a complete backend system from scratch using PostgreSQL, Golang and Docker.

## Contents

[1. Design DB schema and generate SQL code with dbdiagram.io](#1)

[2. Install & use Docker + Postgres + TablePlus to create DB schema](#2)

[3. How to write & run database migration in Golang](#3)

## <a id="1"></a> 1. Design DB schema and generate SQL code with dbdiagram.io

Check the dbdiagram.io [schema](https://dbdiagram.io/d/644e30eadca9fb07c4452f97) described in DBML.

## <a id="2"></a> 2. Install & use Docker + Postgres + TablePlus to create DB schema

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

## <a id="3"></a> 3. How to write & run database migration in Golang

Install the Golang [migrate](https://github.com/golang-migrate/migrate) CLI tool.

Inside the `simplebank` project directory, create a new folder to store all migration files:

```bash
mkdir -p db/migration
```

Create the first migration file to initialize a simple bank's database schema:

```bash
migrate create -ext sql -dir db/migration -seq init_schema
```

Copy and paste all content from the `PostgreSQL schema.sql` into  `000001_init_schema.up.sql` files.

For reverting the changes made by the up script, enter these queries into the `000001_init_schema.down.sql` file:

```sql
DROP TABLE IF EXISTS entries;
DROP TABLE IF EXISTS transfers;
DROP TABLE IF EXISTS accounts;
```

Check if the PostgreSQL container is running:

```bash
docker ps -a  # list all containers
```

```bash
docker stop postgres15  # terminate running container
```

```bash
docker start postgres15  # run exited container
```

Get access to the PostgreSQL server shell:

```bash
docker exec -it postgres15 /bin/sh
```

Inside the shell, use spicified CLI commands to interact directly with the PostgreSQL server:

```bash
createdb --username=root --owner=root simple_bank  # create a new database for the project
```

```bash
psql simple_bank  # access the database console and quit it later with the \q command
```

```bash
dropdb simple_bank  # delete the database
```

Get out of the container shell with the `exit` command.

Outside of the PostgreSQL container, the above commands can be run with the `docker exec` command, for example:

```bash
docker exec -it postgres15 createdb --username=root --owner=root simple_bank

docker exec -it postgres15 psql -U root simple_bank
```

Remove container completely before using `make postgres` command later:

```bash
docker rm postgres15
```

Create and fill the `Makefile` in the project directory to run the above commands with the `make` utility later, and add the migration commands:

```Makefile
postgres:
	docker run --name postgres15 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:15-alpine

createdb:
	docker exec -it postgres15 createdb --username=root --owner=root simple_bank

dropdb:
	docker exec -it postgres15 dropdb simple_bank

migrateup:
	migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose up

migratedown:
	migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose down

.PHONY: postgres createdb dropdb migrateup migratedown
```
