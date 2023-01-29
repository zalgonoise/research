[back](cryptography.md) | [index](index.md)

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
	"regexp"

	"github.com/zalgonoise/x/errors"
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
	"regexp"

	"github.com/zalgonoise/x/errors"
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
	"regexp"

	"github.com/zalgonoise/x/errors"
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

#### `service.CreateUser`

This action receives a request from the transport layer (HTTP) and executes it after validating the input.

For this the following is required:

1. Validating the user's input (username, password and name), with the appropriate validators

```go
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return nil, errors.Join(ErrInvalidPassword, err)
	}
	if err := user.ValidateName(name); err != nil {
		return nil, errors.Join(ErrInvalidName, err)
	}
```

2. Checking if the username already exists. This will be a `Get` call to the user.Repository

```go
	_, err := s.users.Get(ctx, username)
	if err == nil || !errors.Is(sqlite.ErrNotFoundUser, err) {
		return nil, errors.Join(ErrAlreadyExistsUser, err)
	}
```

3. Creating a new salt value
4. Generating the password hash, from the input password with the salt appended to it

```go
	salt := crypt.NewSalt()
	hashedPassword := crypt.Hash([]byte(password), salt[:])
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
		return nil, fmt.Errorf("failed to create user %s: %w", username, err)
	}
	u.ID = id
```
8. Generating a new 32-byte AES key for this user
9. Storing that key in the user's bucket (as a first entry, too), with a `Set` call to the keys.Repository

```go
	key := crypt.New32Key()
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, key[:])
	if err != nil {
		return nil, fmt.Errorf("failed to create user %s: %w", username, err)
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
}

func newTx() Transactioner {
	return &transactioner{}
}

func (tx *transactioner) Rollback(input error) error {
	var errs = make([]error, 0, len(tx.r)+1)
	errs[0] = input

	for _, rb := range tx.r {
		if err := rb(); err != nil {
			errs = append(errs, fmt.Errorf("rollback error: %w", err))
		}
	}
	switch len(errs) {

	case 1:
		return errs[0]
	default:
		return errors.Join(errs...)
	}
}

func (tx *transactioner) Add(r RollbackFn) {
	tx.r = append(tx.r, r)
}
```

Since the user is being created in user.Repository first (to generate an ID for them), I define the opposite action as a transaction's rollback func. If the keys.Repository action fails, I can call it:

```go
	id, err := s.users.Create(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to create user %s: %w", username, err)
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
		return nil, tx.Rollback(fmt.Errorf("failed to create user %s: %w", username, err))
	}
```

The whole thing looks like this:

```go
// CreateUser creates the user under username `username`, with the provided password `password` and name `name`
// It returns a user and an error
func (s service) CreateUser(ctx context.Context, username, password, name string) (*user.User, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return nil, errors.Join(ErrInvalidPassword, err)
	}
	if err := user.ValidateName(name); err != nil {
		return nil, errors.Join(ErrInvalidName, err)
	}

	// check if user exists
	_, err := s.users.Get(ctx, username)
	if err == nil || !errors.Is(sqlite.ErrNotFoundUser, err) {
		return nil, errors.Join(ErrAlreadyExistsUser, err)
	}

	// generate hash from password
	salt := crypt.NewSalt()
	hashedPassword := crypt.Hash([]byte(password), salt[:])

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
		return nil, fmt.Errorf("failed to create user %s: %w", username, err)
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
		return nil, tx.Rollback(fmt.Errorf("failed to create user %s: %w", username, err))
	}

	return u, nil
}
```

#### `service.GetUser`

This method fetches a user from the user.Repository, by its username. It's much simpler than `CreateUser` since it just validates the username and does a `Get` call to the user.Repository:

```go
// GetUser fetches the user with username `username`. Returns a user and an error
func (s service) GetUser(ctx context.Context, username string) (*user.User, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to get user %s: %w", username, err)
	}
	return u, nil
}
```

implementation/http_api_implementation.md
#### `service.ListUsers`

Same goes for `ListUsers` -- but this one doesn't require any validation:

```go
// ListUsers returns all the users in the directory, and an error
func (s service) ListUsers(ctx context.Context) ([]*user.User, error) {
	users, err := s.users.List(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to list users: %w", err)
	}
	return users, nil
}
```


#### `service.UpdateUser`

This method will be used to update a user's (general) information -- in the context of this application, this only applies to the user's name. The password changes are handled by a separate (service) method.

To summarize the actions that I need to take here:

1. Validating the user's input. This is the input username (identifying the user), and a user object with their updated version.

For the username there is already a validator function, but for the updated object:

- It must not be nil
- Its name (and any data being updated) needs to be validated

```go
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if updated == nil {
		return fmt.Errorf("%w: updated user cannot be nil", ErrInvalidUser)
	}
	if err := user.ValidateName(updated.Name); err != nil {
		return errors.Join(ErrInvalidName, err)
	}
```

2. Get this user, from the user.Repository (from their username). This will serve as the template object to alter with the new changes:

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch original user %s: %w", username, err)
	}
```

3. Verify if there are any actual changes to be made in this action. If not, simply return without an error (the desired state is already achieved).

```go
	if updated.Name == u.Name {
		// no changes to be made
		return nil
	}
	u.Name = updated.Name
```

4. Update the user by calling the `Update` user.Repository method

```go
	err = s.users.Update(ctx, username, u)
	if err != nil {
		return fmt.Errorf("failed to update user %s: %w", username, err)
	}

	return nil
```

Here's the whole thing:

```go
// UpdateUser updates the user `username`'s name, found in `updated` user. Returns an error
func (s service) UpdateUser(ctx context.Context, username string, updated *user.User) error {
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if updated == nil {
		return fmt.Errorf("%w: updated user cannot be nil", ErrInvalidUser)
	}
	if err := user.ValidateName(updated.Name); err != nil {
		return errors.Join(ErrInvalidName, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch original user %s: %w", username, err)
	}
	if updated.Name == u.Name {
		// no changes to be made
		return nil
	}
	u.Name = updated.Name

	err = s.users.Update(ctx, username, u)
	if err != nil {
		return fmt.Errorf("failed to update user %s: %w", username, err)
	}

	return nil
}
```



#### `service.DeleteUser`

While this method seems to only remove a user, there is also the user's secrets and shares to consider, which need to be handled appropriately. As such, to avoid cluttering the database and allowing access to secrets beyond an owner's removal, the user's shares must be removed, and then the user's secrets must be removed before finally deleting the user.

These are the required actions to remove a user:

1. Validate the input (the username)

```go
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
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
		return fmt.Errorf("failed to fetch original user %s: %w", username, err)
	}
