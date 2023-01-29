[back](factories.md) | [index](index.md)

### The `cmd` package

This will be a top-level folder in the project to write the app's runtime as a CLI app, as well as a configuration for it (for the caller to set different file paths for the databases and JWT signing key, for example).

```
.
└─ cmd
    ├─ config
	│    ├─ config.go  -- app config structure 
	│    └─ options.go -- configurable options 
    ├─ flags
	│    └─ flags.go -- CLI flag / OS env parser
    └─ server.go -- server runtime
```

#### App configuration

Configuring the app will take a placeholder struct for all of the options. Not that many options to configure at first. Also, adding a `Default` object that is used as reference for configuring the app:

```go
// Config describes the app configuration
type Config struct {
	HTTPPort       int    `json:"http_port,omitempty" yaml:"http_port,omitempty"`
	BoltDBPath     string `json:"bolt_path,omitempty" yaml:"bolt_path,omitempty"`
	SQLiteDBPath   string `json:"sqlite_path,omitempty" yaml:"sqlite_path,omitempty"`
	SigningKeyPath string `json:"jwt_key_path,omitempty" yaml:"jwt_key_path,omitempty"`
}

// Default is a default configuration that the app will kick-off with, if not configured
var Default = Config{
	HTTPPort:       8080,
	BoltDBPath:     "/secr/keys.db",
	SQLiteDBPath:   "/secr/sqlite.db",
	SigningKeyPath: "/secr/server/key",
}
```

Adding the options to the `Config` object will use a neat pattern that I like very much:

```go
// Option describes setter types for a Config
//
// As new options / elements are added to the Config, new data structures can
// implement the Option interface to allow setting these options in the Config
type Option interface {
	// Apply sets the configuration on the input Config `c`
	Apply(c *Config)
}
```

This way, different types can implement the `Option` interface, and can apply that option to the config.

And also, a `New` function will initialize a `Config` based off of the `Default`, and apply all user provided `Option` to the object:

```go
// New initializes a new config with default settings, and then iterates through
// all input Option `opts` applying them to the Config, which is returned
// to the caller
func New(opts ...Option) *Config {
	conf := &Default

	for _, opt := range opts {
		if opt != nil {
			opt.Apply(conf)
		}
	}

	return conf
}
```

Lastly, since I know the app will support both CLI flags and OS environment variables, I may need to merge two configs. This merge method basically checks if the input config's elements are set, and overwrites the caller's with its values. I really wanted to write a generic function for this as the repetition looks silly:

```go
// Merge combines Configs `c` with `input`, returning a merged version
// of the two
//
// All set elements in `input` will be applied to `c`, and the unset elements
// will be ignored (keeps `c`'s data)
func (c *Config) Merge(input *Config) *Config {
	if input.HTTPPort != 0 {
		c.HTTPPort = input.HTTPPort
	}
	if input.BoltDBPath != "" {
		c.BoltDBPath = input.BoltDBPath
	}
	if input.SQLiteDBPath != "" {
		c.SQLiteDBPath = input.SQLiteDBPath
	}
	if input.SigningKeyPath != "" {
		c.SigningKeyPath = input.SigningKeyPath
	}
	return c
}

```

#### Port option

The `Option` will be a type that implements `Apply(*Config)`, and I will expose a function for creating it. The same goes for the other options:

```go
type httpPort int

// Apply sets the configuration on the input Config `c`
func (p httpPort) Apply(c *Config) {
	c.HTTPPort = (int)(p)
}

// Port defines the HTTP port for the server
func Port(port int) Option {
	if port == 0 {
		return nil
	}
	return (httpPort)(port)
}
```


#### Bolt option

```go
type boltPath string

// Apply sets the configuration on the input Config `c`
func (p boltPath) Apply(c *Config) {
	c.BoltDBPath = (string)(p)
}

// BoltDB defines the path for the Bolt database file
func BoltDB(path string) Option {
	if path == "" {
		return nil
	}
	return (boltPath)(path)
}
```


#### SQLite option

```go
type sqlitePath string

// Apply sets the configuration on the input Config `c`
func (p sqlitePath) Apply(c *Config) {
	c.SQLiteDBPath = (string)(p)
}

// SQLiteDB defines the path for the SQLite database file
func SQLiteDB(path string) Option {
	if path == "" {
		return nil
	}
	return (sqlitePath)(path)
}
```

#### JWT Key option

```go
type jwtKeyPath string

// Apply sets the configuration on the input Config `c`
func (p jwtKeyPath) Apply(c *Config) {
	c.SigningKeyPath = (string)(p)
}

// JWTKey defines the path for the JWT signing key file
func JWTKey(path string) Option {
	if path == "" {
		return nil
	}
	return (jwtKeyPath)(path)
}
```

