## Important Notes: `Prepare()` vs `Exec()` / `Query()` in Go

### 1. `Prepare()` is optional

You can execute SQL statements directly using `DB.Exec()` or `DB.Query()`. You don't always need to prepare a statement first.

### 2. When to use each method

| Method          | Purpose                                                             |
| --------------- | ------------------------------------------------------------------- |
| `DB.Exec()`     | INSERT, UPDATE, DELETE, CREATE TABLE                                |
| `DB.Query()`    | SELECT queries that return multiple rows                            |
| `DB.QueryRow()` | SELECT queries that return a single row                             |
| `DB.Prepare()`  | Prepares an SQL statement for execution, potentially multiple times |

### 3. Advantage of `Prepare()`

- Can improve performance when executing the same SQL statement multiple times, potentially with different parameter values.
- The prepared statement can be reused by calling `stmt.Exec()` or `stmt.Query()` multiple times.
- Close the statement with `stmt.Close()` when you're finished using it.

### 4. Important performance consideration

If you prepare a statement and execute it only once before closing it, you generally won't gain much performance from preparing it explicitly.

Example:

```go
stmt, err := DB.Prepare(
    "INSERT INTO users (name) VALUES (?)",
)
if err != nil {
    return err
}
defer stmt.Close()

for _, name := range names {
    _, err := stmt.Exec(name)
    if err != nil {
        return err
    }
}
```

Here, the same prepared statement is reused for multiple inserts.

### 5. Best practice to remember

- Use `Exec()` for operations that don't return rows.
- Use `Query()` or `QueryRow()` to retrieve data.
- Consider `Prepare()` when you need to execute the same SQL statement repeatedly.
- Always handle errors and close resources appropriately.

Key takeaway: `Prepare()` is not mandatory. Its main potential benefit is reusing a prepared SQL statement across multiple executions.