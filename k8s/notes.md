1. main components:
- pod
- service
- ingress
- deployment
- statefulset
- daemonset

2. k8s: container orchestration tool, developed by google

3. for stateful apps --> statefulset is used (like mysql, elastic)
4. for stateless apps --> deployment is used

5. service component has three types
5. service component has static ip address and it's state is independent of pods.
6. pod is ephemeral component

7. `Etcd` holds the current status of any k8s component

---

## YAML: Multiline Strings

YAML provides two operators for writing multiline strings:

### 1. Literal Block Scalar (`|`)

- Preserves line breaks (`\n`) in the string.
- Each line remains a separate line.

**Example:**

```yaml
command:
  - sh
  - -c
  - |
    echo "Hello"
    echo "World"
```

The resulting string is:

```bash
echo "Hello"
echo "World"
```

**Output:**
```text
Hello
World
```

### 2. Folded Block Scalar (`>`)

- Folds most line breaks into spaces.
- The resulting string is treated as a single line, except for blank lines and certain indentation cases.

**Example:**

```yaml
message: >
  Hello
  World
```

The resulting string is:

```text
Hello World
```

### Comparison

| Operator | Name | Behavior |
|---|---|---|
| `|` | Literal block scalar | Preserves line breaks |
| `>` | Folded block scalar | Converts most line breaks into spaces |

### Docker Compose Example

```yaml
services:
  app:
    image: alpine
    command:
      - sh
      - -c
      - |
        echo "Starting app..."
        sleep 5
        echo "App finished."
```

Here, `|` preserves the newlines between shell commands, allowing the shell to execute them as separate commands.

### Important Notes

- Both `|` and `>` introduce multiline strings.
- `|` preserves line breaks; `>` folds most line breaks into spaces.
- Neither operator automatically executes commands. The meaning depends on where the string is used.
- These operators are useful for multiline shell scripts, configuration values, and text blocks.

---