```

3. Get all of the user's shared secrets, and remove them

```go
	shares, err := s.shares.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secrets: %w", err)
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
		return fmt.Errorf("failed to list user secrets: %w", err)
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
		return fmt.Errorf("failed to delete user %s's key: %w", username, err)
	}
```

6. Finally, remove the user

```go
	err = s.users.Delete(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to delete user %s: %w", username, err)
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
		return fmt.Errorf("failed to fetch shared secrets: %w", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})

		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %w", err))
		}
	}
```

4. Get all of the user's secrets, and remove them

```go
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to list user secrets: %w", err)
	}
	for _, secr := range secrets {
		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, secr)
			return err
		})

		err := s.secrets.Delete(ctx, username, secr.Key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove secret: %w", err))
		}
	}
```

5. Remove the user's private key in the keys.Repository

```go
	upk, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return fmt.Errorf("failed to fetch user %s's key: %w", username, err)
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, upk)
	})

	// delete private key
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s's key: %w", username, err))
	}
```

6. Finally, remove the user

```go
	err = s.users.Delete(ctx, username)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s: %w", username, err))
	}

	return nil
```

Again, it's important to note that a (decent) transaction would work directly with the repositories, and this implementation is acting more like a batch executor (in case an error is raised). Knowing this and embracing the size of the app, this is enough.

Here is the whole method body:

```go
// DeleteUser removes the user with username `username`. Returns an error
func (s service) DeleteUser(ctx context.Context, username string) error {
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		if errors.Is(sqlite.ErrNotFoundUser, err) {
			// no change in state
			return nil
		}
		return fmt.Errorf("failed to fetch original user %s: %w", username, err)
	}

	tx := newTx()

	// remove all shares
	shares, err := s.shares.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secrets: %w", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})

		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %w", err))
		}
	}

	// remove all secrets
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to list user secrets: %w", err)
	}
	for _, secr := range secrets {
		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, secr)
			return err
		})

		err := s.secrets.Delete(ctx, username, secr.Key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove secret: %w", err))
		}
	}

	// get user's private key for rollback func
	upk, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return fmt.Errorf("failed to fetch user %s's key: %w", username, err)
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), keys.UniqueID, upk)
	})

	// delete private key
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s's key: %w", username, err))
	}

	// delete user
	err = s.users.Delete(ctx, username)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to delete user %s: %w", username, err))
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
	"regexp"
	"strings"

	"github.com/zalgonoise/x/errors"
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


#### `service.CreateSecret`

This action stores (or overwrites) a secret when called, but there are also the shares to consider. This database schema imposes a link between the **secret ID** and users it is shared with.

This means that overwriting an existing secret with `CreateSecret` must go through fetching and deleting both the existing secret and its shares. The `Transactioner` comes in handy for these types of operations.

After going through the existing secret's ordeal, the new secret can finally be created.

For this, I need to fetch the user's (private) cipher key to encrypt the input value, and store that ciphertext in the keys.Repository, under the user's bucket.

Then, the secret's metadata can be stored in the secret.Repository.

Here is a recap of these actions and code to follow along:

1. Validating the input (username, secret key and secret value); note that the call is rejected when the user attempts to *crate* a secret whose key implies a share. 

```go
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(key); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}
	if err := secret.ValidateValue(value); err != nil {
		return errors.Join(ErrInvalidValue, err)
	}
```

2. Fetching the user from their username

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %w", err)
	}
```

3. Checking if the secret already exists (to overwrite it). Begin a transaction to be able to rollback any deletions

```go
	tx := newTx()

	oldSecr, err := s.secrets.Get(ctx, username, key)
	if err != nil && !errors.Is(sqlite.ErrNotFoundSecret, err) {
		return fmt.Errorf("failed to fetch previous secret under this key: %w", err)
	}
```

4. Overwrite routine: if the secret exists, then first thing's first, fetching any shares for this secret:

```go
	if oldSecr != nil {
		shares, err := s.shares.Get(ctx, username, oldSecr.Key)
		if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
			return fmt.Errorf("failed to fetch previous shared secrets: %w", err)
		}
	// (...)
	}
```

5. Iterating through each share to remove it. Pushing a new rollback function into the transaction as well

```go
	if oldSecr != nil {
	// (...)
		for _, sh := range shares {
			tx.Add(func() error {
				_, err := s.shares.Create(ctx, sh)
				return err
			})
			err := s.shares.Delete(ctx, sh)
			if err != nil {
				return tx.Rollback(fmt.Errorf("failed to remove old share: %w", err))
			}
		}
	// (...)
	}
```

6. Now the actual secret. Fetching the secret's value (still encrypted) for the rollback function

```go
	if oldSecr != nil {
	// (...)
		val, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to fetch old secret's value: %w", err))
		}
		tx.Add(func() error {
			return s.keys.Set(ctx, keys.UserBucket(u.ID), key, val)
		})
	// (...)
	}
```

7. Removing the old secret's value from the keys.Repository


```go
	if oldSecr != nil {
	// (...)
		err = s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove old secret's value: %w", err))
		}
	// (...)
	}
```

8. Remove the old secret's metadata from the secrets.Repository

```go
	if oldSecr != nil {
	// (...)
		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, oldSecr)
			return err
		})
		err = s.secrets.Delete(ctx, username, key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove old secret: %w", err))
		}
	}
```

9. Cool, back to adding the **new** secret. First, getting the user's private key

```go
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to get user's private key: %w", err))
	}
```

10. Encrypting the value with the user's private key

```go
	cipher := crypt.NewCipher(cipherKey)
	encValue, err := cipher.Encrypt(value)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to encrypt value: %w", err))
	}
```

11. Storing the encrypted secret value in the keys.Repository; note that the rollback function will only remove the secret if there wasn't one before (it doesn't apply to overwrites)

```go
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), key, encValue)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to store the secret: %w", err))
	}
	tx.Add(func() error {
		if oldSecr == nil {
			return s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
		}
		return nil
	})
