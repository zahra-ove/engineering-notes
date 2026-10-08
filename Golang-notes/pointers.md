### Dereferencing a Pointer in Go

The `*` operator is used to **dereference a pointer**.

**Dereferencing means accessing the value stored at the memory address that the pointer points to.**

```go
x := 10
p := &x

fmt.Println(p)  // address of x
fmt.Println(*p) // value stored at that address → 10
```

* `&x` → gets the **address** of `x`
* `p` → stores the **address**
* `*p` → gets the **value at that address**

### Example with a slice

```go
numbers := []int{1, 2, 3}
p := &numbers

fmt.Println((*p)[0]) // 1
```

Here:

```go
*p
```

dereferences the pointer and gives us the original slice:

```text
p
↓
address of numbers
↓
*p
↓
[1, 2, 3]
```

Then:

```go
(*p)[0]
```

means:

> **Dereference `p` first, then access the first element of the resulting slice.**

**Key idea:**

> `*pointer` → **dereference the pointer and access the value it points to.**
