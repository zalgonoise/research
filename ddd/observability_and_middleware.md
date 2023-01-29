[back](implementation/cmd_package.md) | [index](index.md)

## Observability and middleware

This section will cover a lightweight approach to observability. I am not going to create a Prometheus + Grafana setup for this app. Instead, I will use my own logger and spanner solely for having a clean API to read. It's not meant to be an advertisement to use it, if anything use [OpenTelemetry](https://opentelemetry.io/), please.

This section is important for the project's organization and structure, when it comes to wrapping interfaces with specific middleware in this case for observability. It's about the coding patterns, not the tools.

Here's how I will instrument my code:

- Logging will be registering errors in the application. So in this particular case, it should handle error levels of *Warn* and above.
- Tracing will be registering the entire transaction from transport to persistence layers. The approach needs to take into consideration that this is a secrets store, so no PII. Traces will be identified by Trace and Span IDs, with timestamps and optionally attributes and events.

To organize this instrumentation, I will create wrappers for some of the interfaces exposed in the app, so that the observability code does not overlap with the business logic.

I will start with the `*.Repository` interfaces. I will only set up tracing for these. For this, I will create a `*_with_trace.go` file in the appropriate folder where I implement the Repository interface, which will wrap another implementation with a tracer. This implementation starts a span for the call, issues the request through the wrapped Repository, and records an event if there is an error.

