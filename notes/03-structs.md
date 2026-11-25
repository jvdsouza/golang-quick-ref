# Structs

## Zero values

Every field starts at its zero value: `0`, `""`, `false`, `nil` (pointers, slices, maps, chans, funcs, interfaces), and zeroed nested structs.

```go
type Config struct {
    Port    int
    Host    string
    Tags    []string          // nil: len, range, and append all work
    Limits  map[string]int    // nil: reads work, WRITES PANIC
}
var c Config                  // fully usable without a constructor? Only if you design for it.
```

**Make the zero value useful** where you can. The stdlib does this:

```go
var mu sync.Mutex       // ready to Lock()
var buf bytes.Buffer    // ready to Write()
var sb strings.Builder  // ready to WriteString()
```

When the zero value can't work (it needs a map, a connection, or validation), provide `NewX(...)`.

## Literals

```go
p := Point{X: 1, Y: 2}   // keyed: prefer this, it survives field reordering
p := Point{1, 2}         // positional: fragile, and go vet flags it for other packages' types
p := &Point{X: 1}        // pointer to a new struct; Y is 0

// Anonymous structs work well for one-offs and test tables
tests := []struct {
    name string
    in   int
    want int
}{
    {"zero", 0, 0},
    {"one", 1, 1},
}
```

Structs are **comparable** with `==` if all their fields are (no slices, maps, or funcs). That means they can be map keys.

## Composition over inheritance

Go has no inheritance. You build types by **holding** other types:

```go
type Server struct {
    log  *slog.Logger   // named field: explicit, nothing promoted
    db   *sql.DB
}
```

## Embedding

An embedded field (type with no field name) **promotes** its fields and methods to the outer type.

```go
type Logger struct{ prefix string }
func (l *Logger) Log(msg string) { fmt.Println(l.prefix + msg) }

type Server struct {
    *Logger            // embedded; the field name is "Logger"
    Addr string
}

s := Server{Logger: &Logger{prefix: "[srv] "}, Addr: ":8080"}
s.Log("start")         // promoted: same as s.Logger.Log("start")
```

Promoted methods also count toward **interface satisfaction**: `Server` now has a `Log` method.

### Embedding is NOT inheritance

The receiver of a promoted method is the **embedded value**, not the outer struct. There's no virtual dispatch.

```go
type Base struct{}
func (Base) Name() string    { return "base" }
func (b Base) Hello() string { return "hi " + b.Name() }

type Derived struct{ Base }
func (Derived) Name() string { return "derived" }

Derived{}.Name()   // "derived": the outer method shadows
Derived{}.Hello()  // "hi base": Hello runs on Base and calls Base.Name
```

### Shadowing and conflicts

- A field or method at a shallower depth wins over a promoted one.
- Two embedded types at the same depth with the same name: **ambiguous**. That's a compile error only if you actually use the name.

### Gotchas

```go
// Embedding a nil pointer compiles, then panics when a promoted method touches it
s := Server{}   // s.Logger == nil
s.Log("x")      // nil pointer dereference inside Log

// Embedding sync.Mutex exports Lock/Unlock as part of your API
type Counter struct {
    sync.Mutex   // callers can now c.Lock(): usually not what you want
    n int
}
type Counter struct {
    mu sync.Mutex   // prefer an unexported named field
    n  int
}
```

### Embedding interfaces in structs

This is useful for decorators: override one method and the rest pass through.

```go
type countingWriter struct {
    io.Writer        // everything delegates to this
    n int
}
func (c *countingWriter) Write(p []byte) (int, error) {
    n, err := c.Writer.Write(p)
    c.n += n
    return n, err
}
```

## Misc

- `struct{}` is zero bytes. Use it for sets and signal channels (`chan struct{}`).
- Unexported fields (lowercase) are invisible to other packages and to `encoding/json`.
- Field order affects padding and size. This rarely matters, but `fieldalignment` can report it.

## Be ready to explain

- The difference between embedding and inheritance (the `Hello` example above).
- What "useful zero value" means, and when you need a constructor.
- Why embedding a mutex is a leaky API.
