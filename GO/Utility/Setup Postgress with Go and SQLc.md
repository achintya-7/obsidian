A [[Utility]] doc to setup and use sqlc with Go and Gin

Containers required 
- PostgresSQL
- SQLc [For Windows]

```
docker run --name postgres -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres
docker pull kjconroy/sqlc
```

Create a mogration dir
```
mkdir -p db/migration
```

Create a Database
```
docker exec -it postgres createdb --username=root --owner=root db
```

Create migration
```
migrate create -ext sql -dir db/migration -seq schema1
```

Run migrations
```
migrate -path db/migration -database "postgresql://root:secret@localhost:5432/db?sslmode=disable" -verbose up
```

*If facing an issue related to auth, Change the port number in docker container OR Delete the postgres app in device*

Run `sqlc init` in the app directory and use this config
```yaml
version: "1"
packages:
  - name: "db"
    path: "./db/sqlc"
    queries: "./db/query/"
    schema: "./db/migration/"
    engine: "postgresql"
    emit_json_tags: true
    emit_prepared_queries: false
    emit_interface: false
    emit_exact_table_names: false
    emit_empty_slices: true
```

Write the Schema Up and Schema Down query

*UP*
```sql
Create Table "users" (
  "id" bigserial PRIMARY KEY,
  "name" varchar NOT NULL,
  "email" varchar NOT NULL
);
```
*If facing no cahnge issue, delete the migrations table*

*DOWN*
```sql
Drop Table If Exists "user";
```

Now write a bunch of sql queries in a new file under db/query dir
```sql
-- name: CreateUser :one
INSERT INTO users (
 name,
 email
) VALUES (
    $1, $2
) RETURNING *;
```

Now run sqlc generate
```
docker run --rm -v ${pwd}:/src -w /src kjconroy/sqlc generate
```

You will get a sqlc dir with all your queries in it

Lets write some tests 
In sqlc make `main_test.go`
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
	dbSource = "postgresql://root:secret@localhost:5432/db?sslmode=disable"
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

Make user_test.go and write the test
```go
package db

import (
	"context"
	"testing"

	"github.com/stretchr/testify/require"
)

func TestCreateUser(t *testing.T) {
	arg := CreateUserParams{
		Name:    "Achintya",
		Email:   "achintya@gmail.com",
	}

	user, err := testQueries.CreateUser(context.Background(), arg)
	
	require.NoError(t, err)
	require.NotEmpty(t, user)
	require.Equal(t, arg.Name, user.Name)
	require.Equal(t, arg.Email, user.Email)
	require.NotZero(t, user.ID)

}
```