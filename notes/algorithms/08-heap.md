# Heap / priority queue

**Recognise it:** "k largest or smallest", "k closest", "always process the most or least urgent next", or merging k sorted things.
**Core trick:** for the **k largest**, keep a **min-heap of size k**. Its root is the kth largest, and anything smaller gets evicted.

## Heaps in Go: `container/heap`

Go has no ready-made priority queue. You implement `heap.Interface` (5 methods) and call the **package functions** `heap.Push` and `heap.Pop`, not the methods directly.

```go
type IntHeap []int                                  // a min-heap

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }   // use > for a max-heap
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntHeap) Push(x any)        { *h = append(*h, x.(int)) }   // pointer receiver: changes length
func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

h := &IntHeap{5, 2, 8}
heap.Init(h)               // O(n) heapify
heap.Push(h, 3)            // O(log n). NOT h.Push(3), which skips the sift-up
x := heap.Pop(h).(int)     // O(log n); returns `any`, so assert the type
top := (*h)[0]             // peek at the min
heap.Fix(h, i)             // after changing (*h)[i] in place
```

### A reusable generic heap

This is handy when you need several heaps with different orderings.

```go
type Heap[T any] struct {
    items []T
    less  func(a, b T) bool
}

func (h *Heap[T]) Len() int           { return len(h.items) }
func (h *Heap[T]) Less(i, j int) bool { return h.less(h.items[i], h.items[j]) }
func (h *Heap[T]) Swap(i, j int)      { h.items[i], h.items[j] = h.items[j], h.items[i] }
func (h *Heap[T]) Push(x any)         { h.items = append(h.items, x.(T)) }
func (h *Heap[T]) Pop() any {
    n := len(h.items)
    x := h.items[n-1]
    h.items = h.items[:n-1]
    return x
}
```

## Kth Largest Element in an Array

Keep a min-heap of the k largest seen so far. Its root is the answer.

```go
func findKthLargest(nums []int, k int) int {
    h := IntHeap(slices.Clone(nums[:k]))   // clone so we don't reorder the caller's slice
    heap.Init(&h)
    for _, n := range nums[k:] {
        if n > h[0] {
            h[0] = n                       // replace the root, then restore heap order
            heap.Fix(&h, 0)
        }
    }
    return h[0]
}
```
O(n log k) time, O(k) space. Alternatives are sorting (O(n log n)) and **quickselect**, which is O(n) on average and O(n²) in the worst case.

## K Closest Points to Origin

You want the k **smallest** distances, so keep a **max-heap** of size k and evict the farthest. Compare squared distances, because `sqrt` isn't needed for ordering.

```go
func kClosest(points [][]int, k int) [][]int {
    dist := func(p []int) int { return p[0]*p[0] + p[1]*p[1] }
    h := &Heap[[]int]{less: func(a, b []int) bool { return dist(a) > dist(b) }}  // max-heap
    for _, p := range points {
        heap.Push(h, p)
        if h.Len() > k {
            heap.Pop(h)                    // drop the farthest
        }
    }
    return h.items
}
```
O(n log k). The simpler O(n log n) version is `slices.SortFunc(points, ...)` followed by `return points[:k]`.

## Task Scheduler

### Greedy formula (O(n))

The most frequent task sets the frame: `(maxCount - 1)` gaps of size `n + 1`, plus one slot for each task tied for the max count. If there are enough distinct tasks to fill every gap, there's no idle time and the answer is just `len(tasks)`.

```go
func leastInterval(tasks []byte, n int) int {
    var count [26]int
    for _, t := range tasks {
        count[t-'A']++
    }
    maxCount := slices.Max(count[:])       // count[:] turns the array into a slice
    numMax := 0
    for _, c := range count {
        if c == maxCount {
            numMax++
        }
    }
    return max(len(tasks), (maxCount-1)*(n+1)+numMax)
}
```

### Heap simulation (the general approach interviewers expect to see)

A max-heap holds the remaining counts. After a task runs, it sits in a cooldown queue until it's allowed to run again.

```go
func leastIntervalHeap(tasks []byte, n int) int {
    var count [26]int
    for _, t := range tasks {
        count[t-'A']++
    }
    h := &Heap[int]{less: func(a, b int) bool { return a > b }}   // max-heap
    for _, c := range count {
        if c > 0 {
            heap.Push(h, c)
        }
    }

    type cooling struct{ remaining, readyAt int }
    var queue []cooling
    clock := 0
    for h.Len() > 0 || len(queue) > 0 {
        clock++
        if h.Len() > 0 {
            if left := heap.Pop(h).(int) - 1; left > 0 {
                queue = append(queue, cooling{left, clock + n})
            }
        }
        if len(queue) > 0 && queue[0].readyAt == clock {
            heap.Push(h, queue[0].remaining)
            queue = queue[1:]
        }
    }
    return clock
}
```
O(n log 26), which is O(n).

## Takeaways

- For k largest, use a min-heap of size k. For k smallest, use a max-heap of size k. It feels backwards, but the root is what you evict.
- In Go, always call `heap.Push(h, x)` and `heap.Pop(h)`, never `h.Push`.
- `Push` and `Pop` need pointer receivers because they change the slice length (see [pointers](../04-pointers.md)).
