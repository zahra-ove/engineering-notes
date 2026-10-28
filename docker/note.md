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

**Docker: Image & Container Layers**

- **Image = read-only layers.** These are shared by all containers created from the same image and stored on disk only once.
- **Each container gets its own thin R/W layer** (the container layer) on top of the image layers.
- **All writes go to the container layer only.** The image and other containers are not affected.
- **Copy-on-Write (CoW):**
  - Modify an image file → it's first copied up into the container layer, then changed.
  - Delete a file → a "whiteout" marker is added; the original stays in the image.
  - Handled by the storage driver (usually `overlay2`).
- **The container layer is ephemeral:**
  - `docker stop` / `start` → data kept.
  - `docker rm` → container layer and its data are deleted.
- **For persistent or shared data → use Volumes / Bind mounts** (`-v name:/path`).
  - Data survives container removal.
  - Can be shared between containers.
  - Faster for heavy writes (DBs, logs).
- **Useful commands:**
  - `docker diff <container>` → changes in the R/W layer (A = added, C = changed, D = deleted)
  - `docker ps -s` → size of the R/W layer (`virtual` = image + R/W layer)

```
[Container A R/W]   [Container B R/W]
         \               /
     [Image Layers – Read-only, Shared]
```


---

# `docker diff` & `docker commit`

## Background: Container Filesystem (overlay2)
- **lowerdir**: the image layers. Read-only and shared between containers.
- **upperdir**: the container's own writable layer, at `/var/lib/docker/overlay2/<id>/diff`.
- **merged**: the combined view the container sees as `/`.
- **Copy-up**: when a file from the image is modified, it is first copied to upperdir, then changed there.
- **Whiteout**: when a file from the image is deleted, a char device `0/0` is created in upperdir to hide it.
- **Opaque dir**: a directory marked with xattr `trusted.overlay.opaque=y`, which hides the lower-layer contents of that directory.
- Find the upperdir: `docker inspect -f '{{.GraphDriver.Data.UpperDir}}' <container>`

> `docker diff` **shows** the upperdir. `docker commit` **turns** the upperdir into a new image layer.

---

## `docker diff`

**Purpose:** Lists filesystem changes in a container compared to its image.

```bash
docker diff <container>
```

| Symbol | Meaning |
|---|---|
| `A` | Added |
| `C` | Changed (parent dirs of any change also show as `C`) |
| `D` | Deleted |

**Internals:**
- The CLI calls `GET /containers/{id}/changes`, and the daemon calls the storage driver's `Changes()`.
- **overlay2 (fast path):** only walks upperdir.
  - File in upperdir that also exists in lower: `C`
  - File only in upperdir: `A`
  - Whiteout: `D`
- **vfs and similar (naive diff):** walks both trees and compares mode, size, mtime, uid/gid. Slower.

**Key points:**
- Volumes and bind mounts are **not** included.
- `/etc/hosts`, `/etc/resolv.conf` and `/etc/hostname` are bind mounts, so they are excluded.
- Works on **stopped** containers too, because the layer exists until `docker rm`.
- A metadata-only change (e.g. `chmod`) still triggers copy-up and shows as `C`.

**Use cases:**
1. **Security / forensics:** detect suspicious files or tampering.
   ```bash
   docker diff app | grep -E '^(A|C) /(bin|usr/bin|etc|tmp|root)'
   ```
2. **Preparing `--read-only`:** find where the app writes, then add `tmpfs` or volumes for those paths.
   ```bash
   docker run --read-only --tmpfs /var/cache/nginx --tmpfs /var/run nginx
   ```
3. **Disk growth debugging:** when `docker ps -s` shows a large container size, find logs or data that should be on a volume.
4. **Reverse-engineering a manual install:** see what an installer changed, then move it to a Dockerfile.
5. **Immutability checks:** in CI or audits, verify the app doesn't write outside allowed paths.

---

## `docker commit`

**Purpose:** Creates a new image from a container's current filesystem state.

```bash
docker commit [OPTIONS] CONTAINER [REPO[:TAG]]
```

**Main flags:**
- `-m`: commit message (shows in `docker history`)
- `-a`: author
- `-c` / `--change`: apply Dockerfile instructions. Allowed: `CMD`, `ENTRYPOINT`, `ENV`, `EXPOSE`, `LABEL`, `USER`, `VOLUME`, `WORKDIR`, `ONBUILD`.
- `-p` / `--pause`: default `true`. Freezes the container (cgroup freezer) for a consistent snapshot.

