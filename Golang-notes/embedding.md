### Go: Embedding vs. Inheritance

In Go, we don't use classical inheritance to extend the functionality of a class. Instead, we use struct embedding, a technique based on composition.

By embedding one struct into another, the outer struct can directly access the embedded struct's fields and methods through method promotion.

This allows us to reuse and extend functionality without creating an inheritance hierarchy.

Example:

Go

```
type Store struct {
    *Queries
    db *sql.DB
}
```

By embedding `*Queries` into `Store`, we can call its methods directly:

Go

```
store.GetAccount(ctx, id)
```

Instead of:

Go

```
store.Queries.GetAccount(ctx, id)
```

Key takeaway: Embedding promotes fields and methods, but it does not establish an inheritance relationship. A `Store` contains a `*Queries`; it is not a subtype of `Queries`.
