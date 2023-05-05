# simplebank
My implementation of the Tech School's [backend master class](https://www.youtube.com/playlist?list=PLy_6D98if3ULEtXtNSY_2qN21VCKgoQAE) about designing, developing and deploying a complete backend system from scratch using PostgreSQL, Golang and Docker.

## Contents

[1. Design DB schema and generate SQL code with dbdiagram.io](#1)

[2. Install & use Docker + Postgres + TablePlus to create DB schema](#2)

[3. How to write & run database migration in Golang](#3)

[4. Generate CRUD Golang code from SQL | Compare db/sql, gorm, sqlx & sqlc](#4)

[5. Write Golang unit tests for database CRUD with random data](#5)

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

## <a id="4"></a> 4. Generate CRUD Golang code from SQL | Compare db/sql, gorm, sqlx & sqlc

DATABASE/SQL:
- Very fast & straightforward
- Manual mapping SQL fields to variables
- Easy to make mistakes, not caught until runtime

GORM:
- CRUD functions already implemented, very short production code
- Must learn to write queries using gorm's function
- Run slowly on high load

SQLX:
- Quite fast & easy to use
- Fields mapping via query text & struct tags
- Failure won't occur until runtime

SQLC:
- Very fast & easy to use
- Automatic code generation
- Catch SQL query errors before generating codes
- Full support PostgreSQL & MySQL

Install the [sqlc](https://sqlc.dev/) tool.

Inside the project directory, run `sqlc init` to create an empty `sqlc.yaml` settings file.

Create the `sqlc` and `query` folders inside the `db` directory:

```bash
mkdir db/sqlc

mkdir db/query
```

Copy and paste the [sample](https://docs.sqlc.dev/en/stable/reference/config.html#version-1) settings into the `sqlc.yaml` file and set these ones to:

```yaml
path: "./db/sqlc"
queries: "./db/query/"
schema: "./db/migration/"
emit_prepared_queries: false
json_tags_case_style: "snake"
```

Remove the `output_batch_file_name: "batch.go"` setting.

Add a new `sqlc` command equivalent to the current OS into `Makefile`:

```Makefile
sqlc:
	sqlc generate

.PHONY: ... sqlc
```

Create the `account.sql` file inside `db/query` folder and enter the following code:

```sql
-- name: CreateAccount :one
INSERT INTO accounts (
  owner,
  balance,
  currency
) VALUES (
  $1, $2, $3
) RETURNING *;
```

Run `make sqlc` command to generate the Go code and explore the generated code in the `db/sqlc` folder.

Append the following code to the `db/query/account.sql` file:

```sql
-- name: GetAccount :one
SELECT * FROM accounts
WHERE id = $1 LIMIT 1;

-- name: ListAccounts :many
SELECT * FROM accounts
ORDER BY id
LIMIT $1
OFFSET $2;
```

Regenerate the Go code with the `make sqlc` command and explore the `db/sqlc/account.sql.go` file.

Append the update account code to the `db/query/account.sql` file:

```sql
-- name: UpdateAccount :one
UPDATE accounts
SET balance = $2
WHERE id = $1
RETURNING *;
```

Run the `make sqlc` command to get the new generated Go code.

Append the delete account code to the `db/query/account.sql` file:

```sql
-- name: DeleteAccount :exec
DELETE FROM accounts
WHERE id = $1;
```

Regenerate the Go code with the `make sqlc` command.

Do the similar thing to the remaining tables `entries` and `transfers`.

Additionally create the `db/schema` folder and move related files into it.

## <a id="5"></a> 5. Write Golang unit tests for database CRUD with random data

Install the PostgreSQL [driver](https://github.com/lib/pq) in order to connect to the database:

```bash
go get github.com/lib/pq
```

Setup the database connection and the Queries object in a new `main_test.go` file inside the `db/sqlc` folder:

```go
package db

import (
	"database/sql"
	"log"
	"os"
	"testing"

	_ "github.com/lib/pq"
)

const (
	dbDriver = "postgres"
	dbSource = "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable"
)

var testQueries *Queries

func TestMain(m *testing.M) {
	conn, err := sql.Open(dbDriver, dbSource)
	if err != nil {
		log.Fatal("cannot connect to db:", err)
	}

	testQueries = New(conn)

	os.Exit(m.Run())
}
```

Run `go mod tidy` inside the project directory to clean up the dependencies.

Install the [testify](https://github.com/stretchr/testify) package to simplify writing unit tests:

```bash
go get github.com/stretchr/testify
```

Create a new `util` folder in the project directory:

```bash
mkdir util
```

Inside the `util` folder, create a `random.go` file to generate the test data:

```go
package util

import (
	"math/rand"
	"strings"
	"time"
)

const alphabet = "abcdefghijklmnopqrstuvwxyz"

func init() {
	rand.Seed(time.Now().UnixNano())
}

// RandomInt generates a random integer between min and max
func RandomInt(min, max int64) int64 {
	return min + rand.Int63n(max-min+1)
}

// RandomString generates a random string of length n
func RandomString(n int) string {
	var sb strings.Builder
	k := len(alphabet)

	for i := 0; i < n; i++ {
		c := alphabet[rand.Intn(k)]
		sb.WriteByte(c)
	}

	return sb.String()
}

// RandomOwner generates a random owner name
func RandomOwner() string {
	return RandomString(6)
}

// RandomMoney generates a random amount of money
func RandomMoney() int64 {
	return RandomInt(0, 1000)
}

// RandomCurrency generates a random currency code
func RandomCurrency() string {
	currencies := []string{"EUR", "USD", "CAD"}
	n := len(currencies)
	return currencies[rand.Intn(n)]
}
```

Create a new `account_test.go` file inside the `db/sqlc` folder and write a unit test for the `CreateAccount` function: 

```go
package db

import (
	"context"
	"testing"

	"github.com/stretchr/testify/require"
	"github.com/vlevko/simplebank/util"
)

func TestCreateAccount(t *testing.T) {
	arg := CreateAccountParams{
		Owner:    util.RandomOwner(),
		Balance:  util.RandomMoney(),
		Currency: util.RandomCurrency(),
	}

	account, err := testQueries.CreateAccount(context.Background(), arg)
	require.NoError(t, err)
	require.NotEmpty(t, account)

	require.Equal(t, arg.Owner, account.Owner)
	require.Equal(t, arg.Balance, account.Balance)
	require.Equal(t, arg.Currency, account.Currency)

	require.NotZero(t, account.ID)
	require.NotZero(t, account.CreatedAt)
}
```

Add a new `test` command to the `Makefile`:

```Makefile
test:
	go test -v -cover ./...

.PHONY: ... test
```

Write unit tests for the rest of the CRUD operations of `accounts`, `entries` and `transfers` tables.
