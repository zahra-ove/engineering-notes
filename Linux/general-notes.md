### 💡 Linux Tip: Encode a String with Base64

You can use `echo -n` together with `base64` to encode a string directly in the terminal:

```bash
echo -n "your-value" | base64
```

* `echo -n` prints the string **without adding a trailing newline**.
* `|` pipes the output of `echo` to the `base64` command.
* `base64` encodes the input into Base64 format.

**Example:**

```bash
echo -n "hello" | base64
```

Output:

```text
aGVsbG8=
```

> **Note:** Using `-n` is important because otherwise `echo` adds a newline character (`\n`) to the input, which will also be encoded.


---


# Linux Signals: SIGTERM, SIGKILL, and SIGINT

### 1. SIGTERM (Terminate)
- Requests a process to terminate gracefully.
- Allows the application to perform cleanup and release resources.
- Can be caught and handled by the application.

```bash
kill -SIGTERM 1234
```

### 2. SIGKILL (Kill)
- Forces a process to terminate immediately.
- Cannot be caught, ignored, or handled.
- Does not allow the application to perform cleanup.

```bash
kill -SIGKILL 1234
```

### 3. SIGINT (Interrupt)
- Requests a process to stop, usually due to a user interrupt.
- Commonly sent when pressing `Ctrl+C` in a terminal.
- Can be caught and handled by the application.

```bash
kill -SIGINT 1234
```

**Key Takeaway:**
- `SIGTERM`: Graceful termination request.
- `SIGKILL`: Immediate, forceful termination.
- `SIGINT`: User interrupt request.

`1234` represents the target process ID (PID).


---

`ps faux`

The ps faux command displays running processes in a tree-like structure, showing the relationships between parent and child processes.

---

