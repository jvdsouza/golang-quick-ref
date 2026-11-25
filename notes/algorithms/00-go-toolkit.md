# Go toolkit for algorithm problems

These are the building blocks the other sheets reuse. Go has no built-in stack, queue, set, or easy heap, so you build them from slices and maps.

## Stack (a slice)

```go
stack := []int{}
stack = append(stack, x)            // push
top := stack[len(stack)-1]          // peek
stack = stack[:len(stack)-1]        // pop
len(stack) == 0                     // empty?
```

## Queue (a slice)

```go
queue := []int{start}
for len(queue) > 0 {
    cur := queue[0]
    queue = queue[1:]               // the backing array isn't freed until a realloc: fine for interviews
    queue = append(queue, next)
}

// Level by level (BFS that tracks depth): swap in a fresh slice per level
for len(queue) > 0 {
    var next []int
    for _, cur := range queue { /* ... */ next = append(next, nb) }
    queue = next
}
```

## Set (a map)

```go
set := map[int]struct{}{}           // or map[int]bool, which reads nicer
set[x] = struct{}{}
if _, ok := set[x]; ok { ... }
delete(set, x)
```

## Fixed-size counters: use arrays, not maps

```go
var count [26]int                    // lowercase letters
for i := 0; i < len(s); i++ { count[s[i]-'a']++ }

// Arrays are COMPARABLE, so they work as map keys (Group Anagrams)
groups := map[[26]int][]string{}
```

## Strings, bytes and runes

```go
s[i]                      // a byte (uint8), not a character
for i, r := range s {}    // a rune (Unicode code point); i is the BYTE offset
len(s)                    // bytes, not characters
b := []byte(s)            // mutable copy; string(b) converts back
r := []rune(s)            // use for non-ASCII problems

var sb strings.Builder    // efficient concatenation
sb.WriteByte('a'); sb.WriteString("bc"); sb.String()

strconv.Itoa(42)          // int to string. NOT string(42), which gives "*"
strconv.Atoi("42")        // string to (int, error)
unicode.IsLetter(r) / unicode.IsDigit(r) / unicode.ToLower(r)
```

Most LeetCode problems are ASCII-only, so byte indexing (`s[i]`) is fine and fast.

## Sorting

```go
slices.Sort(nums)                                    // ascending, in place
slices.SortFunc(iv, func(a, b []int) int {           // custom: return <0, 0, >0
    return cmp.Compare(a[0], b[0])
})
slices.SortStableFunc(...)                           // keeps equal elements in order
sort.Slice(x, func(i, j int) bool { return x[i] < x[j] })   // older API, still common

slices.Reverse(nums)
```

## Binary search helpers

```go
i, found := slices.BinarySearch(sorted, target)  // i = first index with sorted[i] >= target
i := sort.Search(n, func(i int) bool { return pred(i) })  // first i in [0,n) where pred is true
```

## Min, max and infinity

```go
min(a, b, c)  max(a, b)        // built-ins, Go 1.21+
slices.Min(s) slices.Max(s)    // panics on an empty slice
math.MaxInt  math.MinInt       // "infinity" for ints (int is 64-bit on 64-bit platforms)
math.Inf(1)                    // float +inf
abs := func(x int) int { if x < 0 { return -x }; return x }   // no built-in integer abs
```

## 2D grids

```go
grid := make([][]int, rows)
for r := range grid { grid[r] = make([]int, cols) }

dirs := [4][2]int{{1, 0}, {-1, 0}, {0, 1}, {0, -1}}
for _, d := range dirs {
    nr, nc := r+d[0], c+d[1]
    if nr < 0 || nr >= rows || nc < 0 || nc >= cols { continue }
}

for r := range rows { }        // Go 1.22+: range over an int
```

## Recursive closures

A closure can't refer to itself until it's declared, so declare it first:

```go
var dfs func(i int) int
dfs = func(i int) int {
    if i == 0 { return 0 }
    return dfs(i - 1) + 1
}
```

This lets the helper capture `grid`, `memo`, `out` and so on without threading them through as parameters.

## Heap

See [heap](08-heap.md). It uses `container/heap`, which means implementing 5 methods.

## Pitfalls that bite in interviews

```go
// Appending a shared slice into results: later changes overwrite it
out = append(out, path)                  // BUG in backtracking
out = append(out, slices.Clone(path))    // correct

// Integer division truncates toward zero (-7/2 == -3), unlike Python's floor division
// % keeps the sign of the dividend (-7 % 3 == -1)

// Overflow: int is 64-bit, so it's rarely an issue, but sentinels like math.MaxInt + 1 overflow
```
