# MySQL — `SELECT ... FOR UPDATE` vs `SELECT ... FOR SHARE`

In MySQL/InnoDB, a normal `SELECT` is usually a **non-locking consistent read**.  
Sometimes, however, we need to read data while also protecting that data from concurrent transactions.

MySQL provides two main types of **locking reads**:

```sql
SELECT ... FOR SHARE;
```

and

```sql
SELECT ... FOR UPDATE;
```

The locks acquired by these statements are held until the transaction is committed or rolled back.

---

## 1. `SELECT ... FOR SHARE`

`FOR SHARE` acquires a **shared lock (S lock)** on the rows that are read.

Example:

```sql
START TRANSACTION;

SELECT *
FROM accounts
WHERE id = 1
FOR SHARE;

COMMIT;
```

The main idea is:

> "I need to read this row and make sure nobody changes or deletes it while my transaction is using it."

Multiple transactions can generally acquire a shared lock on the same row at the same time.

For example:

```text
Transaction A: FOR SHARE ✅
Transaction B: FOR SHARE ✅
```

Both transactions can hold shared locks simultaneously.

However, while those shared locks exist, another transaction cannot acquire an incompatible exclusive lock in order to modify the row.

For example:

```text
Transaction A: SELECT ... FOR SHARE
Transaction B: UPDATE ...          ⏳ waits
```

The `UPDATE` must wait until Transaction A commits or rolls back.

### Typical use case

Use `FOR SHARE` when:

- You need to read a row.
- You are not planning to modify that row yourself.
- But you need to guarantee that another transaction cannot modify or delete it before your transaction finishes.

A classic example is checking whether a parent record exists before inserting a related child record.

```sql
START TRANSACTION;

SELECT *
FROM users
WHERE id = 10
FOR SHARE;

INSERT INTO orders (user_id, amount)
VALUES (10, 100);

COMMIT;
```

Here we want to make sure that the user we validated cannot be deleted while we are creating the related order.

---

## 2. `SELECT ... FOR UPDATE`

`FOR UPDATE` acquires an **exclusive-style row lock (X lock)** on the rows that are read.

Example:

```sql
START TRANSACTION;

SELECT balance
FROM accounts
WHERE id = 1
FOR UPDATE;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;

COMMIT;
```

The main idea is:

> "I am reading this row because I may modify it, so other transactions must not acquire conflicting locks on it."

If Transaction A executes:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

another transaction trying this:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

must wait.

Likewise:

```sql
UPDATE accounts
SET balance = 500
WHERE id = 1;
```

must wait.

And normally this locking read also conflicts:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR SHARE;
```

So:

```text
Transaction A: FOR UPDATE
Transaction B: FOR UPDATE  ⏳
Transaction C: FOR SHARE   ⏳
Transaction D: UPDATE      ⏳
```

until Transaction A finishes.

---

## Important: `FOR UPDATE` does NOT mean all SELECTs are blocked

This is an important point.

Suppose Transaction A runs:

```sql
START TRANSACTION;

SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

Transaction B can usually still execute a normal:

```sql
SELECT *
FROM accounts
WHERE id = 1;
```

because InnoDB uses **MVCC (Multi-Version Concurrency Control)** for normal consistent reads.

The normal `SELECT` may read a snapshot/version of the row rather than waiting for the lock.

So conceptually:

```text
Transaction A
SELECT ... FOR UPDATE
        │
        ├── UPDATE          ❌ blocked
        ├── FOR UPDATE      ❌ blocked
        ├── FOR SHARE       ❌ blocked
        │
        └── normal SELECT   ✅ usually allowed through MVCC
```

Therefore:

> `FOR UPDATE` does not mean "nobody can read this row."

It means:

> "Nobody can acquire a conflicting lock on this row until my transaction finishes."

---

# Main Difference

The main difference is the **strength and purpose of the lock**.

### `FOR SHARE`

Acquires a shared lock.

Used mainly when:

