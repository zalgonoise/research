[back](service_preparation.md) | [index](index.md)

#### Cryptography

There are cryptographic actions to plan ahead, for authentication and authorization:

- User passwords will be salted and hashed. For this we will need a random source for creating the 128-byte salt.
- When users are created, a unique 32-byte key is created, which is used to AES-encrypt their secrets. This key needs to be generated from a random source. An AES encrypt / decrypter object will be useful too.
- If there is no JWT signing key present, one needs to be generated. A 256-byte key needs to be created from a random source.

So, this will be mapped under `crypt/crypt.go`. 

```
.
└─ crypt
    └─ crypt.go
```


Starting with the `Cryptographer` and `EcryptDecrypter` interfaces: 

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

#### Key generating methods

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

Lastly, to hash the users' passwords I want to use a secure method, so after testing a few combinations I've settled with 128-byte-long PBKDF2 keys over a SHA-512 hash function, with 600k iterations. In my machine this is about 1 second per password, which is a good amount of iterations for an OK-ish amount of time. SHA-3 was taking about 5~10 seconds per password for the same amount of iterations which is crazy.

```go
import (
	"crypto/sha512"

	"golang.org/x/crypto/pbkdf2"
)

const numHashIter = 600_001

func Hash(secret, salt []byte) []byte {
	return pbkdf2.Key(secret, salt, numHashIter, 128, sha512.New)
}
```

This does it for the cryptography needs in this application, at least for the minimal viable product release.

#### Authorization

While it's already possible to generate JWT signing keys, there isn't yet any logic for handling JWT. This is going into the `authz` package. To plan ahead, this is the behavior that I need when handling JWT:

- Creating a new token (for a user)
- Parsing a user's input token (returning a user and an error)

I've added the top-level folder `authz` with a file (`authz/jwt.go`) in it.


```
.
└─ authz
    └─ jwt.go
```


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
		return "", fmt.Errorf("failed to sign token: %w", err)
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
		return nil, fmt.Errorf("failed to parse token: %w", err)
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
		return nil, errors.Join(ErrInvalidMethod, fmt.Errorf("unexpected signing method: %v", token.Header["alg"]))
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

[next](service_implementation.md)