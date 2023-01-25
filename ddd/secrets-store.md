## Concept

This document will cover the process of building a secrets / password store web application, in Go. The main goals and features of the app are the following:

- exposing a key-value store for secrets (confidential data), where the keys are of type `string` and the values are of type `[]byte` (slice of bytes / byte array).
- basic IAM support, with multiple users being able to use the service independently and privately. Users login with a username+password combination
- ability to share secrets among users, optionally scoped for a certain period of time.

This document will cover the entire process of designing, building and deploying the Go backend for this app. This document was written with Go currently on version 1.19, on January 2023.

__________________

## Design 

This will be a monolith code-base with a touch of Domain-Driven Design (DDD) principles, to allow a readable structure and isolated responsibilities among the different packages. As a whole, the project's organization will focus on:

1. Separating transport, service and repositories. Each layer will have their own set of responsibilities.
2. Decoupling the implementations, so that they can be easily refactored and / or re-implemented with a different solution / technology stack.
3. Joining the different modules is deferred to the factory package. This allows a bigger focus on the package that we're working on focusing on inputs and returns.
4. No `pkg`, nor `util`, nor `helpers`, nor `models`; at least for the app's logic. This is done on the corresponding module.

### App structure

The entities in this model are simple, there are users who create secrets, and these secrets can be accessed by either the owner or other users if the secret is shared. The diagram below showcases this model:

![Diagram](./media/Secr_Diagram.jpg)

Of course, the app will have a few other services to support authentication, authorization and encryption, for example. But these services are not the core-purpose of the app.

### Entities

Like the diagram above implies, at least two entities are necessary: User and Secret.

These entities determine the structure of the handled data.  

#### User

```go
package user

// User is a person (or entity) that uses the application to store
// secrets. They will have a unique username.
type User struct {
	ID        uint64    `json:"id"`
	Username  string    `json:"username"`
	Name      string    `json:"name"`
	Hash      string    `json:"-"`
	Salt      string    `json:"-"`
	CreatedAt time.Time `json:"created_at,omitempty"`
	UpdatedAt time.Time `json:"updated_at,omitempty"`
}
```

> A user object will store the user's ID and basic information, a hash of their (salted) password and the salt; as well as created / updated timestamps

#### Secret

```go
package secret

// Secret is a key-value pair where they Key is string type and Value
// is a slice of bytes. Secrets are encrypted then stored with a user-scoped
// private key
type Secret struct {
	ID        uint64    `json:"id"`
	Key       string    `json:"key"`
	Value     string    `json:"value"`
	CreatedAt time.Time `json:"created_at"`
}
```

> A secret object will hold its key-value data, as well as a timestamp value for when it was created

#### Shared

```go
package shared

// DefaultShareDuration sets a maximum of 30 days for shared secrets with no defined time limit
const DefaultShareDuration = time.Hour * 24 * 30

// Shared is metadata for a secret that a user (the owner) shares with a set of users
// optionally within a limited period of time
type Share struct {
	ID        uint64     `json:"id"`
	SecretKey string     `json:"secret_key"`
	Owner     string     `json:"owner"`
	Target    []string   `json:"shared_with"`
	Until     *time.Time `json:"until,omitempty"`
	CreatedAt time.Time  `json:"created_at"`
}
```

> Shared secrets will be objects containing a Secret's key, it's owner's username, and shares (slice of strings representing usernames). Optionally, a shared secret can be scoped for a period of time (a nullable value). For this application, I also define the default share duration (1 month) in case it's not defined by the caller.


### Database design

Storing this data could be easily done in a simple manner, in a relational database. This database would contain at least 3 tables, for users, secrets and shared_secrets. However, this specific implementation will use two types of databases:

1. A **SQLite database** to serve as a relational database for the users and secrets / shares metadata.
2. A **Bolt database** to serve as a key-value store, holding the sensitive information for the user's secrets, the user's private encryption key and any other type of sensitive information.

Please note below how do the entities and model above translate to an RDB table structure, for this project:

#### `users` table

col. name | type | PK | FK
:--:|:--:|:--:|:--:
`id` | INTEGER | Yes |
`username` | VARCHAR(50) | |
`name` | TEXT | |
`hash` | TEXT | |
`salt` | TEXT | |
`created_at` | TIMESTAMP | |
`updated_at` | TIMESTAMP | |

> *Users data will consist on the basic information they will supply when creating a user account (initially username, password and a name); however the database will not store the password -- but a salt generated when the account was created and the hash of the password with the salt appended to it*

#### `secrets` table

col. name | type | PK | FK
:--:|:--:|:--:|:--:
`id` | INTEGER | Yes |
`user_id` | INTEGER | | `users (id)`
`name` | VARCHAR(250) | |
`created_at` | TIMESTAMP | |

> *Secrets data will consist exclusively in secret metadata and its key (as "name"); the actual secret's value is stored in Bolt, encrypted with a key that is exclusive to the user (generated by the service when the user is created)*

#### `shared_secrets` table

col. name | type | PK | FK
:--:|:--:|:--:|:--:
`id` | INTEGER | Yes |
`owner_id` | INTEGER | | `users (id)`
`secret_id` | INTEGER | | `secrets (id)`
`shared_with` | INTEGER | | `users (id)`
`until` | TIMESTAMP | |
`created_at` | TIMESTAMP | |

> *Shared secrets table will hold the metadata about the share; such as from-to user IDs, the secret's ID and a time limit if set. If the secret is shared with multiple users, there will be multiple entries simliar to each other, with different `shared_with` values*

### Actions and operations

This application will require a CRUD implementation for both users and secrets, as well as added actions to share the secrets with one (or more) user(s).

As for the users repository, it will expose a complete set of CRUD operations (with `list`); but secrets will not contain an update operation -- its `create` operation will be designed to overwrite the secret if it already exists.

The shared repository will contain methods to share a secret with users -- optionally until a period in time, or for a certain duration. The removal of an expired shared secret is done when reading it; so if a user lists their secrets which currently include an expired shared secret, it will be removed before the read / list response is returned to the caller.

### Security

Initially, the users will login using a dedicated endpoint, using a username + password combination. The call returns a JWT if their credentials are valid, and the caller can use the JWT as an Authorization HTTP header calling users and secrets endpoints. JWT expire within one hour of being created.

Creating a user does not require authentication.

When a user is created, the service will create an encryption private key in the Bolt database that will be used to encrypt the user's secrets' values. Also, when created, a random 128-byte-long salt value is generated and appended to the user's password; and the resulting value is hashed. The SQLite database will store this salt and hash values, base64-encoded, and not the user's plaintext password.

A user can only access their own resources. A secret shared by user A with user B is perceived as a (new) secret owned by user B in a read operation; which will contain an expiry. However, the actual owner is in control of this secret -- as shared secrets are read-write only for the actual owner. Shared secrets have a different key in the format of `username:secretkey`.

While there is a reserved username (`root`) despite not having a roles / privileges model; the Bolt database will still store user secrets in buckets identified by the user's (internal) ID, and not their username. User secrets are encrypted with AES, using a 32-byte key generated and stored when the user is first created. Like passwords, secrets will not be stored in plaintext (beyond the database's own encryption).

### Product Format

The final application will be initially distributed as a web application, as a HTTP API. It could potentially have a CLI client that communicates with a configured secrets store server via HTTP (performing the same HTTP calls, but from a CLI app).

While there are no plans for an actual frontend with a UI, the simplest approach to doing so would be with a Flutter / Dart project which is simple and distributable on a number of platforms.

_______________

## Implementation

Having a general structure planned ahead is essential to organize the codebase before any code is written. Having the context on what we want from the above, it's easier to sketch out the entities and repositories for this application:

### The `user` package

This package will be a top-level folder in the project named `user`, with:

```
.
└─ user
    ├─ repository.go -- lists the actions supported by the repository
    └─ user.go -- describes the user entity
```

#### `user.go` - defining entities

Just like the snippet a few blocks above, this file is very straight-forward and will contain the user entity, which describes the basic elements of a user. Additionally it will already include the Session type, which is an authenticated user (user with a session token); since this is also within the realm of the *users*. 

While the entities expose the JSON tags which will be used later on in the HTTP responses, note how both the `Hash` and `Salt` fields are omitted.

```go
package user

import (
	"time"
)

// RootUsername is a reserved username for the admin deploying the application
const RootUsername = "root"

// User is a person (or entity) that uses the application to store
// secrets. They will have a unique username.
type User struct {
	ID        uint64    `json:"id"`
	Username  string    `json:"username"`
	Name      string    `json:"name"`
	Hash      string    `json:"-"`
	Salt      string    `json:"-"`
	CreatedAt time.Time `json:"created_at,omitempty"`
	UpdatedAt time.Time `json:"updated_at,omitempty"`
}

// Session is an authorized user, accompanied by a JWT
type Session struct {
	User  `json:"user"`
	Token string `json:"token"`
}
```

#### `repository.go` - defining the actions

The users repository will contain a set of actions which will target the `users` SQL table described in the Design chapter. This means that it does not contain methods related to login; only CRUD operations against actual users:

```go
package user

import "context"

// Repository describes the actions exposed by the users store
type Repository interface {
	// Create will create a user `u`, returning its ID and an error
	Create(ctx context.Context, u *User) (uint64, error)
	// Get returns the user identified by `username`, and an error
	Get(ctx context.Context, username string) (*User, error)
	// List returns all the users, and an error
	List(ctx context.Context) ([]*User, error)
	// Update will update the user `username` with its updated version `updated`. Returns an error
	Update(ctx context.Context, username string, updated *User) error
	// Delete removes the user identified by `username`, returning an error
	Delete(ctx context.Context, username string) error
}
```

> All repository methods accept a context as a first argument to allow retrieving more observability information, as covered further down this document.

#### Users recap

In a nutshell this is the domain abstraction of the data stored in the database. It allows for the persistence layer (the databases) to have different implementations -- while also deferring the responsibility of persisting those objects to a different module.

The domain entity and repository interface state what this object (the user) is and what we can do with it.

The exposed methods are pretty much CRUD operations (create-read-update-delete) with a list operation (which is basically a batch read).
____________

### The `secret` package

This package will be a top-level folder in the project named `secret`, with:

