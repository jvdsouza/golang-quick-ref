# Errors

## Basics

```go
type error interface { Error() string }

f, err := os.Open(path)
if err != nil {
    return fmt.Errorf("open config %q: %w", path, err)
}
```

- The error is the **last** return value.
- Handle it once: either **return it** (with context) or **log/handle it**. Not both.
- Messages are lowercase with no trailing punctuation, because they get chained: `load user 42: query: connection refused`.

## Three ways to define errors

```go
// 1. Sentinel: a fixed value callers compare against
var ErrNotFound = errors.New("not found")

// 2. Custom type: carries data
type ValidationError struct {
    Field string
    Msg   string
}
func (e *ValidationError) Error() string { return e.Field + ": " + e.Msg }

// 3. Ad-hoc: when callers don't need to inspect it
return fmt.Errorf("bad header %q", h)
```

## Wrapping

```go
return fmt.Errorf("get user %d: %w", id, err)   // %w wraps, so the chain is preserved
return fmt.Errorf("get user %d: %v", id, err)   // %v only formats, so the chain is BROKEN
```

Go 1.20+ supports multiple `%w` verbs and `errors.Join`:

```go
err := errors.Join(err1, err2)           // Is/As check both
err := fmt.Errorf("%w; %w", err1, err2)
```

Wrapping makes the inner error **part of your API**. Wrap with `%w` when callers should be able to inspect it, and use `%v` to hide implementation details.

## errors.Is: "is this error (or anything it wraps) X?"

```go
if errors.Is(err, ErrNotFound) { ... }
if errors.Is(err, os.ErrNotExist) { ... }
if errors.Is(err, context.DeadlineExceeded) { ... }

err == ErrNotFound    // WRONG once anything wraps it
```

It walks the `Unwrap()` chain comparing with `==`, or calls the error's own `Is(target) bool` method if it has one.

## errors.As: "find an error of this TYPE in the chain"

```go
var ve *ValidationError
if errors.As(err, &ve) {          // pass a POINTER to your target variable
    fmt.Println("bad field:", ve.Field)
}

var pe *fs.PathError
if errors.As(err, &pe) { fmt.Println(pe.Path) }

// Go 1.26+: a generic version, no target variable needed
if ve, ok := errors.AsType[*ValidationError](err); ok { ... }
```

Pointer-receiver gotcha: `Error()` is defined on `*ValidationError`, so the target is `var ve *ValidationError` and you pass `&ve`. Using `var ve ValidationError` won't match.

## Making your own type wrap

```go
type QueryError struct {
    Query string
    Err   error
}
func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }
func (e *QueryError) Unwrap() error { return e.Err }   // lets Is/As see through it
```

## Returning vs panicking

| Return an `error` | `panic` |
|---|---|
| Expected failures: I/O, network, bad input, not found | Programmer bugs or impossible states (nil deref, index out of range) |
| Anything a caller might reasonably handle | Broken invariants where continuing would corrupt data |
| Library code (almost always) | `MustX` helpers used at init time: `regexp.MustCompile`, `template.Must` |

```go
// recover only works inside a deferred function, in the SAME goroutine
func safe(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    fn()
    return nil
}
```

- An unrecovered panic in **any** goroutine crashes the whole program. A panic doesn't cross to the parent goroutine.
- `net/http` recovers panics per request. Your own background goroutines need their own recovery if you want them to survive.
- Don't use panic/recover as exceptions for control flow.

## Typed nil trap (again)

```go
func f() error {
    var e *MyErr      // nil pointer
    return e          // non-nil error interface!
}
```

Always `return nil` explicitly on success. See [interfaces](05-interfaces.md).

## Be ready to explain

- `%w` vs `%v`, and when you'd deliberately not wrap.
- The difference between `errors.Is` (value) and `errors.As` (type), and why `As` takes a pointer.
- When a panic is acceptable.
