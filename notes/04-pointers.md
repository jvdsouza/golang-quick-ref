# Pointers

## Everything is passed by value

Assignment and function calls **copy** the value. For a struct, that's every field.
"Reference-like" types (slices, maps, channels, funcs, interfaces) are small headers whose copies still point at the same underlying data.

```go
type User struct{ Name string; Tags []string }

u2 := u1          // shallow copy
u2.Name = "x"     // u1.Name unchanged
u2.Tags[0] = "y"  // u1.Tags[0] CHANGED: same backing array
```

## Value vs pointer receivers

```go
type Counter struct{ n int }

func (c Counter) Value() int { return c.n }  // value receiver: works on a copy
func (c *Counter) Inc()      { c.n++ }       // pointer receiver: mutates the original
```

| Use a **pointer receiver** when... | Use a **value receiver** when... |
|---|---|
| the method mutates the receiver | the type is small and immutable-ish (`time.Time`, `Point`) |
| the struct is large (avoid copying) | the type is a map, func, or chan (already reference-like) |
| it contains a `sync.Mutex` or other no-copy field | you want copies to be independent |

**Be consistent**: if any method needs a pointer receiver, give all of them pointer receivers.

## Method sets (why interfaces sometimes "don't match")

- The method set of `T` contains the **value-receiver** methods only.
- The method set of `*T` contains **both** value- and pointer-receiver methods.

```go
type Incer interface{ Inc() }

var _ Incer = &Counter{}  // OK
var _ Incer = Counter{}   // compile error: Counter does not implement Incer
                          // (method Inc has pointer receiver)
```

The reason: a value stored in an interface isn't addressable, so Go can't take its address to call `Inc`.

## Auto-addressing

```go
var c Counter
c.Inc()            // OK: Go rewrites it as (&c).Inc() because c is addressable

m := map[string]Counter{}
m["a"].Inc()       // compile error: map elements aren't addressable

func get() Counter { ... }
get().Inc()        // compile error: return values aren't addressable

p := &Counter{}
p.Value()          // OK: Go dereferences automatically, (*p).Value()
```

## Mutation through pointers

```go
func reset(u User)  { u.Name = "" }   // changes a copy: no effect
func reset(u *User) { u.Name = "" }   // changes the caller's value

// Modifying elements during iteration
for i := range users { users[i].Active = true }   // index
for _, u := range ptrUsers { u.Active = true }    // []*User: works
```

## Copying gotchas

```go
// Copying a struct that holds a mutex copies the lock state: a bug
type Store struct { mu sync.Mutex; m map[string]int }
func (s Store) Get(k string) int { s.mu.Lock(); ... }  // locks a COPY
// go vet reports: "passes lock by value"

// Pointer fields make "copies" share state
type A struct{ cfg *Config }
a2 := a1   // a2.cfg == a1.cfg
```

## Creating pointers

```go
p := &User{Name: "a"}   // most common
p := new(User)          // zeroed *User
p := new(42)            // Go 1.26+: new accepts an expression, giving a *int set to 42

func newUser() *User {
    u := User{}         // returning the address of a local is SAFE:
    return &u           // escape analysis moves it to the heap
}
```

Since Go 1.22, each `for` loop iteration gets a **fresh loop variable**, so `&v` inside the loop no longer aliases one shared variable.

## nil pointers

```go
var p *User
p.Name        // panic: nil pointer dereference

// Methods can be called on nil pointer receivers, and can handle it themselves
func (t *Tree) Sum() int {
    if t == nil { return 0 }
    return t.Val + t.Left.Sum() + t.Right.Sum()
}
```

## Don't

- Don't use `*[]T`, `*map[K]V`, or `*SomeInterface` unless you have a specific reason (for example, a function that must replace the caller's slice header).
- Don't reach for pointers "for performance" on small structs. Copies are cheap, and pointers can cause heap allocation.

## Be ready to explain

- Why `Counter{}` doesn't satisfy `Incer` but `&Counter{}` does.
- What gets shared and what gets copied when you assign a struct.
- When you'd choose a value receiver.
