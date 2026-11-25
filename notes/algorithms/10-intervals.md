# Intervals

**Recognise it:** `[start, end]` pairs, meetings or bookings, overlap, or "minimum rooms or removals".
**Core trick:** **sort** first, usually by start. Sort by **end** for greedy "keep as many as possible" problems. Then a single pass compares each interval with the last kept one.

Two intervals `a` and `b` overlap when `a[0] <= b[1] && b[0] <= a[1]`. Check the problem to see whether touching endpoints count as overlapping.

```go
slices.SortFunc(intervals, func(a, b []int) int { return cmp.Compare(a[0], b[0]) })
```

## Merge Intervals

```go
func merge(intervals [][]int) [][]int {
    if len(intervals) == 0 {
        return nil
    }
    slices.SortFunc(intervals, func(a, b []int) int { return cmp.Compare(a[0], b[0]) })

    out := [][]int{intervals[0]}
    for _, iv := range intervals[1:] {
        last := out[len(out)-1]            // a slice header that ALIASES out's element
        if iv[0] <= last[1] {
            last[1] = max(last[1], iv[1])  // so this updates out in place
        } else {
            out = append(out, iv)
        }
    }
    return out
}
```
O(n log n).

Aliasing note (see [slices](../01-slices.md)): `last` shares memory with `intervals[i]`, so merging also **mutates the input** intervals. LeetCode doesn't care. If the caller does, append `[]int{iv[0], iv[1]}` copies instead.

## Insert Interval

The input is already sorted and non-overlapping. That gives three phases: intervals entirely before, intervals that overlap (absorb them), and intervals entirely after.

```go
func insert(intervals [][]int, newInterval []int) [][]int {
    out := make([][]int, 0, len(intervals)+1)
    i, n := 0, len(intervals)

    for i < n && intervals[i][1] < newInterval[0] {   // ends before the new one starts
        out = append(out, intervals[i])
        i++
    }

    lo, hi := newInterval[0], newInterval[1]
    for i < n && intervals[i][0] <= hi {              // overlaps: widen [lo, hi]
        lo = min(lo, intervals[i][0])
        hi = max(hi, intervals[i][1])
        i++
    }
    out = append(out, []int{lo, hi})

    return append(out, intervals[i:]...)              // the rest come after
}
```
O(n) with no sort needed.

## Non-overlapping Intervals

To keep the maximum number of intervals, always keep the one that **ends earliest**, because it leaves the most room for the others. That means sorting by end. The minimum removals is the total minus the number kept.

```go
func eraseOverlapIntervals(intervals [][]int) int {
    slices.SortFunc(intervals, func(a, b []int) int { return cmp.Compare(a[1], b[1]) })
    removed, end := 0, math.MinInt
    for _, iv := range intervals {
        if iv[0] >= end {                 // no overlap (touching is fine here)
            end = iv[1]                   // keep it
        } else {
            removed++                     // overlaps the kept one, which ends sooner, so drop this
        }
    }
    return removed
}
```
O(n log n). This is the classic activity-selection greedy.

## Related: Meeting Rooms II (minimum rooms)

Sort the starts and ends separately, then sweep with two pointers. Alternatively, use a min-heap of end times.

```go
func minMeetingRooms(intervals [][]int) int {
    starts := make([]int, len(intervals))
    ends := make([]int, len(intervals))
    for i, iv := range intervals {
        starts[i], ends[i] = iv[0], iv[1]
    }
    slices.Sort(starts)
    slices.Sort(ends)
    rooms, best, e := 0, 0, 0
    for _, s := range starts {
        for e < len(ends) && ends[e] <= s {   // free rooms whose meeting ended
            rooms--
            e++
        }
        rooms++
        best = max(best, rooms)
    }
    return best
}
```

## Takeaways

- Sort by start to merge. Sort by end for greedy selection.
- Use `slices.SortFunc` with `cmp.Compare` for `[][]int`.
- Be explicit about whether touching intervals (`[1,2]` and `[2,3]`) overlap.
- Watch slice aliasing when you change the "last" interval in place.
