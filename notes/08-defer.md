# defer

`defer f()` schedules `f` to run when the **surrounding function** returns. That covers a normal return, an early return, and a panic.

## Execution order: LIFO (a stack)

```go
for i := 0; i < 3; i++ {
    defer fmt.Print(i)
}
// prints: 210
```

Resources are released in reverse order of acquisition, which is what you want.

## Arguments are evaluated immediately

The deferred call's **arguments** (and the receiver) are evaluated when the `defer` statement runs. The **call** happens later.

```go
x := 1
defer fmt.Println("arg:", x)                // captures 1 now
defer func() { fmt.Println("closure:", x) }() // reads x at exit
x = 2
// prints:
// closure: 2
// arg: 1
```

```go
start := time.Now()
defer log.Println("took", time.Since(start))   // WRONG: time.Since runs immediately, prints ~0
defer func() { log.Println("took", time.Since(start)) }()  // right
```

## Modifying named results

A deferred closure can read and change **named** return values after the `return` statement sets them.

```go
func f() (n int) {
    defer func() { n *= 2 }()
    return 3        // sets n = 3, then defer runs, so it returns 6
}
```

This is how deferred cleanup reports errors and how `recover` turns a panic into an error.

## Resource cleanup patterns

```go
// Files: defer AFTER checking the error
f, err := os.Open(path)
if err != nil { return err }
defer f.Close()

// HTTP response bodies
resp, err := http.Get(url)
if err != nil { return err }
defer resp.Body.Close()

// Locks
mu.Lock()
defer mu.Unlock()

// Contexts
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

### Don't ignore Close errors on writes

For files you **write**, `Close` can report the real failure (for example, a flush to disk).

```go
func writeFile(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        err = errors.Join(err, f.Close())   // keep both errors
    }()
    _, err = f.Write(data)
    return err
}
```

## Gotchas

```go
// 1. defer in a loop: nothing closes until the FUNCTION returns
for _, p := range paths {
    f, _ := os.Open(p)
    defer f.Close()        // thousands of open files
}
// Fix: move the body into a function so each iteration has its own scope
for _, p := range paths {
    if err := processFile(p); err != nil { return err }
}

// 2. defer doesn't run on os.Exit or log.Fatal
defer cleanup()
log.Fatal("bye")           // cleanup NEVER runs

// 3. Deferring before the error check
f, err := os.Open(p)
defer f.Close()            // f may be nil here; for many types (e.g. resp.Body) this panics
if err != nil { ... }

// 4. Method values evaluate the receiver at defer time
defer s.Close()            // uses the s from THIS moment, even if s is reassigned later
```

## recover

`recover()` only stops a panic when called **directly inside a deferred function**.

```go
defer func() {
    if r := recover(); r != nil {
        log.Printf("recovered: %v\n%s", r, debug.Stack())
    }
}()
```

## Cost

Since Go 1.14, `defer` is close to free in most cases (open-coded defers). Don't avoid it for performance unless a profile says otherwise.

## Be ready to explain

- The output of the loop and `x := 1` examples above.
- Why `defer f.Close()` goes after the `err` check.
- Why `defer` in a long loop leaks, and how to fix it.