```

12. Finally, adding the secret's metadata to the secret.Repository

```go
	secr := &secret.Secret{
		Key: key,
	}
	id, err := s.secrets.Create(ctx, username, secr)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to create the secret: %w", err))
	}
```

Here is how the whole thing looks:

```go
// CreateSecret creates a secret with key `key` and value `value` (as a slice of bytes), for the
// user `username`. It returns an error
func (s service) CreateSecret(ctx context.Context, username string, key string, value []byte) error {
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(key); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}
	if err := secret.ValidateValue(value); err != nil {
		return errors.Join(ErrInvalidValue, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %w", err)
	}

	tx := newTx()

	// check if secret already exists
	oldSecr, err := s.secrets.Get(ctx, username, key)
	if err != nil && !errors.Is(sqlite.ErrNotFoundSecret, err) {
		return fmt.Errorf("failed to fetch previous secret under this key: %w", err)
	}
	if oldSecr != nil {
		// remove old shares if they exist
		shares, err := s.shares.Get(ctx, username, oldSecr.Key)
		if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
			return fmt.Errorf("failed to fetch previous shared secrets: %w", err)
		}
		for _, sh := range shares {
			tx.Add(func() error {
				_, err := s.shares.Create(ctx, sh)
				return err
			})
			err := s.shares.Delete(ctx, sh)
			if err != nil {
				return tx.Rollback(fmt.Errorf("failed to remove old share: %w", err))
			}
		}

		// get encrypted value for existing secret (for RollbackFn)
		val, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to fetch old secret's value: %w", err))
		}
		tx.Add(func() error {
			return s.keys.Set(ctx, keys.UserBucket(u.ID), key, val)
		})

		// remove it
		err = s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove old secret's value: %w", err))
		}

		tx.Add(func() error {
			_, err := s.secrets.Create(ctx, username, oldSecr)
			return err
		})
		// remove the secret's metadata
		err = s.secrets.Delete(ctx, username, key)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove old secret: %w", err))
		}
	}

	// encrypt secret with user's key:
	// fetch the key
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to get user's private key: %w", err))
	}

	// encrypt value with user's private key
	cipher := crypt.NewCipher(cipherKey)
	encValue, err := cipher.Encrypt(value)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to encrypt value: %w", err))
	}

	// store encrypted value
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), key, encValue)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to store the secret: %w", err))
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
		return tx.Rollback(fmt.Errorf("failed to create the secret: %w", err))
	}
	secr.ID = id

	return nil
}
```

I would probably mark as a post-MVP task to refactor this method into smaller functions to allow a cleaner presentation.

#### `service.GetSecret`

Fetching a secret has simply one thing to consider; whether the target secret is shared or not. If it is shared, the key is expected to be in a `username:key` format, which will allow the secret key to be validated accordingly. Afterall, this will be how shared secrets appear to users.

To allow some actions to be reusable in separate methods, I added `getSecret` and `getSharedSecret`.

The `GetSecret` method will simply validate the user's input and depending on the `isShared` boolean from validating the secret key, either route is taken: `getSecret` or `getSharedSecret`

`getSecret` fetches the user's private key and the secret value, deciphers it and returns a `*secret.Secret` already with its metadata.

`getSharedSecret` validates the request by fetching the owner's shares for this secret, and checking whether the share is not expired, and if the user is a target. If they are, then the secret is deciphered with a `getSecret` call, and its key updated to the `username:key` format.

Below is how `GetSecret` looks like. When the secret key validation indicates that it is a shared secret, the key is split into username and key; which is processed with `getSharedSecret`. Otherwise it just goes through the `getSecret` routine:

```go
// GetSecret fetches the secret with key `key`, for user `username`. Returns a secret and an error
func (s service) GetSecret(ctx context.Context, username string, key string) (*secret.Secret, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	isShared, err := secret.ValidateKey(key)
	if err != nil {
		return nil, errors.Join(ErrInvalidKey, err)
	}

	// handle fetching an owned secret vs a shared secret
	if isShared {
		userAndKey := strings.Split(key, ":")
		return s.getSharedSecret(ctx, userAndKey[0], userAndKey[1], username)
	}

	return s.getSecret(ctx, username, key)
}
```

As for `getSecret`, it has the same signature as `GetSecret`:

```go
(s service) getSecret(ctx context.Context, username, key string) (*secret.Secret, error)
```

1. First, it must fetch the user

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user: %w", err)
	}
```

2. Then, the secret's metadata:

```go
	secr, err := s.secrets.Get(ctx, username, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the secret: %w", err)
	}
```

3. Also, the user's cipher key:

```go
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the user's private key: %w", err)
	}
	cipher := crypt.NewCipher(cipherKey)
```

4. And finally, the (encrypted) secret's value:

```go
	encValue, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the secret: %w", err)
	}
```

5. With all the needed data, it's a matter of deciphering the secret and composing the returned object:

```go
	decValue, err := cipher.Decrypt(encValue)
	if err != nil {
		return nil, fmt.Errorf("failed to decrypt value: %w", err)
	}

	secr.Value = string(decValue)
```

This is how `getSecret` looks like as a whole:

```go
func (s service) getSecret(ctx context.Context, username, key string) (*secret.Secret, error) {
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user: %w", err)
	}

	// fetch secret('s metadata )
	secr, err := s.secrets.Get(ctx, username, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the secret: %w", err)
	}

	// fetch user's private key to decode encrypted secret
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the user's private key: %w", err)
	}
	cipher := crypt.NewCipher(cipherKey)

	// fetch secret's value
	encValue, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the secret: %w", err)
	}

	// decrypt value with user's private key
	decValue, err := cipher.Decrypt(encValue)
	if err != nil {
		return nil, fmt.Errorf("failed to decrypt value: %w", err)
	}

	secr.Value = string(decValue)
	return secr, nil
}
```

Now for the `getSharedSecret` method; its signature is accepting an owner username, a secret key and a target username:

```go
(s service) getSharedSecret(ctx context.Context, owner, key, target string) (*secret.Secret, error)
```

Now for the shared secret routine:

1. Fetch the share metadata on behalf of the owner, for the input secret key

```go
	sh, err := s.shares.Get(ctx, owner, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret metadata: %w", err)
	}
```

2. Iterate through all the shares to find the target. For this I initialize a pointer to a `shared.Share` which I will use to store a valid share if found.

