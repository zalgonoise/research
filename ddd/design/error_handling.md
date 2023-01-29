[back](app_structure.md) | [index](index.md)


### Error handling
This app will work with several errors, sometimes with multiple errors wrapped together (a general database error and the actual error raised).

Since as of now (Go v1.19) the standard library does not support wrapping multiple errors together, the statement below **is not** possible:

```go
err := fmt.Errorf("errors: %w -- %w", err1, err2)
```

However, with Go 1.20, the `errors` package is greatly improved with joined errors.

I am eager to use this feature. So much that I externalized in in my [`zalgonoise/x/errors`](https://github.com/zalgonoise/x/tree/master/errors) package. This is an exact replica of [Go 1.20's `errors`](https://github.com/golang/go/tree/master/src/errors), with `reflectlite` replaced with regular `reflect`.

This allows me to join errors together like the statement below (and still having `errors.Is` verify if the nested error is included or not):


```go
err := errors.Join(err1, err2)
```

As such, you will find in this repo both references to `fmt.Errorf("blablabla: %w", err)` and `errors.Join(ErrSomething, err)`.


[next](entities.md)