```
.
└─ secret
    ├─ repository.go -- lists the actions supported by the repository
    └─ secret.go -- describes the secret entity
```

#### `secret.go` - defining entities

Very similar to the user; the Secret type will contain the basic elements of a Secret. Note how a (basic) secret does not reference a user in it:

```go
package secret

import (
	"time"
)

// Secret is a key-value pair where they Key is string type and Value
// is a slice of bytes. Secrets are encrypted then stored with a user-scoped
// private key
type Secret struct {
	ID        uint64    `json:"id"`
	Key       string    `json:"key"`
	Value     string    `json:"value"`
	CreatedAt time.Time `json:"created_at"`
}
```


#### `repository.go` - defining the actions 

The `secret.Repository` will expose CRUD operations against the `secrets` SQL table, similar to `users.Repository` -- however it does not expose an Update method; as secrets' values are simply overwritten if the key already exists:

```go
package secret

import (
	"context"
)

// Repository describes the actions exposed by the secrets store
type Repository interface {
	// Create will create (or overwrite) the secret identified by `s.Key`, for user `username`,
	// returning its ID and an error
	Create(ctx context.Context, username string, s *Secret) (uint64, error)
	// Get fetches a secret identified by `key` for user `username`. Returns a secret and an error
	Get(ctx context.Context, username string, key string) (*Secret, error)
	// List returns all secrets belonging to user `username`, and an error
	List(ctx context.Context, username string) ([]*Secret, error)
	// Delete removes the secret identified by `key`, for user `username`. Returns an error
	Delete(ctx context.Context, username string, key string) error
}
```

> All repository methods accept a context as a first argument to allow retrieving more observability information, as covered further down this document.

#### Secrets recap

