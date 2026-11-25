# Linked list

**Recognise it:** pointer rewiring, needing O(1) insertion or removal, or cycle detection.
**Core tricks:** a **dummy head** node avoids special-casing the first element, **fast and slow pointers** find cycles and middles, and you should **save `next` before you overwrite it**.

```go
type ListNode struct {
    Val  int
    Next *ListNode
}
```

## Reverse Linked List

```go
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode                // nil: the new tail points at nothing
    cur := head
    for cur != nil {
        next := cur.Next              // save it before overwriting
        cur.Next = prev
        prev, cur = cur, next
    }
    return prev
}
```
O(n) time, O(1) space.

Go one-liner: `cur.Next, prev, cur = prev, cur, cur.Next`. Go evaluates every right-hand side (and the `cur` in `cur.Next` on the left) **before** assigning, so this is safe. Write it out longhand in an interview if clarity matters more.

Recursive version, which uses O(n) stack:

```go
func reverseListRec(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseListRec(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```

## Merge Two Sorted Lists

The dummy node gives you a `tail` to append to from the very first step.

```go
func mergeTwoLists(a, b *ListNode) *ListNode {
    dummy := &ListNode{}
    tail := dummy
    for a != nil && b != nil {
        if a.Val <= b.Val {
            tail.Next, a = a, a.Next
        } else {
            tail.Next, b = b, b.Next
        }
        tail = tail.Next
    }
    if a != nil {
        tail.Next = a                 // attach the remainder; no loop needed
    } else {
        tail.Next = b
    }
    return dummy.Next
}
```
O(n + m) time, O(1) space, and it reuses the existing nodes.

## Linked List Cycle (Floyd's tortoise and hare)

```go
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow, fast = slow.Next, fast.Next.Next
        if slow == fast {             // pointer equality: the same node
            return true
        }
    }
    return false
}
```
O(n) time, O(1) space. The easy alternative is a `map[*ListNode]bool` of visited nodes, with O(n) space. Pointers are comparable, so they work as map keys.

To find where the cycle **starts**: after they meet, reset `slow` to `head` and move both pointers one step at a time. They meet again at the cycle's entry.

## LRU Cache

Combine a **map** (key → node) for O(1) lookup with a **doubly linked list** for O(1) reordering. The most recent entry sits at the front, and the eviction candidate at the back. Sentinel `head` and `tail` nodes remove all nil checks.

```go
type node struct {
    key, val   int
    prev, next *node
}

type LRUCache struct {
    cap        int
    items      map[int]*node
    head, tail *node      // sentinels: head.next = most recent, tail.prev = least recent
}

func Constructor(capacity int) LRUCache {
    head, tail := &node{}, &node{}
    head.next, tail.prev = tail, head
    return LRUCache{cap: capacity, items: make(map[int]*node, capacity), head: head, tail: tail}
}

func (c *LRUCache) unlink(n *node) {
    n.prev.next, n.next.prev = n.next, n.prev
}

func (c *LRUCache) pushFront(n *node) {
    n.prev, n.next = c.head, c.head.next
    c.head.next.prev = n
    c.head.next = n
}

func (c *LRUCache) Get(key int) int {
    n, ok := c.items[key]
    if !ok {
        return -1
    }
    c.unlink(n)
    c.pushFront(n)                    // touched, so now it's the most recent
    return n.val
}

func (c *LRUCache) Put(key, value int) {
    if n, ok := c.items[key]; ok {
        n.val = value
        c.unlink(n)
        c.pushFront(n)
        return
    }
    if len(c.items) == c.cap {
        lru := c.tail.prev
        c.unlink(lru)
        delete(c.items, lru.key)      // that's why the node stores its key
    }
    n := &node{key: key, val: value}
    c.items[key] = n
    c.pushFront(n)
}
```
O(1) for both operations. In real code, add a `sync.Mutex`, because even `Get` mutates the list.

### With `container/list` from the standard library

```go
type entry struct{ key, val int }

type LRU struct {
    cap   int
    ll    *list.List                  // front = most recent
    items map[int]*list.Element
}

func NewLRU(capacity int) *LRU {
    return &LRU{cap: capacity, ll: list.New(), items: map[int]*list.Element{}}
}

func (c *LRU) Get(key int) int {
    e, ok := c.items[key]
    if !ok {
        return -1
    }
    c.ll.MoveToFront(e)
    return e.Value.(*entry).val       // Value is `any`, so it needs a type assertion
}

func (c *LRU) Put(key, val int) {
    if e, ok := c.items[key]; ok {
        e.Value.(*entry).val = val
        c.ll.MoveToFront(e)
        return
    }
    if c.ll.Len() == c.cap {
        back := c.ll.Back()
        c.ll.Remove(back)
        delete(c.items, back.Value.(*entry).key)
    }
    c.items[key] = c.ll.PushFront(&entry{key, val})
}
```

Interviewers usually want the hand-rolled version, but mentioning `container/list` shows you know the standard library.

## Takeaways

- Dummy or sentinel nodes remove edge cases at the head and tail.
- Fast and slow pointers solve cycle detection, cycle start, middle of the list, and "kth from the end" (with a gap of k).
- LRU = hash map + doubly linked list. The node stores its key so eviction can delete it from the map.
