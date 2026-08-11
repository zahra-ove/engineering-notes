1. Go has value semantics and pointer semantics.

2. A pointer return means the caller gets shared access,
   not its own copy.

3. Escape analysis is a form of static code analysis
   performed by the Go compiler.

4. Escape analysis determines whether a value should be
   constructed on the stack or heap.

5. "Escape" does NOT mean Stack → Heap migration.
   It means the value is constructed on the heap in the first place.

6. Escape analysis is primarily about how a value is shared,
   not simply where it is constructed.

7. Stack memory is cheaper to manage; heap memory is managed
   by the garbage collector.

8. Heap allocation can introduce additional latency/GC cost.

9. Don't optimize for performance based on guesses.
   Optimize for correctness, readability and simplicity first.

10. Use compiler tooling such as:
    go build -gcflags="-m=2"
    to inspect escape analysis decisions.