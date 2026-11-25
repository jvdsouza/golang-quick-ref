# Binary search

**Recognise it:** sorted input, O(log n) required, or "find the minimum or maximum value such that a condition holds" (binary search on the **answer**).
**Core trick:** keep an invariant about `[lo, hi]` and halve the range every step.

## Two templates

```go
// A) Find an exact match. The range is [lo, hi] inclusive.
lo, hi := 0, len(a)-1
for lo <= hi {
    mid := lo + (hi-lo)/2          // avoids overflow (habit from other languages; int is 64-bit in Go)
    switch {
    case a[mid] == target: return mid
    case a[mid] < target:  lo = mid + 1
    default:               hi = mid - 1
    }
}
return -1

// B) First value where ok(x) is true. ok must be false...false,true...true.
lo, hi := minAnswer, maxAnswer
for lo < hi {
    mid := lo + (hi-lo)/2
    if ok(mid) { hi = mid } else { lo = mid + 1 }
}
return lo
```

The standard library does template B for you:

```go
i, found := slices.BinarySearch(sorted, target)   // i = first index with sorted[i] >= target
i := sort.Search(n, func(i int) bool { return ok(i) })  // first i in [0,n) with ok(i), or n
```

## Binary Search

```go
func search(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        switch {
        case nums[mid] == target:
            return mid
        case nums[mid] < target:
            lo = mid + 1
        default:
            hi = mid - 1
        }
    }
    return -1
}

// Or, with the standard library:
func searchStd(nums []int, target int) int {
    if i, ok := slices.BinarySearch(nums, target); ok {
        return i
    }
    return -1
}
```
O(log n).

## Koko Eating Bananas (binary search on the answer)

The speed `k` ranges from 1 to `max(piles)`. Hours needed decreases as `k` grows, so "can finish in h hours" flips from false to true exactly once. Find the first `true`.

```go
func minEatingSpeed(piles []int, h int) int {
    lo, hi := 1, slices.Max(piles)
    for lo < hi {
        k := lo + (hi-lo)/2
        if hoursAt(piles, k) <= h {
            hi = k                    // k works; maybe something smaller does too
        } else {
            lo = k + 1                // too slow
        }
    }
    return lo
}

func hoursAt(piles []int, k int) int {
    total := 0
    for _, p := range piles {
        total += (p + k - 1) / k      // ceil(p / k) using integer math
    }
    return total
}

// The same thing with sort.Search (it searches i in [0, n), so shift by 1)
func minEatingSpeedStd(piles []int, h int) int {
    maxPile := slices.Max(piles)
    return 1 + sort.Search(maxPile, func(i int) bool { return hoursAt(piles, i+1) <= h })
}
```
O(n log m), where m = max pile.

## Search in Rotated Sorted Array

At any `mid`, **one half is sorted**. Check whether the target falls inside the sorted half's range. If it does, search there. Otherwise, search the other half.

```go
func searchRotated(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        if nums[mid] == target {
            return mid
        }
        if nums[lo] <= nums[mid] {                       // left half [lo..mid] is sorted
            if nums[lo] <= target && target < nums[mid] {
                hi = mid - 1
            } else {
                lo = mid + 1
            }
        } else {                                         // right half [mid..hi] is sorted
            if nums[mid] < target && target <= nums[hi] {
                lo = mid + 1
            } else {
                hi = mid - 1
            }
        }
    }
    return -1
}
```
O(log n). Use `<=` in `nums[lo] <= nums[mid]` to handle `lo == mid` when two elements remain.

## Takeaways

- Decide up front whether you're finding a value (template A) or a boundary (template B). Mixing `lo <= hi` with `hi = mid` loops forever.
- "Minimum X such that it's feasible" means binary searching the answer with a feasibility function.
- Integer ceiling division: `(a + b - 1) / b`.
