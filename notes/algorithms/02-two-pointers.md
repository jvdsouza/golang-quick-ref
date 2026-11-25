# Two pointers

**Recognise it:** a sorted array (or one you can sort), palindromes, "pair or triplet that sums to X", or "maximize something between two ends".
**Core trick:** two indices move toward each other (or in the same direction). Each step rules out a whole set of candidates, so O(n²) becomes O(n).

```go
l, r := 0, len(a)-1
for l < r {
    if condition(a[l], a[r]) { /* record */ }
    if needBigger { l++ } else { r-- }
}
```

## Valid Palindrome

Skip non-alphanumeric characters from both ends, and compare case-insensitively. Working on bytes is fine because the input is ASCII.

```go
func isPalindrome(s string) bool {
    l, r := 0, len(s)-1
    for l < r {
        for l < r && !isAlnum(s[l]) { l++ }
        for l < r && !isAlnum(s[r]) { r-- }
        if toLower(s[l]) != toLower(s[r]) {
            return false
        }
        l++
        r--
    }
    return true
}

func isAlnum(c byte) bool {
    return 'a' <= c && c <= 'z' || 'A' <= c && c <= 'Z' || '0' <= c && c <= '9'
}

func toLower(c byte) byte {
    if 'A' <= c && c <= 'Z' {
        return c + ('a' - 'A')
    }
    return c
}
```
O(n) time, O(1) space. With Unicode input, convert to `[]rune` and use `unicode.IsLetter`, `unicode.IsDigit` and `unicode.ToLower`.

## 3Sum

Sort the array. Fix `nums[i]`, then run a two-pointer Two Sum on the rest for `-nums[i]`. Skip duplicates at **both** levels.

```go
func threeSum(nums []int) [][]int {
    slices.Sort(nums)                    // NOTE: sorts the caller's slice in place
    var out [][]int
    for i := 0; i < len(nums)-2; i++ {
        if nums[i] > 0 {
            break                        // sorted, so no three positives can sum to 0
        }
        if i > 0 && nums[i] == nums[i-1] {
            continue                     // skip a duplicate anchor
        }
        l, r := i+1, len(nums)-1
        for l < r {
            switch sum := nums[i] + nums[l] + nums[r]; {
            case sum < 0:
                l++
            case sum > 0:
                r--
            default:
                out = append(out, []int{nums[i], nums[l], nums[r]})
                l++
                r--
                for l < r && nums[l] == nums[l-1] {
                    l++                  // skip a duplicate second element
                }
            }
        }
    }
    return out
}
```
O(n²) time, and O(1) extra space apart from the sort. `switch x := expr; { case ... }` is a tagless switch with an init statement.

## Container With Most Water

Start with the widest container. The area is limited by the **shorter** wall, so move that pointer inward. Moving the taller one can never help, because the width shrinks and the height is still capped by the short wall.

```go
func maxArea(height []int) int {
    l, r, best := 0, len(height)-1, 0
    for l < r {
        best = max(best, (r-l)*min(height[l], height[r]))
        if height[l] < height[r] {
            l++
        } else {
            r--
        }
    }
    return best
}
```
O(n) time, O(1) space.

## Takeaways

- Sorting first (O(n log n)) often unlocks a two-pointer pass.
- To skip duplicates, compare with the **previous** element after moving the pointer.
- Know *why* moving a pointer is safe. Interviewers ask for the argument, not just the code.
