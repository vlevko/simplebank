# simplebank
My implementation of the Tech School's [backend master class](https://www.youtube.com/playlist?list=PLy_6D98if3ULEtXtNSY_2qN21VCKgoQAE) about designing, developing and deploying a complete backend system from scratch using PostgreSQL, Golang and Docker.

## Contents

[1. Design DB schema and generate SQL code with dbdiagram.io](#1)

[2. Install & use Docker + Postgres + TablePlus to create DB schema](#2)

[3. How to write & run database migration in Golang](#3)

[4. Generate CRUD Golang code from SQL | Compare db/sql, gorm, sqlx & sqlc](#4)

[5. Write Golang unit tests for database CRUD with random data](#5)

[6. A clean way to implement database transaction in Golang](#6)

[7. DB transaction lock & How to handle deadlock in Golang](#7)

[8. How to avoid deadlock in DB transaction? Queries order matters!](#8)

[9. Understand isolation levels & read phenomena in MySQL & PostgreSQL via examples](#9)

[10. Setup Github Actions for Golang + Postgres to run automated tests](#10)

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

## <a id="6"></a> 6. A clean way to implement database transaction in Golang

What is a db transaction?

> A single unit of work that's often made up of multiple db operations.

Why do we need db transaction?

> 1\. To provide a reliable and consistent unit of work, even in case of system failure.\
> 2\. To provide isolation between programs that access the database concurrently.

ACID property:

- Atomacity (A)\
Either all operations complete successfully or the transaction fails and the db is unchanged.
- Consistency (C)\
The db state must be valid after the transaction. All constraints must be satisfied.
- Isolation (I)\
Concurrent transactions must not affect each other.
- Durability (D)\
Data written by a successful transaction must be recorded in persistent storage.

Create a new file `store.go` inside the `db/sqlc` folder:

```go
package db

import (
	"context"
	"database/sql"
	"fmt"
)

// Store provides all functions to execute db queries and transactions
type Store struct {
	*Queries
	db *sql.DB
}

// NewStore creates a new Store
func NewStore(db *sql.DB) *Store {
	return &Store{
		db:      db,
		Queries: New(db),
	}
}

// execTx executes a function within a database transaction
func (store *Store) execTx(ctx context.Context, fn func(*Queries) error) error {
	tx, err := store.db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}

	q := New(tx)
	err = fn(q)
	if err != nil {
		if rbErr := tx.Rollback(); rbErr != nil {
			return fmt.Errorf("tx err: %v, rb err: %v", err, rbErr)
		}
		return err
	}

	return tx.Commit()
}

// TransferTxParams contains the input parameters of the transfer transaction
type TransferTxParams struct {
	FromAccountID int64 `json:"from_account_id"`
	ToAccountID   int64 `json:"to_account_id"`
	Amount        int64 `json:"amount"`
}

// TransferTxResult is the result of the transfer transaction
type TransferTxResult struct {
	Transfer    Transfer `json:"transfer"`
	FromAccount Account  `json:"from_account"`
	ToAccount   Account  `json:"to_account"`
	FromEntry   Entry    `json:"from_entry"`
	ToEntry     Entry    `json:"to_entry"`
}

// TransferTx performs a money transfer from one account to the other.
// It creates a transfer record, adds account entries, and updates accounts' balance whithin a single database transaction
func (store *Store) TransferTx(ctx context.Context, arg TransferTxParams) (TransferTxResult, error) {
	var result TransferTxResult

	err := store.execTx(ctx, func(q *Queries) error {
		var err error

		result.Transfer, err = q.CreateTransfer(ctx, CreateTransferParams{
			FromAccountID: arg.FromAccountID,
			ToAccountID:   arg.ToAccountID,
			Amount:        arg.Amount,
		})
		if err != nil {
			return err
		}

		result.FromEntry, err = q.CreateEntry(ctx, CreateEntryParams{
			AccountID: arg.FromAccountID,
			Amount:    -arg.Amount,
		})
		if err != nil {
			return err
		}

		result.ToEntry, err = q.CreateEntry(ctx, CreateEntryParams{
			AccountID: arg.ToAccountID,
			Amount:    arg.Amount,
		})
		if err != nil {
			return err
		}

		// TODO: update accounts' balance

		return nil
	})

	return result, err
}
```

Creaet a new `store_test.go` file in the same folder to test the `TransferTx` function:

```go
package db

import (
	"context"
	"testing"

	"github.com/stretchr/testify/require"
)

func TestTransferTx(t *testing.T) {
	store := NewStore(testDB)

	account1 := createRandomAccount(t)
	account2 := createRandomAccount(t)

	// run n concurrent transfer transactions
	n := 5
	amount := int64(10)

	errs := make(chan error)
	results := make(chan TransferTxResult)

	for i := 0; i < n; i++ {
		go func() {
			result, err := store.TransferTx(context.Background(), TransferTxParams{
				FromAccountID: account1.ID,
				ToAccountID:   account2.ID,
				Amount:        amount,
			})

			errs <- err
			results <- result
		}()
	}

	// check results
	for i := 0; i < n; i++ {
		err := <-errs
		require.NoError(t, err)

		result := <-results
		require.NotEmpty(t, result)

		// check transfer
		transfer := result.Transfer
		require.NotEmpty(t, transfer)
		require.Equal(t, account1.ID, transfer.FromAccountID)
		require.Equal(t, account2.ID, transfer.ToAccountID)
		require.Equal(t, amount, transfer.Amount)
		require.NotZero(t, transfer.ID)
		require.NotZero(t, transfer.CreatedAt)

		_, err = store.GetTransfer(context.Background(), transfer.ID)
		require.NoError(t, err)

		// check entries
		fromEntry := result.FromEntry
		require.NotEmpty(t, fromEntry)
		require.Equal(t, account1.ID, fromEntry.AccountID)
		require.Equal(t, -amount, fromEntry.Amount)
		require.NotZero(t, fromEntry.ID)
		require.NotZero(t, fromEntry.CreatedAt)

		_, err = store.GetEntry(context.Background(), fromEntry.ID)
		require.NoError(t, err)

		toEntry := result.ToEntry
		require.NotEmpty(t, toEntry)
		require.Equal(t, account2.ID, toEntry.AccountID)
		require.Equal(t, amount, toEntry.Amount)
		require.NotZero(t, toEntry.ID)
		require.NotZero(t, toEntry.CreatedAt)

		_, err = store.GetEntry(context.Background(), toEntry.ID)
		require.NoError(t, err)

		// TODO: check accounts' balance
	}
}
```

Make the necessary changes in the `main_test.go` file:

```go
...
var testDB *sql.DB

func TestMain(m *testing.M) {
	var err error
	testDB, err = sql.Open(dbDriver, dbSource)
	...
	testQueries = New(testDB)
	...
}
```

## <a id="7"></a> 7. DB transaction lock & How to handle deadlock in Golang

Extend the `store_test.go` file with the testing of `accounts` table by applying a Test Driven Development (TDD) approach:

```go
...
import (
	"fmt"
	...
)
...
func TestTransferTx(t *testing.T) {
	...
	fmt.Println(">> before:", account1.Balance, account2.Balance)

	// run n concurrent transfer transactions
	...
	// check results
	existed := make(map[int]bool)

	for i := 0; i < n; i++ {
		...
		// check accounts
		fromAccount := result.FromAccount
		require.NotEmpty(t, fromAccount)
		require.Equal(t, account1.ID, fromAccount.ID)

		toAccount := result.ToAccount
		require.NotEmpty(t, toAccount)
		require.Equal(t, account2.ID, toAccount.ID)

		// check accounts' balance
		fmt.Println(">> tx:", fromAccount.Balance, toAccount.Balance)
		diff1 := account1.Balance - fromAccount.Balance
		diff2 := toAccount.Balance - account2.Balance
		require.Equal(t, diff1, diff2)
		require.True(t, diff1 > 0)
		require.True(t, diff1%amount == 0) // 1 * amount, 2 * amount, 3 * amount, ..., n * amount

		k := int(diff1 / amount)
		require.True(t, k >= 1 && k <= n)
		require.NotContains(t, existed, k)
		existed[k] = true
	}

	// check the final updated balances
	updatedAccount1, err := testQueries.GetAccount(context.Background(), account1.ID)
	require.NoError(t, err)

	updatedAccount2, err := testQueries.GetAccount(context.Background(), account2.ID)
	require.NoError(t, err)

	fmt.Println(">> after:", updatedAccount1, updatedAccount2)
	require.Equal(t, account1.Balance-int64(n)*amount, updatedAccount1.Balance)
	require.Equal(t, account2.Balance+int64(n)*amount, updatedAccount2.Balance)
}
```

Implement the updating accounts' balance feature in the `store.go` file:

```go
...
func (store *Store) TransferTx(ctx context.Context, arg TransferTxParams) (TransferTxResult, error) {
	...
	err := store.execTx(ctx, func(q *Queries) error {
		...
		result.FromAccount, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
			ID:     arg.FromAccountID,
			Amount: -arg.Amount,
		})
		if err != nil {
			return err
		}

		result.ToAccount, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
			ID:     arg.ToAccountID,
			Amount: arg.Amount,
		})
		if err != nil {
			return err
		}

		return nil
	})

	return result, err
}
```

Update the `account.sql` file inside the `db/query` folder by adding a new query to get account for update:
```sql
...
-- name: GetAccountForUpdate :one
SELECT * FROM accounts
WHERE id = $1 LIMIT 1
FOR NO KEY UPDATE;

-- name: ListAccounts :many
...
-- name: AddAccountBalance :one
UPDATE accounts
SET balance = balance + sqlc.arg(amount)
WHERE id = sqlc.arg(id)
RETURNING *;

-- name: DeleteAccount :exec
...
```

Regenerate the code by running the `make sqlc` command.

## <a id="8"></a> 8. How to avoid deadlock in DB transaction? Queries order matters!

The best way to deal with deadlock is to avoid it.

In the `store_test.go` file inside the `db/sqlc` folder add a new `TestTransferTxDeadlock` function to show how a deadlock error can occur:

```go
func TestTransferTxDeadlock(t *testing.T) {
	store := NewStore(testDB)

	account1 := createRandomAccount(t)
	account2 := createRandomAccount(t)
	fmt.Println(">> before:", account1.Balance, account2.Balance)

	// run n concurrent transfer transactions
	n := 10
	amount := int64(10)
	errs := make(chan error)

	for i := 0; i < n; i++ {
		fromAccountID := account1.ID
		toAccountID := account2.ID

		if i%2 == 1 {
			fromAccountID = account2.ID
			toAccountID = account1.ID
		}

		go func() {
			_, err := store.TransferTx(context.Background(), TransferTxParams{
				FromAccountID: fromAccountID,
				ToAccountID:   toAccountID,
				Amount:        amount,
			})

			errs <- err
		}()
	}

	// check results
	for i := 0; i < n; i++ {
		err := <-errs
		require.NoError(t, err)
	}

	// check the final updated balances
	updatedAccount1, err := testQueries.GetAccount(context.Background(), account1.ID)
	require.NoError(t, err)

	updatedAccount2, err := testQueries.GetAccount(context.Background(), account2.ID)
	require.NoError(t, err)

	fmt.Println(">> after:", updatedAccount1, updatedAccount2)
	require.Equal(t, account1.Balance, updatedAccount1.Balance)
	require.Equal(t, account2.Balance, updatedAccount2.Balance)
}
```

Avoid the deadlock error by making the following changes in the `store.go` file inside the `db/sqlc` folder so that the application always acquires locks in a consistent order, in particular so that it always updates the account with smaller ID first:

```go
...
func (store *Store) TransferTx(ctx context.Context, arg TransferTxParams) (TransferTxResult, error) {
	...
	err := store.execTx(ctx, func(q *Queries) error {
		...
		result.ToEntry, err = q.CreateEntry(ctx, CreateEntryParams{
			AccountID: arg.ToAccountID,
			Amount:    arg.Amount,
		})
		if err != nil {
			return err
		}
		
		if arg.FromAccountID < arg.ToAccountID {
			result.FromAccount, result.ToAccount, err = addMoney(ctx, q, arg.FromAccountID, -arg.Amount, arg.ToAccountID, arg.Amount)
		} else {
			result.ToAccount, result.FromAccount, err = addMoney(ctx, q, arg.ToAccountID, arg.Amount, arg.FromAccountID, -arg.Amount)
		}

		return nil
	})

	return result, err
}

func addMoney(
	ctx context.Context,
	q *Queries,
	accountID1 int64,
	amount1 int64,
	accountID2 int64,
	amount2 int64
) (account1 Account, account2 Account, err error) {
	account1, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
		ID: accountID1,
		Amount: amount1,
	})
	if err != nil {
		return
	}

	account2, err = q.AddAccountBalance(ctx, AddAccountBalanceParams{
		ID: accountID2,
		Amount: amount2,
	})
	return
}
```

## <a id="9"></a> 9. Understand isolation levels & read phenomena in MySQL & PostgreSQL via examples

Isolation is a part of the ACID property (along with Atomacity, Consistency and Durability), which must be satisfied by a database transaction, meaning that concurrent transactions must not affect each other.

Read Phenomena:
- DIRTY READ\
A transaction **reads** data written by other concurrent **uncommitted** transaction
- NON-REPEATABLE READ\
A transaction **reads** the **same row twice** and sees different value because it has been **modified** by other **committed** transaction
- PHANTOM READ\
A transaction **re-executes** a query to **find rows** that satisfy a condition and sees a **different set** of rows, due to changes by other **commited** transaction
- SERIALIZATION ANOMALY\
The result of a **group** of concurrent **committed transactions** is **impossible to achieve** if we try to run them **sequentially** in any order without overlapping

4 Standard Isolation Level according to American National Standards Institute (ANSI):
1. READ UNCOMMITTED\
Can see data written by uncommitted transaction
2. READ COMMITTED\
Only see data written by committed transaction
3. REPEATABLE READ\
Sama read query always returns same result
4. SERIALIZABLE\
Can achieve same result if execute transactions serially in some order instead of concurrenctly

### Isolation Levels in MySQL

|                           | READ<br/>UNCOMMITTED | READ<br/>COMMITTED | REPEATABLE<br/>READ | SERIALIZABLE |
|:-------------------------:|:--------------------:|:------------------:|:-------------------:|:------------:|
| DIRTY<br/>READ            | +                    | -                  | -                   | -            |
| <nobr>NON-REPEATABLE<nobr/><br/>READ | +         | +                  | -                   | -            |
| PHANTOM<br/>READ          | +                    | +                  | -                   | -            |
| SERIALIZATION<br/>ANOMALY | +                    | +                  | +                   | -            |

To get the transaction isolation level of the current session run inside the database console:

```sql
select @@transaction_isolation;  -- default: REPEATABLE-READ
```

To get the global transaction isolation level applied to all sessions:

```sql
select @@global.transaction_isolation;  -- default: REPEATABLE-READ
```

To change the isolation level of current session:

```sql
set session transaction isolation level read uncommitted;  -- or: "read committed", "repeatable read", "serializable"
```

To start a new transaction:

```sql
start transaction;
```

or:

```sql
begin;
```

To commit changes of the current transaction:

```sql
commit;
```

To cancel changes made by the current transaction:

```sql
rollback;
```

### Isolation Levels in PostgreSQL

|                           | READ<br/>UNCOMMITTED | READ<br/>COMMITTED | REPEATABLE<br/>READ | SERIALIZABLE |
|:-------------------------:|:--------------------:|:------------------:|:-------------------:|:------------:|
| DIRTY<br/>READ            | -                    | -                  | -                   | -            |
| <nobr>NON-REPEATABLE<nobr/><br/>READ | +         | +                  | -                   | -            |
| PHANTOM<br/>READ          | +                    | +                  | -                   | -            |
| SERIALIZATION<br/>ANOMALY | +                    | +                  | +                   | -            |

To get the current isolation level run inside the database console:

```sql
show transaction isolation level;  -- default: read committed
```

To set the isolation level within the current transaction:

```sql
begin;

set transaction isolation level read uncommitted;  -- or: "read committed", "repeatable read", "serializable"

-- "commit" or "rollback" this transaction later
```

### Compare MySQL vs PostgreSQL

| MySQL                     | PostgreSQL               |
|:--------------------------|:-------------------------|
| 4 ISOLATION LEVELS        | 3 ISOLATION LEVELS       |
| LOCKING MECHANISM         | DEPENDENCIES DETECTION   |
|REPEATABLE READ BY DEFAULT | READ COMMITED BY DEFAULT |

Keep in mind:
- RETRY MECHANISM\
There might be errors, timeout or deadlock
- READ DOCUMENTATION\
Each database engine might implement isolation level differently

## <a id="10"></a> 10. Setup Github Actions for Golang + Postgres to run automated tests

Workflow:
- is an automated procedure
- made up of 1+ jobs
- triggered by events, scheduled, or manually
- add .yml file to repository

Runner:
- is a server to run the jobs
- run 1 job at a time
- GitHub hosted or self hosted
- report progress, logs & result to GitHub

Job:
- is a set of steps execute on the same runner
- normal jobs run in parallel
- dependent jobs run serially

Step:
- is an individual task
- run serially within a job
- contain 1+ actions

Action:
- is a standalone command
- run serially within a step
- can be reused

Within the project directory, create a new folder `.github/workflows`:

```bash
mkdir -p .github/workflows
```

Create a new YAML file `ci.yml` for the workflow inside this folder:

```yaml
name: ci-test

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:

  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: root
          POSTGRES_PASSWORD: secret
          POSTGRES_DB: simple_bank
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:

    - name: Check out code into the Go module directory
      uses: actions/checkout@v3

    - name: Set up Go
      uses: actions/setup-go@v3
      with:
        go-version: ^1.20
      id: go

    - name: Install golang-migrate
      run: |
        curl -L https://github.com/golang-migrate/migrate/releases/download/v4.15.2/migrate.linux-amd64.tar.gz | tar xvz
        sudo mv migrate /usr/bin/
        which migrate

    - name: Run migrations
      run: make migrateup

    - name: Test
      run: make test
```