```bash
docker commit -a "zahra" -m "add curl" tmp myubuntu:curl
docker commit -c 'CMD ["nginx","-g","daemon off;"]' -c 'EXPOSE 8080' web myweb:v1
```

**Internals:**
1. Pause the container (if `-p`).
2. The storage driver's `Diff()` creates a **tar stream** of upperdir.
   - Deleted file is stored as `.wh.<name>`.
   - Opaque dir is stored as `.wh..wh..opq`.
3. SHA256 of the tar gives the **diffID**, and a new layer is registered.
4. A new **image config** is built:
   - `rootfs.diff_ids` = base layers + new layer
   - `config` is copied from the container, then `--change` is applied
   - a new `history` entry is added
5. SHA256 of the config gives the **Image ID**, and the tag is applied.
6. Unpause the container.

**NOT included in the image:**
- ❌ Volume data. For example, committing a MySQL container does **not** save `/var/lib/mysql`.
- ❌ Running processes and memory. For that, use `docker checkpoint` (CRIU).
- ❌ Runtime settings: `-p`, `--network`, resource limits, bind mounts.

**✅ Good use cases:**
1. **Incident snapshot / forensics:** preserve a broken container before restarting it.
   ```bash
   docker commit -m "incident" broken-app evidence/app:1
   docker save evidence/app:1 | gzip > incident.tar.gz
   ```
2. **Debugging a crashing container:**
   ```bash
   docker commit crashed debug:1
   docker run -it --rm --entrypoint sh debug:1
   ```
3. **Quick experiments:** checkpoint your progress while testing.
4. **Legacy software** with interactive or GUI installers.
5. **Learning:** understand how images and layers work.

