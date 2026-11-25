# Trees

**Recognise it:** hierarchical data or a "binary tree" in the problem statement. Most answers are DFS (recursion), and "level by level" problems are BFS (a queue).
**Core trick:** define what the recursive function **returns** for a subtree, then combine the left and right results.

```go
type TreeNode struct {
    Val         int
    Left, Right *TreeNode
}

// The DFS skeleton
func dfs(n *TreeNode) int {
    if n == nil {
        return 0                      // base case: an empty subtree
    }
    l, r := dfs(n.Left), dfs(n.Right)
    return combine(n.Val, l, r)
}
```

Calling `dfs(n.Left)` on a nil child is fine. The nil check sits at the top of the function, so every call is safe.

## Maximum Depth of Binary Tree

```go
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    return 1 + max(maxDepth(root.Left), maxDepth(root.Right))
}
```
O(n) time, O(h) stack, where h is the height (O(n) worst case for a skewed tree).

## Invert Binary Tree

```go
func invertTree(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    root.Left, root.Right = invertTree(root.Right), invertTree(root.Left)  // parallel assignment
    return root
}
```
O(n). Go's tuple assignment swaps without a temporary variable.

## Binary Tree Level Order Traversal (BFS)

Process one level at a time: everything currently in the queue *is* the level.

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var out [][]int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        level := make([]int, 0, len(queue))
        var next []*TreeNode
        for _, n := range queue {
            level = append(level, n.Val)
            if n.Left != nil {
                next = append(next, n.Left)
            }
            if n.Right != nil {
                next = append(next, n.Right)
            }
        }
        out = append(out, level)
        queue = next
    }
    return out
}
```
O(n). The same shape solves right side view (take the last item of each level), zigzag order, and minimum depth.

## Validate Binary Search Tree

Checking each node only against its direct children is **wrong**. Every node must fall inside an `(lo, hi)` range inherited from its ancestors.

```go
func isValidBST(root *TreeNode) bool {
    var valid func(n *TreeNode, lo, hi int) bool
    valid = func(n *TreeNode, lo, hi int) bool {
        if n == nil {
            return true
        }
        if n.Val <= lo || n.Val >= hi {
            return false
        }
        return valid(n.Left, lo, n.Val) && valid(n.Right, n.Val, hi)
    }
    return valid(root, math.MinInt, math.MaxInt)   // int is 64-bit, so int32 node values are safely inside
}
```
O(n). The alternative is an in-order traversal, which must produce **strictly increasing** values: track `prev` and compare.

## Lowest Common Ancestor of a BST

Use the BST ordering. If both values are smaller than the node, go left. If both are larger, go right. Otherwise the node splits them, so it's the LCA.

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    n := root
    for n != nil {
        switch {
        case p.Val < n.Val && q.Val < n.Val:
            n = n.Left
        case p.Val > n.Val && q.Val > n.Val:
            n = n.Right
        default:
            return n                  // split point, or n is p or q itself
        }
    }
    return nil
}
```
O(h) time, O(1) space, iterative. For a **general** binary tree, use the recursive version: return the node if it's p or q, recurse left and right, and if both sides return non-nil the current node is the LCA.

## Iterative DFS (when recursion depth is a concern)

```go
stack := []*TreeNode{root}
for len(stack) > 0 {
    n := stack[len(stack)-1]
    stack = stack[:len(stack)-1]
    if n == nil {
        continue
    }
    visit(n)
    stack = append(stack, n.Right, n.Left)   // push right first so left is processed first
}
```

Goroutine stacks grow dynamically, so deep recursion is rarely a problem in Go. Still, mention the iterative option if asked.

## Takeaways

- Most tree problems ask: "what do I return for a nil subtree, and how do I combine left and right?"
- Use BFS with per-level slices for anything "by level".
- Validating a BST needs ancestor bounds, not parent-only checks.
- Recursive closures (`var f func(...); f = func(...) {...}`) let helpers capture state.
