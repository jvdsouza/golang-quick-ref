# Generics (Go 1.18+)

## Type parameters on functions

```go
func Map[T, U any](s []T, f func(T) U) []U {
    out := make([]U, 0, len(s))
    for _, v := range s {
        out = append(out, f(v))
    }
    return out
}

lens := Map([]string{"a", "bb"}, func(s string) int { return len(s) })  // T, U inferred
Map[string, int](xs, f)   // explicit instantiation, when inference can't figure it out
```

## Constraints

A constraint is an interface that describes the allowed **type set**.

```go
any                 // anything (alias for interface{})
comparable          // supports == and !=, so it can be a map key
cmp.Ordered         // supports < <= > >= (ints, uints, floats, strings)

type Number interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}
// ~int means "any type whose UNDERLYING type is int", e.g. `type Celsius int`

func Sum[T Number](xs []T) T {
    var total T          // zero value of T
    for _, x := range xs { total += x }
    return total
}

func Keys[K comparable, V any](m map[K]V) []K { ... }
```

Constraints that contain a union (`|`) or `~` can **only** be used as constraints, not as ordinary variable types.

Constraints can also require methods:

```go
type Shape interface { Area() float64 }
func TotalArea[S Shape](shapes []S) float64 { ... }
```

## Generic types

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    v := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return v, true
}

var s Stack[int]
s.Push(1)
```

## Limitations to know

```go
// Methods can't declare their OWN type parameters
func (s *Stack[T]) Map[U any](f func(T) U) Stack[U]   // compile error
// Workaround: write a top-level function: func MapStack[T, U any](s *Stack[T], f func(T) U) ...

// You can't type-switch directly on a T value; convert it to any first
func describe[T any](v T) string {
    switch any(v).(type) {
    case int:    return "int"
    case string: return "string"
    }
    return "other"
}
```

## When abstraction is worthwhile

**Use generics when** the *logic is identical* and only the *type* varies:
- Container types: stacks, queues, sets, caches, trees, ordered maps
- Slice and map utilities: `Map`, `Filter`, `Keys`, `GroupBy` (check `slices` and `maps` first, they cover a lot)
- Type-safe wrappers that would otherwise need `any` plus type assertions (for example, `sync.Pool`-style or `atomic.Pointer[T]`-style APIs)

**Use an interface instead when** the *behavior* varies by type:
- `io.Reader` doesn't need generics. Each implementation does something different.
- If your function body calls a method on the value, a plain interface parameter is usually simpler.

**Don't use generics when:**
- There's only one concrete type in practice. Write the plain function, and generalize when the second caller shows up.
- It makes signatures harder to read than a small amount of duplication would.
- You're trying to imitate inheritance or build class hierarchies.

> Rule of thumb: write the concrete code first. Reach for type parameters when you notice yourself copy-pasting the same function for `int`, `string`, and `float64`.

## Built-ins and stdlib that replace hand-rolled generics

```go
min(a, b, c)  max(a, b)              // built-ins, Go 1.21+
slices.Contains / Index / Sort / Max / Reverse / Compact
maps.Keys / Values / Clone           // Keys/Values return iterators (1.23+)
cmp.Compare(a, b)  cmp.Or(a, b, c)   // cmp.Or returns the first non-zero value
```

## Be ready to explain

- What `~` means and why `type Celsius int` needs it.
- `any` vs `comparable` vs `cmp.Ordered`.
- A concrete case where you'd pick an interface over a type parameter.