```go
	var validShare *shared.Share
	for _, share := range sh {
	// (...)
	}
```

3. First thing's first, even before looking into the targets, it's important to validate the share's expiry date. The plan was to remove expired shares on read actions, and this is one of them. 

```go
	for _, share := range sh {
		if share.Until != nil {
			if time.Now().After(*share.Until) {
				err := s.shares.Delete(ctx, share)
				if err != nil {
					return nil, fmt.Errorf("failed to remove expired shared secret: %w", err)
				}
				continue
			}
		}
	// (...)
	}
```

4. Then, looking for the requesting user in the `share.Target` list of usernames. If there is a match, the `validShare` pointer is set


```go
	var validShare *shared.Share
	for _, share := range sh {
	// (...)
		for _, t := range share.Target {
			if t == target {
				validShare = share
				break
			}
		}	
	}
```

5. Having gone through the shares, I can evaluate the state of `validShare`:

```go
	if validShare == nil {
		return nil, ErrZeroShares
	}
```

6. With a non-nil `validShare`, however, I can continue fetching the actual secret. For this I will just use the `getSecret` method on behalf of the owner

```go
	secr, err := s.getSecret(ctx, owner, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret: %w", err)
	}
```

7. Lastly, I erase the creation date (as the only form of PII besides the secret's value), as it only concerns the owner and set the key to a `username:key` format:

```go
	secr.CreatedAt = time.Time{}
	secr.Key = fmt.Sprintf("%s:%s", owner, secr.Key)
```

Here is how the whole thing looks:

```go
func (s service) getSharedSecret(ctx context.Context, owner, key, target string) (*secret.Secret, error) {
	// get the original share (as if it was the owner)
	sh, err := s.shares.Get(ctx, owner, key)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret metadata: %w", err)
	}

	var validShare *shared.Share
	for _, share := range sh {
		// validate share's deadline first
		if share.Until != nil {
			if time.Now().After(*share.Until) {
				// remove it if expired
				err := s.shares.Delete(ctx, share)
				if err != nil {
					return nil, fmt.Errorf("failed to remove expired shared secret: %w", err)
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
		return nil, fmt.Errorf("failed to fetch shared secret: %w", err)
	}
	// erase creation time; set key as `user:key`
	secr.CreatedAt = time.Time{}
	secr.Key = fmt.Sprintf("%s:%s", owner, secr.Key)
	return secr, nil
}
```

#### `service.ListSecrets`

This operation does the same as a `GetSecret` operation, but obviously as a batch process (for all of the input user's owned secrets and secrets shared with them).

There are a lot of similarities from the previous chapter, with differences in the repository methods being called:

1. Validation, validation, validation!

```go
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
```

2. Fetching the user:

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user: %w", err)
	}
```

3. And fetching *all* of the secrets' metadata (with `secrets.List`)

```go
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to list the user's secrets: %w", err)
	}
```

4. Fetch the user's cipher key

```go
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the user's private key: %w", err)
	}
	cipher := crypt.NewCipher(cipherKey)
```

5. Then, having already a list of `*secret.Secret`, I can decode the secret's values in-place, when iterating through them:

```go
	for _, secr := range secrets {
		encValue, err := s.keys.Get(ctx, keys.UserBucket(u.ID), secr.Key)
		if err != nil {
			return nil, fmt.Errorf("failed to fetch the secret: %w", err)
		}

		decValue, err := cipher.Decrypt(encValue)
		if err != nil {
			return nil, fmt.Errorf("failed to decrypt value: %w", err)
		}

		secr.Value = string(decValue)
	}
```

6. And now for the shared secrets I can call `shares.ListTarget`, to fetch all secrets shared with this user:

```go
	sharedSecrets, err := s.shares.ListTarget(ctx, username)
	if err != nil {
		return secrets, fmt.Errorf("failed to fetch secrets shared with %s: %w", username, err)
	}
```

7. When iterating through all shares, I can call `getSharedSecret` for each share. If the error is not having any shares (expired, for instance), just continue. Otherwise I can return the user's secrets *and* the error. If all goes well, each (shared) secret is appended to the returned list of secrets:

```go
	for _, sh := range sharedSecrets {
		sharedSecr, err := s.getSharedSecret(ctx, sh.Owner, sh.SecretKey, username)
		if err != nil {
			if errors.Is(ErrZeroShares, err) {
				continue
			}
			return secrets, fmt.Errorf("failed to fetch secrets shared with %s: %w", username, err)
		}
		secrets = append(secrets, sharedSecr)
	}
```

Here is how the entire thing looks:

```go
// ListSecrets retuns all secrets for user `username`. Returns a list of secrets and an error
func (s service) ListSecrets(ctx context.Context, username string) ([]*secret.Secret, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user: %w", err)
	}

	// fetch secrets(' metadata )
	secrets, err := s.secrets.List(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to list the user's secrets: %w", err)
	}

	// fetch user's private key to decode encrypted secret
	cipherKey, err := s.keys.Get(ctx, keys.UserBucket(u.ID), keys.UniqueID)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch the user's private key: %w", err)
	}
	cipher := crypt.NewCipher(cipherKey)

	// fetch and decode the value for each secret
	for _, secr := range secrets {
		// fetch secret's value
		encValue, err := s.keys.Get(ctx, keys.UserBucket(u.ID), secr.Key)
		if err != nil {
			return nil, fmt.Errorf("failed to fetch the secret: %w", err)
		}

		// decrypt value with user's private key
		decValue, err := cipher.Decrypt(encValue)
		if err != nil {
			return nil, fmt.Errorf("failed to decrypt value: %w", err)
		}

		secr.Value = string(decValue)
	}

	// aggregate secrets that are shared with this user
	sharedSecrets, err := s.shares.ListTarget(ctx, username)
	if err != nil {
		return secrets, fmt.Errorf("failed to fetch secrets shared with %s: %w", username, err)
	}
	for _, sh := range sharedSecrets {
		// extract secret from shared secret
		sharedSecr, err := s.getSharedSecret(ctx, sh.Owner, sh.SecretKey, username)
		if err != nil {
			if errors.Is(ErrZeroShares, err) {
				continue
			}
			return secrets, fmt.Errorf("failed to fetch secrets shared with %s: %w", username, err)
		}
		// append results
		secrets = append(secrets, sharedSecr)
	}

	return secrets, nil
}
```

#### `service.DeleteSecret`

Deleting a secret will involve a transaction, since its shares will also be removed; so it's nice to consider rolling back any deletions in case an error occurs.

1. It all starts with validation (for username and secret key):

```go
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(key); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}
```

2. Then, fetching the user:

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %w", err)
	}
```

