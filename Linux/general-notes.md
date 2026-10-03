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