Very similar [to the User](#users-recap).

_______

### The `shared` package


This package will be a top-level folder in the project named `shared`, with:

```
.
└─ shared
    ├─ repository.go -- lists the actions supported by the repository
    └─ shared.go -- describes the secret entity
```

#### `shared.go` - defining entities

A Share type will represent a shared secret, holding the secret key, the username of the owner, and a list of target usernames that the secret is shared with. Optionally the object can include an `Until` value to scope the access of the secret until a certain point in time. This is a nullable field, but there is a default value whenever it is not populated (of 1 month).

A shared secret is usually returned in a list, even in a GET (read / list) operation.

```go
package shared

import (
	"time"
)

// DefaultShareDuration sets a maximum of 30 days for shared secrets with no defined time limit
const DefaultShareDuration = time.Hour * 24 * 30

// Shared is metadata for a secret that a user (the owner) shares with a set of users
// optionally within a limited period of time
type Share struct {
	ID        uint64     `json:"id"`
	SecretKey string     `json:"secret_key"`
	Owner     string     `json:"owner"`
	Target    []string   `json:"shared_with"`
	Until     *time.Time `json:"until,omitempty"`
	CreatedAt time.Time  `json:"created_at"`
}

```

#### `repository.go` - defining the actions

The `shared.Repository` is just as simple as the secrets': a set of CRUD operations without Update. The persistence layer (which implements the repository) will be solely responsible of saving the shared secrets state. Any features to this implementation (sharing for a duration of time, sharing until a point in time) will be handled by the service layer.

There are also two versions of the List operation, one for the shared secrets' owner, and one for the shared secrets' target user.

```go
package shared

import (
	"context"
)

// Repository describes the actions exposed by the shared secrets store
type Repository interface {
	// Create shares the secret identified by `secretName`, owned by `owner`, with
	// user `target`. Returns its ID and an error
	Create(ctx context.Context, s *Share) (uint64, error)
	// Get fetches the secret's share metadata for a given owner's username and secret key
	Get(ctx context.Context, owner, secretName string) ([]*Share, error)
	// List fetches all shared secrets for a given owner's username
	List(ctx context.Context, owner string) ([]*Share, error)
	// ListTarget is similar to List, but returns secrets that are shared with a target user
	ListTarget(ctx context.Context, target string) ([]*Share, error)
	// Delete removes the user `target` from the secret share
	Delete(ctx context.Context, s *Share) error
}
```


> All repository methods accept a context as a first argument to allow retrieving more observability information, as covered further down this document.

#### Shared recap

Very similar [to the User](#users-recap); but even simpler (and with two "list" operations) -- one lists the owner's shared secrets, while the other (`ListTarget`) lists the secrets that are shared with a certain target user.

___________

### The `keys` package

This package will be a top-level folder in the project named `keys` -- this will be a basic interface to the Bolt instance (as a key-value store), with:

```
.
└─ keys
    ├─ repository.go -- lists the actions supported by the repository
    └─ key.go -- contains constants used in the keys context, and 
	             domain-specific helper functions 
```


#### `key.go` - defining constants and identifiers

The key-value store is, as the name implies a database storing a value represented by a key. Bolt allows to nest these key-value pairs (a key that contains more key-value pairs) with buckets. 

A bucket can also be nested (buckets inside buckets). 

This use-case will be super simple where the service stores the user's secrets in a bucket identified by their ID (and a special prefix). This way, secrets are exclusive to a certain user, and other business-logic content can still be stored -- provided it is prefixed / named differently.

An alternative would be to have a bucket for secrets, that will store buckets for users, that will store key-value pairs for their secrets. To avoid this level of nesting, I've opted to go for the first approach.

Also -- as user secrets are encrypted with a unique key (which is generated on user creation), the key will also be stored in the user's bucket, under a unique key (`unique_identifier`).

Lastly, active JWT are stored in Bolt as well, under the user's bucket, with a unique key (`active-token`).

These constants are declared in this file, however it will be a secrets validator that will ensure that user-input secrets do not contain this key. More about validation further down this document.


```go
package keys

import "fmt"

const (
	UniqueID = "unique_identifier"
	TokenKey = "active-token"
)

// UserBucket formats the input user ID as a user bucket identifier (`uid:###`)
func UserBucket(id uint64) string {
	return fmt.Sprintf("uid:%d", id)
}
```


#### `repository.go` - defining the actions

The keys repository interface will be an *updateless CRUD* (CRD?); it also contains a `Purge()` method to destroy a bucket's contents completely.

```go
package keys

import "context"

// Repository describes the action exposed by the keys store
type Repository interface {
	// Set creates or overwrites a secret identified by `k` with value `v`, in
	// bucket `bucket`. Returns an error
	Set(ctx context.Context, bucket, k string, v []byte) error
	// Get fetches the secret identified by `k` in the bucket `bucket`,
	// returning a slice of bytes for the value and an error
	Get(ctx context.Context, bucket, k string) ([]byte, error)
	// Delete removes the secret identified by `k` in bucket `bucket`, returning an error
	Delete(ctx context.Context, bucket, k string) error
	// Purge removes all the secrets in the bucket `bucket`, returning an error
	Purge(ctx context.Context, bucket string) error
}
```

> All repository methods accept a context as a first argument to allow retrieving more observability information, as covered further down this document.

#### Keys recap

This section will be even simpler than the entities referred before, as it only handles actions against the (Bolt DB) key-value store. Since the data will always be a slice of bytes, there is no point in incrementing it much more than this; as it will be seen as a *plugin* in some packages to handle this type of (confidential content) storage.

___________

### Bolt DB implementation

The Bolt DB implementation will be placed in a top-level folder within the project, named `bolt`. This will be a very clean implementation considering how Bolt makes it simple to work with transactions -- meaning that you're able to issue a batch of operations to the database and rollback if an error is raised.

```
.
└─ bolt
    ├─ bolt.go -- exposes function(s) to initialize a Bolt DB instance
    └─ keys.go -- implements the keys.Repository interface
```

#### Initializing a Bolt DB instance

Initializing a Bolt DB instance simply takes a path to a file. This function is simply calling `bbolt.Open()`, but its signature will be common with the SQLite implementation (later in this document); which is useful for the factory functions -- it keeps a similar structure and signature:

```go
package bolt

import (
	"go.etcd.io/bbolt"
)

// Open will initialize a Bolt DB based on the `.db` file in `path`,
// returning a pointer to a bbolt.DB and an error
func Open(path string) (*bbolt.DB, error) {
	return bbolt.Open(path, 0600, nil)
}
```

#### Implementing `keys.Repository`

The keys repository will be very simple both thanks to Bolt's API and user experience -- but also because of the low complexity this feature has in the project. This repository is like a side-car to the rest of the application, as a means to isolate sensitive information in a key-value database.

Usually I start by laying down the type to represent this implementation, the methods that I need to implement and a function to initialize this type:

```go
package bolt

import (
	"context"

	"github.com/zalgonoise/x/secr/keys"
	"go.etcd.io/bbolt"
)

type keysRepository struct {
	db *bbolt.DB
}

// NewKeysRepository creates a keys.Repository from the Bolt DB `db`
func NewKeysRepository(db *bbolt.DB) keys.Repository {
	return &keysRepository{db}
}

// Get fetches the secret identified by `k` in the bucket `bucket`,
// returning a slice of bytes for the value and an error
func (ukr *keysRepository) Get(ctx context.Context, bucket, k string) ([]byte, error) {
	return nil, nil
}

// Set creates or overwrites a secret identified by `k` with value `v`, in
// bucket `bucket`. Returns an error
func (ukr *keysRepository) Set(ctx context.Context, bucket, k string, v []byte) error {
	return nil
}

// Delete removes the secret identified by `k` in bucket `bucket`, returning an error
func (ukr *keysRepository) Delete(ctx context.Context, bucket, k string) error {
	return nil
}

// Purge removes all the secrets in the bucket `bucket`, returning an error
func (ukr *keysRepository) Purge(ctx context.Context, bucket string) error {
	return nil
}
```

From here, it's much easier to isolate the task at hand and writing the logic. This is usually the format for my `unimplemented.go` files, if existing.

#### Implementing `keysRepository.Get`

Like mentioned before, the Bolt instance will contain buckets (for users, as an example) that will contain key-value pairs in them (for secrets). A `Get` call to this repository will be most likely for fetching secrets but it could also be to fetch the user's encryption key, and the user's active JWT. 

To ensure that the bucket and key are valid, this implementation will verify if the input strings are empty (otherwise it would be checked on the service level). Then, a read request is sent to the DB (with [`bbolt.View`](https://pkg.go.dev/go.etcd.io/bbolt#DB.View)), to read a key from an (assumed to be existing) bucket. The transaction will fail if the bucket does not exist.

```go
// Get fetches the secret identified by `k` in the bucket `bucket`,
// returning a slice of bytes for the value and an error
func (ukr *keysRepository) Get(ctx context.Context, bucket, k string) ([]byte, error) {
	if bucket == "" {
		return nil, ErrEmptyBucket
	}
	if k == "" {
		return nil, ErrEmptyKey
	}

	var v []byte

	err := ukr.db.View(func(tx *bbolt.Tx) error {
		b := tx.Bucket([]byte(bucket))
		if b == nil {
			return ErrEmptyBucket
		}
		v = b.Get([]byte(k))
		return nil
	})
	if err != nil {
		if errors.Is(err, ErrEmptyBucket) {
			return nil, err
		}
		return nil, fmt.Errorf("%w: %v", ErrDBError, err)
	}

	return v, nil
}
```

#### Implementing `keysRepository.Set`

Like `Get`; this call checks for empty values in the input. The `Set` call is a DB-write, thus using the [`bbolt.Update`](https://pkg.go.dev/go.etcd.io/bbolt#DB.Update) method. Since this call will be prepared to create new buckets and key-value pairs, it calls the [`tx.CreateBucketIfNotExists`](https://pkg.go.dev/go.etcd.io/bbolt#Tx.CreateBucketIfNotExists) method, accordingly.

The transaction returns an error if it fails when fetching / creating the bucket, or when storing the key-value pair.

```go
// Set creates or overwrites a secret identified by `k` with value `v`, in
// bucket `bucket`. Returns an error
func (ukr *keysRepository) Set(ctx context.Context, bucket, k string, v []byte) error {
	if bucket == "" {
		return ErrEmptyBucket
	}
	if k == "" {
		return ErrEmptyKey
	}
	if len(v) == 0 {
		return ErrEmptyValue
	}

	err := ukr.db.Update(func(tx *bbolt.Tx) error {
		b, err := tx.CreateBucketIfNotExists([]byte(bucket))
		if err != nil {
			return fmt.Errorf("failed to get / create bucket: %v", err)
		}

		err = b.Put([]byte(k), []byte(v))
		if err != nil {
			return fmt.Errorf("failed to set key-value: %v", err)
		}

		return nil
	})
	if err != nil {
		return fmt.Errorf("%w: %v", ErrDBError, err)
	}
	return nil
}
```


#### Implementing `keysRepository.Delete` and `keysRepository.Purge`

Both `Delete` and `Purge` calls are very similar, in the sense that the former removes a key-value pair while the latter removes the bucket entirely. These will be write operations, similar to the `Set` call, in which the `bbolt.Update` is called.

```go

// Delete removes the secret identified by `k` in bucket `bucket`, returning an error
func (ukr *keysRepository) Delete(ctx context.Context, bucket, k string) error {
	if bucket == "" {
		return ErrEmptyBucket
	}
	if k == "" {
		return ErrEmptyKey
	}

	err := ukr.db.Update(func(tx *bbolt.Tx) error {
		b := tx.Bucket([]byte(bucket))
		if b == nil {
			return ErrEmptyBucket
		}

		err := b.Delete([]byte(k))
		if err != nil {
			return fmt.Errorf("failed to delete key %s in the bucket %s: %v", k, bucket, err)
		}
		return nil
	})

	if err != nil {
		if errors.Is(err, ErrEmptyBucket) {
			return err
		}
		return fmt.Errorf("%w: %v", ErrDBError, err)
	}
	return nil
}
```

Even simpler, `Purge` doesn't even fetch the bucket:

```go
// Purge removes all the secrets in the bucket `bucket`, returning an error
func (ukr *keysRepository) Purge(ctx context.Context, bucket string) error {
	if bucket == "" {
		return ErrEmptyBucket
	}

	err := ukr.db.Update(func(tx *bbolt.Tx) error {
		err := tx.DeleteBucket([]byte(bucket))
		if err != nil {
			return fmt.Errorf("failed to delete the bucket %s: %v", bucket, err)
		}
		return nil
	})

	if err != nil {
		return fmt.Errorf("%w: %v", ErrDBError, err)
	}
	return nil
}
```

#### Defining errors and updating `import`s

Since the methods refer some error types that seem to be part of this package, they need to be defined; these are super simple errors to identify the issue better with, for example, `errors.Is()`.

```go
import (
	"context"
	"errors"
	"fmt"

	"github.com/zalgonoise/x/secr/keys"
	"go.etcd.io/bbolt"
)

var (
	ErrDBError     = errors.New("database error")
	ErrNotFoundKey = errors.New("couldn't find the key")
	ErrEmptyKey    = errors.New("key cannot be empty")
	ErrEmptyValue  = errors.New("username cannot be empty")
	ErrEmptyBucket = errors.New("empty bucket")
	ErrForbidden   = errors.New("unable to modify this resource")
)
```

#### BoltDB recap

This is the shortest and cleanest repository implementations in this application and for the same reason it was presented first. This design model will allow the level of abstraction as you see above, where the Bolt DB implementation only cares about what is being stored / read from Bolt DB.

The important take-aways from this one is that:

1. This is a feature that only the service layer will use / have access to.
2. Trying to stick to the necessary actions for the transaction to go through (on the persistence layer)
3. Even initializing the database file (creating it, loading it) is being deferred to a different package

___________

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

**SQL Querier types**

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


**SQL-type converters**

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

**IsEntityFound functions**

```go
import (
	"database/sql"
	"fmt"
)

// IsUserFound returns an error if the entity is not found
func IsUserFound(res sql.Result) error {
	n, err := res.RowsAffected()
	if err != nil {
		return fmt.Errorf("%w: %v", ErrDBError, err)
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
		return fmt.Errorf("%w: %v", ErrDBError, err)
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
		return fmt.Errorf("%w: %v", ErrDBError, err)
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
		return 0, fmt.Errorf("%w: failed to create user %s: %v", ErrDBError, u.Username, err)
	}

	id, err := res.LastInsertId()
	if err != nil {
		return 0, fmt.Errorf("%w: failed to create user %s: %v", ErrDBError, u.Username, err)
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
		return nil, fmt.Errorf("%w: failed to scan DB row: %v", ErrDBError, err)
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
		return nil, fmt.Errorf("%w: failed to list users: %v", ErrDBError, err)
	}

	users, err := ur.scanUsers(rows)
	if err != nil {
		return nil, fmt.Errorf("%w: failed to list users: %v", ErrDBError, err)
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
			return nil, fmt.Errorf("failed to scan row: %v", err)
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
		return fmt.Errorf("%w: failed to update user %s: %v", ErrDBError, username, err)
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
		return fmt.Errorf("%w: failed to delete user %s: %v", ErrDBError, username, err)
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
	"errors"
	"fmt"

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
		return nil, fmt.Errorf("%w: failed to scan DB row: %v", ErrDBError, err)
	}
	return dbs.toDomainEntity(), nil
}

func (sr *secretRepository) scanSecrets(rs *sql.Rows) ([]*secret.Secret, error) {
	var secrets = []*secret.Secret{}

	defer rs.Close()
	for rs.Next() {
		s, err := sr.scanSecret(rs)
		if err != nil {
			return nil, fmt.Errorf("failed to scan row: %v", err)
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
		return 0, fmt.Errorf("%w: failed to create secret %s: %v", ErrDBError, s.Key, err)
	}

	id, err := res.LastInsertId()
	if err != nil {
		return 0, fmt.Errorf("%w: failed to create secret %s: %v", ErrDBError, s.Key, err)
	}
	if id == 0 {
		return 0, fmt.Errorf("%w: secret was not created %s", ErrDBError, s.Key)
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
		return nil, fmt.Errorf("%w: failed to list secrets: %v", ErrDBError, err)
	}

	secrets, err := sr.scanSecrets(rows)
	if err != nil {
		return nil, fmt.Errorf("%w: failed to list secrets: %v", ErrDBError, err)
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
		return fmt.Errorf("%w: failed to delete secret %s: %v", ErrDBError, key, err)
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
	"errors"
	"fmt"

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
	"errors"
	"fmt"

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
		return nil, fmt.Errorf("%w: failed to scan DB row: %v", ErrDBError, err)
	}
	return dbs, nil
}

func (sr *sharedRepository) scanShares(rs *sql.Rows) ([]*shared.Share, error) {
	var shares []*dbShare

	defer rs.Close()
	for rs.Next() {
		dbs, err := sr.scanShare(rs)
		if err != nil {
			return nil, fmt.Errorf("failed to scan row: %v", err)
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
		return 0, fmt.Errorf("failed to begin transaction: %v", err)
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
		return 0, fmt.Errorf("failed to create shared secret: %v", err)
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
		return nil, fmt.Errorf("%w: failed to list shared secrets: %v", ErrDBError, err)
	}

	shares, err := sr.scanShares(rows)
	if err != nil {
		return nil, fmt.Errorf("%w: failed to list shared secrets: %v", ErrDBError, err)
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
		return nil, fmt.Errorf("%w: failed to list shared secrets: %v", ErrDBError, err)
	}

	shares, err := sr.scanShares(rows)
	if err != nil {
		return nil, fmt.Errorf("%w: failed to list shared secrets: %v", ErrDBError, err)
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
		return nil, fmt.Errorf("%w: failed to list shared secrets: %v", ErrDBError, err)
	}

	shares, err := sr.scanShares(rows)
	if err != nil {
		return nil, fmt.Errorf("%w: failed to list shared secrets: %v", ErrDBError, err)
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
		return fmt.Errorf("failed to begin transaction: %v", err)
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
			return fmt.Errorf("%w: %v", ErrDBError, err)
		}
		err = IsShareFound(res)
		if err != nil {
			return err
		}
	}

	err = tx.Commit()
	if err != nil {
		return fmt.Errorf("%w: shared secret was not deleted: %v", ErrDBError, err)
	}
	return nil
}
```


#### Defining errors and updating `import`s


```go
import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"time"

	"github.com/zalgonoise/x/secr/shared"
)

var (
	ErrNotFoundShare = errors.New("shared secret not found")
)
```

___________

### Service preparation

The service layer will have access to the repositories and is the part of the application that will handle the business logic, the flow of the calls, and the validation for the requests.

Besides users and secrets and shares, the service will also handle the authorization and authentication features, which haven't yet been discussed.

For the moment, this is the current state of the service structure (knowing that there isn't yet a service here, but the modules that it will have access to).

```
─ service
   ├─ user.Repository
   ├─ secret.Repository
   ├─ shared.Repository
   └─ keys.Repository
```

To handle authentication and authorization, two specific packages will be added:

- `crypt`: handles general cryptography-related actions for this application
- `authz`: handles authorization, issuing and parsing JWT

Let's prepare those first:

#### Cryptography

There are cryptographic actions to plan ahead, for authentication and authorization:

- User passwords will be salted and hashed. For this we will need a random source for creating the 128-byte salt.
- When users are created, a unique 32-byte key is created, which is used to AES-encrypt their secrets. This key needs to be generated from a random source. An AES encrypt / decrypter object will be useful too.
- If there is no JWT signing key present, one needs to be generated. A 256-byte key needs to be created from a random source.

So, this will be mapped under `crypt/crypt.go`. Starting with the `Cryptographer` and `EcryptDecrypter` interfaces: 

```go
package crypt

// Cryptographer describes the set of cryptographic actions required by the app
type Cryptographer interface {
	// NewSalt generates a new random salt value, of 128 bytes in size
	NewSalt() [128]byte
	// New256Key generates a new random key value, of 256 bytes in size
	New256Key() [256]byte
	// New32Key generates a new random key value, of 256 bytes in size
	New32Key() [32]byte
	// NewCipher generates a new AES cipher based on the input key
	NewCipher(key []byte) EncryptDecrypter
	// Random reads random bytes into the input byte slice
	Random(buffer []byte)
}

// EncrypterDecrypter is a general purpose encryption interface that supports
// an Encrypt and a Decrypt method
type EncryptDecrypter interface {
	// Encrypt will encrypt the input bytes `v` with the EncryptDecrypter key,
	// returning the ciphertext of `v` as a byte slice, and an error
	Encrypt(v []byte) ([]byte, error)

	// Decrypt will decipher the input bytes `v` with the EncryptDecrypter key,
	// returning the plaintext of `v` as a byte slice, and an error
	Decrypt(v []byte) ([]byte, error)
}
```

The `EncryptDecrypter` interface will only have one implementation (for AES), so implementing that, below. This will be a private type that exposes the two methods (`Encrypt` and `Decrypt`). The struct is initialized with a key, and is able to encrypt and decrypt interchangeably, provided it has access to the same key (as in, I shouldn't need the exact same `EncryptDecrypter` to decipher the input text). This implementation allows the `Cryptographer` object to create a new `EncryptDecrypter` from its `NewCipher` method, that takes in the AES cipher key:

```go
type aesEncrypter struct {
	key []byte
}

// Encrypt will encrypt the input bytes `v` with the EncryptDecrypter key,
// returning the ciphertext of `v` as a byte slice, and an error
func (enc aesEncrypter) Encrypt(plaintext []byte) ([]byte, error) {
	c, err := aes.NewCipher(enc.key)
	if err != nil {
		return nil, err
	}

	gcm, err := cipher.NewGCM(c)
	if err != nil {
		return nil, err
	}

	nonce := make([]byte, gcm.NonceSize())
	cryptog.Random(nonce)

	return gcm.Seal(nonce, nonce, plaintext, nil), nil
}

// Decrypt will decipher the input bytes `v` with the EncryptDecrypter key,
// returning the plaintext of `v` as a byte slice, and an error
func (enc aesEncrypter) Decrypt(ciphertext []byte) ([]byte, error) {
	c, err := aes.NewCipher(enc.key)
	if err != nil {
		return nil, err
	}

	gcm, err := cipher.NewGCM(c)
	if err != nil {
		return nil, err
	}

	nonceSize := gcm.NonceSize()
	if len(ciphertext) < nonceSize {
		return nil, errors.New("invalid ciphertext length")
	}

	nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
	plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
	if err != nil {
		return nil, err
	}
	return plaintext, nil
}
```

Now, to implement the `Cryptographer` interface, with a private type (`cryptog`). This package contains an `init()` function that initializes a package-level `Cryptographer` while running a `crypto/rand` random source with a current Unix timestamp seed:

```go
import (
	cryptorand "crypto/rand"
	"encoding/binary"
	"math/rand"
	"sync"
	"time"
)

type cryptographer struct {
	sync.Mutex
	random *rand.Rand
}

var cryptog Cryptographer

func init() {
	cryptog = &cryptographer{}
	var rngSeed int64 = time.Now().Unix()
	_ = binary.Read(cryptorand.Reader, binary.LittleEndian, &rngSeed)
	cryptog.(*cryptographer).random = rand.New(rand.NewSource(rngSeed))
}
```

Cool! The `cryptog` object will be ready to spit out random zeros and ones whenever the package is loaded. Now, for the `Cryptographer` implementation:

**Key generating methods**

The flow of these methods is simple:

- create the output byte array (or take the input slice)
- lock the mutex in the `Cryptographer`
- read random bytes until the array (or slice) is full
- unlock the mutex
- return the output byte array

```go
// NewSalt generates a new random salt value, of 128 bytes in size
func (g *cryptographer) NewSalt() [128]byte {
	salt := [128]byte{}
	g.Lock()
	_, _ = g.random.Read(salt[:])
	g.Unlock()
	return salt
}

// New256Key generates a new random key value, of 256 bytes in size
func (g *cryptographer) New256Key() [256]byte {
	key := [256]byte{}
	g.Lock()
	_, _ = g.random.Read(key[:])
	g.Unlock()
	return key
}

// New32Key generates a new random key value, of 256 bytes in size
func (g *cryptographer) New32Key() [32]byte {
	key := [32]byte{}
	g.Lock()
	_, _ = g.random.Read(key[:])
	g.Unlock()
	return key
}

// Random reads random bytes into the input byte slice
func (g *cryptographer) Random(buffer []byte) {
	g.Lock()
	_, _ = g.random.Read(buffer)
	g.Unlock()
}
```

The `NewCipher` method is simply returning an `aesEncrypter` as a `EncryptDecrypter` interface, with the input key:

```go
// NewCipher generates a new AES cipher based on the input key
func (g *cryptographer) NewCipher(key []byte) EncryptDecrypter {
	return aesEncrypter{
		key: key,
	}
}
```

Awesome. Interface implemented!

Since it's not actually necessary to give access to all these methods (and since both the `cryptog` instance and `cryptographer` type are private, they cannot be yet used outside this package), I've exposed only the methods that matter, as public functions in the package. In this particular case, the `Random()` method is exclusively used on the `aesEcrypter.Encrypt` method call, so it's not exposed:

```go
// NewSalt generates a new random salt value, of 128 bytes in size
func NewSalt() [128]byte {
	return cryptog.NewSalt()
}

// NewKey generates a new random key value, of 256 bytes in size
func New256Key() [256]byte {
	return cryptog.New256Key()
}

// NewKey generates a new random key value, of 256 bytes in size
func New32Key() [32]byte {
	return cryptog.New32Key()
}

// NewCipher generates a new AES cipher based on the input key
func NewCipher(key []byte) EncryptDecrypter {
	return cryptog.NewCipher(key)
}
```

This does it for the cryptography needs in this application, at least for the minimal viable product release.

#### Authorization

While it's already possible to generate JWT signing keys, there isn't yet any logic for handling JWT. This is going into the `authz` package. To plan ahead, this is the behavior that I need when handling JWT:

- Creating a new token (for a user)
- Parsing a user's input token (returning a user and an error)

Let's describe the `Authorizer` interface to do that exactly:

```go
import (
	"context"
	
	"github.com/zalgonoise/x/secr/user"
)

// Authorizer is responsible for generating, refreshing and validating JWT
type Authorizer interface {
	// NewToken returns a new JWT for the user `u`, and an error
	NewToken(ctx context.Context, u *user.User) (string, error)
	// Parse returns the data from a valid JWT
	Parse(ctx context.Context, token string) (*user.User, error)
}
```

Since the `Authorizer` is initialized with a JWT signing key, its type and `new` function can be added too:

```go
type authz struct {
	signingKey []byte
}

// NewAuthorizer initializes an Authorizer with the signing key `signingKey`
func NewAuthorizer(signingKey []byte) Authorizer {
	return &authz{signingKey}
}
```

To implement the `NewToken` method, a data structure is necessary to represent the user object, within the JWT (`jwtUser`). The actual method will create a JWT with a HMAC-SHA256 algorithm; which is set to expire one hour, with this `jwtUser` object embed. The JWT is signed with the `Authorizer`'s signing key and the token (or an error) is returned:


```go
import (
	"context"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt"
	"github.com/zalgonoise/x/secr/user"
)

const jwtExpiryTime = time.Hour * 1

type jwtUser struct {
	Username  string    `json:"username"`
	Name      string    `json:"name"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
}

// NewToken returns a new JWT for the user `u`, and an error
func (a *authz) NewToken(ctx context.Context, u *user.User) (string, error) {
	tok := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
		"exp": time.Now().Add(jwtExpiryTime).Unix(),
		"user": jwtUser{
			Username:  u.Username,
			Name:      u.Name,
			CreatedAt: u.CreatedAt,
			UpdatedAt: u.UpdatedAt,
		},
	})

	token, err := tok.SignedString(a.signingKey)
	if err != nil {
		return "", fmt.Errorf("failed to sign token: %v", err)
	}
	return token, nil
}
```

The `Parse` method runs the input token through the `jwt.Parse()` function, configured with the `parseToken` method. if this operation is successful, the first step is to ensure the expiry claim is present and valid (that the token is not yet expired). Then, the user object is extracted (username and name fields) which are returned as a `*user.User` object.

The `parseToken` method simply configures the parser to accept HMAC-SHA256 algorithms, returning the signing key for the parser to use.

```go
import (
	"context"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt"
	"github.com/zalgonoise/x/secr/user"
)

// Parse returns the data from a valid JWT
func (a *authz) Parse(ctx context.Context, token string) (*user.User, error) {
	tok, err := jwt.Parse(token, a.parseToken)

	if err != nil {
		return nil, fmt.Errorf("failed to parse token: %v", err)
	}
	claims := tok.Claims.(jwt.MapClaims)

	exp, ok := claims["exp"]
	if !ok {
		return nil, ErrMissingExpiry
	}
	expTime := time.Unix(int64(exp.(float64)), 0)
	if time.Now().After(expTime) {
		return nil, ErrExpired
	}
	v, ok := claims["user"]
	if !ok {
		return nil, ErrMissingUser
	}

	valmap := v.(map[string]interface{})
	vUsername := valmap["username"].(string)
	vName := valmap["name"].(string)

	return &user.User{
		Name:     vName,
		Username: vUsername,
	}, nil
}

func (a *authz) parseToken(token *jwt.Token) (interface{}, error) {
	if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
		return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
	}
	return a.signingKey, nil
}
```

Lastly -- and only knowing in advance that the authorization verification will be delivered as HTTP middleware, and that its design decision is to include the (validated) username on the request's context, two helper functions are added, too.

`SignRequest` sets the context value in the request to contain the validated username under a unique context value (string) type.

`GetCaller` extracts this username from an input context

```go
import (
	"context"
	"net/http"
)

type usernameIdentifier string

const contextUsername usernameIdentifier = "secr:username"

// SignRequest sets the input username `u` as a contextUsername context value for
// the HTTP Request `r`'s context
func SignRequest(u string, r *http.Request) *http.Request {
	return r.WithContext(context.WithValue(r.Context(), contextUsername, u))
}

// GetCaller returns the username associated with the HTTP Request `r`, as extracted
// from the request's context, under its contextUsername value (if existing).
//
// Returns the username and an OK-boolean.
func GetCaller(r *http.Request) (string, bool) {
	v := r.Context().Value(contextUsername)
	if v == nil {
		return "", false
	}
	if u, ok := v.(string); ok {
		return u, true
	}
	return "", false
}
```


Now the service will also have access to the Authorizer, to sign JWT as needed:


```
─ service
   ├─ user.Repository
   ├─ secret.Repository
   ├─ shared.Repository
   ├─ keys.Repository
   └─ authz.Authorizer
```


___________

### Service implementation

The service will have access to all the repositories and will contain the core business logic for accessing and writing data within the application. To layout the implementation it is usually easier to follow the same pattern as before: define an interface, then implement it. 

However due to the compexity of the `Service` interface, I will start with its type and the function to create it -- as it is exactly like the diagram above:

```go
type service struct {
	users   user.Repository
	secrets secret.Repository
	shares  shared.Repository
	keys    keys.Repository
	auth    authz.Authorizer
}

// NewService creates a service instance from the input repositories and authorizer
func NewService(
	users user.Repository,
	secrets secret.Repository,
	shares shared.Repository,
	keys keys.Repository,
	auth authz.Authorizer,
) Service {
	return service{
		users:   users,
		secrets: secrets,
		shares:  shares,
		keys:    keys,
		auth:    auth,
	}
}
```

For the `Service` interface, it will contain methods related to the actions exposed by the repositories. This doesn't mean that it will have a literal representation of the methods exposed in the repositories (e.g., it will not simply contain `UsersCreate`, `UsersGet`, etc); the service layer can expose methods that are appropriate for features of the application, which don't really require any added functionality in the corresponding repositories.

An example of that is with shares, as a user can create a share (without limit), create a share with a duration, and create a share with a time limit. However the `shared.Repository` only exposes a single `Create` method. It's the responsibility of the service layer to organize the data to be queried / executed on the persistence layer.

Here is the `Service` interface, containing auth, users, secrets and shares methods:

```go
import (
	"context"
	"time"

	"github.com/zalgonoise/x/secr/authz"
	"github.com/zalgonoise/x/secr/keys"
	"github.com/zalgonoise/x/secr/secret"
	"github.com/zalgonoise/x/secr/shared"
	"github.com/zalgonoise/x/secr/user"
)

// Service defines all the exposed features and functionalities of the secrets store
type Service interface {
	// Login verifies the user's credentials and returns a session and an error
	Login(ctx context.Context, username, password string) (*user.Session, error)
	// Logout signs-out the user `username`
	Logout(ctx context.Context, username string) error
	// ChangePassword updates user `username`'s password after verifying the old one, returning an error
	ChangePassword(ctx context.Context, username, password, newPassword string) error
	// Refresh renews a user's JWT provided it is a valid one. Returns a session and an error
	Refresh(ctx context.Context, username, token string) (*user.Session, error)
	// Validate verifies if a user's JWT is a valid one, returning a boolean and an error
	Validate(ctx context.Context, username, token string) (bool, error)
	// ParseToken reads the input token string and returns the corresponding user in it, or an error
	ParseToken(ctx context.Context, token string) (*user.User, error)

	// CreateUser creates the user under username `username`, with the provided password `password` and name `name`
	// It returns a user and an error
	CreateUser(ctx context.Context, username, password, name string) (*user.User, error)
	// GetUser fetches the user with username `username`. Returns a user and an error
	GetUser(ctx context.Context, username string) (*user.User, error)
	// ListUsers returns all the users in the directory, and an error
	ListUsers(ctx context.Context) ([]*user.User, error)
	// UpdateUser updates the user `username`'s name, found in `updated` user. Returns an error
	UpdateUser(ctx context.Context, username string, updated *user.User) error
	// DeleteUser removes the user with username `username`. Returns an error
	DeleteUser(ctx context.Context, username string) error

	// CreateSecret creates a secret with key `key` and value `value` (as a slice of bytes), for the
	// user `username`. It returns an error
	CreateSecret(ctx context.Context, username string, key string, value []byte) error
	// GetSecret fetches the secret with key `key`, for user `username`. Returns a secret and an error
	GetSecret(ctx context.Context, username string, key string) (*secret.Secret, error)
	// ListSecrets retuns all secrets for user `username`. Returns a list of secrets and an error
	ListSecrets(ctx context.Context, username string) ([]*secret.Secret, error)
	// DeleteSecret removes a secret with key `key` from the user `username`. Returns an error
	DeleteSecret(ctx context.Context, username string, key string) error

	// CreateShare shares the secret with key `secretKey` belonging to user with username `owner`, with users `targets`.
	// Returns the resulting shared secret, and an error
	CreateShare(ctx context.Context, owner, secretKey string, targets ...string) (*shared.Share, error)
	// ShareFor is similar to CreateShare, but sets the shared secret to expire after `dur` Duration
	ShareFor(ctx context.Context, owner, secretKey string, dur time.Duration, targets ...string) (*shared.Share, error)
	// ShareFor is similar to CreateShare, but sets the shared secret to expire after `until` Time
	ShareUntil(ctx context.Context, owner, secretKey string, until time.Time, targets ...string) (*shared.Share, error)
	// GetShare fetches the shared secret belonging to `username`, with key `secretKey`, returning it as a
	// shared secret and an error
	GetShare(ctx context.Context, username, secretKey string) ([]*shared.Share, error)
	// ListShares fetches all the secrets the user with username `username` has shared with other users
	ListShares(ctx context.Context, username string) ([]*shared.Share, error)
	// DeleteShare removes the users `targets` from a shared secret with key `secretKey`, belonging to `username`. Returns
	// an error
	DeleteShare(ctx context.Context, username, secretKey string, targets ...string) error
	// PurgeShares removes the shared secret completely, so it's no longer available to the users it was
	// shared with. Returns an error
	PurgeShares(ctx context.Context, username, secretKey string) error
}
```

This seems like a lot! Well it's not as much as it seems, especially when broken down. While DDD can be a bit more verbose, I take it that the main point is to delimit the responsibility of a certain domain within that domain. This allows code to look more readable and organized as the codebase expands. Let's create a few files in the `service` directory:

```
.
└─ service
    ├─ secrets.go -- implements Secrets-related methods
    ├─ service.go  -- describes the Service interface and type 
    ├─ sessions.go -- implements Sessions-related methods
    ├─ shares.go -- implements Shares-related methods
    └─ users.go -- implements Users-related methods
```

Now, to implement the `Service` interface!


#### Implementing Service Users methods

Taking a look at the interface methods beforehand, it's noticeable that this is the point where the caller input is *processed* into a domain object. A method takes in some arguments like a username, password and name, and returns a `*user.User`.

Validating this input happens on this layer, precisely, and input validation is the first thing that happens in all of these methods. To better organize validation, I create a `validate.go` file under a certain entity's folder:

```
.
└─ user
    ├─ repository.go
    ├─ user.go 
    └─ validate.go -- user-related validator functions
```

For users, it's important to validate the input username, name and password (as those are the customizable fields that makes up a user).

Here's the username validation:

```go
package user

import (
	"errors"
	"regexp"
)

var (
	ErrEmptyUsername   = errors.New("username cannot be empty")
	ErrShortUsername   = errors.New("username is too short")
	ErrLongUsername    = errors.New("username is too long")
	ErrInvalidUsername = errors.New("invalid username")
)

const (
	usernameMinLength = 3
	usernameMaxLength = 25
)

var usernameRegex = regexp.MustCompile(`[a-z0-9]+[a-z0-9\-_]+[a-z0-9]+`)

// ValidateUsername verifies if the input username is valid, returning an error
// if invalid
func ValidateUsername(username string) error {
	if username == "" {
		return ErrEmptyUsername
	}
	if len(username) < usernameMinLength {
		return ErrShortUsername
	}
	if len(username) > usernameMaxLength {
		return ErrLongUsername
	}
	if match := usernameRegex.FindString(username); match != username || username == RootUsername {
		return ErrInvalidUsername
	}
	return nil
}
```

For the username, it has to be an alphanumeric string with dashes and underscores as (optional) separators. There is also a size range (between 3 and 25 characters). 

This is very straight-forward using a regular expression, with some checks for length (and empty string), so that the appropriate error is returned.

The name validation is very similar:

```go
import (
	"errors"
	"regexp"
)


var (
	ErrEmptyName   = errors.New("name cannot be empty")
	ErrShortName   = errors.New("name is too short")
	ErrLongName    = errors.New("name is too long")
	ErrInvalidName = errors.New("invalid name")
)

const (
	nameMinLength = 2
	nameMaxLength = 25
)

var nameRegex     = regexp.MustCompile(`[a-zA-Z]+[\s]?[a-zA-Z]+`)

// ValidateName verifies if the input name is valid, returning an error
// if invalid
func ValidateName(name string) error {
	if name == "" {
		return ErrEmptyName
	}
	if len(name) < nameMinLength {
		return ErrShortName
	}
	if len(name) > nameMaxLength {
		return ErrLongName
	}
	if match := nameRegex.FindString(name); match != name {
		return ErrInvalidName
	}
	return nil
}
```

It's super similar to the username validation, with a different regular expression to allow capital letters and spaces (instead of underscores and dashes).

The password validation has a slightly different approach:

```go

import (
	"errors"
	"regexp"
)

var (
	ErrEmptyPassword   = errors.New("password cannot be empty")
	ErrShortPassword   = errors.New("password is too short")
	ErrLongPassword    = errors.New("password is too long")
	ErrInvalidPassword = errors.New("invalid password")
)

const (
	passwordMinLength       = 7
	passwordMaxLength       = 300
	PasswordCharRepeatLimit = 4
	passwordAllowedChars    = `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-_~!@#$%^&*()=[]{}'\"|,./<>?;:`
)

var passwordCharMap = map[rune]struct{}{}

func init() {
	for _, c := range passwordAllowedChars {
		passwordCharMap[c] = struct{}{}
	}
}

// ValidatePassword verifies if the input password is valid, returning an error
// if invalid
func ValidatePassword(password string) error {
	if password == "" {
		return ErrEmptyPassword
	}
	if len(password) < passwordMinLength {
		return ErrShortPassword
	}
	if len(password) > passwordMaxLength {
		return ErrLongPassword
	}
	return validatePasswordCharacters(password)
}

func validatePasswordCharacters(password string) error {
	var cur rune
	var count int = 1
	for _, c := range password {
		switch c {
		case cur:
			count++
			if count >= PasswordCharRepeatLimit {
				return ErrInvalidPassword
			}
		default:
			cur = c
			count = 1
		}
		if _, ok := passwordCharMap[c]; !ok {
			return ErrInvalidPassword
		}
	}
	return nil
}
```

For the user passwords, there is no need to have the overhead of a regular expression (I benchmarked it, takes about 60% the time when not using regular expressions for this one).

I define all valid characters as a string constant and populate a rune-to-empty-struct map (`struct{}` has a zero-memory footprint) with these chars on an `init()` function. This allows to build the map when the package is loaded.

The caracter validator function is (in a nutshell) running through all characters in the password and checking if it's present in `passwordCharMap`:

```go
	for _, c := range password {
		// (...)
		if _, ok := passwordCharMap[c]; !ok {
			return ErrInvalidPassword
		}
	}
	return nil
```

I also added a constraint for a user not to repeat the same character more than 3 times (so, no `000000` passwords and similar), and all of these additional checks are optional. These could be refactored in a different way (maybe to run several checks simultaneously); however validation isn't of the biggest importance in this app, at this point of implementation.

Great! User validation is ready, and it's time to jump into the service implementation for user-related methods.

Just like before, starting off with a blank canvas:

```go
package service

import (
	"context"

	"github.com/zalgonoise/x/secr/user"
)

// CreateUser creates the user under username `username`, with the provided password `password` and name `name`
// It returns a user and an error
func (s service) CreateUser(ctx context.Context, username, password, name string) (*user.User, error) {
	return nil, nil
}

// GetUser fetches the user with username `username`. Returns a user and an error
func (s service) GetUser(ctx context.Context, username string) (*user.User, error) {
	return nil, nil
}

// ListUsers returns all the users in the directory, and an error
func (s service) ListUsers(ctx context.Context) ([]*user.User, error) {
	return nil, nil
}

// UpdateUser updates the user `username`'s name, found in `updated` user. Returns an error
func (s service) UpdateUser(ctx context.Context, username string, updated *user.User) error {
	return nil
}

// DeleteUser removes the user with username `username`. Returns an error
func (s service) DeleteUser(ctx context.Context, username string) error {
	return nil
}
```

**`service.CreateUser`**

This action receives a request from the transport layer (HTTP) and executes it after validating the input.

For this the following is required:

1. Validating the user's input (username, password and name), with the appropriate validators

```go
	if err := user.ValidateUsername(username); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidPassword, err)
	}
	if err := user.ValidateName(name); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidName, err)
	}
