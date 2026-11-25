# Sliding window

**Recognise it:** "longest or shortest **contiguous** subarray or substring such that...", "best over a range", or "at most k changes or distinct values".
**Core trick:** grow the window with `right`. When it becomes invalid, shrink it from `left`. Each index enters and leaves once, so the whole pass is O(n).

```go
left := 0
for right := 0; right < len(s); right++ {
    add(s[right])
    for windowInvalid() {
        remove(s[left])
        left++
    }
    best = max(best, right-left+1)
}
```

## Best Time to Buy and Sell Stock

The simplest window: track the lowest price so far, which is the left edge, and the best profit if you sell today.

```go
func maxProfit(prices []int) int {
    minPrice, best := math.MaxInt, 0
    for _, p := range prices {
        minPrice = min(minPrice, p)
        best = max(best, p-minPrice)
    }
    return best
}
```
O(n) time, O(1) space.

## Longest Substring Without Repeating Characters

Remember where each byte was last seen. When `s[right]` repeats inside the window, **jump** `left` past its previous position.

```go
func lengthOfLongestSubstring(s string) int {
    var last [256]int            // last[c] = (index of c's last occurrence) + 1; 0 means unseen
    best, left := 0, 0
    for right := 0; right < len(s); right++ {
        c := s[right]
        if last[c] > left {
            left = last[c]       // only move forward: an old occurrence may sit left of the window
        }
        last[c] = right + 1
        best = max(best, right-left+1)
    }
    return best
}
```
O(n) time, O(1) space (256 slots). Storing `index+1` avoids having to fill the array with `-1` first. For Unicode input, use `map[rune]int` and range over the string.

## Longest Repeating Character Replacement

A window is valid if `windowLen - countOfMostFrequentChar <= k`, because you can replace the rest.

```go
func characterReplacement(s string, k int) int {
    var count [26]int
    left, maxFreq, best := 0, 0, 0
    for right := 0; right < len(s); right++ {
        idx := s[right] - 'A'
        count[idx]++
        maxFreq = max(maxFreq, count[idx])

        if (right-left+1)-maxFreq > k {   // invalid: slide left by one
            count[s[left]-'A']--
            left++
        }
        best = max(best, right-left+1)
    }
    return best
}
```
O(n) time. The subtle part: `maxFreq` is **never decreased** when shrinking. A stale, too-high `maxFreq` can only keep the window at its current size, never grow it, and the answer only improves when a genuinely higher frequency appears. That's also why a single `if` is enough here instead of a `for`.

## Takeaways

- Contiguous plus an optimum means you should think about a window first.
- For small alphabets, use a fixed `[N]int` for counts or last-seen positions instead of a map.
- Be able to state the window's **invariant** ("no duplicates", "replacements needed <= k").
