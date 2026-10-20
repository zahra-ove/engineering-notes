1. docker run -it ubuntu
`-it` => interactive mode
it means, we get access to terminal of this container by using `it` flag

2. to count approximately number of installed packages in ubuntu system:

`dpkg -l | wc -l`

---
3. 
# Docker Graceful Shutdown

## How `docker stop` Works

The `docker stop` command initiates a graceful shutdown by following these steps:

1. **Send `SIGTERM`:** Docker asks the main process to terminate.
2. **Graceful Shutdown:** The application can finish ongoing requests, close database connections, and release resources if it handles the signal correctly.
3. **Wait for Timeout:** Docker waits for a grace period (10 seconds by default on Linux).
4. **Send `SIGKILL`:** If the process has not exited, Docker forcibly terminates it.

## Difference Between `docker stop` and `docker kill`

| Command | Behavior |
|---|---|
| `docker stop <container>` | Sends `SIGTERM`, waits, then sends `SIGKILL` if necessary. |
| `docker kill <container>` | Sends `SIGKILL` by default, terminating the process immediately. |
| `docker stop -t 30 <container>` | Waits up to 30 seconds before forcefully terminating the process. |

## Key Takeaway

`docker stop` **allows graceful shutdown but does not guarantee it**. The application must handle termination signals correctly and complete its cleanup before the timeout expires.

---

`docker ps -l` displays the latest created container, regardless of its current state.

- docker: Docker CLI.
- ps: Lists containers.
- -l: Short for --latest; shows the most recently created container.

---

`docker ps -q` displays only the container IDs of running containers, without additional details.

---

# Docker: `stop` vs `kill`

### `docker stop`
- Sends `SIGTERM` to the container's main process.
- Allows the application to shut down gracefully and clean up resources.
- If the process does not exit within the timeout, Docker sends `SIGKILL`.
- Default timeout on Linux: **10 seconds**.

```bash
docker stop <container>
```

### `docker kill`
- Sends `SIGKILL` by default.
- Terminates the process immediately without allowing graceful cleanup.
- Can send a different signal using the `--signal` option.

```bash
docker kill <container>
```

### Key Differences

| Feature | `docker stop` | `docker kill` |
|---|---|---|
| Default signal | `SIGTERM` | `SIGKILL` |
| Graceful shutdown | Yes, if supported by the application | No, by default |
| Waits for timeout | Yes | No |
| Typical use | Normal shutdown | Immediate termination |

**Key takeaway:** Use `docker stop` for normal shutdowns and `docker kill` when immediate termination is required.

---

`docker kill $(docker ps -q)`
kills all running containers immediately

---

## `docker diff`: See What Changed in the R/W Layer

Syntax: `docker diff <container-name-or-id>`
(newer form: `docker container diff <container>`, which does the same thing)

**What it does:** Lists every file and directory that was added, changed, or deleted
in a container's filesystem compared to its image.
In other words, it shows **the contents of the container's R/W layer**.

| Symbol | Meaning                          |
|--------|----------------------------------|
| `A`    | **Added**: file/dir was created  |
| `C`    | **Changed**: file/dir was modified |
| `D`    | **Deleted**: file/dir was removed |

### Example
    docker run -it --name test ubuntu bash
    # inside the container:
    echo hi > /root/note.txt
    rm /etc/legal
    exit

    docker diff test

Output:
    C /root
    A /root/note.txt
    C /etc
    D /etc/legal

### Notes
- A parent directory shows as `C` when something inside it is added or deleted
  (e.g., `C /root` because `note.txt` was added).
- It also works on **stopped** containers. The R/W layer exists until `docker rm`.
- Files written to **volumes** are NOT shown. Volumes are not part of the R/W layer.
- The image itself is never changed. `docker diff` proves that all changes live
  only in the container layer.

### Use cases
- **Debugging:** find out where an app writes files at runtime (logs, cache, temp files).
- **Disk usage:** find out why a container keeps growing in size.
- **Before `docker commit`:** check exactly what will be saved into a new image.
- **Security/auditing:** detect unexpected file changes inside a container.

---

## Docker: Image Layers vs. Container R/W Layer

- An **image** is a stack of **read-only (R/O) layers**. Each Dockerfile
  instruction (FROM, RUN, COPY, ...) creates one layer. Once built, layers never change.
- When you `docker run` an image, Docker adds a **thin read-write (R/W) layer**
  on top. This is the **container layer**. It starts empty.
- **All changes made inside a container go only into this top R/W layer.**

### How file operations work
| Operation        | What happens                                                        |
|------------------|---------------------------------------------------------------------|
| Read a file      | Docker searches layers top → bottom and returns the first match.    |
| Create a file    | Written only to the R/W layer.                                      |
| Modify a file    | **Copy-on-Write (CoW):** the file is copied up from the image layer |
|                  | into the R/W layer, then the copy is modified. Original untouched.  |
| Delete a file    | A "whiteout" marker is added in the R/W layer to hide it.           |
|                  | The file still exists in the image layer.                           |

### Why it matters
1. **Sharing:** Many containers can share the same image layers on disk.
   Each container only adds its own small R/W layer.
2. **Immutability:** Containers cannot change or corrupt the image.
3. **Ephemeral data:** `docker rm` deletes the R/W layer and everything written to it.
   → Use **volumes** for data that must persist (e.g., databases).

### Analogy
Image = a printed book (read-only).
R/W layer = a transparent sheet on top. You write on the sheet, the book stays unchanged.

---