```

2. Checking if the username already exists. This will be a `Get` call to the user.Repository

```go
	_, err := s.users.Get(ctx, username)
	if err == nil || !errors.Is(sqlite.ErrNotFoundUser, err) {
		return nil, fmt.Errorf("failed to create user: %v", ErrAlreadyExistsUser)
	}
```

3. Creating a new salt value
4. Generating the password hash, from the input password with the salt appended to it

```go
	salt := crypt.NewSalt()
	hashedPassword := sha256.Sum256(append([]byte(password), salt[:]...))
```

5. Base64-encoding both the salt and the password hash

```go
	encSalt := base64.StdEncoding.EncodeToString(salt[:])
	encHash := base64.StdEncoding.EncodeToString(hashedPassword[:])
```

6. Creating the (domain) user object with the input data and hash / salt

```go
	u := &user.User{
		Username: username,
		Hash:     encHash,
		Salt:     encSalt,
		Name:     name,
	}
```


7. Creating the user with this object, with a `Create` call to the user.Repository

```go
	id, err := s.users.Create(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to create user %s: %v", username, err)
	}
	u.ID = id
```
8. Generating a new 32-byte AES key for this user
9. Storing that key in the user's bucket (as a first entry, too), with a `Set` call to the keys.Repository

```go
	key := crypt.New32Key()
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, key[:])
	if err != nil {
		return nil, fmt.Errorf("failed to create user %s: %v", username, err)
	}

	return u, nil
