# Slices

## Mental model

A slice is a 3-word header: **pointer** to a backing array, **len**, **cap**.
Passing a slice copies the header, not the array.

```go
s := make([]int, 3, 5) // [0 0 0], len 3, cap 5
len(s)                 // elements you can index: s[0..2]
cap(s)                 // room in the backing array from s[0] onward
s[3]                   // panic: index out of range (len is 3, not cap)
s = s[:5]              // OK: reslicing up to cap is allowed
```

## Backing arrays and aliasing

Slicing does **not** copy. Sub-slices share memory.

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]            // [2 3], len 2, cap 4 (runs to end of a)
b[0] = 99              // a == [1 99 3 4 5]
b = append(b, 100)     // fits in cap, so it writes into a[3]!
                       // a == [1 99 3 100 5]
```

Fix: cap the sub-slice with a **full slice expression** `s[low:high:max]`.

```go
b := a[1:3:3]          // len 2, cap 2
b = append(b, 100)     // no room, so it allocates a new array; a is untouched
```

## append

```go
s = append(s, x)           // ALWAYS reassign the result
s = append(s, other...)    // append a slice
```

- If `len < cap`, append writes in place and returns a header that shares the array.
- If not, it allocates a bigger array (growth roughly 2x when small, about 1.25x when large; this is an implementation detail, so don't rely on it), copies, and returns a header pointing at the **new** array.
- That means after `append` you may or may not still alias the original. Treat it as unknown.

```go
func add(s []int) { s = append(s, 4) } // caller's len doesn't change

func add(s []int) []int { return append(s, 4) } // return it instead
```

Preallocate when you know the size:

```go
out := make([]T, 0, len(in)) // len 0, cap n, then append
```

## Copying

```go
dst := make([]int, len(src))
n := copy(dst, src)        // copies min(len(dst), len(src)); returns count

var dst []int
copy(dst, src)             // copies 0, because dst has len 0 (cap doesn't count)

c := slices.Clone(src)     // Go 1.21+, the idiomatic clone
c := append([]int(nil), src...) // older idiom
```

`copy` handles overlapping ranges correctly, e.g. shifting: `copy(s[i:], s[i+1:])`.

## nil vs empty

```go
var a []int     // nil:      a == nil, len 0
b := []int{}    // empty:    b != nil, len 0
```

Both work with `len`, `range`, and `append`. Differences:
- `json.Marshal`: nil gives `null`, empty gives `[]`
- `reflect.DeepEqual(a, b)` is false

Check emptiness with `len(s) == 0`, never `s == nil`.

## Gotchas

```go
// range gives a COPY of each element
for _, u := range users { u.Active = true }      // no effect
for i := range users { users[i].Active = true }  // works

// A small slice can pin a huge array in memory
head := huge[:10]                  // keeps all of huge alive
head := slices.Clone(huge[:10])    // lets huge be collected

// Slices aren't comparable with == (except to nil)
slices.Equal(a, b)                 // Go 1.21+
```

## Handy `slices` package (Go 1.21+)

```go
slices.Contains(s, v)       slices.Index(s, v)
slices.Sort(s)              slices.SortFunc(s, func(a, b T) int { return cmp.Compare(a.X, b.X) })
slices.Insert(s, i, v...)   slices.Delete(s, i, j)   // Delete zeros the freed tail (1.22+)
slices.Reverse(s)           slices.Max(s) / slices.Min(s)
slices.Compact(s)           slices.BinarySearch(s, v)
```

## Be ready to explain

- Why `append` must be reassigned.
- Why changing `b[0]` changed `a`, and how `a[i:j:j]` prevents append clobbering.
- What happens to the caller's slice when a function appends to it or changes its elements.