3. To remove the shares, I will fetch them first, to iterate over all of the secret's shares, deleting them as I go:

```go
	tx := newTx()
	shares, err := s.shares.Get(ctx, username, key)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return fmt.Errorf("failed to list shared secrets: %w", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})
		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %w", err))
		}
	}
```

4. Now, to delete the secret's value from the `keys.Repository`, I will fetch it first (for the rollback function). I won't decrypt it for this action:

```go
	secr, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		if errors.Is(bolt.ErrEmptyBucket, err) {
			// nothing to delete, no changes in state
			return nil
		}
		return tx.Rollback(fmt.Errorf("failed to fetch the secret: %w", err))
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), key, secr)
	})
```

5. Delete the secret's value

```go
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to remove secret: %w", err))
	}
```

6. Now the same for the secret's metadata. First, to fetch it for the rollback function:

```go
	secretMeta, err := s.secrets.Get(ctx, username, key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to fetch secret: %w", err))
	}
	tx.Add(func() error {
		_, err := s.secrets.Create(ctx, username, secretMeta)
		return err
	})
```

7. Then, to finally delete it:

```go
	err = s.secrets.Delete(ctx, username, key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to remove secret: %w", err))
	}
```

Here is the whole method:

```go
// DeleteSecret removes a secret with key `key` from the user `username`. Returns an error
func (s service) DeleteSecret(ctx context.Context, username string, key string) error {
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(key); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %w", err)
	}

	tx := newTx()

	// remove shares for this secret
	shares, err := s.shares.Get(ctx, username, key)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return fmt.Errorf("failed to list shared secrets: %w", err)
	}
	for _, sh := range shares {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, sh)
			return err
		})
		err := s.shares.Delete(ctx, sh)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %w", err))
		}
	}

	// fetch the encrypted secret (for RollbackFn)
	secr, err := s.keys.Get(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		if errors.Is(bolt.ErrEmptyBucket, err) {
			// nothing to delete, no changes in state
			return nil
		}
		return tx.Rollback(fmt.Errorf("failed to fetch the secret: %w", err))
	}
	tx.Add(func() error {
		return s.keys.Set(ctx, keys.UserBucket(u.ID), key, secr)
	})

	// delete it
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to remove secret: %w", err))
	}

	// fetch the secret's metadata (for RollbackFn)
	secretMeta, err := s.secrets.Get(ctx, username, key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to fetch secret: %w", err))
	}

	tx.Add(func() error {
		_, err := s.secrets.Create(ctx, username, secretMeta)
		return err
	})

	// delete it
	err = s.secrets.Delete(ctx, username, key)
	if err != nil {
		return tx.Rollback(fmt.Errorf("failed to remove secret: %w", err))
	}

	return nil
}
```

#### Implementing Service Shares methods

For the shared secrets, the only new element (that needs to be validated) is the input time / duration for the share's expiry.

To validate this type of input, a `validate.go` file is added to the `shared` package folder:

```
.
└─ shared
    ├─ repository.go
    ├─ shared.go 
    └─ validate.go -- shares-related validator functions
```

This file will contain two functions:

`ValidateDuration` ensures that the input duration is not zero:

```go
import (
	"time"

	"github.com/zalgonoise/x/errors"
)

var ErrEmptyDuration = errors.New("duration cannot be zero")

// ValidateDuration verifies if the input duration is valid, returning an error
// if otherwise
func ValidateDuration(dur time.Duration) error {
	if dur == 0 {
		return ErrEmptyDuration
	}
	return nil
}
```

`ValidateTime` ensures that the input time is not zero and is not expired (pointing to a past date):

```go
import (
	"time"

	"github.com/zalgonoise/x/errors"
)

var zeroTime = time.Time{}

var (
	ErrEmptyTime     = errors.New("time cannot be zero")
	ErrExpired       = errors.New("input time is already expired")
)

// ValidateTime verifies if the input time is valid, returning an error
// if otherwise
func ValidateTime(t time.Time) error {
	if t.IsZero() || t == zeroTime {
		return ErrEmptyTime
	}
	if time.Now().After(t) {
		return ErrExpired
	}
	return nil
}
```

Having these validator functions defined, it's time to *share* a blank canvas for the service implementation:

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



#### `service.CreateShare` / `ShareFor` / `ShareUntil`

The shares create actions allow 3 different approaches:
- one where the user does not define an expiry (which is imposed by the service)
- the user defines a date when the share should expire
- the user defines a duration for it to exist

All in all, these are very similar approaches, with a single difference (apart from the signature). This section covers `service.CreateShare`, and at the end positions how `ShareFor` and `ShareUntil` differ from it.

1. Validate the user's input for the owner, secret key and length of targets. The targets usernames will be validated afterwards:

```go
	if err := user.ValidateUsername(owner); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return nil, errors.Join(ErrInvalidKey, err)
	}
	if len(targets) == 0 {
		return nil, ErrZeroTargets
	}
```

2. Initialize a `*shared.Share` object with the secret key, owner and `Until` field -- in this case defined with the default share duration (of 1 month)

> Note: this is the step that will differ in `ShareFor` and `ShareUntil`, as the `*shared.Share`'s Until field is configured differently

```go
	sh := &shared.Share{
		SecretKey: secretKey,
		Owner:     owner,
		Until:     ptr.To(time.Now().Add(shared.DefaultShareDuration)),
	}
```

3. Iterate through all input targets, validating their username and appending them to the list of targets, if valid

```go
	for _, t := range targets {
		if err := user.ValidateUsername(t); err != nil {
			return nil, errors.Join(ErrInvalidUser, err)
		}
		sh.Target = append(sh.Target, t)
	}
```

4. The database will constrain the shares with unique values between the secret and the target user. As such, clear any existing shares of this secret with these target users, first:

```go
	err := s.shares.Delete(ctx, sh)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return nil, fmt.Errorf("failed to remove previous shared secrets: %w", err)
	}
```

5. Finally, create the share and return the object with the new ID