**❌ Anti-patterns (don't use for production images):**
- Not reproducible: no source of truth, unlike a Dockerfile.
- Bloated images: caches, temp files, shell history.
- Secret leakage: `~/.bash_history`, `~/.ssh`, typed passwords.
- No layer caching, and hard to rebuild on a new base image (security patches).
- No review, audit or version control.
- Not suitable for database backups (volumes are excluded).

> 💡 The legacy Docker builder (pre-BuildKit) ran each `RUN` step as container → execute → commit. BuildKit uses snapshotters directly.

---

## Combined Workflow
```bash
docker run -it --name lab debian:12 bash      # 1. manual install / experiment
docker diff lab | grep -v '^C' > changes.txt  # 2. see what changed
docker commit -m "checkpoint" lab lab:wip     # 3. temporary snapshot
# 4. write a clean Dockerfile from changes.txt, then docker build
```

## Summary
- **`docker diff`** is an observation tool for debugging, security, understanding app behavior and preparing `--read-only`. Safe and very useful.
- **`docker commit`** is a snapshot tool for forensics, debugging and experiments. Always build production images with a **Dockerfile**.



---

# `docker build --no-cache` vs `--pull`

**Basic syntax:**
```bash
docker build -t <name>:<tag> <context-path>
docker build -t myapp:1.0 .
```

## How the build cache works
- `FROM`: uses the **local** base image if it exists.
- `RUN`: cache key = **command text only**. External changes (internet, apt repos) are not detected.
- `COPY` / `ADD`: cache key = **file content checksums**.
- One cache miss → **all following steps** are rebuilt.

## `--no-cache`
- **What it does:** ignores the build cache and re-executes **every** instruction.
- **Does NOT** re-pull the base image (`FROM` uses the local copy).
- **Use when:**
  - `RUN apt-get update` / `apk add` is cached and packages are outdated
  - `RUN curl` / `git clone` must fetch the latest content
  - debugging suspected stale or broken cache layers
  - making clean release builds
- **Cost:** slow, so don't use it for daily dev builds.

## `--pull`
- **What it does:** always checks the registry for a newer version of the `FROM` image.
- **Cache is still used.** If the base image changed, the following layers rebuild automatically.
- **Use when:**
  - mutable tags (`python:3.12`, `node:20`, `ubuntu:24.04`) receive security patches
  - running CI/CD, so every runner uses the same up-to-date base
  - running scheduled rebuilds
- **Cost:** needs registry access (network, Docker Hub rate limits).

## Comparison
| | `--no-cache` | `--pull` |
|---|---|---|
| Disables layer cache | ✅ | ❌ |
| Fetches newest base image | ❌ | ✅ |
| Speed | slow | fast (if base unchanged) |
| Main goal | fresh build steps | fresh base image |

## Both together, for a fully fresh build
```bash
docker build --pull --no-cache -t myapp:1.0 .
```
Use for releases, nightly builds and CVE rebuilds.

## Finer-grained alternatives
- `--no-cache-filter <stage>`: skip the cache for a specific stage only (BuildKit).
- Cache-bust with `ARG`:
  ```dockerfile
  ARG CACHEBUST=1
  RUN git clone https://github.com/x/y.git
  ```
  ```bash
  docker build --build-arg CACHEBUST=$(date +%s) .
  ```
- Code changes don't need `--no-cache`, because `COPY` detects them via checksums.

## Key takeaway
> `--no-cache` means **fresh build steps**. `--pull` means **fresh base image**.
> Only `--no-cache` actually disables the cache.

---


برداشت من از «داک انگ» این بود که مستند `docker history` را به انگلیسی برای جزوه‌ات می‌خواهی، مثل نکته‌های قبلی. اگر منظورت چیز دیگری بود بگو.

---


#  `docker history`

## Purpose
Shows the **build history of an image**: each step that created it, the instruction used, and the size it added.

## Syntax
```bash
docker history [OPTIONS] IMAGE
docker image history [OPTIONS] IMAGE   # same command
```

## Example
```bash
docker history nginx:latest
```
```
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
a8758716bb6a   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   RUN /bin/sh -c set -x && apt-get update ...     112MB     buildkit.dockerfile.v0
<missing>      3 weeks ago   /bin/sh -c #(nop) ADD file:... in /             74.8MB
```
- The order is **newest first**: the top row is the last Dockerfile step.

## Output Columns
| Column | Meaning |
|---|---|
| `IMAGE` | Image ID of that step. `<missing>` if it wasn't built locally |
| `CREATED` | When the step was built |
| `CREATED BY` | The instruction / command that created the step |
| `SIZE` | Size added by this step (uncompressed) |
| `COMMENT` | Builder info (e.g. `buildkit.dockerfile.v0`) or a `docker commit -m` message |


## Key Points / Gotchas
- `0B` means a metadata step, not a filesystem layer.
- `<missing>` means the step was built on another machine (pulled image). It is **normal, not an error**.
- **Multi-stage builds:** only the **final stage** appears. Earlier stages are not in the history.
- History is **metadata written by the builder**. It can be incomplete or edited, so don't treat it as proof.
- ⚠️ **Secrets leak:** `ARG`/`ENV` values and commands containing tokens are visible in `CREATED BY`. Use `RUN --mount=type=secret` instead.
- Deleting a file in a later layer does **not** shrink the image. The earlier layer still contains it, and history shows that size.

## Use Cases
1. **Understand how an image was built**, roughly reconstructing a Dockerfile without the source.
   ```bash
   docker history --no-trunc --format "{{.CreatedBy}}" myimg
   ```
2. **Optimize image size:** find the biggest layers.
   ```bash
   docker history --format "{{.Size}}\t{{.CreatedBy}}" myimg | sort -h -r | head
   ```
3. **Security audit:** check for leaked passwords, tokens, or suspicious `curl | sh` commands.
   ```bash
   docker history --no-trunc myimg | grep -iE 'pass|token|secret|key'
   ```
4. **Check the base image and build tool** (BuildKit vs legacy, or `docker commit`).
5. **Compare two versions** of an image to see which steps changed.

## Summary
> `docker history` = **"how was this image built, and which step made it big?"**
> Local, metadata-based. Watch out for `0B`, `<missing>` and leaked secrets.

---

note: every line in Dockerfile will be translated to a layer in docker image

---

# Shell form vs Exec form

```dockerfile
RUN apt-get install figlet              # shell form
RUN ["apt-get", "install", "figlet"]    # exec form (JSON array)
```

| | Shell form | Exec form |
|---|---|---|
| Actually runs | `/bin/sh -c "apt-get install figlet"` | `execve("apt-get", [...])`, no shell |
| `docker history` | `/bin/sh -c apt-get install figlet` | `apt-get install figlet` |
| `$VAR` expansion | ✅ | ❌ |
| `&&`, `\|`, `>`, `*` | ✅ | ❌ |
| Needs `/bin/sh` in image | ✅ yes | ❌ no (works on scratch/distroless) |
| Syntax | plain string | valid JSON, **double quotes only** |

- The shell prefix comes from the `SHELL` instruction (default `["/bin/sh","-c"]`).
- Need shell features in exec form? Call the shell explicitly: `RUN ["sh", "-c", "echo $HOME"]`.
- **Best practice:**
  - `RUN` uses **shell form** (chaining, variables, cleanup).
  - `CMD` / `ENTRYPOINT` use **exec form**, so the app is PID 1 and receives signals (graceful `docker stop`).
- `history` shows exactly what was executed, which is why the `/bin/sh -c` prefix appears only for shell form.

---

