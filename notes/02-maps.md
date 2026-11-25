# Maps

## Basics and lookup with `ok`

```go
m := map[string]int{"a": 1}
m := make(map[string]int, 100)   // size hint, optional

v := m["missing"]        // 0: the zero value, no error
v, ok := m["missing"]    // 0, false: the "comma ok" idiom
if v, ok := m[k]; ok { ... }

m[k]++                   // fine on a missing key (starts from 0)
delete(m, k)             // no-op if the key is absent
clear(m)                 // Go 1.21+, removes all entries
len(m)
```

Use `ok` whenever the zero value is also a valid stored value (`0`, `""`, `false`).

## nil maps

```go
var m map[string]int     // nil
m["x"]                   // 0: reads are fine
len(m)                   // 0
for k := range m {}      // fine, zero iterations
delete(m, "x")           // fine
m["x"] = 1               // PANIC: assignment to entry in nil map
```

A struct field of map type is nil by default. Initialize it in a constructor or lazily:

```go
func (c *Cache) Set(k string, v int) {
    if c.data == nil { c.data = make(map[string]int) }
    c.data[k] = v
}
```

## Unordered iteration

Iteration order is **deliberately randomized**. Never depend on it, including in tests.

```go
keys := slices.Sorted(maps.Keys(m))   // Go 1.23+ (maps.Keys returns an iterator)
for _, k := range keys { fmt.Println(k, m[k]) }
```

- Deleting entries during `range` is safe.
- Entries added during `range` may or may not be visited.
- `fmt.Println(m)` and `json.Marshal` print keys sorted (that's the printer sorting, not map order).

## Concurrent access

Maps are **not** safe for concurrent use if any goroutine writes.
The runtime detects it and crashes with `fatal error: concurrent map writes` (or `concurrent map read and map write`). That is a fatal error, **not a recoverable panic**.

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Get(k string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[k]
    return v, ok
}

func (s *SafeMap) Set(k string, v int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[k] = v
}
```

`sync.Map` is only better for two cases: keys written once and read many times, or goroutines working on disjoint key sets. Otherwise a mutex plus a map is simpler and usually faster.

Run `go test -race` to catch data races.

## Gotchas

```go
// Map elements are not addressable
m := map[string]Point{}
m["a"].X = 1          // compile error: cannot assign to struct field m["a"].X in map
p := m["a"]; p.X = 1; m["a"] = p   // copy, change, write back
// or use map[string]*Point

// Keys must be comparable: no slices, maps, or funcs as keys.
// Structs and arrays of comparable types work:
type key struct{ x, y int }
grid := map[key]bool{}

// Maps behave like references: passing one to a function lets it mutate yours
func fill(m map[string]int) { m["x"] = 1 }   // the caller sees this

// Copying a map needs an explicit copy
c := maps.Clone(m)    // Go 1.21+, shallow
```

## Set idiom

```go
seen := map[string]struct{}{}   // struct{} takes 0 bytes
seen["a"] = struct{}{}
if _, ok := seen["a"]; ok { ... }
// map[string]bool also works and reads nicer: if seen[k] { ... }
```

## Be ready to explain

- Why reading a nil map is fine and writing one panics.
- Why the iteration order changes between runs, and how to get a deterministic order.
- Why concurrent writes crash the program, and the mutex vs `sync.Map` trade-off.
