[back](keys_package.md) | [index](index.md)

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
		return nil, errors.Join(ErrDBError, err)
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
			return fmt.Errorf("failed to get / create bucket: %w", err)
		}

		err = b.Put([]byte(k), []byte(v))
		if err != nil {
			return fmt.Errorf("failed to set key-value: %w", err)
		}

		return nil
	})
	if err != nil {
		return errors.Join(ErrDBError, err)
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
			return fmt.Errorf("failed to delete key %s in the bucket %s: %w", k, bucket, err)
		}
		return nil
	})

	if err != nil {
		if errors.Is(err, ErrEmptyBucket) {
			return err
		}
		return errors.Join(ErrDBError, err)
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
			return fmt.Errorf("failed to delete the bucket %s: %w", bucket, err)
		}
		return nil
	})

	if err != nil {
		return errors.Join(ErrDBError, err)
	}
	return nil
}
```

#### Defining errors and updating `import`s

Since the methods refer some error types that seem to be part of this package, they need to be defined; these are super simple errors to identify the issue better with, for example, `errors.Is()`.

```go
import (
	"context"
	"fmt"

	"github.com/zalgonoise/x/errors"
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

[next](sqlite.md)