```

There is something to consider, as I am working with Bolt DB *and* with SQLite. If I create the user in the user.Repository and the keys.Repository call fails, I should rollback the user creation.

Usually this is done with a service that will act as a transactioner, and has access to all repositories that the service needs. The transactioner is able to commit and rollback a transaction.

For a simpler approach, I define a transactioner as an interface with two methods:
- `Add(func() error)` - appends a new rollback function to the transactioner
- `Rollback(error) error` - executes all rollback functions, and stores their errors. Returns the input error wrapping the rollback errors (if any)

For this, I added the `service/transactioner.go` file, to act as such a rollback machine. The point is being able to generate a transaction, and to append rollback functions to the transaction. If an error is raised, the transaction is rolledback and an error is returned accordingly.

Take a look at `service/transactioner.go`:

```go
package service

import "fmt"

type Transactioner interface {
	Rollback(error) error
	Add(RollbackFn)
}

type RollbackFn func() error

type transactioner struct {
	r   []RollbackFn
	err error
}

func newTx() Transactioner {
	return &transactioner{}
}

func (tx *transactioner) Rollback(input error) error {
	for _, rb := range tx.r {
		err := rb()
		if err != nil {
			if tx.err == nil {
				tx.err = err
				continue
			}
			tx.err = fmt.Errorf("%w -- %v", tx.err, err)
		}
	}
	if tx.err == nil {
		return input
	}
	return fmt.Errorf("%w -- rollback error: %v", input, tx.err)
}