```text
Read
↓
Need data to remain unchanged
↓
Do NOT necessarily plan to update it
```

Example:

```sql
SELECT *
FROM parent
WHERE id = 10
FOR SHARE;
```

You are essentially saying:

> "I depend on this row existing and remaining unchanged during my transaction."

---

### `FOR UPDATE`

Acquires a stronger, exclusive lock.

Used mainly when:

```text
Read
↓
Make a decision/business calculation
↓
Modify the same row
```

Example:

```sql
SELECT balance
FROM accounts
WHERE id = 1
FOR UPDATE;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;
```

You are essentially saying:

> "I am reserving this row because I am probably going to change it."

---

# Which one is more exclusive?

`FOR UPDATE` is more exclusive than `FOR SHARE`.

Think of it like this:

```text
Normal SELECT
      ↓
No locking read

FOR SHARE
      ↓
Shared lock
Multiple readers can share it

FOR UPDATE
      ↓
Exclusive lock
Only one transaction can hold the conflicting write lock
```

Or:

```text
Lock strength:

Normal SELECT
     <
FOR SHARE
     <
FOR UPDATE
```

---

# Compatibility Table

| Transaction A | Transaction B | Result |
|---|---|---|
| `FOR SHARE` | `FOR SHARE` | ✅ Allowed |
| `FOR SHARE` | `FOR UPDATE` | ⏳ Wait |
| `FOR SHARE` | `UPDATE` | ⏳ Wait |
| `FOR UPDATE` | `FOR UPDATE` | ⏳ Wait |
| `FOR UPDATE` | `FOR SHARE` | ⏳ Wait |
| `FOR UPDATE` | `UPDATE` | ⏳ Wait |
| `FOR UPDATE` | Normal `SELECT` | ✅ Usually allowed |

---

# When should I use `FOR SHARE`?

Use `FOR SHARE` when you need to **protect something you have read**, but you do not intend to update it.

Example situations:

```text
Verify parent exists
        ↓
Prevent parent from being deleted
        ↓
Insert related child record
```

For example:

```sql
START TRANSACTION;

SELECT id
FROM customers
WHERE id = 100
FOR SHARE;

INSERT INTO orders(customer_id, amount)
VALUES (100, 500);

COMMIT;
```

---

# When should I use `FOR UPDATE`?

Use `FOR UPDATE` when the value you read will influence a subsequent write.

Typical flow:

```text
Read value
   ↓
Business logic
   ↓
Update value
```

Examples include:

- Account balances
- Inventory quantities
- Counters
- Seat reservations
- Wallet balances
- Stock allocation
- Order state transitions

Example:

```sql
START TRANSACTION;

SELECT stock
FROM products
WHERE id = 5
FOR UPDATE;

-- Application checks whether stock is enough

UPDATE products
SET stock = stock - 1
WHERE id = 5;

COMMIT;
```

Without the lock, two transactions could potentially read the same stock value and both make decisions based on stale/concurrent information.

---

# Simple Mental Model

Remember it like this:

```text
FOR SHARE
=
"I'm reading this.
Others may also read it,
but nobody should change it."


FOR UPDATE
=
"I'm reading this because
I'm probably going to change it.
Reserve it for me."
```

---

# Final Summary

```text
SELECT
    → Normal read
    → Usually MVCC
    → No locking read

SELECT ... FOR SHARE
    → Shared lock
    → Other FOR SHARE reads allowed
    → Prevents conflicting modifications
    → Useful when you depend on the row remaining stable

SELECT ... FOR UPDATE
    → Exclusive lock
    → Blocks other conflicting locking reads/writes
    → Useful when read → business logic → update
    → More exclusive than FOR SHARE
```

### Key rule

If your workflow is:

```text
SELECT
↓
calculate / validate
↓
UPDATE the same row
```

`FOR UPDATE` is usually the locking read you should think about.

If your workflow is:

```text
SELECT
↓
depend on that row remaining valid
↓
work with another related resource
```

`FOR SHARE` may be the more appropriate choice.