```go
	id, err := s.shares.Create(ctx, sh)
	if err != nil {
		return nil, fmt.Errorf("failed to create shared secret: %w", err)
	}

	sh.ID = id
	return sh, nil
```

Here is the complete method:

```go
// CreateShare shares the secret with key `secretKey` belonging to user with username `owner`, with users `targets`.
// Returns the resulting shared secret, and an error
func (s service) CreateShare(ctx context.Context, owner, secretKey string, targets ...string) (*shared.Share, error) {
	if err := user.ValidateUsername(owner); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return nil, errors.Join(ErrInvalidKey, err)
	}
	if len(targets) == 0 {
		return nil, ErrZeroTargets
	}

	sh := &shared.Share{
		SecretKey: secretKey,
		Owner:     owner,
		Until:     ptr.To(time.Now().Add(shared.DefaultShareDuration)),
	}
	for _, t := range targets {
		if err := user.ValidateUsername(t); err != nil {
			return nil, errors.Join(ErrInvalidUser, err)
		}
		sh.Target = append(sh.Target, t)
	}
	err := s.shares.Delete(ctx, sh)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return nil, fmt.Errorf("failed to remove previous shared secrets: %w", err)
	}

	id, err := s.shares.Create(ctx, sh)
	if err != nil {
		return nil, fmt.Errorf("failed to create shared secret: %w", err)
	}

	sh.ID = id
	return sh, nil
}
```

The `ShareFor` method will have a different steps 1. and 2., for validation and applying the input duration:

1. Validate the user's input for the owner, secret key, length of targets and duration. The targets usernames will be validated afterwards:

```go
	if err := user.ValidateUsername(owner); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return nil, errors.Join(ErrInvalidKey, err)
	}
	if len(targets) == 0 {
		return nil, ErrZeroTargets
	}
	if err := shared.ValidateDuration(dur); err != nil {
		return nil, errors.Join(ErrInvalidTime, err)
	}
```

2. Initialize a `*shared.Share` object with the secret key, owner and `Until` field -- in this case defined with the current time plus the input duration

> Note: this is the step that will differ in `CreateShare` and `ShareUntil`, as the `*shared.Share`'s Until field is configured differently

```go
	sh := &shared.Share{
		SecretKey: secretKey,
		Owner:     owner,
		Until:     ptr.To(time.Now().Add(dur)),
	}
```

The `ShareUntil` method will also have slightly different steps 1. and 2., for validation and applying the input time:


1. Validate the user's input for the owner, secret key, length of targets and time. The targets usernames will be validated afterwards:

```go
	if err := user.ValidateUsername(owner); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return nil, errors.Join(ErrInvalidKey, err)
	}
	if len(targets) == 0 {
		return nil, ErrZeroTargets
	}
	if err := shared.ValidateTime(until); err != nil {
		return nil, errors.Join(ErrInvalidTime, err)
	}
```

2. Initialize a `*shared.Share` object with the secret key, owner and `Until` field -- in this case defined with a pointer to the input time:

> Note: this is the step that will differ in `CreateShare` and `ShareFor`, as the `*shared.Share`'s Until field is configured differently

```go
	sh := &shared.Share{
		SecretKey: secretKey,
		Owner:     owner,
		Until:     &until,
	}
```


#### `service.GetShare`

Fetching a shared secret is simple as most of the heavy-lifting is done in SQL

1. Validation, validation, validation. This method will not be used to fetch a secret shared with the caller; but the owner's shares (thus rejecting a call for a shared secret's key)

```go
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return nil, errors.Join(ErrInvalidKey, err)
	}
```

2. Then, just fetch the shares from the shared.Repository

```go
	sh, err := s.shares.Get(ctx, username, secretKey)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret: %w", err)
	}
	return sh, nil
```

The entire method:

```go
// GetShare fetches the shared secret belonging to `username`, with key `secretKey`, returning it as a
// shared secret and an error
func (s service) GetShare(ctx context.Context, username, secretKey string) ([]*shared.Share, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return nil, errors.Join(ErrInvalidKey, err)
	}

	sh, err := s.shares.Get(ctx, username, secretKey)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch shared secret: %w", err)
	}
	return sh, nil
}
```

#### `service.ListShares`

Just like the above, but without a key validation, and with a call to `shared.Repository.List`:

```go
// ListShares fetches all the secrets the user with username `username` has shared with other users
func (s service) ListShares(ctx context.Context, username string) ([]*shared.Share, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}

	sh, err := s.shares.List(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to list shared secrets: %w", err)
	}
	return sh, nil
}
```

#### `service.DeleteShare`

The approach to this call is similar to the one in `CreateShare` (with the opposite purpose). That will be:

1. Validating the input owner's username, secret key and length of the list of targets. If no targets are provided, the call will be handled as a `service.PurgeShares` call:

```go
	if err := user.ValidateUsername(owner); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}
	if len(targets) == 0 {
		return s.PurgeShares(ctx, owner, secretKey)
	}
```

2. Create a `*shared.Share` object with the owner and secret key, then iterate through all targets validating their username, then appending them to the targets list if valid:

```go
	sh := &shared.Share{
		SecretKey: secretKey,
		Owner:     owner,
	}
	for _, t := range targets {
		if err := user.ValidateUsername(t); err != nil {
			return errors.Join(ErrInvalidUser, err)
		}
		sh.Target = append(sh.Target, t)
	}
```

3. Finally, remove the shared secret:

```go
	err := s.shares.Delete(ctx, sh)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return fmt.Errorf("failed to delete shared secret: %w", err)
	}
```

Here is the entire method:


```go
// DeleteShare removes the users `targets` from a shared secret with key `secretKey`, belonging to `username`. Returns
// an error
func (s service) DeleteShare(ctx context.Context, owner, secretKey string, targets ...string) error {
	if err := user.ValidateUsername(owner); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}
	if len(targets) == 0 {
		return s.PurgeShares(ctx, owner, secretKey)
	}

	sh := &shared.Share{
		SecretKey: secretKey,
		Owner:     owner,
	}
	for _, t := range targets {
		if err := user.ValidateUsername(t); err != nil {
			return errors.Join(ErrInvalidUser, err)
		}
		sh.Target = append(sh.Target, t)
	}

	err := s.shares.Delete(ctx, sh)
	if err != nil && !errors.Is(sqlite.ErrNotFoundShare, err) {
		return fmt.Errorf("failed to delete shared secret: %w", err)
	}
	return nil
}
```