func (tx *transactioner) Add(r RollbackFn) {
	tx.r = append(tx.r, r)
}
```

Since the user is being created in user.Repository first (to generate an ID for them), I define the opposite action as a transaction's rollback func. If the keys.Repository action fails, I can call it:

```go
	id, err := s.users.Create(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to create user %s: %v", username, err)
	}
	u.ID = id

	tx := newTx()
	tx.Add(func() error {
		return s.users.Delete(ctx, username)
	})

	// generate a new private key for this user, and store it
	key := crypt.New32Key()
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, key[:])
	if err != nil {
		return nil, tx.Rollback(fmt.Errorf("failed to create user %s: %v", username, err))
	}
```

The whole thing looks like this:

```go
// CreateUser creates the user under username `username`, with the provided password `password` and name `name`
// It returns a user and an error
func (s service) CreateUser(ctx context.Context, username, password, name string) (*user.User, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidPassword, err)
	}
	if err := user.ValidateName(name); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidName, err)
	}

	// check if user exists
	_, err := s.users.Get(ctx, username)
	if err == nil || !errors.Is(sqlite.ErrNotFoundUser, err) {
		return nil, fmt.Errorf("failed to create user: %v", ErrAlreadyExistsUser)
	}

	// generate hash from password
	salt := crypt.NewSalt()
	hashedPassword := sha256.Sum256(append([]byte(password), salt[:]...))

	encSalt := base64.StdEncoding.EncodeToString(salt[:])
	encHash := base64.StdEncoding.EncodeToString(hashedPassword[:])

	// create the user
	u := &user.User{
		Username: username,
		Hash:     encHash,
		Salt:     encSalt,
		Name:     name,
	}
	id, err := s.users.Create(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to create user %s: %v", username, err)
	}
	u.ID = id

	tx := newTx()
	tx.Add(func() error {
		return s.users.Delete(ctx, username)
	})

	// generate a new private key for this user, and store it
	key := crypt.New32Key()
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, key[:])
	if err != nil {
		return nil, tx.Rollback(fmt.Errorf("failed to create user %s: %v", username, err))
	}

	return u, nil
}
```

**`service.GetUser`**

This method fetches a user from the user.Repository, by its username. It's much simpler than `CreateUser` since it just validates the username and does a `Get` call to the user.Repository:

```go
// GetUser fetches the user with username `username`. Returns a user and an error
func (s service) GetUser(ctx context.Context, username string) (*user.User, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to get user %s: %v", username, err)
	}
	return u, nil
}
```


**`service.ListUsers`**

Same goes for `ListUsers` -- but this one doesn't require any validation:

```go
// ListUsers returns all the users in the directory, and an error
func (s service) ListUsers(ctx context.Context) ([]*user.User, error) {
	users, err := s.users.List(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to list users: %v", err)
	}
	return users, nil
}
```


**`service.UpdateUser`**

This method will be used to update a user's (general) information -- in the context of this application, this only applies to the user's name. The password changes are handled by a separate (service) method.

To summarize the actions that I need to take here:

1. Validating the user's input. This is the input username (identifying the user), and a user object with their updated version.

For the username there is already a validator function, but for the updated object:

- It must not be nil
- Its name (and any data being updated) needs to be validated

```go
	if err := user.ValidateUsername(username); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	if updated == nil {
		return fmt.Errorf("%w: updated user cannot be nil", ErrInvalidUser)
	}
	if err := user.ValidateName(updated.Name); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidName, err)
	}
