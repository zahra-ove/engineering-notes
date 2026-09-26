# Transaction Isolation Levels — MySQL/InnoDB and PostgreSQL

## Common Read Anomalies

### Dirty Read
A transaction reads data written by another transaction that has not committed yet.

```text
Tx A:
UPDATE balance = 500
-- not committed

Tx B:
SELECT balance
→ sees 500
```

If Tx A rolls back, Tx B has read data that never became permanent.

---

### Non-repeatable Read
The same row is read twice inside one transaction, but another transaction modifies and commits it between the two reads.

```text
Tx A:
SELECT balance
→ 1000

Tx B:
UPDATE balance = 500
COMMIT

Tx A:
SELECT balance
→ 500
```

---

### Phantom Read
The same query returns a different set of rows because another transaction inserts or deletes rows that match the query condition.

```text
Tx A:
SELECT *
FROM users
WHERE age > 18;
→ 10 rows

Tx B:
INSERT INTO users(age) VALUES (25);
COMMIT

Tx A:
SELECT *
FROM users
WHERE age > 18;
→ 11 rows
```

---

### Serialization Anomaly
Concurrent transactions produce a result that could not have happened if the transactions had executed one by one in some serial order.

This is a broader anomaly than Dirty Read, Non-repeatable Read, or Phantom Read.

---

# MySQL / InnoDB

InnoDB supports:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

Default:

```text
REPEATABLE READ
```

---

## 1. READ UNCOMMITTED

Weakest isolation level.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

Behavior:

```text
Dirty Read            possible
Non-repeatable Read   possible
Phantom Read          possible
```

A normal `SELECT` can potentially see uncommitted changes from another transaction.

### Locking behavior

A plain `SELECT` is normally not a locking read.

```sql
SELECT *
FROM accounts;
```

But explicit locking reads still work:

```sql
SELECT ... FOR SHARE;
SELECT ... FOR UPDATE;
```

### Use case

Useful only when maximum concurrency matters more than consistency.

Usually not appropriate for financial or business-critical logic.

---

# 2. READ COMMITTED

Only committed data is visible.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

Each normal `SELECT` gets a fresh snapshot.

Example:

```text
Tx A:
SELECT balance
→ 1000

Tx B:
UPDATE balance = 500
COMMIT

Tx A:
SELECT balance
→ 500
```

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   possible
Phantom Read          possible
```

### Important behavior

Each statement gets its own snapshot:

```text
SELECT #1
→ snapshot A

SELECT #2
→ snapshot B
```

Therefore two reads in the same transaction can return different committed values.

### Locking behavior

Plain `SELECT` is normally a non-locking consistent read.

```sql
SELECT *
FROM accounts;
```

It usually does not acquire row locks.

Explicit locking reads do:

```sql
SELECT ... FOR SHARE;
SELECT ... FOR UPDATE;
```

### Gap locking

In Read Committed, gap locking is much less aggressive than in Repeatable Read.

It is mainly retained for cases such as:

```text
foreign-key checking
duplicate-key checking
```

This improves concurrency.

### UPDATE / DELETE behavior

Rows that turn out not to match the `WHERE` condition can often have their locks released earlier.

This also improves concurrency.

### Typical use case

Good when you want:

```text
fresh committed data
+
high concurrency
```

and do not require the same snapshot for the whole transaction.

---

# 3. REPEATABLE READ

Default isolation level in InnoDB.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

The first consistent read creates a transaction snapshot.

Later consistent reads use that same snapshot.

Example:

```text
Tx A:
SELECT balance
→ 1000

Tx B:
UPDATE balance = 500
COMMIT

Tx A:
SELECT balance
→ still 1000
```

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   prevented
Phantom Read          prevented for normal consistent reads
```

### Important point about phantoms

For normal snapshot reads, newly committed rows from other transactions are not visible because the transaction keeps using the same snapshot.

For locking reads, InnoDB may use:

```text
record lock
gap lock
next-key lock
```

to protect a searched range.

---

## Record Lock

Locks an index record.

Example:

```sql
SELECT *
FROM users
WHERE id = 10
FOR UPDATE;
```

If `id` is a unique indexed column, InnoDB may lock only that record.

---

## Gap Lock

Locks the space between index records.

Example:

```text
existing keys:

10
20

gap:

(10,20)
```

A gap lock can prevent another transaction from inserting a new row into that range.

---

## Next-Key Lock

Combination of:

```text
Record Lock
+
Gap Lock
```

Used to protect both existing records and the surrounding index range.

---

### Very important nuance

In Repeatable Read:

```sql
SELECT ...
```

and:

```sql
SELECT ... FOR UPDATE;
```

do not necessarily behave as the same kind of read.

A normal `SELECT` uses:

```text
snapshot / historical view
```

A locking read uses:

```text
current relevant committed state
+
locking
```

So mixing locking and non-locking reads in the same transaction can sometimes be confusing.

### Typical use case

Useful when you need:

```text
stable transaction snapshot
+
stronger protection against range changes
```

---

# 4. SERIALIZABLE

Strongest isolation level.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