#### `service.PurgeShares`

`PurgeShares` will clear all shares for a secret. To do so, I need to fetch the shares for the input owner and secret key combination, and remove them.

1. First thing's first, validation:

```go
	if err := user.ValidateUsername(owner); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}
```

2. Then, fetching the secrets for this owner and key:

```go
	sh, err := s.shares.Get(ctx, owner, secretKey)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secret: %w", err)
	}
```

3. Finally, within a transaction, remove all shares for this secret

```go
	tx := newTx()
	for _, share := range sh {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, share)
			return err
		})
		err := s.shares.Delete(ctx, share)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %w", err))
		}
	}
```

Here is the whole method:

```go
// PurgeShares removes the shared secret completely, so it's no longer available to the users it was
// shared with. Returns an error
func (s service) PurgeShares(ctx context.Context, owner, secretKey string) error {
	if err := user.ValidateUsername(owner); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if isShared, err := secret.ValidateKey(secretKey); err != nil || isShared {
		return errors.Join(ErrInvalidKey, err)
	}

	sh, err := s.shares.Get(ctx, owner, secretKey)
	if err != nil {
		return fmt.Errorf("failed to fetch shared secret: %w", err)
	}

	tx := newTx()

	for _, share := range sh {
		tx.Add(func() error {
			_, err := s.shares.Create(ctx, share)
			return err
		})
		err := s.shares.Delete(ctx, share)
		if err != nil {
			return tx.Rollback(fmt.Errorf("failed to remove shared secret: %w", err))
		}
	}
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

// ParseToken reads the input token string and returns the corresponding user in it, or an error
func (s service) ParseToken(ctx context.Context, token string) (*user.User, error) {
	return nil, nil
}
```

These methods will handle authentication and authorization-related requests.


#### `service.Login`

Starting with `Login`; the action needs to validate the input username and password; fetch the user from the user.Repository, and match their password to the stored hash.

From here, it's a matter of creating a new JWT for the user, storing it in the keys.Repository and returning a `*user.Session`.

To handle the credentials verification, I will add a separate `login` method that can be used on other session-related service methods. Let's start with that one, here is the signature:

```go
(s service) login(ctx context.Context, u *user.User, password string) error
```

1. For this procedure, starting by base64-decoding the hash and salt values

```go
	hash, err := base64.StdEncoding.DecodeString(u.Hash)
	if err != nil {
		return fmt.Errorf("failed to decode hash: %w", err)
	}
	salt, err := base64.StdEncoding.DecodeString(u.Salt)
	if err != nil {
		return fmt.Errorf("failed to decode salt: %w", err)
	}
```

2. Hash the input password with the salt appended to it

```go
	hashedPassword := crypt.Hash([]byte(password), salt[:])
```

3. Compare this value with the stored hash. If it doesn't match it is an incorrect password

```go
	if string(hashedPassword[:]) != string(hash) {
		return ErrIncorrectPassword
	}
	return nil
```

The entire function:

```go
func (s service) login(ctx context.Context, u *user.User, password string) error {
	hash, err := base64.StdEncoding.DecodeString(u.Hash)
	if err != nil {
		return fmt.Errorf("failed to decode hash: %w", err)
	}
	salt, err := base64.StdEncoding.DecodeString(u.Salt)
	if err != nil {
		return fmt.Errorf("failed to decode salt: %w", err)
	}
	hashedPassword := crypt.Hash([]byte(password), salt)

	if string(hashedPassword[:]) != string(hash) {
		return ErrIncorrectPassword
	}
	return nil
}
```

Now, for `service.Login`:

1. Validate the input username and password:

```go
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return nil, errors.Join(ErrInvalidPassword, err)
	}
```

2. Fetch the user by their username

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user %s: %w", username, err)
	}
```

3. Validate the user credentials with the `login` method

```go
	if err := s.login(ctx, u, password); err != nil {
		return nil, fmt.Errorf("failed to validate user credentials: %w", err)
	}
```

4. If there isn't an error, everything checks out. Time to issue a new token:

```go
	token, err := s.auth.NewToken(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to create a new session token: %w", err)
	}
```

5. Then, to store it in the `keys.Repository`, in the user's bucket, under the `keys.TokenKey` reserved identifier

```go
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.TokenKey, []byte(token))
	if err != nil {
		return nil, fmt.Errorf("failed to store the new session token: %w", err)
	}

	return &user.Session{
		User:  *u,
		Token: token,
	}, nil
```

Here's the entire method:

```go
// Login verifies the user's credentials and returns a session and an error
func (s service) Login(ctx context.Context, username, password string) (*user.Session, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return nil, errors.Join(ErrInvalidPassword, err)
	}

	// fetch user
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user %s: %w", username, err)
	}

	// validate credentials
	if err := s.login(ctx, u, password); err != nil {
		return nil, fmt.Errorf("failed to validate user credentials: %w", err)
	}

	// issue token
	token, err := s.auth.NewToken(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to create a new session token: %w", err)
	}

	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.TokenKey, []byte(token))
	if err != nil {
		return nil, fmt.Errorf("failed to store the new session token: %w", err)
	}

	return &user.Session{
		User:  *u,
		Token: token,
	}, nil
}
```


#### `service.Logout`

Logout just involves validating the caller's user, and deleting the JWT from the `keys.Repository`:

1. Validate the input username


```go
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
```

2. Fetch the user

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %w", err)
	}
```

3. Delete the stored JWT from the `keys.Repository`

```go
	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.TokenKey)
	if err != nil {
		return fmt.Errorf("failed to log user out: %w", err)
	}
	return nil
```

Here's the whole `Logout` method:

```go
// Logout signs-out the user `username`
func (s service) Logout(ctx context.Context, username string) error {
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}

	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user: %w", err)
	}

	err = s.keys.Delete(ctx, keys.UserBucket(u.ID), keys.TokenKey)
	if err != nil {
		return fmt.Errorf("failed to log user out: %w", err)
	}
	return nil
}
```

#### `service.ChangePassword`

This method updates a user's password. For this, I need to ensure the current credentials are valid, and then update the `user.Repository` with the new password hash.

1. Validate the user input (username, old password and new password)

```go
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return errors.Join(ErrInvalidPassword, err)
	}
	if err := user.ValidatePassword(newPassword); err != nil {
		return errors.Join(ErrInvalidPassword, err)
	}
```

