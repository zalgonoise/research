[back](boltdb.md) | [index](index.md)

### SQLite implementation

The SQLite DB implementation will be placed in a top-level folder within the project, named `sqlite`. This folder will contain a `sqlite.go` file to initialize the DB instance, and 3 files for the 3 repositories it implements. Besides these, it will also have a `helper.go` file to aid with reusable functions, and a migrations folder for database migrations.

Note that this implementation is not leveraging [`migrate`](https://github.com/golang-migrate/migrate) which is an awesome library to manage SQL migrations in Go.

The `helper.go` file exposes types that are used to narrow-down the methods used for a particular action (usually the `Context` variants of SQL method calls).

```
.
└─ sqlite
    ├─ migrations
    │    └─ 1672703190_initial_up.sql -- initial migration to create the database tables
    ├─ helper.go -- contains reusable functions and types used for SQLite transactions  
    ├─ secrets.go -- implements the secret.Repository interface
    ├─ shared.go -- implements the shared.Repository interface 
    ├─ sqlite.go -- exposes function(s) to initialize a SQLite DB instance
    └─ users.go -- implements the user.Repository interface
```

#### Defining the initial migration

The initial migration file will create the tables as described in the [Database Design section](#database-design), with the `IF NOT EXISTS` clause.

This is the chance to define the unique fields which weren't yet set at first, to limit the relations between the objects within a set of boundaries. Take a look at the migration query first:

```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    name TEXT NOT NULL,
    hash TEXT NOT NULL,
    salt TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS secrets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    name VARCHAR(250) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users (id),
    UNIQUE(user_id, name)
);

CREATE TABLE IF NOT EXISTS shared_secrets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    owner_id INTEGER NOT NULL,
    secret_id INTEGER NOT NULL,
    shared_with INTEGER NOT NULL,
    until TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (owner_id) REFERENCES users (id),
    FOREIGN KEY (secret_id) REFERENCES secrets (id),
    FOREIGN KEY (shared_with) REFERENCES users (id),
    UNIQUE(secret_id, shared_with)
);
```

The migration above sets the following `UNIQUE` constraints:
- `secrets`: secrets have unique keys per user. Means that the database does not allow `joe` to store a secret with key `access` when `access` already exists. The entry needs to be removed before added again
- `shared_secrets`: secrets are unique per (target) user when shared. Means that the database does not allow sharing a secret with key `access` with user `joe` if such a share already exists (scoped with time for example). The entry needs to be removed before added again

As for `FOREIGN KEY` constraints:
- `secrets`: its `user_id` field will reference the `users.id` field
- `shared_secrets`: its `owner_id` field will reference the `users.id` field
- `shared_secrets`: its `secret_id` field will reference the `secrets.id` field
- `shared_secrets`: its `shared_with` field will reference the `users.id` field

As for the unique constraints, the service layer is responsible for fetching the data before mutating it -- so if a user tries to overwrite a secret (which is allowed), the service will first fetch it; delete it; then create it.

#### Initializing a SQLite DB instance

Initializing a SQLite DB instance simply takes a path to a file. This function is calling `sql.Open()`, and running the initial migration as set into `initialMigration` using `go:embed`

This implementation uses [`mattn/go-sqlite3`](https://github.com/mattn/go-sqlite3) as a SQLite driver.

```go
package sqlite

import (
	"database/sql"

	_ "embed"

	_ "github.com/mattn/go-sqlite3"
)

//go:embed migrations/1672703190_initial_up.sql
var initialMigration string

// Open will initialize a SQLite DB based on the `.sql` file in `path`,
// returning a pointer to a sql.DB and an error
//
// It executes the initial migration, as well.
func Open(path string) (*sql.DB, error) {
	db, err := sql.Open("sqlite3", path)
	if err != nil {
		return nil, err
	}

	_, err = db.Exec(initialMigration)
	if err != nil {
		return nil, err
	}
	return db, nil
}
```

#### Defining helper functions

Of course, most of these come up when you follow the same pattern more than once, be it in the same projects or over time. Some of these functions are added to the file as the logic in it is reusable across repository implementations, or simply to avoid clogging up the implemenation files. This isn't a file I start with when designing a SQL-based persistence layer, but often emerges over time when I do.

Continuing with the contents of this file since they are referenced through the implementations, for better context later:

#### SQL Querier types

```go
import (
	"context"
	"database/sql"
)

// Scanner describes an object that scans values into destination pointers
type Scanner interface {
	Scan(dest ...interface{}) error
}

// RowQuerier describes an object that queries a single row in a SQL database
type RowQuerier interface {
	QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
}

// RowsQuerier describes an object that queries multiple rows in a SQL database
type RowsQuerier interface {
	QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
}

// Executer describes an object that executes a mutation in a SQL database
type Executer interface {
	ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
}

// Querier encapsulates a RowQuerier, RowsQuerier and Executer implementations
type Querier interface {
	RowQuerier
	RowsQuerier
	Executer
}
```

These types narrow down the methods used when interacting with the database, often picking their `Context` method variants; often used as input parameters and returns, in functions and methods.


#### SQL-type converters

```go
import (
	"database/sql"
	"time"

	"golang.org/x/exp/constraints"
)

// ToSQLString converts the input string into a sql.NullString
func ToSQLString(s string) sql.NullString {
	return sql.NullString{String: s, Valid: s != ""}
}

// ToSQLInt64 converts the generic integer into a sql.NullInt64
func ToSQLInt64[T constraints.Integer](v T) sql.NullInt64 {
	return sql.NullInt64{Int64: int64(v), Valid: v >= 0}
}

// ToSQLTime converts the input time into a sql.NullTime
func ToSQLTime(t time.Time) sql.NullTime {
	return sql.NullTime{Time: t, Valid: t != time.Time{} && t.Unix() != 0}
}
```

These `ToSQLXxx()` functions are often used to convert primitive types (and `time.Time`) into its `sql.NullXxx` variant. This is especially useful to quickly convert types for queries.

#### IsEntityFound functions

```go
import (
	"database/sql"
	"fmt"
)

// IsUserFound returns an error if the entity is not found
func IsUserFound(res sql.Result) error {
	n, err := res.RowsAffected()
	if err != nil {
		return errors.Join(ErrDBError, err)
	}
	if n == 0 {
		return ErrNotFoundUser
	}
	return nil
}

// IsSecretFound returns an error if the entity is not found
func IsSecretFound(res sql.Result) error {
	n, err := res.RowsAffected()
	if err != nil {
		return errors.Join(ErrDBError, err)
	}
	if n == 0 {
		return ErrNotFoundSecret
	}
	return nil
}

// IsShareFound returns an error if the entity is not found
func IsShareFound(res sql.Result) error {
	n, err := res.RowsAffected()
	if err != nil {
		return errors.Join(ErrDBError, err)
	}
	if n == 0 {
		return ErrNotFoundShare
	}
	return nil
}
```

The `IsXxxFound()` functions are wrappers for `sql.ExecContext()` returns, to verify if the issue was a target where the entry didn't exist, returning an appropriate error for the action.

Note that this application does not implement any custom error type nor is it templating entities for usage in this type of errors -- otherwise it could be done in one-go with a generic function (that accepts an entity type) or with a different error handling pattern. I chose to simply re-write the same function three times since they're used very little in the implementations, and didn't seem like a solid reason to implement custom errors for this app.


#### Implementing `user.Repository`

A SQL implementation will need a data structure that is specific to the database, with the appropriate `sql.NullXxx` types and with the same structure as the table schema.

It will also contain a type to implement the repository and function to initialize said type. Lastly, it contains a few helper methods to convert between the database data structure and the domain one, as well as functions / methods to read the rows' contents.

Here is the first layout of the `sqlite/users.go` file:

```go
package sqlite

import (
	"context"
	"database/sql"

	"github.com/zalgonoise/x/secr/user"
)

type dbUser struct {
	ID        sql.NullInt64
	Username  sql.NullString
	Name      sql.NullString
	Hash      sql.NullString
	Salt      sql.NullString
	CreatedAt sql.NullTime
	UpdatedAt sql.NullTime
}

var _ user.Repository = &userRepository{nil}

type userRepository struct {
	db *sql.DB
}

// NewUserRepository creates a user.Repository from the SQL DB `db`
func NewUserRepository(db *sql.DB) user.Repository {
	return &userRepository{db}
}

// Create will create a user `u`, returning its ID and an error
func (ur *userRepository) Create(ctx context.Context, u *user.User) (uint64, error) {
	return 0, nil
}

// Get returns the user identified by `username`, and an error
func (ur *userRepository) Get(ctx context.Context, username string) (*user.User, error) {
	return nil, nil
}

// List returns all the users, and an error
func (ur *userRepository) List(ctx context.Context) ([]*user.User, error) {
	return nil, nil
}

// Update will update the user `username` with its updated version `updated`. Returns an error
func (ur *userRepository) Update(ctx context.Context, username string, updated *user.User) error {
	return nil
}

// Delete removes the user identified by `username`, returning an error
func (ur *userRepository) Delete(ctx context.Context, username string) error {
	return nil
}

func (u *dbUser) toDomainEntity() *user.User {
	return &user.User{
		ID:        uint64(u.ID.Int64),
		Username:  u.Username.String,
		Name:      u.Name.String,
		Hash:      u.Hash.String,
		Salt:      u.Salt.String,
		CreatedAt: u.CreatedAt.Time,
		UpdatedAt: u.UpdatedAt.Time,
	}
}

func newDBUser(u *user.User) *dbUser {
	return &dbUser{
		Username: ToSQLString(u.Username),
		Name:     ToSQLString(u.Name),
		Hash:     ToSQLString(u.Hash),
		Salt:     ToSQLString(u.Salt),
	}
}

```

This is a very similar preparation to the one done in Bolt DB's [`keys.Repository` implementation](#implementing-keysrepository). The only difference is that for SQL I start by defining the `dbUser` struct, as a database entity. Since these types need to be interchangeable, I expand on that too by writing a `newDBXxx()` function and a `(x dbXxx) toDomainEntity() Xxx` method.

#### Implementing `userRepository.Create`

This call will push the input `*user.User` into the database, by converting it into a `dbUser` and writing its fields to the `users` table.

It will return an error if the transaction fails, or if the last inserted ID errors or is zero.


```go
// Create will create a user `u`, returning its ID and an error
func (ur *userRepository) Create(ctx context.Context, u *user.User) (uint64, error) {
	dbu := newDBUser(u)
	res, err := ur.db.ExecContext(ctx, `
INSERT INTO users (username, name, hash, salt)
VALUES (?, ?, ?, ?)
`, dbu.Username, dbu.Name, dbu.Hash, dbu.Salt)

	if err != nil {
		return 0, errors.Join(ErrDBError, fmt.Errorf("failed to create user %s: %w",u.Username, err))
	}

	id, err := res.LastInsertId()
	if err != nil {
		return 0, errors.Join(ErrDBError, fmt.Errorf("failed to create user %s: %w", u.Username, err))
	}
	if id == 0 {
		return 0, fmt.Errorf("%w: user was not created %s", ErrDBError, u.Username)
	}
	return uint64(id), nil
}
```


#### Implementing `userRepository.Get`

The `Get` method is very simple in nature, in the sense that it fetches a user identified by username `username`. This is a `SELECT` operation in SQL filtering by the username field in the users table.

On this method, it's not important to create a new `dbUser` object, since the `ToSQLString()` function can be called directly on the input `username` parameter.

From this query, the resulting row is being passed to a `scanUser` method which extracts the user from the `sql.Row`.

> Note -- for SQL folks that are not familiar with Go: the question marks `?` in the queries represent a template, where the query will receive some data instead of that question mark token.
>
> The following function argument here, `ToSQLString(username)` converts a primitive `string` type into sql.NullString (a compatible type), which is parsed as `(...) WHERE u.username = 'myusername';`

```go
// Get returns the user identified by `username`, and an error
func (ur *userRepository) Get(ctx context.Context, username string) (*user.User, error) {
	row := ur.db.QueryRowContext(ctx, `
SELECT u.id, u.username, u.name, u.hash, u.salt, u.created_at, u.updated_at
FROM users AS u
WHERE u.username = ?
	`, ToSQLString(username))

	user, err := ur.scanUser(row)
	if err != nil {
		return nil, err
	}
	return user, nil
}

// (...)

func (ur *userRepository) scanUser(r Scanner) (u *user.User, err error) {
	if r == nil {
		return nil, fmt.Errorf("%w: failed to find this user", ErrNotFoundUser)
	}
	dbu := new(dbUser)
	err = r.Scan(
		&dbu.ID,
		&dbu.Username,
		&dbu.Name,
		&dbu.Hash,
		&dbu.Salt,
		&dbu.CreatedAt,
		&dbu.UpdatedAt,
	)
	if err != nil {
		if errors.Is(sql.ErrNoRows, err) {
			return nil, ErrNotFoundUser
		}
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to scan DB row: %w", err))
	}
	return dbu.toDomainEntity(), nil
}
```

The `scanUser` method will be reused, thus being a separate method, and basically leverages the `Scan` method to extract the row fields into one or more target pointers.

The function returns a user (already converted to a domain entity, as `*user.User`) and an error -- be it a *Not Found* error or a generic DB error.

#### Implementing `userRepository.List`

The `List` method is even simpler than `Get`, since it will just collect all users in the database.

If the query is successful, the rows are passed onto the `scanUsers` method (below) which is sort of like a batch `scanUser` method.

```go
// List returns all the users, and an error
func (ur *userRepository) List(ctx context.Context) ([]*user.User, error) {
	rows, err := ur.db.QueryContext(ctx, `
SELECT u.id, u.username, u.name, u.created_at, u.updated_at
FROM users AS u
	`)
	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list users: %w", err))
	}

	users, err := ur.scanUsers(rows)
	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list users: %w", err))
	}
	return users, nil
}

// (...)


func (ur *userRepository) scanUsers(rs *sql.Rows) ([]*user.User, error) {
	var users = []*user.User{}

	defer rs.Close()
	for rs.Next() {
		u, err := ur.scanUser(rs)
		if err != nil {
			return nil, fmt.Errorf("failed to scan row: %w", err)
		}
		users = append(users, u)
	}
	return users, nil
}
```

`scanUsers` is a simple method that navigates through all rows in the input cursor, while extracting each user from each row, appending it to the `users` slice.



#### Implementing `userRepository.Update`

The `Update` operation is only intended to target the user's name changes and password changes.

This isn't a dynamic operation where only the set values are updated, so the caller (in this case the service) is responsible for assuring that the name and hash data in the `updated *user.User` is not empty.

Both fields are updated in the database, and an error is returned if the user is not found, or on a database error.

```go
// Update will update the user `username` with its updated version `updated`. Returns an error
func (ur *userRepository) Update(ctx context.Context, username string, updated *user.User) error {
	dbu := newDBUser(updated)
	res, err := ur.db.ExecContext(ctx, `
UPDATE users
SET name = ?, hash = ?
WHERE u.username = ?
`, dbu.Name, dbu.Hash, ToSQLString(username))

	if err != nil {
		return errors.Join(ErrDBError, fmt.Errorf("failed to update user %s: %w", username, err))
	}

	err = IsUserFound(res)
	if err != nil {
		return err
	}
	return nil
}
```

#### Implementing `userRepository.Delete`

The `Delete` operation will remove a user by their username. It has a very similar flow as in the `Update` operation in the sense that it executes an update on the database and can return an error if the user is not found, or on a database error.

```go
// Delete removes the user identified by `username`, returning an error
func (ur *userRepository) Delete(ctx context.Context, username string) error {
	res, err := ur.db.ExecContext(ctx, `
	DELETE FROM users WHERE username = ?
	`, ToSQLString(username))

	if err != nil {
		return errors.Join(ErrDBError, fmt.Errorf("failed to delete user %s: %w", username, err))
	}

	err = IsUserFound(res)
	if err != nil {
		return err
	}

	return nil
}
```


#### Defining errors and updating `import`s

```go
import (
	"context"
	"database/sql"
	"fmt"

	"github.com/zalgonoise/x/errors"
	"github.com/zalgonoise/x/secr/user"
)

var (
	ErrDBError      = errors.New("database error")
	ErrNotFoundUser = errors.New("user not found")
)
```


#### Implementing `secret.Repository`

Let's mimick the flow for writing the `user.Repository` implementation.

Just for context, the secrets repository stores secrets metadata, not the secret's values. That will be the service's responsibility to also call the `keys.Repository` for the same values.

Here is the first layout of the `sqlite/secrets.go` file:

```go
package sqlite

import (
	"context"
	"database/sql"

	"github.com/zalgonoise/x/secr/secret"
)


type dbSecret struct {
	ID        sql.NullInt64
	Name      sql.NullString
	CreatedAt sql.NullTime
}

var _ secret.Repository = &secretRepository{nil}

type secretRepository struct {
	db *sql.DB
}

// NewSecretRepository creates a secret.Repository from the SQL DB `db`
func NewSecretRepository(db *sql.DB) secret.Repository {
	return &secretRepository{db}
}

// Create will create (or overwrite) the secret identified by `s.Key`, for user `username`,
// returning an error
func (sr *secretRepository) Create(ctx context.Context, username string, s *secret.Secret) (uint64, error) {
	return 0, nil
}

// Get fetches a secret identified by `key` for user `username`. Returns a secret and an error
func (sr *secretRepository) Get(ctx context.Context, username string, key string) (*secret.Secret, error) {
	return nil, nil
}

// List returns all secrets belonging to user `username`, and an error
func (sr *secretRepository) List(ctx context.Context, username string) ([]*secret.Secret, error) {
	return nil, nil
}

// Delete removes the secret identified by `key`, for user `username`. Returns an error
func (sr *secretRepository) Delete(ctx context.Context, username string, key string) error {
	return nil
}

func (s *dbSecret) toDomainEntity() *secret.Secret {
	return &secret.Secret{
		ID:        uint64(s.ID.Int64),
		Key:       s.Name.String,
		CreatedAt: s.CreatedAt.Time,
	}
}

func newDBSecret(s *secret.Secret) *dbSecret {
	return &dbSecret{
		Name: ToSQLString(s.Key),
	}
}
```

Since I know it will be needed, I will now sketch out the `scanSecret()` and `scanSecrets()` methods in advance:

```go
func (sr *secretRepository) scanSecret(r Scanner) (s *secret.Secret, err error) {
	if r == nil {
		return nil, fmt.Errorf("%w: failed to find this secret", ErrNotFoundSecret)
	}
	dbs := new(dbSecret)
	err = r.Scan(
		&dbs.ID,
		&dbs.Name,
		&dbs.CreatedAt,
	)
	if err != nil {
		if errors.Is(sql.ErrNoRows, err) {
			return nil, ErrNotFoundSecret
		}
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to scan DB row: %w", err))
	}
	return dbs.toDomainEntity(), nil
}

func (sr *secretRepository) scanSecrets(rs *sql.Rows) ([]*secret.Secret, error) {
	var secrets = []*secret.Secret{}

	defer rs.Close()
	for rs.Next() {
		s, err := sr.scanSecret(rs)
		if err != nil {
			return nil, fmt.Errorf("failed to scan row: %w", err)
		}
		secrets = append(secrets, s)
	}
	return secrets, nil
}
```


#### Implementing `secretRepository.Create`

Knowing that the `secrets` table references the `users` table for the `users.id` value, the SQL queries raise a tiny bit in complexity. Other than that, it's the same flow as seen in the `userRepository.Create` implementation. The same is seen in the other methods, too:

```go
// Create will create (or overwrite) the secret identified by `s.Key`, for user `username`,
// returning an error
func (sr *secretRepository) Create(ctx context.Context, username string, s *secret.Secret) (uint64, error) {
	dbs := newDBSecret(s)
	res, err := sr.db.ExecContext(ctx, `
INSERT INTO secrets (user_id, name)
VALUES (
	(SELECT u.id FROM users AS u WHERE u.username = ?), 
	?)
`, ToSQLString(username), dbs.Name)

	if err != nil {
		return 0, errors.Join(ErrDBError, fmt.Errorf("failed to create secret %s: %w", s.Key, err))
	}

	id, err := res.LastInsertId()
	if err != nil {
		return 0, errors.Join(ErrDBError, fmt.Errorf("failed to create secret %s: %w",  s.Key, err))
	}
	if id == 0 {
		return 0, fmt.Errorf("secret was not created %s", ErrDBError, s.Key)
	}

	return uint64(id), nil
}
```



#### Implementing `secretRepository.Get`

Same flow as the `userRepository.Get` implementation. I am joining the user's table to scope the query to the correct `secrets.user_id`:

```go
// Get fetches a secret identified by `key` for user `username`. Returns a secret and an error
func (sr *secretRepository) Get(ctx context.Context, username string, key string) (*secret.Secret, error) {
	row := sr.db.QueryRowContext(ctx, `
SELECT s.id, s.name, s.created_at
FROM secrets AS s
	JOIN users AS u ON u.id = s.user_id
WHERE u.username = ?
	AND s.name = ?
	`, ToSQLString(username), ToSQLString(key))

	s, err := sr.scanSecret(row)
	if err != nil {
		return nil, err
	}
	return s, nil
}
```

#### Implementing `secretRepository.List`

Exactly like the `Get` method, but only filters by `users.username`:

```go
// List returns all secrets belonging to user `username`, and an error
func (sr *secretRepository) List(ctx context.Context, username string) ([]*secret.Secret, error) {
	rows, err := sr.db.QueryContext(ctx, `
SELECT s.id, s.name, s.created_at
FROM secrets AS s
	JOIN users AS u ON u.id = s.user_id
WHERE u.username = ?
		`, ToSQLString(username))

	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list secrets: %w", err))
	}

	secrets, err := sr.scanSecrets(rows)
	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list secrets: %w", err))
	}

	return secrets, nil
}
```

#### Implementing `secretRepository.Delete`

The `Delete` operation will remove the entry where the ID matches the following query, as SQLite has some limitations (that are a pain when you're used to MySQL / MariaDB). Besides this, it's a regular DB-write:

```go
// Delete removes the secret identified by `key`, for user `username`. Returns an error
func (sr *secretRepository) Delete(ctx context.Context, username string, key string) error {
	res, err := sr.db.ExecContext(ctx, `
	DELETE FROM secrets WHERE id = (
		SELECT s.id FROM secrets AS s
			JOIN users AS u ON u.id = s.user_id
		WHERE u.username = ? 
			AND s.name = ?
	)
	`, ToSQLString(username), ToSQLString(key))

	if err != nil {
		return errors.Join(ErrDBError, fmt.Errorf("failed to delete secret %s: %w", key, err))
	}

	err = IsSecretFound(res)
	if err != nil {
		return err
	}

	return nil
}
```


#### Defining errors and updating `import`s

```go
import (
	"context"
	"database/sql"
	"fmt"

	"github.com/zalgonoise/x/errors"
	"github.com/zalgonoise/x/secr/secret"
)

var (
	ErrNotFoundSecret = errors.New("secret not found")
)
```

#### Implementing `shared.Repository`

Just like the `secret.Repository` implementation, but where the SQL queries kick it up a notch. And also the domain-to-database object conversions.

Mostly because this is dealing with the following characteristics:
- the domain entity describes the targets as a list of strings (their usernames). The database entity describes the target as a single string. Means that when creating a share that has multiple targets, it will break down the (domain) share into several (db) shares and process them individually.
- merging the resulting `dbShare` list into one or more domain shares requires processing them, evaluating if the `until` time value is the same for each target to aggregate them together (within the same key-owner object).

Here is the first layout of the `sqlite/shared.go` file:

```go
package sqlite

import (
	"context"
	"database/sql"
	"fmt"

	"github.com/zalgonoise/x/errors"
	"github.com/zalgonoise/x/secr/shared"
)

var (
	ErrNotFoundShare = errors.New("shared secret not found")
)

var _ shared.Repository = &sharedRepository{nil}

type dbShare struct {
	ID        sql.NullInt64
	Secret    sql.NullString
	Owner     sql.NullString
	Target    sql.NullString
	Until     sql.NullTime
	CreatedAt sql.NullTime
}

type sharedRepository struct {
	db *sql.DB
}

// NewSharedRepository creates a shared.Repository from the SQL DB `db`
func NewSharedRepository(db *sql.DB) shared.Repository {
	return &sharedRepository{db}
}

// Create shares the secret identified by `secretName`, owned by `owner`, with
// user `target`. Returns an error
func (sr *sharedRepository) Create(ctx context.Context, sh *shared.Share) (uint64, error) {
	return nil, nil
}

// Get fetches the secret's share metadata for a given username and secret key
func (sr *sharedRepository) Get(ctx context.Context, username, secretName string) ([]*shared.Share, error) {
	return nil, nil
}

func (sr *sharedRepository) List(ctx context.Context, username string) ([]*shared.Share, error) {
	return nil, nil
}

// ListTarget is similar to List, but returns secrets that are shared with a target user
func (sr *sharedRepository) ListTarget(ctx context.Context, target string) ([]*shared.Share, error) {
	return nil, nil
}

// Delete removes the user `target` from the secret share
func (sr *sharedRepository) Delete(ctx context.Context, sh *shared.Share) error {
	return nil
}
```

Let's start with the basic converter functions, for `dbShare`:

```go
func newDBShare(s *shared.Share) []*dbShare {
	var sqlT sql.NullTime
	switch s.Until {
	case nil:
		sqlT = ToSQLTime(time.Now().Add(shared.DefaultShareDuration))
	default:
		sqlT = ToSQLTime(*s.Until)
	}

	shares := make([]*dbShare, 0, len(s.Target))

	for _, t := range s.Target {
		shares = append(shares, &dbShare{
			Owner:  ToSQLString(s.Owner),
			Secret: ToSQLString(s.SecretKey),
			Target: ToSQLString(t),
			Until:  sqlT,
		})
	}
	return shares
}
```

Breaking this down: 

- Since the `*shared.Share.Until` value is nullable, the function checks on it. If it is in fact `nil` (it shouldn't as covered by the service, but worth checking), set the `until` time to now plus the default duration.
- Create a list of `*dbShare` the same size as the targets in the input share.
- Iterate through each target in the share appending a new `dbShare` to the list containing the owner, secret key and target.

For the other way around (say, listing all secrets I've shared with other users), the logic is slightly different as it's important to check if the secrets can be merged into a domain share or not. For this, they must match the same secret key and owner, as well as `until` time:

```go
func toDomainShare(shares ...*dbShare) []*shared.Share {
	if len(shares) == 0 {
		return nil
	}

	s := make([]*shared.Share, 0, len(shares))
	s = append(s, &shared.Share{
		ID:        uint64(shares[0].ID.Int64),
		SecretKey: shares[0].Secret.String,
		Owner:     shares[0].Owner.String,
		Target:    []string{shares[0].Target.String},
		Until:     &shares[0].Until.Time,
		CreatedAt: shares[0].CreatedAt.Time,
	})

	if len(shares) == 1 {
		return s
	}

inputLoop:
	for i := 1; i < len(shares); i++ {
		for _, sh := range s {
			if shares[i].Owner.String == sh.Owner &&
				shares[i].Secret.String == sh.SecretKey &&
				sh.Until.Unix() == shares[i].Until.Time.Unix() {
				for _, t := range sh.Target {
					if t == shares[i].Target.String {
						continue
					}
					sh.Target = append(sh.Target, shares[i].Target.String)
					continue inputLoop
				}
			}
		}
		s = append(s, &shared.Share{
			ID:        uint64(shares[i].ID.Int64),
			SecretKey: shares[i].Secret.String,
			Owner:     shares[i].Owner.String,
			Target:    []string{shares[i].Target.String},
			Until:     &shares[i].Until.Time,
			CreatedAt: shares[i].CreatedAt.Time,
		})
	}

	return s
}
```

Breaking this function down:
- It uses a variadic parameter to accept any number of `*dbShare`, making it useful all use-cases (short-circuiting out when no shares are passed in as arguments).
- It initializes the returned list of `*shared.Share` with the first element in the input `*dbShare`, appending it to the returned list.
- Then, it loops over the remainder of the input (starting on index 1) where:
  - it loops through each item already collected.
  - checks if it matches owner, secret key **and** `until` time
  - if passes all three checks, appends the target to the matched object
  - otherwise, creates a new `*shared.Share` object that is appended to the output list.

> `inputLoop:` is named loop in Go. This means that you can specify control flow on a specific loop by name. [See an article on this topic](https://www.ardanlabs.com/blog/2013/11/label-breaks-in-go.html)


Having these ready, it's time to write the scanner methods, similar to the previous implementations. The major difference on this one is the converter to domain entity is not a method but a function, since we're handling a list of shares:

```go
func (sr *sharedRepository) scanShare(r Scanner) (dbs *dbShare, err error) {
	if r == nil {
		return nil, fmt.Errorf("%w: failed to find this share", ErrNotFoundShare)
	}
	dbs = new(dbShare)
	err = r.Scan(
		&dbs.ID,
		&dbs.Secret,
		&dbs.Owner,
		&dbs.Target,
		&dbs.Until,
		&dbs.CreatedAt,
	)
	if err != nil {
		if errors.Is(sql.ErrNoRows, err) {
			return nil, ErrNotFoundShare
		}
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to scan DB row: %w", err))
	}
	return dbs, nil
}

func (sr *sharedRepository) scanShares(rs *sql.Rows) ([]*shared.Share, error) {
	var shares []*dbShare

	defer rs.Close()
	for rs.Next() {
		dbs, err := sr.scanShare(rs)
		if err != nil {
			return nil, fmt.Errorf("failed to scan row: %w", err)
		}
		shares = append(shares, dbs)
	}

	return toDomainShare(shares...), nil
}
```

#### Implementing `sharedRepository.Create`

Taking into account the other implementations, this will be the most complex one.

Shares are created usually in batches, and (for any reason) the implementation should be ready to handle errors in a transaction.

For this, the `Create` (and `Delete`) methods will use a `*sql.Tx`. The `tx.Rollback()` method is deferred in case an error is raised.

With this out of the way, it's a matter of iterating through the `[]*dbShare` (extracted from the input `*shared.Share`) and inserting the share in the DB. As for the share ID, only the last ID is returned (it's only used for reference, and not in any practical way).

```go
// Create shares the secret identified by `secretName`, owned by `owner`, with
// user `target`. Returns an error
func (sr *sharedRepository) Create(ctx context.Context, sh *shared.Share) (uint64, error) {
	shares := newDBShare(sh)
	tx, err := sr.db.Begin()
	if err != nil {
		return 0, fmt.Errorf("failed to begin transaction: %w", err)
	}

	defer tx.Rollback()

	var lastID uint64

	for _, dbs := range shares {
		res, err := tx.ExecContext(ctx, `
		INSERT INTO shared_secrets (owner_id, secret_id, shared_with, until)
		VALUES (
			(SELECT id FROM users WHERE username = ?),
			(SELECT id FROM secrets WHERE name = ?),
			(SELECT id FROM users WHERE username = ?),
			?
		)
		`, dbs.Owner, dbs.Secret, dbs.Target, dbs.Until)

		if err != nil {
			return 0, err
		}
		id, err := res.LastInsertId()
		if err != nil {
			return 0, err
		}
		lastID = uint64(id)
	}

	err = tx.Commit()
	if err != nil {
		return 0, fmt.Errorf("failed to create shared secret: %w", err)
	}
	return lastID, nil
}
```


#### Implementing `sharedRepository.Get`

`Get` is nothing fancy besides having a longer SQL query. The query itself joins the `users` table twice for the owner and the target(s) and the `secrets` table for the secret key. 

As the signature implies, it filters by the owner's name in `users.username` and the secret's key in `secrets.name`.

```go
// Get fetches the secret's share metadata for a given username and secret key
func (sr *sharedRepository) Get(ctx context.Context, username, secretName string) ([]*shared.Share, error) {
	rows, err := sr.db.QueryContext(ctx, `
SELECT s.id, x.name, o.username, t.username, s.until, s.created_at
FROM shared_secrets AS s
	JOIN users AS o ON o.id = s.owner_id
	JOIN users AS t ON t.id = s.shared_with
	JOIN secrets AS x ON x.id = s.secret_id
WHERE o.username = ?
	AND x.name = ?
`, ToSQLString(username), ToSQLString(secretName))

	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list shared secrets: %w", err))
	}

	shares, err := sr.scanShares(rows)
	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list shared secrets: %w", err))
	}
	return shares, nil
}
```


#### Implementing `sharedRepository.List`

Exactly the same as `Get`, but only has a filter for the owner's username.

```go
func (sr *sharedRepository) List(ctx context.Context, username string) ([]*shared.Share, error) {
	rows, err := sr.db.QueryContext(ctx, `
SELECT s.id, x.name, o.username, t.username, s.until, s.created_at
FROM shared_secrets AS s
	JOIN users AS o ON o.id = s.owner_id
	JOIN users AS t ON t.id = s.shared_with
	JOIN secrets AS x ON x.id = s.secret_id
WHERE o.username = ?
`, ToSQLString(username))

	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list shared secrets: %w", err))
	}

	shares, err := sr.scanShares(rows)
	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list shared secrets: %w", err))
	}
	return shares, nil
}
```



#### Implementing `sharedRepository.ListTarget`

Exactly the same as `List`, but its filter is for the **target**'s username.

```go
// ListTarget is similar to List, but returns secrets that are shared with a target user
func (sr *sharedRepository) ListTarget(ctx context.Context, target string) ([]*shared.Share, error) {
	rows, err := sr.db.QueryContext(ctx, `
	SELECT s.id, x.name, o.username, t.username, s.until, s.created_at
	FROM shared_secrets AS s
		JOIN users AS o ON o.id = s.owner_id
		JOIN users AS t ON t.id = s.shared_with
		JOIN secrets AS x ON x.id = s.secret_id
	WHERE t.username = ?
	`, ToSQLString(target))

	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list shared secrets: %w", err))
	}

	shares, err := sr.scanShares(rows)
	if err != nil {
		return nil, errors.Join(ErrDBError, fmt.Errorf("failed to list shared secrets: %w", err))
	}
	return shares, nil
}
```



#### Implementing `sharedRepository.Delete`

The `Delete` call is similar to the `Create` operation, in the sense that it will break down the input `*shared.Share` into (possibly multiple) `*dbShare`, and within a SQL transaction it will delete the corresponding share.


```go
// Delete removes the user `target` from the secret share
func (sr *sharedRepository) Delete(ctx context.Context, sh *shared.Share) error {
	dbs := newDBShare(sh)
	tx, err := sr.db.Begin()
	if err != nil {
		return fmt.Errorf("failed to begin transaction: %w", err)
	}

	// defer rollback in case an error occurs
	defer tx.Rollback()

	for _, share := range dbs {
		res, err := tx.ExecContext(ctx, `
DELETE FROM shared_secrets WHERE id = (
SELECT s.id FROM shared_secrets AS s
	JOIN users AS o ON o.id = s.owner_id
	JOIN users AS t ON t.id = s.shared_with
	JOIN secrets AS x ON x.id = s.secret_id
WHERE o.username = ?
	AND x.name = ?
	AND t.username = ?
)`,
			share.Owner, share.Secret, share.Target)

		if err != nil {
			return errors.Join(ErrDBError, err)
		}
		err = IsShareFound(res)
		if err != nil {
			return err
		}
	}

	err = tx.Commit()
	if err != nil {
		return errors.Join(ErrDBError, fmt.Errorf("shared secret was not deleted: %w", err))
	}
	return nil
}
```


#### Defining errors and updating `import`s


```go
import (
	"context"
	"database/sql"
	"fmt"
	"time"

	"github.com/zalgonoise/x/errors"
	"github.com/zalgonoise/x/secr/shared"
)

var (
	ErrNotFoundShare = errors.New("shared secret not found")
)
```


[next](service_preparation.md)