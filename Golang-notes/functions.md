### anonymous function

```go
func(a int, b int) int {
    return a + b
}
```

how to call it?

```go
add := func(a int, b int) int {
    return a + b
}

result := add(10, 20)

fmt.Println(result) // 30
```

### immediately invoked function:

```go
result := func(a int, b int) int {
    return a + b
}(10, 20)

fmt.Println(result) // 30
```


### closure function
### What does “capture a variable” mean?

When a function **captures a variable**, it means the function keeps access to a variable from its **outer scope**, even though that variable is not defined inside the function.

**Example 1 — Capturing a variable:**

```go
func makeMultiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}

double := makeMultiplier(2)

double(5) // 10
```

Here, `factor` is **captured** by the inner function.

* `factor` → captured from the outer scope
* `n` → normal function parameter

**Example 2 — Capturing state:**

```go
func counter() func() int {
    count := 0

    return func() int {
        count++
        return count
    }
}

c := counter()

c() // 1
c() // 2
c() // 3
```

The inner function captures `count`, so it keeps access to the same `count` between calls.

> **Closure = a function + the variables it captures from its surrounding scope.**
