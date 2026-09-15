Yes. If you want **`golang-migrate` to run as a temporary Docker container**, while your migration files stay in your Go project, this is a clean setup.

The official project supports exactly this Docker usage: mount your migration directory into `/migrations`, then run the `migrate/migrate` image. ([GitHub][1])

Assuming:

* PostgreSQL
* your project root is the current directory
* migrations will live in `db/migrations`
* database is reachable from your machine/container

### 1. Create the migration files

From your project root:

```bash
mkdir -p db/migrations
```

Then use a temporary migrate container to **create** the migration files:

```bash
docker run --rm \
  -v "$(pwd)/db/migrations:/migrations" \
  migrate/migrate \
  create -ext sql -dir /migrations -seq create_users_table
```

This will create:

```text
db/
└── migrations/
    ├── 000001_create_users_table.up.sql
    └── 000001_create_users_table.down.sql
```

That's the standard structure recommended by `golang-migrate`: every migration has an `up` and a `down` file. ([GitHub][2])

---

### 2. Put the SQL in `up.sql`

For example:

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

And in:

```text
000001_create_users_table.down.sql
```

put:

```sql
DROP TABLE IF EXISTS users;
```

So:

```text
db/migrations/
├── 000001_create_users_table.up.sql
└── 000001_create_users_table.down.sql
```

---

### 3. Run the migration using a temporary container

Suppose your PostgreSQL connection is:

```text
postgres://postgres:password@localhost:5432/mydb?sslmode=disable
```

You might initially think:

```bash
docker run --rm \
  -v "$(pwd)/db/migrations:/migrations" \
  migrate/migrate \
  -path=/migrations \
  -database "postgres://postgres:password@localhost:5432/mydb?sslmode=disable" \
  up
```

**But there is an important Docker networking issue here.**

Inside the migrate container, `localhost` means **the migrate container itself**, not your host machine.

If PostgreSQL is running on your host, on Linux you can use:

```bash
docker run --rm \
  -v "$(pwd)/db/migrations:/migrations" \
  --network host \
  migrate/migrate \
  -path=/migrations \
  -database "postgres://postgres:password@localhost:5432/mydb?sslmode=disable" \
  up
```

This is also the networking pattern shown in the project's Docker example. ([GitHub][1])

---

## If PostgreSQL is another Docker container

This is actually the setup I'd recommend for your Go project.

For example, if your PostgreSQL container is on a Docker network called `myapp`:

```bash
docker run --rm \
  --network myapp \
  -v "$(pwd)/db/migrations:/migrations" \
  migrate/migrate \
  -path=/migrations \
  -database "postgres://postgres:password@postgres:5432/mydb?sslmode=disable" \
  up
```

Notice:

```text
postgres:5432
```

Here `postgres` is the **PostgreSQL container/service name**, not `localhost`.

---

### 4. Check migration status

You can also run:

```bash
docker run --rm \
  -v "$(pwd)/db/migrations:/migrations" \
  --network host \
  migrate/migrate \
  -path=/migrations \
  -database "postgres://postgres:password@localhost:5432/mydb?sslmode=disable" \
  version
```

You'll get something like:

```text
1
```

meaning migration version `1` has been applied.

---

### 5. Roll it back

To undo the latest migration:

```bash
docker run --rm \
  -v "$(pwd)/db/migrations:/migrations" \
  --network host \
  migrate/migrate \
  -path=/migrations \
  -database "postgres://postgres:password@localhost:5432/mydb?sslmode=disable" \
  down 1
```

Then the `users` table will be dropped.

---

### The mental model

Think of it like this:

```text
Your Go project
│
├── cmd/
├── internal/
├── go.mod
│
└── db/
    └── migrations/
        ├── 000001_create_users_table.up.sql
        └── 000001_create_users_table.down.sql
             │
             │ mounted as /migrations
             ▼
       ┌──────────────────────┐
       │ migrate/migrate      │
       │ temporary container  │
       └──────────┬───────────┘
                  │
                  │ SQL
                  ▼
       ┌──────────────────────┐
       │ PostgreSQL           │
       │ mydb                 │
       └──────────────────────┘
```

The container is **ephemeral** because of:

```bash
--rm
```

So you don't need to install `migrate` on your machine and you don't need to keep a migrate container running. The migration files themselves remain in your project because `db/migrations` is bind-mounted into the temporary container.

The official CLI also supports `create`, `up`, `down`, `version`, `force`, etc. ([GitHub][3])

If you're using **Docker Compose for your Go project + PostgreSQL**, I can also show you the cleaner `docker compose run --rm migrate ...` setup, which is probably the one I'd use for your project.

[1]: https://github.com/golang-migrate/migrate?utm_source=chatgpt.com "GitHub - golang-migrate/migrate: Database migrations. CLI and Golang library. · GitHub"
[2]: https://github.com/golang-migrate/migrate/blob/master/GETTING_STARTED.md?utm_source=chatgpt.com "migrate/GETTING_STARTED.md at master · golang-migrate/migrate · GitHub"
[3]: https://github.com/golang-migrate/migrate/blob/master/cmd/migrate/README.md?utm_source=chatgpt.com "migrate/cmd/migrate/README.md at master · golang-migrate/migrate · GitHub"
