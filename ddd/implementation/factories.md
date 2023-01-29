[back](http_api_implementation.md) | [index](index.md)

### Setting up factories

The factory functions are entirely optional and I only include it in projects that make sense, namely those that use different backends and tech stacks for particular features (say, one of the app's options allows to pick different databases).

These will still be implemented for the sake of documenting the process, and because it's a bit simpler to work with configuration and startup later.

I will create a top-level directory on the project named `factory`:

```
.
└─ factory
    ├─ authz.go -- contains a function to initialize the authz.Authorizer
    ├─ database.go -- contains functions to initialize the DB-based repositories
    └─ factory.go -- contains functions to create the service and server
```

#### Authorizer

To initialize the Authorizer (to sign JWT), the caller may or may not supply an already-existing key. If the server restarts for a couple of seconds, you don't necessarily want to revoke all active sessions just because the key was overwritten.

In case it is not supplied or it is not valid, the key should then be created. Lastly, the function will accept a path, but also has a fallback default path in case the one provided is invalid.

So the `Authorizer` function will load the key from the input path or the default path if errored:

```go
// Authorizer creates a new authorizer with the input key `key`, or creates a new
// one under the preset folder if it doesn't yet exist and is not provided
func Authorizer(path string) (auth authz.Authorizer, err error) {
	var defErr error

	auth, err = loadKey(path)
	if err != nil {
		auth, defErr = loadKey(signingKeyPath)
		if defErr != nil {
			return nil, errors.Join(defErr, fmt.Errorf("failed to load key from input path and from default path: %w", err))
		}
	}
	return auth, nil
}
```

Now, to write the `loadKey` function. This function will try to read from the file in the input path, returning an `authz.Authorizer` or an error. However, I want it to fallback to creating the key if the file does not exist or if it is empty. If everything works, it will return a call to `authz.NewAuthorizer` with the key:

```go
func loadKey(path string) (authz.Authorizer, error) {
	fs, err := os.Stat(path)
	if (err != nil && os.IsNotExist(err)) || (fs != nil && fs.Size() == 0) {
		return createKey(path)
	}

	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	b, err := io.ReadAll(f)

	if err != nil {
		return nil, err
	}
	if len(b) == 0 {
		return nil, errors.New("zero bytes read")
	}

	return authz.NewAuthorizer(b), nil
}
```

Finally for the `createKey` function -- trying to create a new file with a new key. The task is to generate the key, writing it to the file, and finally returning a new Authorizer based off of it. The order is important since it needs to exit if it fails to write the key to the file (it cannot just work for this instance of the app):

```go
func createKey(path string) (authz.Authorizer, error) {
	// try to create local key
	k := crypt.New256Key()
	f, err := os.Create(path)
	if err != nil {
		return nil, err
	}
	n, err := f.Write(k[:])
	if err != nil {
		return nil, err
	}
	if n == 0 {
		return nil, errors.New("zero bytes written")
	}
	return authz.NewAuthorizer(k[:]), nil
}
```

#### SQLite

The `SQLite` function creates the user, secret and shared repositories from a SQLite DB file as indicated in the input path. Just like the authorizer, it will try to load the file first, and if it raises an error to create it. If that fails too, the factory function will try to fallback to a default path:

```go
const sqliteDbPath = "/secr/sqlite.db"

// SQLite creates user and secret repositories based on the defined SQLite DB path
func SQLite(path string) (user.Repository, secret.Repository, shared.Repository, error) {
	fs, err := os.Stat(path)
	if (err != nil && os.IsNotExist(err)) || (fs != nil && fs.Size() == 0) {
		_, err := os.Create(path)
		if err != nil {
			if path == sqliteDbPath {
				return nil, nil, nil, err
			}
			return SQLite(sqliteDbPath)
		}
	}

	db, err := sqlite.Open(path)
	if err != nil {
		if path == sqliteDbPath {
			return nil, nil, nil, err
		}
		return SQLite(sqliteDbPath)
	}

	return sqlite.NewUserRepository(db),
		sqlite.NewSecretRepository(db),
		sqlite.NewSharedRepository(db),
		nil
}
```

#### Bolt

For Bolt, it's almost the same (but will only return the `keys.Repository` and an error):

```go
const boltDbPath = "/secr/keys.db"

// Bolt creates a key repository based on the defined Bolt DB path
func Bolt(path string) (keys.Repository, error) {
	fs, err := os.Stat(path)
	if (err != nil && os.IsNotExist(err)) || (fs != nil && fs.Size() == 0) {
		_, err := os.Create(path)
		if err != nil {
			if path == boltDbPath {
				return nil, err
			}
			return Bolt(boltDbPath)
		}
	}

	db, err := bolt.Open(path)
	if err != nil {
		if path == boltDbPath {
			return nil, err
		}
		return Bolt(boltDbPath)
	}
	return bolt.NewKeysRepository(db), nil
}
```

#### Service

The factory function for the service will call the other factory functions, so it has a longer signature. All in all, it's about factory function calls, error checking and returning a new service:

```go
// Service creates a new service based on the signing key path `authKeyPath`,
// Bolt DB path `boltDBPath`, and SQLite DB path `sqliteDBPath`
func Service(authKeyPath, boltDBPath, sqliteDBPath string) (service.Service, error) {
	authorizer, err := Authorizer(authKeyPath)
	if err != nil {
		return nil, err
	}

	keys, err := Bolt(boltDBPath)
	if err != nil {
		return nil, err
	}

	users, secrets, shares, err := SQLite(sqliteDBPath)
	if err != nil {
		return nil, err
	}

	return service.NewService(
		users, secrets, shares, keys, authorizer,
	), nil
}
```

#### Server

For this one, it's just a matter of calling the app's `NewServer`, supplying a port and the service.


[next](cmd_package.md)