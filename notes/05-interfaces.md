# Interfaces

## Implicit satisfaction

A type satisfies an interface by having the methods. There's no `implements` keyword.

```go
type Stringer interface{ String() string }

type Temp float64
func (t Temp) String() string { return fmt.Sprintf("%.1f°C", float64(t)) }

var s Stringer = Temp(21.5)   // just works
```

A compile-time assertion documents intent and catches breakage early:

```go
var _ io.Writer = (*MyWriter)(nil)   // fails to compile if *MyWriter stops implementing Write
```

## Small interfaces

"The bigger the interface, the weaker the abstraction." Most good Go interfaces have 1 to 3 methods.

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }
type ReadWriter interface { Reader; Writer }   // composition by embedding

type error interface { Error() string }
type Stringer interface { String() string }
```

### Define interfaces where they're used (the consumer side)

```go
// package billing: declare only what billing needs
type invoiceStore interface {
    Save(ctx context.Context, inv Invoice) error
}

func NewService(s invoiceStore) *Service { ... }
```

- **Accept interfaces, return concrete types.** The constructor returns `*PostgresStore`, and callers narrow it to whatever interface they need.
- Don't create an interface before there's a second implementation or a test seam that needs one.

## Type assertions and switches

```go
var x any = "hi"

s := x.(string)          // panics if x isn't a string
s, ok := x.(string)      // safe form

switch v := x.(type) {
case string:  fmt.Println("string", len(v))
case int:     fmt.Println("int", v+1)
case nil:     fmt.Println("nil interface")
default:      fmt.Printf("other %T\n", v)
}

// Checking for an optional capability
if sw, ok := w.(io.StringWriter); ok {
    sw.WriteString(s)
}
```

`any` is an alias for `interface{}`.

## Interface values: (type, value) pairs

An interface value holds **two** things: the dynamic type and the dynamic value.
It is `nil` only when **both** are nil.

### The typed-nil trap

```go
type MyErr struct{}
func (*MyErr) Error() string { return "boom" }

func do() error {
    var e *MyErr = nil
    return e                 // interface = (*MyErr, nil), which is NOT nil
}

err := do()
fmt.Println(err == nil)      // false!
fmt.Printf("%T %v\n", err, err == (*MyErr)(nil)) // *main.MyErr true
```

Fix: return a literal `nil` on the success path. Don't route concrete nil pointers through an interface.

```go
func do() error {
    var e *MyErr
    if somethingFailed { e = &MyErr{} }
    if e != nil { return e }
    return nil               // untyped nil gives a nil interface
}
```

The same trap applies to any interface, such as `io.Reader` or `http.Handler`.

## Other gotchas

```go
// == on interfaces compares (type, value). It PANICS if the dynamic type isn't comparable
var a, b any = []int{1}, []int{1}
a == b   // runtime panic: comparing uncomparable type []int

// Pointer receivers: only *T satisfies the interface (see [pointers](04-pointers.md))
var _ Incer = Counter{}   // compile error if Inc has a pointer receiver

// An interface holding a struct holds a COPY of it
var s fmt.Stringer = myStruct   // later changes to myStruct aren't reflected
```

## Be ready to explain

- Why Go uses implicit interfaces (decoupling: the consumer defines the contract, and the producer doesn't import it).
- Why `err != nil` can be true when the pointer inside is nil.
- Why you'd keep an interface to 1 or 2 methods and define it in the consuming package.