The goal is to make concurrent transactions behave as though they had executed one by one.

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   prevented
Phantom Read          prevented
Serialization anomaly prevented
```

### Plain SELECT behavior

Inside a transaction with autocommit disabled, a normal `SELECT` behaves approximately like:

```sql
SELECT ... FOR SHARE;
```

So it becomes a locking read.

Example:

```sql
START TRANSACTION;

SELECT *
FROM accounts
WHERE id = 1;
```

Conceptually behaves like:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR SHARE;
```

### Lock type

For a single indexed record:

```text
Shared Lock / S Lock
```

may be acquired.

For range queries:

```sql
SELECT *
FROM accounts
WHERE id BETWEEN 10 AND 20;
```

InnoDB may use:

```text
record locks
gap locks
next-key locks
```

depending on the index and query plan.

### Important autocommit nuance

Do not simplify Serializable to:

```text
"every SELECT always takes a shared lock"
```

That statement is incomplete.

The locking behavior matters especially when the `SELECT` is inside a transaction and autocommit is disabled.

### Blocking

Serializable increases blocking because reads can conflict with writes.

### Lock wait timeout

If one transaction waits for a lock held by another transaction, MySQL uses:

```text
innodb_lock_wait_timeout
```

to limit how long the statement waits.

Important:

```text
Serializable itself does not have a special timeout.
```

The timeout belongs to InnoDB lock waiting.

### Lock timeout vs deadlock

These are different.

```text
Lock wait:
Tx B waits for Tx A
→ may continue waiting until timeout

Deadlock:
Tx A waits for Tx B
Tx B waits for Tx A
→ deadlock detector chooses a victim
→ one transaction is rolled back
```

A deadlock can be detected before the normal lock wait timeout expires.

### Typical use case

Use Serializable when correctness requires the strongest isolation and you accept:

```text
more blocking
more deadlocks
lower concurrency
more retry requirements
```

---

# MySQL Summary

```text
READ UNCOMMITTED
→ Dirty reads possible
→ weakest isolation

READ COMMITTED
→ only committed data
→ fresh snapshot per statement
→ non-repeatable reads possible
→ phantoms possible

REPEATABLE READ
→ stable snapshot for consistent reads
→ default InnoDB level
→ gap / next-key locking for locking ranges

SERIALIZABLE
→ strongest isolation
→ plain SELECT can become locking read
→ more blocking and less concurrency
```

---

# PostgreSQL

PostgreSQL accepts these four names:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

But in practice:

```text
READ UNCOMMITTED
=
READ COMMITTED
```

Default:

```text
READ COMMITTED
```

---

# 1. PostgreSQL READ UNCOMMITTED

PostgreSQL does not provide true dirty reads.

If you request:

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

it behaves like:

```text
READ COMMITTED
```

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   possible
Phantom Read          possible
Serialization anomaly possible
```

So unlike MySQL, PostgreSQL does not expose uncommitted row versions.

---

# 2. PostgreSQL READ COMMITTED

Default PostgreSQL isolation level.

Each statement gets a new snapshot.

Example:

```text
Tx A:
SELECT balance
→ 1000

Tx B:
UPDATE balance = 500
COMMIT

Tx A:
SELECT balance
→ 500
```

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   possible
Phantom Read          possible
Serialization anomaly possible
```

### Important behavior for UPDATE and locking reads

Statements such as:

```sql
UPDATE ...
DELETE ...
SELECT ... FOR UPDATE
SELECT ... FOR SHARE
```

may encounter a row currently modified by another transaction.

In that case they can wait.

After the other transaction commits, PostgreSQL can re-check the `WHERE` condition against the updated row version.

This is an important detail of Read Committed.

### Typical use case

Good general-purpose level when:

```text
fresh committed data
+
high concurrency
```

is preferred.

---

# 3. PostgreSQL REPEATABLE READ

Uses snapshot isolation.

The transaction works with a stable snapshot.

Example:

```text
Tx A starts
↓
snapshot created

Tx B modifies data
COMMIT

Tx A
↓
still sees its original snapshot
```

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   prevented
Phantom Read          prevented in PostgreSQL
Serialization anomaly still possible
```

### Important difference from standard expectations

PostgreSQL Repeatable Read is stronger than the minimum SQL requirement because it also prevents phantom reads.

### But it is not Serializable

Two transactions can still create a business-level serialization anomaly.

So:

```text
Repeatable Read
≠
Serializable
```

### Concurrent update behavior

If a Repeatable Read transaction tries to modify a row that has been changed since its snapshot, PostgreSQL may abort the transaction.

Example error:

```text
could not serialize access due to concurrent update
```

The application should then:

```text
retry the entire transaction
```

### Typical use case

Good when you need:

```text
stable snapshot
+
no phantoms
```

but can tolerate retrying on certain concurrency conflicts.

---

# 4. PostgreSQL SERIALIZABLE

Strongest PostgreSQL isolation level.

PostgreSQL Serializable is not implemented simply by turning every `SELECT` into `FOR SHARE`.

Instead, it uses:

```text
Snapshot Isolation
+
dependency tracking
+
serialization conflict detection
```

Behavior:

```text
Dirty Read            prevented
Non-repeatable Read   prevented
Phantom Read          prevented
Serialization anomaly prevented
```

### Predicate tracking

PostgreSQL tracks read dependencies using predicate-style locks.

These can appear internally as:

```text
SIReadLock
```

Important:

```text
Predicate locks are not ordinary blocking locks.
```

They are mainly used to detect unsafe dependency patterns.

Instead of always blocking another transaction, PostgreSQL may allow both transactions to continue and later abort one with a serialization failure.

### Retry requirement

Applications using Serializable must be prepared for:

```text
serialization failure
```

commonly identified by:

```text
SQLSTATE 40001
```

The correct response is:

```text
retry the entire transaction
```

### Key difference from MySQL Serializable

MySQL tends to rely more heavily on:

```text
blocking locks
shared locks
range locks
```

PostgreSQL Serializable relies more heavily on:

```text
snapshot isolation
dependency detection
abort + retry
```

### Typical use case

Use when you need the strongest correctness guarantee and are prepared to handle retries.

---

# PostgreSQL Explicit Row Locks

Important lock modes:

```text
FOR KEY SHARE
FOR SHARE
FOR NO KEY UPDATE
FOR UPDATE
```

Approximate strength order:

```text
FOR KEY SHARE
<
FOR SHARE
<
FOR NO KEY UPDATE
<
FOR UPDATE
```

---

## FOR SHARE

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR SHARE;
```

Takes a shared row lock.

Multiple compatible shared locks can coexist.

Conflicting writes must wait.

---

## FOR UPDATE

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

Takes a stronger row lock.

Used when you read a row because you intend to modify it.

Conflicting operations on the same row must wait.

A plain `SELECT` is still usually allowed because PostgreSQL uses MVCC.

---

# PostgreSQL Lock Timeout

PostgreSQL provides:

```text
lock_timeout
```

This limits how long a statement waits to acquire a lock.

Default:

```text
0
```

which means disabled.

Important:

```text
Serializable does not automatically define a lock timeout.
```

Lock timeout is a separate setting.

---

# PostgreSQL Deadlock Timeout

PostgreSQL also has:

```text
deadlock_timeout
```

This is not the same as lock timeout.

Conceptually:

```text
wait for some time
↓
check whether a deadlock exists
```

If a deadlock exists, one transaction is aborted.

If no deadlock exists, waiting may continue.

So:

```text
deadlock_timeout
≠
maximum lock wait time
```

---

# MySQL vs PostgreSQL — Important Differences

## Repeatable Read

### MySQL

```text
stable snapshot
+
gap locks
+
next-key locks
for locking range reads
```

### PostgreSQL

```text
snapshot isolation
+
no InnoDB-style gap locking
```

---

## Serializable

### MySQL

```text
more blocking
plain SELECT may become FOR SHARE
range locks possible
```

### PostgreSQL

```text
snapshot isolation
+
predicate dependency tracking
+
serialization failure
+
retry
```

---

# Practical Rules

```text
1. Isolation level and explicit lock are not the same thing.

2. MVCC allows many plain SELECTs to avoid blocking writers.

3. Use FOR UPDATE when:
   read → validate/calculate → update

4. Use FOR SHARE when:
   read → depend on row remaining stable

5. MySQL Repeatable Read may use gap and next-key locks
   for locking range queries.

6. PostgreSQL Repeatable Read prevents phantom reads
   through snapshot isolation, not InnoDB-style gap locking.

7. PostgreSQL Serializable relies heavily on retry.

8. Lock timeout and deadlock detection are different concepts.

9. Higher isolation generally gives:
   more consistency
   less concurrency
   more blocking or retries

10. Serializable does not eliminate the need for:
    transaction retry
    idempotency
    correct lock ordering
    short transactions
```

# Good Mental Model

```text
MySQL:

READ UNCOMMITTED
→ almost no read isolation

READ COMMITTED
→ new committed snapshot per statement

REPEATABLE READ
→ same transaction snapshot

SERIALIZABLE
→ locking reads + strongest isolation
```

```text
PostgreSQL:

READ UNCOMMITTED
→ behaves as READ COMMITTED

READ COMMITTED
→ new snapshot per statement

REPEATABLE READ
→ stable snapshot isolation

SERIALIZABLE
→ stable snapshot
  + dependency detection
  + abort/retry if unsafe
```


| Phenomenon | READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|---|---|---|---|---|
| Dirty Read | ❌ Prevented | ❌ Prevented | ❌ Prevented | ❌ Prevented |
| Non-repeatable Read | ✅ Possible | ✅ Possible | ❌ Prevented | ❌ Prevented |
| Phantom Read | ✅ Possible | ✅ Possible | ❌ Prevented | ❌ Prevented |
| Serialization Anomaly | ✅ Possible | ✅ Possible | ✅ Possible | ❌ Prevented |

**PostgreSQL note:** `READ UNCOMMITTED` behaves the same as `READ COMMITTED`, so Dirty Reads are still prevented.