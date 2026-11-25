# Graphs

**Recognise it:** grids ("islands", "rooms", "oranges"), dependencies ("prerequisites", "build order"), or anything about connections and reachability.

- **DFS**: connectivity, counting components, flood fill, cycle detection.
- **BFS**: shortest path in an **unweighted** graph, "minimum steps or minutes", and spreading from multiple sources at once.
- **Topological sort**: ordering with dependencies, and detecting cycles in a **directed** graph.

## Representations in Go

```go
adj := make([][]int, n)                    // adjacency list for nodes 0..n-1 (the usual choice)
adj[u] = append(adj[u], v)

adj := map[string][]string{}               // when nodes aren't dense ints

visited := make([]bool, n)                 // or map[T]bool, or mark the grid in place
dirs := [4][2]int{{1, 0}, {-1, 0}, {0, 1}, {0, -1}}   // grid neighbours
```

## Number of Islands (DFS flood fill)

Every unvisited `'1'` starts a new island. Sink the whole island so it's never counted again.

```go
func numIslands(grid [][]byte) int {
    rows, cols := len(grid), len(grid[0])

    var sink func(r, c int)
    sink = func(r, c int) {
        if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1' {
            return
        }
        grid[r][c] = '0'                   // mark visited by mutating the grid
        sink(r+1, c)
        sink(r-1, c)
        sink(r, c+1)
        sink(r, c-1)
    }

    count := 0
    for r := range rows {                  // Go 1.22+ range over int
        for c := range cols {
            if grid[r][c] == '1' {
                count++
                sink(r, c)
            }
        }
    }
    return count
}
```
O(rows × cols). It mutates the input. If that isn't allowed, use a `visited [][]bool`.

## Rotting Oranges (multi-source BFS)

Start the BFS from **every** rotten orange at once. Each BFS level is one minute.

```go
func orangesRotting(grid [][]int) int {
    rows, cols := len(grid), len(grid[0])
    var queue [][2]int                     // [2]int is a cheap, comparable (r, c) pair
    fresh := 0
    for r := range rows {
        for c := range cols {
            switch grid[r][c] {
            case 2:
                queue = append(queue, [2]int{r, c})
            case 1:
                fresh++
            }
        }
    }

    dirs := [4][2]int{{1, 0}, {-1, 0}, {0, 1}, {0, -1}}
    minutes := 0
    for len(queue) > 0 && fresh > 0 {
        minutes++
        var next [][2]int
        for _, p := range queue {
            for _, d := range dirs {
                r, c := p[0]+d[0], p[1]+d[1]
                if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != 1 {
                    continue
                }
                grid[r][c] = 2             // mark when ENQUEUED, not when dequeued
                fresh--
                next = append(next, [2]int{r, c})
            }
        }
        queue = next
    }
    if fresh > 0 {
        return -1                          // some orange was unreachable
    }
    return minutes
}
```
O(rows × cols). Marking cells when you enqueue them prevents duplicates in the queue.

## Clone Graph

Map each original node to its clone. **Register the clone before recursing**, so cycles find the existing copy.

```go
type Node struct {
    Val       int
    Neighbors []*Node
}

func cloneGraph(node *Node) *Node {
    clones := map[*Node]*Node{}            // pointers are comparable, so they work as keys

    var clone func(n *Node) *Node
    clone = func(n *Node) *Node {
        if n == nil {
            return nil
        }
        if c, ok := clones[n]; ok {
            return c
        }
        c := &Node{Val: n.Val, Neighbors: make([]*Node, 0, len(n.Neighbors))}
        clones[n] = c                      // BEFORE visiting neighbours
        for _, nb := range n.Neighbors {
            c.Neighbors = append(c.Neighbors, clone(nb))
        }
        return c
    }
    return clone(node)
}
```
O(V + E).

## Course Schedule (topological sort, Kahn's algorithm)

`[course, prereq]` means an edge `prereq → course`. Repeatedly take the courses with no remaining prerequisites (in-degree 0). If you can't take them all, there's a cycle.

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    adj := make([][]int, numCourses)
    indegree := make([]int, numCourses)
    for _, p := range prerequisites {
        course, pre := p[0], p[1]
        adj[pre] = append(adj[pre], course)
        indegree[course]++
    }

    var queue []int
    for c, d := range indegree {
        if d == 0 {
            queue = append(queue, c)
        }
    }

    taken := 0
    for len(queue) > 0 {
        c := queue[0]
        queue = queue[1:]
        taken++                            // for Course Schedule II, append c to an order slice
        for _, next := range adj[c] {
            indegree[next]--
            if indegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }
    return taken == numCourses
}
```
O(V + E).

### Alternative: DFS with three colours

```go
const unvisited, visiting, done = 0, 1, 2
state := make([]int, numCourses)
var hasCycle func(u int) bool
hasCycle = func(u int) bool {
    switch state[u] {
    case visiting:
        return true                        // a back edge means a cycle
    case done:
        return false
    }
    state[u] = visiting
    for _, v := range adj[u] {
        if hasCycle(v) {
            return true
        }
    }
    state[u] = done
    return false
}
```

## Takeaways

- Grid problems are graph problems: neighbours come from the `dirs` array, and bounds checks happen first.
- Use BFS for "minimum steps", and use multi-source BFS when there are several starting points.
- Mark nodes visited when you **enqueue or enter** them, never later.
- For directed dependencies, use topological sort. Kahn's algorithm also gives you the order.