#### CLI flags

There are great CLI flags packages out there like Viper and Cobra, however I will use the standard library's `flag` package.

This is a function that is called on the (CLI) server runtime to read the options passed on the command-line.

```go
// ParseFlags will consume the CLI flags as the app is executed
func ParseFlags() *config.Config {
	var conf = &config.Default

	boltDBPath := flag.String("bolt-path", conf.BoltDBPath, "path to the Bolt database file")
	sqliteDBPath := flag.String("sqlite-path", conf.SQLiteDBPath, "path to the SQLite database file")
	signingKeyPath := flag.String("jwt-key", conf.SigningKeyPath, "path to the JWT signing key file")
	logfilePath := flag.String("logfile-path", conf.LogFilePath, "path to the logfile stored in the service")
	tracefilePath := flag.String("tracefile-path", conf.TraceFilePath, "path to the tracefile stored in the service")

	flag.Parse()

	return conf.Apply(
		config.BoltDB(*boltDBPath),
		config.SQLiteDB(*sqliteDBPath),
		config.JWTKey(*signingKeyPath),
		config.Logfile(*logfilePath),
		config.Tracefile(*tracefilePath),
	)
}
```

#### OS environment variables

Since the app may be containerized (or the user may just like OS environment variables more), I want to support those as a way to configure the app, too.

Adding a `ParseOSEnv` function that will fetch a specific environment variable, regardless if it's empty or not:

```go
// ParseOSEnv will consume the OS environment variables associated with this app, when executed
func ParseOSEnv() *config.Config {
	return &config.Config{
		BoltDBPath:     os.Getenv("SECR_BOLT_PATH"),
		SQLiteDBPath:   os.Getenv("SECR_SQLITE_PATH"),
		SigningKeyPath: os.Getenv("SECR_JWT_KEY_PATH"),
		LogFilePath:    os.Getenv("SECR_LOGFILE_PATH"),
		TraceFilePath:  os.Getenv("SECR_TRACEFILE_PATH"),
	}
}
```

Then, I can update the `ParseFlags` function to merge in any set OS environment variables. This means that setting an environment variable takes precedence over the same config issued on the CLI:

```go
// ParseFlags will consume the CLI flags as the app is executed
func ParseFlags() *config.Config {
	var conf = &config.Default
	// (...)
	flag.Parse()
	osFlags := ParseOSEnv()

	conf.Apply(
		config.BoltDB(*boltDBPath),
		config.SQLiteDB(*sqliteDBPath),
		config.JWTKey(*signingKeyPath),
		config.Logfile(*logfilePath),
		config.Tracefile(*tracefilePath),
	)

	return conf.Merge(osFlags)
}
```

#### Improving the factory

I will now add a new factory function in `factory/factory.go`, `From(*config.Config)`, to get a server ready just from a configuration object:

```go
// From creates a HTTP server for the Secrets service based on the input config
func From(conf *config.Config) (http.Server, error) {
	svc, err := Service(conf.SigningKeyPath, conf.BoltDBPath, conf.SQLiteDBPath)
	if err != nil {
		return nil, err
	}

	return http.NewServer(conf.HTTPPort, svc), nil
}
```

#### Server runtime

This file (`cmd/server.go`) will determine how the server runtime should be. It's not a `package main` but it will by called by `main.go` in the root of the project.

The `Run` function in this file will not do anything major besides reading the configuration (from CLI flags and OS env), creating the server, and starting it:


```go
import (
	"context"
	"os"

	"github.com/zalgonoise/attr"
	"github.com/zalgonoise/logx"
	"github.com/zalgonoise/x/secr/cmd/flags"
	"github.com/zalgonoise/x/secr/factory"
)

// Run executes the app by initializing its configuration and running the HTTP server
func Run() {
	// temp logger
	log := logx.Default()

	conf := flags.ParseFlags()
	server, err := factory.From(conf)
	if err != nil {
		log.Fatal("failed to initialize secrets server", attr.String("error", err.Error()))
		os.Exit(1)
	}

	err = server.Start(ctx)
	if err != nil {
		log.Fatal("failed to start the secrets server", attr.String("error", err.Error()))
		os.Exit(1)
	}
}
```

Lastly, the actual `package main` for this app, in `main.go`, nice and short:

```go
package main

import "github.com/zalgonoise/x/secr/cmd"

func main() {
	cmd.Run()
}
```

Surprisingly that's 90% it. The remaining 10% are to ensure that if the app breaks we know why, so it's time to add some logging and traces in the observability section!


[next](../observability_and_middleware.md)