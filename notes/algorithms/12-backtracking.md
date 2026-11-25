# Backtracking

**Recognise it:** "all subsets, permutations, or combinations", "every valid arrangement", or constraint search (N-Queens, Sudoku, word search).
**Core trick:** build a candidate incrementally. **Choose** an option, **explore** recursively, **un-choose** it, and record the candidate when it's complete. Prune early when a branch can't succeed.

```go
var out [][]int
var path []int
var dfs func(start int)
dfs = func(start int) {
    if done() {
        out = append(out, slices.Clone(path))   // COPY: see the pitfall below
        return
    }
    for i := start; i < len(choices); i++ {
        path = append(path, choices[i])          // choose
        dfs(i + 1)                               // explore
        path = path[:len(path)-1]                // un-choose
    }
}
```

## The Go-specific pitfall: slice aliasing

```go
out = append(out, path)        // BUG
```

`path` is a slice header that points at a shared backing array. Later appends and truncations **overwrite** what you stored, so every result ends up looking like the last path (or garbage). Always store `slices.Clone(path)`, or `append([]int(nil), path...)` on Go versions before 1.21. See [slices](../01-slices.md).

## Subsets

Every node in the recursion tree is a valid subset, so record it on entry.

```go
func subsets(nums []int) [][]int {
    var out [][]int
    var path []int
    var dfs func(start int)
    dfs = func(start int) {
        out = append(out, slices.Clone(path))
        for i := start; i < len(nums); i++ {
            path = append(path, nums[i])
            dfs(i + 1)                         // i+1: each element used at most once, order ignored
            path = path[:len(path)-1]
        }
    }
    dfs(0)
    return out
}
```
O(n · 2ⁿ).

Bitmask alternative: each subset corresponds to a number from 0 to 2ⁿ-1.

```go
for mask := range 1 << len(nums) {
    var s []int
    for i, n := range nums {
        if mask&(1<<i) != 0 {
            s = append(s, n)
        }
    }
    out = append(out, s)
}
```

For **Subsets II** (duplicates in the input), sort first and add `if i > start && nums[i] == nums[i-1] { continue }`.

## Permutations

Order matters, so every position can use any **unused** element. Track them with `used []bool`.

```go
func permute(nums []int) [][]int {
    var out [][]int
    path := make([]int, 0, len(nums))
    used := make([]bool, len(nums))

    var dfs func()
    dfs = func() {
        if len(path) == len(nums) {
            out = append(out, slices.Clone(path))
            return
        }
        for i, n := range nums {
            if used[i] {
                continue
            }
            used[i] = true
            path = append(path, n)
            dfs()
            path = path[:len(path)-1]
            used[i] = false
        }
    }
    dfs()
    return out
}
```
O(n · n!).

## Combination Sum

Elements can be reused, so recurse with `i` rather than `i+1`. Sorting lets you `break` as soon as a candidate exceeds what's left.

```go
func combinationSum(candidates []int, target int) [][]int {
    slices.Sort(candidates)
    var out [][]int
    var path []int

    var dfs func(start, remaining int)
    dfs = func(start, remaining int) {
        if remaining == 0 {
            out = append(out, slices.Clone(path))
            return
        }
        for i := start; i < len(candidates); i++ {
            c := candidates[i]
            if c > remaining {
                break                          // sorted, so nothing later fits either (pruning)
            }
            path = append(path, c)
            dfs(i, remaining-c)                // i, not i+1: c can be reused
            path = path[:len(path)-1]
        }
    }
    dfs(0, target)
    return out
}
```
Exponential in the worst case. Starting each loop at `start` prevents duplicate orderings such as `[2,3]` vs `[3,2]`.

## Cheat table

| Problem | Loop start | Recurse with | Record when | Duplicate handling |
|---|---|---|---|---|
| Subsets | `start` | `i+1` | every call | sort and skip equal siblings (II) |
| Combinations / Combination Sum | `start` | `i` (reuse) or `i+1` (no reuse) | target hit | sort and skip equal siblings |
| Permutations | `0` | (all) + `used[]` | `len(path) == n` | sort, and skip if `nums[i]==nums[i-1] && !used[i-1]` |

## Takeaways

- Every backtracking solution is choose, explore, un-choose.
- **Clone the path** when recording it. This is the most common Go bug in these problems.
- Sort the input to make pruning (`break`) and duplicate skipping possible.
- Closures over `out`, `path`, and `used` keep the signatures short.