```

2. Get this user, from the user.Repository (from their username)

```go
	currentUser, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch original user %s: %v", username, err)
	}
```

3. Verify if there are any actual changes to be made in this action. If not, simply return without an error (the desired state is already achieved).

```go
	if updated.Name == currentUser.Name && updated.Hash == currentUser.Hash {
		return nil
	}
```

4. Update the user by calling the `Update` user.Repository method

```go
	err = s.users.Update(ctx, username, updated)
	if err != nil {
		return fmt.Errorf("failed to update user %s: %v", username, err)
	}

	return nil
```

Here's the whole thing:

```go
// UpdateUser updates the user `username`'s name, found in `updated` user. Returns an error
func (s service) UpdateUser(ctx context.Context, username string, updated *user.User) error {
	if err := user.ValidateUsername(username); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	if updated == nil {
		return fmt.Errorf("%w: updated user cannot be nil", ErrInvalidUser)
	}
	if err := user.ValidateName(updated.Name); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidName, err)
	}

	currentUser, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch original user %s: %v", username, err)
	}
	if updated.Name == currentUser.Name && updated.Hash == currentUser.Hash {
		// no changes to be made
		return nil
	}

	err = s.users.Update(ctx, username, updated)
	if err != nil {
		return fmt.Errorf("failed to update user %s: %v", username, err)
	}

	return nil
}
```



**`service.DeleteUser`**

While this method seems to only remove a user, there is also the user's secrets and shares to consider, which need to be handled appropriately. As such, to avoid cluttering the database and allowing access to secrets beyond an owner's removal, the user's shares must be removed, and then the user's secrets must be removed before finally deleting the user.

These are the required actions to remove a user:

1. Validate the input (the username)

```go
	if err := user.ValidateUsername(username); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
```

2. Fetch the user by this username. If no user is found under this username, simply return (no change in state) 

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		if errors.Is(sqlite.ErrNotFoundUser, err) {
			// no change in state
			return nil
		}
		return fmt.Errorf("failed to fetch original user %s: %v", username, err)
	}
```

3. Get all of the user's shared secrets, and remove them

```go
	shares, err := s.shares.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secrets: %v", err)
	}
	for _, sh := range shares {
		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return err
		}
	}
```

4. Get all of the user's secrets, and remove them

```go
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to list user secrets: %v", err)
	}
	for _, secr := range secrets {
		err := s.secrets.Delete(ctx, username, secr.Key)
		if err != nil {
			return err
		}
	}
```

5. Remove the user's private key in the keys.Repository

```go
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return fmt.Errorf("failed to delete user %s's key: %v", username, err)
	}
```

6. Finally, remove the user

```go
	err = s.users.Delete(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to delete user %s: %v", username, err)
	}

	return nil
```

This is great, but like in the `CreateUser` method, its best to rollback any changes in case an error is raised.

The basic rollback model will be OK here, as it is capable of accumulating multiple `RollbackFn` and execute them all if an error is raised in any of the calls.

Reviewing some of the steps with this in mind:

3. Get all of the user's shared secrets, and remove them

```go
	tx := newTx()

	shares, err := s.shares.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secrets: %v", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})

		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %v", err))
		}
	}
```

4. Get all of the user's secrets, and remove them

```go
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to list user secrets: %v", err)
	}
	for _, secr := range secrets {
		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, secr)
			return err
		})

		err := s.secrets.Delete(ctx, username, secr.Key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove secret: %v", err))
		}
	}
```

5. Remove the user's private key in the keys.Repository

```go
	upk, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return fmt.Errorf("failed to fetch user %s's key: %v", username, err)
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, upk)
	})

	// delete private key
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s's key: %v", username, err))
	}
```

6. Finally, remove the user

```go
	err = s.users.Delete(ctx, username)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s: %v", username, err))
	}

	return nil
```

Again, it's important to note that a (decent) transaction would work directly with the repositories, and this implementation is acting more like a batch executor (in case an error is raised). Knowing this and embracing the size of the app, this is enough.

Here is the whole method body:

```go
// DeleteUser removes the user with username `username`. Returns an error
func (s service) DeleteUser(ctx context.Context, username string) error {
	if err := user.ValidateUsername(username); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		if errors.Is(sqlite.ErrNotFoundUser, err) {
			// no change in state
			return nil
		}
		return fmt.Errorf("failed to fetch original user %s: %v", username, err)
	}

	tx := newTx()

	// remove all shares
	shares, err := s.shares.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secrets: %v", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})

		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %v", err))
		}
	}

	// remove all secrets
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to list user secrets: %v", err)
	}
	for _, secr := range secrets {
		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, secr)
			return err
		})

		err := s.secrets.Delete(ctx, username, secr.Key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove secret: %v", err))
		}
	}

	// get user's private key for rollback func
	upk, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return fmt.Errorf("failed to fetch user %s's key: %v", username, err)
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, upk)
	})

	// delete private key
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s's key: %v", username, err))
	}

	// delete user
	err = s.users.Delete(ctx, username)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s: %v", username, err))
	}

	return nil
}
```


#### Implementing Service Secrets methods

Just like the user, the secrets will need some sort of validation when being handled, for the keys and the values.

I added a `validate.go` file under the secrets directory for this purpose: 

```
.
└─ secret
    ├─ repository.go
    ├─ secret.go
    └─ validate.go
```

Here is the keys validation:

```go
import (
	"errors"
	"regexp"
	"strings"

	"github.com/zalgonoise/x/secr/keys"
	"github.com/zalgonoise/x/secr/user"
)

var (
	ErrEmptyKey   = errors.New("key cannot be empty")
	ErrLongKey    = errors.New("key is too long")
	ErrInvalidKey = errors.New("invalid key")
	ErrInvalidSharedKey = errors.New("invalid shared key")
)

const keyMaxLength   = 20

var (
	keyRegex = regexp.MustCompile(`[a-z0-9]+[a-z0-9\-_:]+[a-z0-9]+`)
)

// ValidateKey verifies if the input secret's key is valid, returning an error
// if invalid
func ValidateKey(key string) (bool, error) {
	if key == "" {
		return false, ErrEmptyKey
	}
	if len(key) > keyMaxLength {
		return false, ErrLongKey
	}
	if match := keyRegex.FindString(key); match != key {
		return false, ErrInvalidKey
	}
	if key == keys.UniqueID || key == keys.TokenKey {
		return false, ErrEmptyKey
	}
	if strings.Contains(key, ":") {
		split := strings.SplitN(key, ":", 1)
		if len(split) != 2 {
			return false, ErrInvalidSharedKey
		}
		if err := user.ValidateUsername(split[0]); err != nil {
			return false, err
		}
		if isShared, err := ValidateKey(split[1]); err != nil || isShared {
			return false, err
		}
		return true, nil
	}
	return false, nil
}
```


Keys will have a limit in size, and a regex matcher similar to the usernames (that also allows `:`, more on that soon). Also, secret keys must not contain the reserved keys for JWT and user's private keys. As a bit of sugar, this validator will return a boolean and an error.

The thing about `:` in secret keys is that later, in the shared package and List / Get operations, shared secrets append the owner's username to the key with `:` as separator (like `joe:secret` is a secret key visible to users that `joe` has shared `secret` with). The boolean from this validator is an `isShared` boolean, determining if the `:` is present thus if it is a shared secret.

> Note: for shared secrets, as the key is validated and determined there is a `:`; the secret is split by this symbol. The first half is validated as a username, and the second half is validated as a secret key, whereas this one must not be a shared key.


Here is the values validator: 

```go
var (
	ErrEmptyValue = errors.New("value cannot be empty")
	ErrLongValue  = errors.New("value is too long")
)

const valueMaxLength = 8192

// ValidateValue verifies if the input secret's value is valid, returning an error
// if invalid
func ValidateValue(value []byte) error {
	if len(value) == 0 {
		return ErrEmptyValue
	}
	if len(value) > valueMaxLength {
		return ErrLongValue
	}
	return nil
}
```

Validating the secret values is merely whether the value is empty, or if it exceeds a maximum length (currently set to 8kb).

Having the secrets validation done, means it is time to jump into the service implementation.

It begins with a blank canvas:

```go
package service

import (
	"context"

	"github.com/zalgonoise/x/secr/secret"
)

// CreateSecret creates a secret with key `key` and value `value` (as a slice of bytes), for the
// user `username`. It returns an error
func (s service) CreateSecret(ctx context.Context, username string, key string, value []byte) error {
	return nil
}

// GetSecret fetches the secret with key `key`, for user `username`. Returns a secret and an error
func (s service) GetSecret(ctx context.Context, username string, key string) (*secret.Secret, error) {
	return nil, nil
}

// ListSecrets retuns all secrets for user `username`. Returns a list of secrets and an error
func (s service) ListSecrets(ctx context.Context, username string) ([]*secret.Secret, error) {
	return nil, nil
}

// DeleteSecret removes a secret with key `key` from the user `username`. Returns an error
func (s service) DeleteSecret(ctx context.Context, username string, key string) error {
	return nil
}
```


**`service.CreateSecret`**

This action stores (or overwrites) a secret