I am using my tracer repository for this ([`zalgonoise/spanner`](https://github.com/zalgonoise/spanner)) for simplicity, which in the end writes traces to a file. In a production enviroment you'd use OpenTelemetry with a reliable exporter platform / traces backend.

#### `keys.Repository` observability

Adding a new file to the `keys` folder, for the tracer wrapper:

```
.
└─ keys
    ├─ key.go
    ├─ keys_with_trace.go -- tracing wrapper for keys.Repository
    └─ repository.go
```

In this file, I will create a type for the wrapper:

```go
type withTrace struct {
	r Repository
}

func WithTrace(r Repository) Repository {
	return withTrace{
		r: r,
	}
}
```

And now, to implement the methods in the `keys.Repository`. Starting with `Set` to go through one of them as an example. All of the others will be exactly the same:

1. Create a base method that just returns the wrapped repository's call. This is the base for all wrappers of this kind. I can then add functionalities that take place before or after the call:

```go
// Set creates or overwrites a secret identified by `k` with value `v`, in
// bucket `bucket`. Returns an error
func (t withTrace) Set(ctx context.Context, bucket, k string, v []byte) error {
	return t.r.Set(ctx, bucket, k, v)
}
```

2. For tracing, I will start a span identifying this action, under the same context. I will shadow the incoming context with the one created by the tracer so that it can cascade through calls as required:

```go
func (t withTrace) Set(ctx context.Context, bucket, k string, v []byte) error {
	ctx, s := spanner.Start(ctx, "keys.Set")
	defer s.End()
	s.Add(
		attr.String("in_bucket", bucket),
	)

	return t.r.Set(ctx, bucket, k, v)
}
```

> In the above action, I create a span and add an attribute to it with the target bucket name (no PII; I will not register the key or value)

3. Additionally, I will want to register an event in the span when an error is raised on this call. Time to change that return statement; I will accept the error value, check if it's nil and register an event if so:

```go
func (t withTrace) Set(ctx context.Context, bucket, k string, v []byte) error {
	ctx, s := spanner.Start(ctx, "keys.Set")
	defer s.End()
	s.Add(
		attr.String("in_bucket", bucket),
	)

	err := t.r.Set(ctx, bucket, k, v)
	if err != nil {
		s.Event("error setting key", attr.New("error", err.Error()))
		return err
	}
	return nil
}
```

And that's all for tracing! I will repeat exactly the same pattern for the other 3 methods:

```go
// Get fetches the secret identified by `k` in the bucket `bucket`,
// returning a slice of bytes for the value and an error
func (t withTrace) Get(ctx context.Context, bucket, k string) ([]byte, error) {
	ctx, s := spanner.Start(ctx, "keys.Get")
	defer s.End()
	s.Add(
		attr.String("in_bucket", bucket),
	)

	value, err := t.r.Get(ctx, bucket, k)
	if err != nil {
		s.Event("error fetching key", attr.New("error", err.Error()))
		return value, err
	}
	return value, nil
}

// Delete removes the secret identified by `k` in bucket `bucket`, returning an error
func (t withTrace) Delete(ctx context.Context, bucket, k string) error {
	ctx, s := spanner.Start(ctx, "keys.Delete")
	defer s.End()
	s.Add(
		attr.String("in_bucket", bucket),
	)

	err := t.r.Delete(ctx, bucket, k)
	if err != nil {
		s.Event("error deleting key", attr.New("error", err.Error()))
		return err
	}
	return nil
}

// Purge removes all the secrets in the bucket `bucket`, returning an error
func (t withTrace) Purge(ctx context.Context, bucket string) error {
	ctx, s := spanner.Start(ctx, "keys.Purge")
	defer s.End()
	s.Add(
		attr.String("in_bucket", bucket),
	)

	err := t.r.Purge(ctx, bucket)
	if err != nil {
		s.Event("error deleting bucket", attr.New("error", err.Error()))
		return err
	}
	return nil
}
```


#### `user.Repository` observability

Just like the above approach, I will add a file in `user/user_with_trace.go`:

```
.
└─ user
    ├─ repository.go 
    ├─ user_with_trace.go -- tracing wrapper for user.Repository
    ├─ user.go 
    └─ validate.go
```

And exactly like before, it will wrap an input `user.Repository`. Here's the entire contents of the file, since it's just like the keys approach. I am only pasting the first method's implementation to avoid the repetition:

```go
package user

import (
	"context"

	"github.com/zalgonoise/attr"
	"github.com/zalgonoise/spanner"
)

type withTrace struct {
	r Repository
}

func WithTrace(r Repository) Repository {
	return withTrace{
		r: r,
	}
}

// Create will create a user `u`, returning its ID and an error
func (t withTrace) Create(ctx context.Context, u *User) (uint64, error) {
	ctx, s := spanner.Start(ctx, "user.Create")
	defer s.End()
	s.Add(
		attr.String("username", u.Username),
	)

	id, err := t.r.Create(ctx, u)
	if err != nil {
		s.Event("error creating user", attr.New("error", err.Error()))
		return id, err
	}
	return id, nil
}

// (...)
```


#### `secret.Repository` observability

Same goes for secrets, I will add a file in `secret/secret_with_trace.go`:

```
.
└─ secret
    ├─ repository.go 
    ├─ secret_with_trace.go -- tracing wrapper for secret.Repository
    ├─ secret.go 
    └─ validate.go
```

Here's the contents of this file. I am only pasting the first method's implementation to avoid the repetition:

```go
package secret

import (
	"context"

	"github.com/zalgonoise/attr"
	"github.com/zalgonoise/spanner"
)

type withTrace struct {
	r Repository
}

func WithTrace(r Repository) Repository {
	return withTrace{
		r: r,
	}
}

// Create will create (or overwrite) the secret identified by `s.Key`, for user `username`,
// returning an error
func (t withTrace) Create(ctx context.Context, username string, secr *Secret) (uint64, error) {
	ctx, s := spanner.Start(ctx, "secret.Create")
	defer s.End()
	s.Add(
		attr.String("for_user", username),
	)

	id, err := t.r.Create(ctx, username, secr)
	if err != nil {
		s.Event("error creating secret", attr.New("error", err.Error()))
		return id, err
	}
	return id, nil
}

// (...)
```

#### `shared.Repository` observability

And now for shares, I will add a file in `shared/shared_with_trace.go`:

```
.
└─ secret
    ├─ repository.go 
    ├─ shared_with_trace.go -- tracing wrapper for secret.Repository
    ├─ shared.go 
    └─ validate.go
```

And here is the content of this file. I am only pasting the first method's implementation to avoid the repetition:

```go
package shared

import (
	"context"

	"github.com/zalgonoise/attr"
	"github.com/zalgonoise/spanner"
)

type withTrace struct {
	r Repository
}

func WithTrace(r Repository) Repository {
	return withTrace{
		r: r,
	}
}

// Create shares the secret identified by `secretName`, owned by `owner`, with
// user `target`. Returns an error
func (t withTrace) Create(ctx context.Context, sh *Share) (uint64, error) {
	ctx, s := spanner.Start(ctx, "secret.Create")
	defer s.End()
	s.Add(
		attr.String("from_user", sh.Owner),
	)

	id, err := t.r.Create(ctx, sh)
	if err != nil {
		s.Event("error creating shared secret", attr.New("error", err.Error()))
		return id, err
	}
	return id, nil
}

// (...)
```


#### `authz.Authorizer` observability

The Authorizer interface can and should also be instrumented. For this, I will add a file: `authz/authz_with_trace.go`:


```
.
└─ authz
    ├─ authz_with_trace.go
    └─ jwt.go
```

Here's the contents of this file, for `NewToken` and `Parse` methods:

```go
package authz

import (
	"context"

	"github.com/zalgonoise/attr"
	"github.com/zalgonoise/spanner"
	"github.com/zalgonoise/x/secr/user"
)

type withTrace struct {
	r Authorizer
}

func WithTrace(r Authorizer) Authorizer {
	return withTrace{
		r: r,
	}
}

// NewToken returns a new JWT for the user `u`, and an error
func (t withTrace) NewToken(ctx context.Context, u *user.User) (string, error) {
	ctx, s := spanner.Start(ctx, "authz.NewToken")
	defer s.End()
	s.Add(
		attr.String("username", u.Username),
	)

	token, err := t.r.NewToken(ctx, u)
	if err != nil {
		s.Event("error creating token for user", attr.New("error", err.Error()))
		return token, err
	}
	return token, nil
}

// Parse returns the data from a valid JWT
func (t withTrace) Parse(ctx context.Context, token string) (*user.User, error) {
	ctx, s := spanner.Start(ctx, "authz.Parse")
	defer s.End()

	u, err := t.r.Parse(ctx, token)
	if err != nil {
		s.Event("error parsing token", attr.New("error", err.Error()))
		return u, err
	}
	s.Add(
		attr.String("username", u.Username),
	)

	return u, nil
}
```


#### `service.Service` observability

This will be a big one (in size)! The service has tons of methods.

Instead of pasting all of the methods' contents, I will go through only the first one. 

The service will have a tracer and a logger. There are solutions where this can be done in one-shot (*screaming OpenTelemetry*), but notably shows how a different wrapper would be implemented (the logger only reacts to errors).

I will add two files to the `service` folder:

```
.
└─ service
    ├─ secrets.go
    ├─ service_with_logger.go
    ├─ service_with_trace.go
    ├─ service.go
    ├─ sessions.go
    ├─ shares.go
    ├─ transaction.go
    └─ users.go
```

For **`service/service_with_trace.go`**:

Just like in the repositories, I add a new type that wraps a `service.Service`:

```go
type withTrace struct {
	r Service
}

func WithTrace(r Service) Service {
	return withTrace{
		r: r,
	}
}
```

Then, I implement each of the service methods, starting a span and calling the wrapped service method. If there is an error I issue an event (on the span) for it, and I return the call's data as normal:

```go
// Login verifies the user's credentials and returns a session and an error
func (t withTrace) Login(ctx context.Context, username, password string) (*user.Session, error) {
	ctx, s := spanner.Start(ctx, "service.Login")
	defer s.End()
	s.Add(
		attr.String("username", username),
	)

	session, err := t.r.Login(ctx, username, password)
	if err != nil {
		s.Event("error logging user in", attr.New("error", err.Error()))
		return session, err
	}
	return session, nil
}
```


This pattern will be repeated for all methods in the service.

For **`service/service_with_logger.go`**:

The type wrapping the service will also contain a logger (that would be picked by the developer implementing their app). It is assumed that this would be an already configured logger that is ready to be used, so the function creating this type just takes in a logger and service, returning the wrapper type:


```go
type withLogger struct {
	r Service
	l logx.Logger
}

func WithLogger(l logx.Logger, r Service) Service {
	return withLogger{
		r: r,
		l: l,
	}
}
```

As for each of the method's implementation, in this case I just want to register errors in the service layer. So I start by calling the wrapped service method and checking on the returned error, logging it with an `Error` level with some metadata, none of it should be PII:

```go
// Login verifies the user's credentials and returns a session and an error
func (l withLogger) Login(ctx context.Context, username, password string) (*user.Session, error) {
	session, err := l.r.Login(ctx, username, password)
	if err != nil {
		l.l.Error(
			err.Error(),
			attr.String("service", "service.Login"),
			attr.String("username", username),
		)
		return session, err
	}
	return session, nil
}
```

The remaining methods will be just like these.


#### Updating factories

To make use of the instrumentation above, it should wrap the repositories and service when created. This is done on the `factory` package.

Taking a look at `factory/database.go`, I can update the function returns to add tracing to the repositories:

```go
// SQLite creates user and secret repositories based on the defined SQLite DB path
func SQLite(path string) (user.Repository, secret.Repository, shared.Repository, error) {
	// (...)
	return user.WithTrace(sqlite.NewUserRepository(db)),
		secret.WithTrace(sqlite.NewSecretRepository(db)),
		shared.WithTrace(sqlite.NewSharedRepository(db)),
		nil
}

// Bolt creates a key repository based on the defined Bolt DB path
func Bolt(path string) (keys.Repository, error) {
	// (...)
	return keys.WithTrace(bolt.NewKeysRepository(db)), nil
}
```

I will do the same for `factory/authz.go`, for the `Authorizer` function:

```go
// Authorizer creates a new authorizer with the input key `key`, or creates a new
// one under the preset folder if it doesn't yet exist and is not provided
func Authorizer(path string) (auth authz.Authorizer, err error) {
	// (...)
	return authz.WithTrace(auth), nil
}
```

Before doing the same for `factory/factory.go` (for the service), I need to actually configure a logger (that is used as input). 

I can take the chance to add `factory/logger.go` and `factory/spanner.go` to initialize the configuration for both.

For `factory/logger.go` I will use my [`zalgonoise/logx`](https://github.com/zalgonoise/logx) package, only because I am familiar with it (and has decent performance so I know it's not hurting this app). In a nutshell, similar to the other file-based factory functions, I check if the input path is valid, try to work with it (creating the file if empty or non-existing), otherwise fallback to the default path. If that doesn't work either, I will stick to standard-out. Otherwise, the log entries are piped to standard out in text format *and* to the file, in JSON format.

```go
import (
	"os"

	"github.com/zalgonoise/logx"
	"github.com/zalgonoise/logx/handlers"
	"github.com/zalgonoise/logx/handlers/jsonh"
	"github.com/zalgonoise/logx/handlers/texth"
)

const (
	logFilePath = "/secr/error.log"
)

// Logger loads the file in the path `path`, to store the error log entries in,
// defaulting to a std.out output if the path is empty or invalid
func Logger(path string) logx.Logger {
	if path == "" {
		return noLogfile()
	}
	l, err := createLogfile(path)
	if err != nil {
		return noLogfile()
	}
	return l
}

func createLogfile(p string) (logx.Logger, error) {
	f, err := os.Create(p)
	if err != nil {
		if p == logFilePath {
			return nil, err
		}
		return createLogfile(logFilePath)
	}
	return logx.New(handlers.Multi(
		texth.New(os.Stdout),
		jsonh.New(f),
	)), nil
}

func noLogfile() logx.Logger {
	return logx.New(texth.New(os.Stdout))
}
```

As for the spanner, it will follow the same pattern: try to use the input path; fallback to the default path; default to standard error. If the path works fine, it will pipe traces to both standard error and to the trace file:

```go
import (
	"io"
	"os"

	"github.com/zalgonoise/spanner"
)

const (
	traceFilePath = "/secr/trace.json"
)

// Spanner loads the file in the path `path`, to store the spanner entries in,
// defaulting to a std.err output if the path is empty or invalid
func Spanner(path string) {
	err := createTracefile(path)
	if err != nil {
		noTracefile()
	}
}

func createTracefile(p string) error {
	f, err := os.Create(p)
	if err != nil {
		if p == traceFilePath {
			return err
		}
		return createTracefile(traceFilePath)
	}

	spanner.To(spanner.Writer(io.MultiWriter(f, os.Stderr)))
	return nil
}

func noTracefile() {
	spanner.To(spanner.Writer(os.Stderr))
}
```

Now, going back to `factory/factory.go` I am able to configure the service with the logger and tracer:


```go
package factory

import (
	"github.com/zalgonoise/x/secr/cmd/config"
	"github.com/zalgonoise/x/secr/service"
	"github.com/zalgonoise/x/secr/transport/http"
)

// (...)

// WithLogAndTrace configures a service to write on a specific trace file and log file
func WithLogAndTrace(traceFilePath, logFilePath string, svc service.Service) service.Service {
	Spanner(traceFilePath)
	return service.WithLogger(
		Logger(logFilePath),
		service.WithTrace(svc),
	)
}

// From creates a HTTP server for the Secrets service based on the input config
func From(conf *config.Config) (http.Server, error) {
	svc, err := Service(conf.SigningKeyPath, conf.BoltDBPath, conf.SQLiteDBPath)
	if err != nil {
		return nil, err
	}

	loggedSvc := WithLogAndTrace(
		conf.TraceFilePath,
		conf.LogFilePath,
		svc,
	)

	return http.NewServer(conf.HTTPPort, loggedSvc), nil
}
```


#### HTTP observability

It's all fine and well with the service and repository instrumentation but what about HTTP? That's next on the list.

HTTP will not have its own wrapper but good news: the `ghttp` package already starts up a context with a span (with UUID, caller's IP, and caller's user-agent), via its embed `NewCtxAndSpan` function:


```go
// NewCtxAndSpan creates a new context for service `service` with attributes `attrs`, scoped to
// a "req" namespace that includes a UUID for the request and the service string `service`,
// and creates a Span for the given request.
// The context is also wrapped with a JSON encoder so the response writer can use it
//
// The input *http.Request `r` is used to register the remote address and user agent in the root span
//
// The resulting context is returned alongside the created Span
func NewCtxAndSpan(r *http.Request, service string, attrs ...attr.Attr) (context.Context, spanner.Span) {
	ctx, span := spanner.Start(r.Context(), service)
	span.Add(
		attr.New("req", attr.Attrs{
			attr.String("module", service),
			attr.String("req_id", uuid.New().String()),
			attr.String("remote_addr", r.RemoteAddr),
			attr.String("user_agent", r.UserAgent()),
		}),
	)
	span.Add(attrs...)

	return ctx, span
}
```

And refreshing on `ghttp.Do`:

```go
// Do is a generic function that creates a HandlerFunc which will take in a context and a query object, and returns
// a HTTP status, a response message, a response object and an error
func Do[Q any, A any](name string, parseFn ParseFn[Q], queryFn ExecFn[Q, A]) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx, s := NewCtxAndSpan(r, name)
		defer s.End()

		// (...)
	}
}
```

This means that all calls to `ghttp.Do` were already passing in a service name that identifies the span. While it doesn't end there, this is halfway of instrumenting HTTP.

Now, it is important to go through each HandlerFunc and understanding what should be instrumented and where. Since this goes on top of the http logic, I will want to reduce the added LOC to the least possible.

Let me use `server.usersUpdate` as an example since it contains a convoluted `ParseFn` and `ExecFn`.

It's not necessary to add tracing to the `ParseFn` since it's taken care of by `ghttp.Do`, since it is returning an error already.

The `ExecFn` will have instrumentation to identify (some) of the added parameters, errors, and successful requests if it's the case.

Starting with the existing `ExecFn` for `server.usersUpdate`:

```go
	var execFn = func(ctx context.Context, q *usersUpdateRequest) *ghttp.Response[user.User] {
		if q == nil {
			return ghttp.NewResponse[user.User](http.StatusBadRequest, "invalid request")
		}

		u := &user.User{
			Username: q.Username,
		}

		err := s.s.UpdateUser(ctx, q.Username, u)
		if err != nil {
			return ghttp.NewResponse[user.User](http.StatusInternalServerError, err.Error())
		}

		dbUser, err := s.s.GetUser(ctx, q.Username)
		if err != nil {
			return ghttp.NewResponse[user.User](http.StatusInternalServerError, err.Error())
		}

		return ghttp.NewResponse[user.User](http.StatusOK, "user updated successfully").WithData(dbUser)
	}
```

1. I will start with a new span, specific to the exec function (there is already a span for this call as `UpdateUser`):

```go
	var execFn = func(ctx context.Context, q *usersUpdateRequest) *ghttp.Response[user.User] {
		ctx, span := spanner.Start(ctx, "http.UpdateUser:exec")
		defer span.End()
	// (...)
	}
```

2. Next, I will register meaningful attributes if the object is not nil (in this case, username and new name):

```go
		if q == nil {
			return ghttp.NewResponse[user.User](http.StatusBadRequest, "invalid request")
		}
		span.Add(
			attr.String("for_user", q.Username),
			attr.String("new_name", q.Name),
		)
```

3. As for the service calls, any errors will result in a status code above 399, which will trigger `ghttp.Do` to register an appropriate event with the error message in the Response. So this will do.

This pattern will be seen in all HTTP endpoints.

Peeking into a trace file after a login event, I can extract a good amount of metadata without breaking confidentiality in any way. Here is the trace collected for a user's login event:

```json
{
  "name": "http.readBody",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "b71e6bddb80be89a"
  },
  "parent_id": "98e1532617a5ed5e",
  "start_time": "2023-01-28T18:11:51.19229419+01:00",
  "end_time": "2023-01-28T18:11:51.192398266+01:00",
  "attributes": {
    "for_type": "http.loginRequest"
  }
}
{
  "name": "user.Get",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "87e44f9c5bb2c87c"
  },
  "parent_id": "d2dcfb08082d8acc",
  "start_time": "2023-01-28T18:11:51.192433463+01:00",
  "end_time": "2023-01-28T18:11:51.192718701+01:00",
  "attributes": {
    "username": "zalgo"
  }
}
{
  "name": "authz.NewToken",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "4c1740c775ba71e3"
  },
  "parent_id": "d2dcfb08082d8acc",
  "start_time": "2023-01-28T18:11:52.144512065+01:00",
  "end_time": "2023-01-28T18:11:52.144639025+01:00",
  "attributes": {
    "username": "zalgo"
  }
}
{
  "name": "keys.Set",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "b7fab38da61c4977"
  },
  "parent_id": "d2dcfb08082d8acc",
  "start_time": "2023-01-28T18:11:52.144647912+01:00",
  "end_time": "2023-01-28T18:11:52.144692596+01:00",
  "attributes": {
    "in_bucket": "uid:1"
  }
}
{
  "name": "service.Login",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "d2dcfb08082d8acc"
  },
  "parent_id": "f6ed9705ebddba03",
  "start_time": "2023-01-28T18:11:51.192407814+01:00",
  "end_time": "2023-01-28T18:11:52.144698317+01:00",
  "attributes": {
    "username": "zalgo"
  }
}
{
  "name": "http.Login:exec",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "f6ed9705ebddba03"
  },
  "parent_id": "98e1532617a5ed5e",
  "start_time": "2023-01-28T18:11:51.192404598+01:00",
  "end_time": "2023-01-28T18:11:52.1447001+01:00",
  "attributes": {
    "for_user": "zalgo"
  }
}
{
  "name": "http.HttpResponse.WriteHTTP",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "b10d3ffedf31c9a0"
  },
  "parent_id": "98e1532617a5ed5e",
  "start_time": "2023-01-28T18:11:52.144703076+01:00",
  "end_time": "2023-01-28T18:11:52.144836177+01:00",
  "attributes": {
    "for_type": "user.Session",
    "http_status": 200
  }
}
{
  "name": "Login",
  "context": {
    "trace_id": "dc7298d52f5e2b076e0ae15ef57c2068",
    "span_id": "98e1532617a5ed5e"
  },
  "parent_id": null,
  "start_time": "2023-01-28T18:11:51.192282157+01:00",
  "end_time": "2023-01-28T18:11:52.14483802+01:00",
  "attributes": {
    "req": {
      "module": "Login",
      "remote_addr": "127.0.0.1:40586",
      "req_id": "87bf64f3-1c53-404a-a132-328591a6295c",
      "user_agent": "curl/7.87.0"
    }
  }
}
```

This was the original HTTP request and response:

```
curl -v -X POST 127.0.0.1:8080/login -d '{"username":"zalgo","password":"secretpassword"}'  
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying 127.0.0.1:8080...
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> POST /login HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.87.0
> Accept: */*
> Content-Length: 42
> Content-Type: application/x-www-form-urlencoded
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Sat, 28 Jan 2023 17:11:52 GMT
< Content-Length: 443
< Content-Type: text/plain; charset=utf-8
< 
```

```json
{
  "message": "user logged in successfully",
  "data": {
    "user": {
      "id": 1,
      "username": "zalgo",
      "name": "Zalgo",
      "created_at": "2023-01-28T17:11:47Z",
      "updated_at": "2023-01-28T17:11:47Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2NzQ5Mjk1MTIsInVzZXIiOnsidXNlcm5hbWUiOiJ6YWxnbyIsIm5hbWUiOiJaYWxnbyIsImNyZWF0ZWRfYXQiOiIyMDIzLTAxLTI4VDE3OjExOjQ3WiIsInVwZGF0ZWRfYXQiOiIyMDIzLTAxLTI4VDE3OjExOjQ3WiJ9fQ.z-8MgM5HX63PFr91AF7KMChuCOTtcsyspqQfHweEJ-4"
  }
}
```

Decoded JWT's payload:

```json
{
  "exp": 1674929512,
  "user": {
    "username": "zalgo",
    "name": "Zalgo",
    "created_at": "2023-01-28T17:11:47Z",
    "updated_at": "2023-01-28T17:11:47Z"
  }
}
```

[next](wrapping_up.md)