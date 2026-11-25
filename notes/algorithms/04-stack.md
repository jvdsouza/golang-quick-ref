# Stack

**Recognise it:** matching or nesting (brackets, tags), "next greater or smaller element", undo, or evaluating expressions.
**In Go:** a slice. Push with `append`, pop with `s = s[:len(s)-1]`.

## Valid Parentheses

Push opening brackets. On a closing bracket, the top of the stack must be its matching opener.

```go
func isValid(s string) bool {
    pairs := map[byte]byte{')': '(', ']': '[', '}': '{'}
    stack := make([]byte, 0, len(s))
    for i := 0; i < len(s); i++ {
        c := s[i]
        if open, isClose := pairs[c]; isClose {
            if len(stack) == 0 || stack[len(stack)-1] != open {
                return false
            }
            stack = stack[:len(stack)-1]
        } else {
            stack = append(stack, c)
        }
    }
    return len(stack) == 0      // leftover openers mean the string is invalid
}
```
O(n) time and space. Odd-length strings can return `false` immediately.

## Daily Temperatures (monotonic stack)

Keep a stack of **indices** whose temperatures are decreasing. A warmer day pops every colder day it resolves.

```go
func dailyTemperatures(temps []int) []int {
    ans := make([]int, len(temps))   // zero value 0 = "no warmer day", already the default
    var stack []int                  // indices; temps[stack] is decreasing
    for i, t := range temps {
        for len(stack) > 0 && temps[stack[len(stack)-1]] < t {
            j := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            ans[j] = i - j
        }
        stack = append(stack, i)
    }
    return ans
}
```
O(n) time, because each index is pushed and popped at most once. The same template solves next greater element, largest rectangle in histogram, and stock span.

## Evaluate Reverse Polish Notation

Push numbers. An operator pops two operands. The **second** pop is the left operand.

```go
func evalRPN(tokens []string) int {
    stack := make([]int, 0, len(tokens))
    for _, tok := range tokens {
        switch tok {
        case "+", "-", "*", "/":
            b, a := stack[len(stack)-1], stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            var v int
            switch tok {
            case "+": v = a + b
            case "-": v = a - b
            case "*": v = a * b
            case "/": v = a / b      // Go truncates toward zero, which is exactly what the problem wants
            }
            stack = append(stack, v)
        default:
            n, _ := strconv.Atoi(tok) // input is guaranteed valid; check the error in real code
            stack = append(stack, n)
        }
    }
    return stack[0]
}
```
O(n) time. Go's `/` truncation matches the spec. In Python you'd need `int(a / b)`, but in Go it just works.

## Bonus: a generic stack

See [generics](../07-generics.md).

```go
type Stack[T any] struct{ items []T }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    v := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return v, true
}
```

## Takeaways

- For "next greater or warmer" problems, use a monotonic stack of **indices**.
- Always guard `len(stack) == 0` before peeking, or you'll get an index-out-of-range panic.
- Pop order matters for non-commutative operators.