2. Fetch the user by their username

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user %s: %w", username, err)
	}
```

3. Validate the current password with the `login` method

```go
	if err := s.login(ctx, u, password); err != nil {
		return fmt.Errorf("failed to validate user credentials: %w", err)
	}
```

4. Base64-decode the salt value

```go
	salt, err := base64.StdEncoding.DecodeString(u.Salt)
	if err != nil {
		return fmt.Errorf("failed to decode salt: %w", err)
	}
```


5. Generate a hash of the new password with the user's salt appended to it 

```go
	u.Hash = string(crypt.Hash([]byte(newPassword), salt))
```

6. Update the user with the new hash value

```go
	err = s.users.Update(ctx, username, u)
	if err != nil {
		return fmt.Errorf("failed to update user %s's password: %w", username, err)
	}
	return nil
```

Behold, the entire method:

```go
// ChangePassword updates user `username`'s password after verifying the old one, returning an error
func (s service) ChangePassword(ctx context.Context, username, password, newPassword string) error {
	if err := user.ValidateUsername(username); err != nil {
		return errors.Join(ErrInvalidUser, err)
	}
	if err := user.ValidatePassword(password); err != nil {
		return errors.Join(ErrInvalidPassword, err)
	}
	if err := user.ValidatePassword(newPassword); err != nil {
		return errors.Join(ErrInvalidPassword, err)
	}

	// fetch user
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return fmt.Errorf("failed to fetch user %s: %w", username, err)
	}

	if err := s.login(ctx, u, password); err != nil {
		return fmt.Errorf("failed to validate user credentials: %w", err)
	}

	salt, err := base64.StdEncoding.DecodeString(u.Salt)
	if err != nil {
		return fmt.Errorf("failed to decode salt: %w", err)
	}

	u.Hash = string(crypt.Hash([]byte(newPassword), salt))

	err = s.users.Update(ctx, username, u)
	if err != nil {
		return fmt.Errorf("failed to update user %s's password: %w", username, err)
	}
	return nil
}
```


#### `service.Refresh`

This method allows extending a user's session in exchange for a currently valid token. Since the tokens have expiry of one hour, a user can call this method to get a new token, valid for another hour.

So, in a nutshell:
- ensure token is valid
- create a new token for the same user
- store it in the `keys.Repository`
- return the new token to the user

Here's how it's implemented:

1. Validate the input username and token (ensuring the token is not an empty string; its validation will come on step 3.)

```go
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if token == "" {
		return nil, fmt.Errorf("%w: token cannot be empty", ErrInvalidPassword)
	}
```

2. Fetch the user

```go
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user %s: %w", username, err)
	}
```

3. Parse the input token to ensure it is valid

```go
	jwtUser, err := s.auth.Parse(ctx, token)
	if err != nil {
		return nil, fmt.Errorf("failed to validate token: %w", err)
	}
```

4. Ensure that the token belongs to the caller

```go
	if jwtUser.Username != u.Username {
		return nil, ErrIncorrectPassword
	}
```

5. Generate a new JWT

```go
	newToken, err := s.auth.NewToken(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to refresh token: %w", err)
	}
```

6. Store it in the `keys.Repository`

```go
	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.TokenKey, []byte(newToken))
	if err != nil {
		return nil, fmt.Errorf("failed to store the new session token: %w", err)
	}
```

7. Return the token to the caller

```go
	return &user.Session{
		User:  *u,
		Token: newToken,
	}, nil
```

Here's the entire `Refresh` method:


```go
// Refresh renews a user's JWT provided it is a valid one. Returns a session and an error
func (s service) Refresh(ctx context.Context, username, token string) (*user.Session, error) {
	if err := user.ValidateUsername(username); err != nil {
		return nil, errors.Join(ErrInvalidUser, err)
	}
	if token == "" {
		return nil, fmt.Errorf("%w: token cannot be empty", ErrInvalidPassword)
	}

	// fetch user
	u, err := s.users.Get(ctx, username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user %s: %w", username, err)
	}

	jwtUser, err := s.auth.Parse(ctx, token)
	if err != nil {
		return nil, fmt.Errorf("failed to validate token: %w", err)
	}

	if jwtUser.Username != u.Username {
		return nil, ErrIncorrectPassword
	}

	newToken, err := s.auth.NewToken(ctx, u)
	if err != nil {
		return nil, fmt.Errorf("failed to refresh token: %w", err)
	}

	err = s.keys.Set(ctx, keys.UserBucket(u.ID), keys.TokenKey, []byte(newToken))
	if err != nil {
		return nil, fmt.Errorf("failed to store the new session token: %w", err)
	}

	return &user.Session{
		User:  *u,
		Token: newToken,
	}, nil
}
```


#### `service.ParseToken`

`ParseToken` will be a "helper" method to identify a user by a JWT. It will be used on the transport layer since the service has access the user.Repository, to verify if that the user actually exists.

1. Starting by the token itself, with its validation (ensuring it's not empty, and parsing it):

```go
	if token == "" {
		return nil, fmt.Errorf("%w: token cannot be empty", ErrInvalidPassword)
	}
	t, err := s.auth.Parse(ctx, token)
	if err != nil {
		return nil, fmt.Errorf("failed to parse token: %w", err)
	}
```

2. If valid, return the user identified by this username

```go
	u, err := s.users.Get(ctx, t.Username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user %s: %w", t.Username, err)
	}

	return u, nil
```

Quick and simple, here's the entire method:

```go
// ParseToken reads the input token string and returns the corresponding user in it, or an error
func (s service) ParseToken(ctx context.Context, token string) (*user.User, error) {
	if token == "" {
		return nil, fmt.Errorf("%w: token cannot be empty", ErrInvalidPassword)
	}

	t, err := s.auth.Parse(ctx, token)
	if err != nil {
		return nil, fmt.Errorf("failed to parse token: %w", err)
	}

	u, err := s.users.Get(ctx, t.Username)
	if err != nil {
		return nil, fmt.Errorf("failed to fetch user %s: %w", t.Username, err)
	}

	return u, nil
}
```

Fantastic! The Service interface is implemented. Technically the app already works as a library, right now so cheers to that!

Next step is to move over to the transport layer, as a HTTP API with JSON documents. Simple and straight-forward.

[next](http_api_implementation.md)