```go
// CreateSecret creates a secret with key `key` and value `value` (as a slice of bytes), for the
// user `username`. It returns an error
func (s service) CreateSecret(ctx context.Context, username string, key string, value []byte) error {
	if err := user.ValidateUsername(username); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(key); err != nil || isShared {
		return fmt.Errorf("%w: %v", ErrInvalidKey, err)
	}
	if err := secret.ValidateValue(value); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidValue, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %v", err)
	}

	tx := newTx()

	// check if secret already exists
	oldSecr, err := s.secrets.Get(ctx, username, key)
	if err != nil && !errors.Is(sqlite.ErrNotFoundSecret, err) {
		return fmt.Errorf("failed to fetch previous secret under this key: %v", err)
	}
	if oldSecr != nil {
		// remove old shares if they exist
		shares, err := s.shares.Get(ctx, username, oldSecr.Key)
		if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
			return fmt.Errorf("failed to fetch previous shared secrets: %v", err)
		}
		for _, sh := range shares {
			tx.Add(func() error {
				_, err := s.shares.Create(ctx, sh)
				return err
			})
			err := s.shares.Delete(ctx, sh)
			if err != nil {
				return tx.Rollback(fmt.Errorf("failed to remove old share: %v", err))
			}
		}

		// get encrypted value for existing secret (for RollbackFn)
		val, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to fetch old secret's value: %v", err))
		}

		tx.Add(func() error {
			return s.keys.Set(ctx, keys.UserBucket(u.ID), key, val)
		})

		// remove it
		err = s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove old secret's value: %v", err))
		}

		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, oldSecr)
			return err
		})

		// remove the secret's metadata
		err = s.secrets.Delete(ctx, username, key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove old secret: %v", err))
		}
	}

	// encrypt secret with user's key:
	// fetch the key
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to get user's private key: %v", err))
	}

	// encrypt value with user's private key
	cipher := crypt.NewCipher(cipherKey)
	encValue, err := cipher.Encrypt(value)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to encrypt value: %v", err))
	}

	// store encrypted value
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), key, encValue)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to store the secret: %v", err))
	}
	tx.Add(func() error {
		if oldSecr == nil {
			return s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
		}
		return nil
	})

	secr := &secret.Secret{
		Key: key,
	}

	// create secret
	id, err := s.secrets.Create(ctx, username, secr)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to create the secret: %v", err))
	}
	secr.ID = id

	return nil
}
```

**`service.GetSecret`**

```go
// GetSecret fetches the secret with key `key`, for user `username`. Returns a secret and an error
func (s service) GetSecret(ctx context.Context, username string, key string) (*secret.Secret, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	isShared, err := secret.ValidateKey(key)
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidKey, err)
	}

	// handle fetching an owned secret vs a shared secret
	if isShared {
		userAndKey := strings.Split(key, ":")
		return s.getSharedSecret(ctx, userAndKey[0], userAndKey[1], username)
	}

	return s.getSecret(ctx, username, key)
}

func (s service) getSecret(ctx context.Context, username, key string) (*secret.Secret, error) {
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user: %v", err)
	}

	// fetch secret('s metadata )
	secr, err := s.secrets.Get(ctx, username, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the secret: %v", err)
	}

	// fetch user's private key to decode encrypted secret
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the user's private key: %v", err)
	}
	cipher := crypt.NewCipher(cipherKey)

	// fetch secret's value
	encValue, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the secret: %v", err)
	}

	// decrypt value with user's private key
	decValue, err := cipher.Decrypt(encValue)
	if err != nil {
		return nil, fmt.Errorf("failed to decrypt value: %v", err)
	}

	secr.Value = string(decValue)
	return secr, nil
}

func (s service) getSharedSecret(ctx context.Context, owner, key, target string) (*secret.Secret, error) {
	// get the original share (as if it was the owner)
	sh, err := s.shares.Get(ctx, owner, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret metadata: %v", err)
	}

	var validShare *shared.Share
	for _, share := range sh {
		// validate share's deadline first
		if share.Until != nil {
			if time.Now().After(*share.Until) {
				// remove it if expired
				err := s.shares.Delete(ctx, share)
				if err != nil {
					return nil, fmt.Errorf("failed to remove expired shared secret: %v", err)
				}
				continue
			}
		}
		// check if the caller is one of the targets
		for _, t := range share.Target {
			if t == target {
				validShare = share
				break
			}
		}
	}

	// no share found, caller requests a share that doesn't exist
	// or doesn't have access to
	if validShare == nil {
		return nil, ErrZeroShares
	}

	// fetch the deciphered secret
	secr, err := s.getSecret(ctx, owner, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret: %v", err)
	}
	// erase creation time; set key as `user:key`
	secr.CreatedAt = time.Time{}
	secr.Key = fmt.Sprintf("%s:%s", owner, secr.Key)
	return secr, nil
}
```

**`service.ListSecrets`**

```go
// ListSecrets retuns all secrets for user `username`. Returns a list of secrets and an error
func (s service) ListSecrets(ctx context.Context, username string) ([]*secret.Secret, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user: %v", err)
	}

	// fetch secret('s metadata )
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to list the user's secrets: %v", err)
	}

	// fetch user's private key to decode encrypted secret
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the user's private key: %v", err)
	}
	cipher := crypt.NewCipher(cipherKey)

	// fetch and decode the value for each secret
	for _, secr := range secrets {
		// fetch secret's value
		encValue, err := s.keys.Get(ctx, keys.UserBucket(u.ID), secr.Key)
		if err != nil {
			return nil, fmt.Errorf("failed to fetch the secret: %v", err)
		}

		// decrypt value with user's private key
		decValue, err := cipher.Decrypt(encValue)
		if err != nil {
			return nil, fmt.Errorf("failed to decrypt value: %v", err)
		}

		secr.Value = string(decValue)
	}

	// aggregate secrets that are shared with this user
	sharedSecrets, err := s.shares.ListTarget(ctx, username)
	if err != nil {
		return secrets, fmt.Errorf("failed to fetch secrets shared with %s: %v", username, err)
	}
	for _, sh := range sharedSecrets {
		// extract secret from shared secret
		sharedSecr, err := s.getSharedSecret(ctx, sh.Owner, sh.SecretKey, username)
		if err != nil {
			if errors.Is(ErrZeroShares, err) {
				continue
			}
			return secrets, fmt.Errorf("failed to fetch secrets shared with %s: %v", username, err)
		}
		// append results
		secrets = append(secrets, sharedSecr)
	}

	return secrets, nil
}
```

**`service.DeleteSecret`**

```go
// DeleteSecret removes a secret with key `key` from the user `username`. Returns an error
func (s service) DeleteSecret(ctx context.Context, username string, key string) error {
	if err := user.ValidateUsername(username); err != nil {
		return fmt.Errorf("%w: %v", ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(key); err != nil || isShared {
		return fmt.Errorf("%w: %v", ErrInvalidKey, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %v", err)
	}

	tx := newTx()

	// remove shares for this secret
	shares, err := s.shares.Get(ctx, username, key)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return fmt.Errorf("failed to list shared secrets: %v", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})
		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %v", err))
		}
	}

	// fetch the encrypted secret (for RollbackFn)
	secr, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		if errors.Is(bolt.ErrEmptyBucket, err) {
			// nothing to delete, no changes in state
			return nil
		}
		return tx.Rollback(fmt.Errorf("failed to fetch the secret: %v", err))
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), key, secr)
	})

	// delete it
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to remove secret: %v", err))
	}

	// fetch the secret's metadata (for RollbackFn)
	secretMeta, err := s.secrets.Get(ctx, username, key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to fetch secret: %v", err))
	}

	tx.Add(func() error {
		_, err := s.secrets.Create(ctx, username, secretMeta)
		return err
	})

	// delete it
	err = s.secrets.Delete(ctx, username, key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to remove secret: %v", err))
	}

	return nil
}
```

#### Implementing Service Shares methods

Let's *share* a blank canvas for this one:

```go
package service

import (
	"context"
	"time"

	"github.com/zalgonoise/x/secr/shared"
)

// CreateShare shares the secret with key `secretKey` belonging to user with username `owner`, with users `targets`.
// Returns the resulting shared secret, and an error
func (s service) CreateShare(ctx context.Context, owner, secretKey string, targets ...string) (*shared.Share, error) {
	return nil, nil
}

// ShareFor is similar to CreateShare, but sets the shared secret to expire after `dur` Duration
func (s service) ShareFor(ctx context.Context, owner, secretKey string, dur time.Duration, targets ...string) (*shared.Share, error) {
	return nil, nil
}

// ShareFor is similar to CreateShare, but sets the shared secret to expire after `until` Time
func (s service) ShareUntil(ctx context.Context, owner, secretKey string, until time.Time, targets ...string) (*shared.Share, error) {
	return nil, nil
}

// GetShare fetches the shared secret belonging to `username`, with key `secretKey`, returning it as a
// shared secret and an error
func (s service) GetShare(ctx context.Context, username, secretKey string) ([]*shared.Share, error) {
	return nil, nil
}

// ListShares fetches all the secrets the user with username `username` has shared with other users
func (s service) ListShares(ctx context.Context, username string) ([]*shared.Share, error) {
	return nil, nil
}

// DeleteShare removes the users `targets` from a shared secret with key `secretKey`, belonging to `username`. Returns
// an error
func (s service) DeleteShare(ctx context.Context, owner, secretKey string, targets ...string) error {
	return nil
}

// PurgeShares removes the shared secret completely, so it's no longer available to the users it was
// shared with. Returns an error
func (s service) PurgeShares(ctx context.Context, owner, secretKey string) error {
	return nil
}
```

#### Implementing Service Sessions methods

A blank canvas *session* for this implementation, too:

```go
package service

import (
	"context"

	"github.com/zalgonoise/x/secr/user"
)

// Login verifies the user's credentials and returns a session and an error
func (s service) Login(ctx context.Context, username, password string) (*user.Session, error) {
	return nil, nil
}

// Logout signs-out the user `username`
func (s service) Logout(ctx context.Context, username string) error {
	return nil
}

// ChangePassword updates user `username`'s password after verifying the old one, returning an error
func (s service) ChangePassword(ctx context.Context, username, password, newPassword string) error {
	return nil
}

// Refresh renews a user's JWT provided it is a valid one. Returns a session and an error
func (s service) Refresh(ctx context.Context, username, token string) (*user.Session, error) {
	return nil, nil
}

// Validate verifies if a user's JWT is a valid one, returning a boolean and an error
func (s service) Validate(ctx context.Context, username, token string) (bool, error) {
	return false, nil
}

func (s service) ParseToken(ctx context.Context, token string) (*user.User, error) {
	return nil, nil
}
```

___________

### HTTP API implementation 

#### Handling endpoints that require auth
___________

### Setting up factories


___________

### The `cmd` package

#### App configuration

#### CLI flags

#### OS environment variables



___________

### Observability and middleware


___________

### Wrapping up


___________

### Final thoughts