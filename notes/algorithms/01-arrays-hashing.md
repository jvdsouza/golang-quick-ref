# Arrays & hashing

**Recognise it:** "find a pair or group", "have I seen this before?", "count occurrences", or the need to beat O(n²).
**Core trick:** trade memory for time. A map gives O(1) lookups, so one pass replaces nested loops.

## Two Sum

Store `value -> index` while scanning. For each `n`, check whether `target - n` has been seen.

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int, len(nums))   // value -> index
    for i, n := range nums {
        if j, ok := seen[target-n]; ok {   // comma-ok: index 0 is a valid result
            return []int{j, i}
        }
        seen[n] = i
    }
    return nil
}
```
O(n) time, O(n) space. Check the map **before** inserting, so an element isn't paired with itself.

## Group Anagrams

Anagrams share a letter-count signature. In Go, a `[26]byte` **array** is comparable, so it works directly as a map key. There's no sorting and no string building.

```go
func groupAnagrams(strs []string) [][]string {
    groups := map[[26]byte][]string{}
    for _, s := range strs {
        var key [26]byte                   // counts fit in a byte because LeetCode words are <= 100 chars
        for i := 0; i < len(s); i++ {
            key[s[i]-'a']++
        }
        groups[key] = append(groups[key], s)   // append on a nil slice is fine
    }
    out := make([][]string, 0, len(groups))
    for _, g := range groups {
        out = append(out, g)
    }
    return out
}
```
O(n·k) time for n words of length k. The alternative key is the sorted string: `b := []byte(s); slices.Sort(b); key := string(b)`, which is O(n·k log k).

## Top K Frequent Elements

Count, then **bucket sort by frequency**. Bucket `f` holds the numbers that appear exactly `f` times. Walk the buckets from high to low.

```go
func topKFrequent(nums []int, k int) []int {
    count := map[int]int{}
    for _, n := range nums {
        count[n]++
    }
    buckets := make([][]int, len(nums)+1)   // a frequency can't exceed len(nums)
    for n, c := range count {
        buckets[c] = append(buckets[c], n)
    }
    out := make([]int, 0, k)
    for f := len(buckets) - 1; f > 0 && len(out) < k; f-- {
        out = append(out, buckets[f]...)
    }
    return out[:k]
}
```
O(n) time. The heap alternative is O(n log k) (see [heap](08-heap.md)).

## Product of Array Except Self

`out[i]` = (product of everything left of i) × (product of everything right of i). Do two passes, and reuse the output array for the prefix products so space stays O(1) extra.

```go
func productExceptSelf(nums []int) []int {
    n := len(nums)
    out := make([]int, n)

    prefix := 1
    for i := 0; i < n; i++ {
        out[i] = prefix          // product of nums[0..i-1]
        prefix *= nums[i]
    }
    suffix := 1
    for i := n - 1; i >= 0; i-- {
        out[i] *= suffix         // times the product of nums[i+1..n-1]
        suffix *= nums[i]
    }
    return out
}
```
O(n) time, O(1) extra space. Division is banned, and it would break on zeros anyway.

## Longest Consecutive Sequence

Put everything in a set. Only start counting from numbers that **begin** a run (`n-1` isn't in the set). Each number is then visited at most twice, so the whole thing is O(n).

```go
func longestConsecutive(nums []int) int {
    set := make(map[int]struct{}, len(nums))
    for _, n := range nums {
        set[n] = struct{}{}
    }
    best := 0
    for n := range set {                    // range over the SET, not nums: skips duplicates
        if _, ok := set[n-1]; ok {
            continue                        // not the start of a run
        }
        length := 1
        for {
            if _, ok := set[n+length]; !ok {
                break
            }
            length++
        }
        best = max(best, length)
    }
    return best
}
```
O(n) time. Sorting first is the easier O(n log n) answer.

## Takeaways

- A map from "what I need" to "where I saw it" turns pair searches into one pass.
- In Go, fixed-size arrays (`[26]int`) beat maps for small alphabets, and they work as map keys.
- "Start of a run" checks (`n-1` not present) keep set scans linear.
