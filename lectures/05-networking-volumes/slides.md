---
theme: seriph
title: "INF345 — Lecture 5: Container Networking & Volumes"
info: |
  INF345 — Fundamentals of DevOps
  Lecture 5 of 15
background: /cover-bg.svg
transition: fade
mdc: true
download: true
---

# INF 345 — Fundamentals of DevOps

## Lecture 5: Container Networking & Volumes

<div class="pt-8 opacity-70">
Adil Akhmetov · Lesson 5
</div>

---
layout: default
---

# Recap — Lesson 4 (Images, Layers & Multi-Stage Builds)

<v-clicks>

- Why does a multi-stage build produce a smaller final image? <span v-click class="opacity-60">(only the last stage ships; earlier stages — compilers included — are thrown away except for what you `COPY --from=`)</span>
- What does `podman history` show you? <span v-click class="opacity-60">(every layer, its size, and the instruction that created it)</span>
- Why copy `requirements.txt` and install it before copying the rest of the app? <span v-click class="opacity-60">(keeps that layer's cache valid when only app code changes)</span>

</v-clicks>

---
---

# Today's agenda

<v-clicks>

- [ ] Why containers need networking, and network namespaces
- [ ] Publishing ports: `-p`, `EXPOSE`, and where to bind
- [ ] Default bridge vs user-defined networks, and DNS by name
- [ ] Podman specifics: rootless networking, pods
- [ ] Volumes: named, bind mounts, tmpfs — and where your data really lives
- [ ] Compose: wiring it all together
- [ ] Live demo → straight into today's practice

</v-clicks>

---
layout: center
class: text-center
---

# The scenario

<div class="text-lg text-left mt-4 max-w-2xl mx-auto">

Your week 4 app runs great as a single container. Now it needs a database.
You start a second container, hardcode its IP in the app's config, and it
works — until you restart the database container and get a different IP.
Or you publish the database's port to the host "just to debug it" and
forget to remove that flag.

</div>

<div v-click class="mt-8 text-xl font-bold">
Today is about the two things that make that scenario safe: how
containers find each other, and where their data actually lives.
</div>

---
layout: section
transition: slide-left
---

# Block 1
## Networking fundamentals

---
---

# Isolation by default

<v-clicks>

- Every container gets its own **network namespace** by default — its own
  loopback, its own interfaces, its own routing table.
- Two containers can't see each other's ports or processes unless you
  explicitly connect them.
- Same namespace mechanism from lesson 3 (containers 101) — just applied
  to networking instead of the filesystem.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
A container's <code>localhost</code> is its own — it is <b>not</b> the
host's localhost, and it is not another container's localhost either
(with one exception we'll get to: pods).
</div>

---
---

# Publishing a port

```bash
podman run -d -p 8080:80 nginx
# host:container — host port 8080 forwards to container port 80
```

<v-clicks>

- Without `-p`, the container's port exists only inside its own network
  namespace — nothing on the host can reach it directly.
- `EXPOSE 80` in a Containerfile is **documentation** — it records intent,
  it does not publish anything by itself.
- `-p host:container` at run time is what actually punches the hole.
- (docker: identical flag — `docker run -p 8080:80 ...`)

</v-clicks>

---
---

# Binding to 127.0.0.1 vs 0.0.0.0

```bash
podman run -d -p 127.0.0.1:8080:80 nginx   # host-only
podman run -d -p 8080:80 nginx             # binds 0.0.0.0
```

<v-clicks>

- `-p 127.0.0.1:8080:80` binds only the host's loopback — reachable only
  from this machine.
- `-p 8080:80` (no address) binds `0.0.0.0` — reachable from any network
  this host is on.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-red-500/10 text-sm">
On a shared or cloud host, <code>-p 8080:80</code> is reachable from the
network unless a firewall stops it. Bind to <code>127.0.0.1:</code> for
anything that should only be reached through a reverse proxy on the same
machine.
</div>

---
---

# Default bridge vs user-defined networks

| | Default bridge | User-defined network |
|---|---|---|
| Created by | Automatically | You: `podman network create` |
| DNS by container name | **No** | **Yes** |
| Reach other containers by | IP address only | Container name |
| Isolation from other networks | One flat shared network | Only attached containers can talk |

<div v-click class="mt-6 text-sm opacity-70">
This is the single most common "my app can't reach my database" bug: two
containers on the default bridge, app trying to connect to <code>db</code>
by name — and it fails, because only user-defined networks get built-in
DNS.
</div>

---
---

# `podman network create/ls/inspect`

```bash{1|2|3|4-6|7}
podman network create app-net
podman network ls
podman network inspect app-net
podman run -d --name db --network app-net postgres:16
podman run -d --name api --network app-net myapp
# from inside "api":
#   curl http://db:5432   →  resolves "db" by name — DNS on app-net
```

<div v-click class="mt-4 text-sm opacity-70">
(docker: identical commands — <code>docker network create</code>,
<code>docker network ls</code>, <code>docker network inspect</code>)
</div>

---
---

# Container-to-container communication

<v-clicks>

- On a shared user-defined network, containers reach each other by name
  over their **container** ports — no `-p` needed at all.
- `-p` is for the outside world (your browser, a client, a script on the
  host). It is not what lets two containers talk to each other.
- Rule: only publish the ports that something *outside* the container
  network actually needs to reach.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-red-500/10 text-sm">
<b>Never publish a database port.</b> If your app and your database are on
the same user-defined network, the app reaches the database by name on
its container port — publishing 5432 to the host just gives every scanner
on the internet a free shot at your data.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
Your app container can <code>curl http://db:5432</code> successfully with
no <code>-p</code> flag anywhere. Why does that work?
</div>

<div v-click class="mt-8 text-lg opacity-70">
Both containers are on the same user-defined network — its built-in DNS
resolves <code>db</code> to the container's address, and
container-to-container traffic doesn't need a host port published at all.
</div>

---
layout: section
transition: slide-left
---

# Block 2
## Podman specifics: rootless networking & pods

---
---

# Rootless networking: netavark & pasta

<v-clicks>

- Podman runs containers as your normal user by default — no root daemon.
  That changes how networking has to work under the hood.
- **Netavark** is Podman's network stack for container-to-container and
  bridge networking (the default since Podman 4).
- **Pasta** handles a rootless container's connection out to the
  host/internet, without needing root privileges or special capabilities.
- You won't configure these directly in this course — just recognize the
  names if you see them in `podman network inspect` or troubleshooting
  docs.

</v-clicks>

---
---

# Pods: containers that share a network namespace

<v-clicks>

- A **pod** in Podman groups containers so they share **one** network
  namespace — the same idea as a Kubernetes pod, running locally.
- Containers in the same pod reach each other over `localhost` — no
  network, no DNS needed.
- Useful for a tightly-coupled sidecar next to your app — not a
  replacement for a user-defined network between independent services.

</v-clicks>

```bash
podman pod create --name mypod -p 8080:80
podman run -d --pod mypod --name web nginx
podman run -d --pod mypod --name sidecar myapp
# inside "sidecar": curl http://localhost:80  →  reaches "web"
```

---
layout: section
transition: slide-left
---

# Block 3
## Storage & volumes

---
---

# The writable layer is ephemeral

<v-clicks>

- Every running container gets a thin **writable layer** on top of its
  image's read-only layers (lesson 4).
- Anything written there — logs, uploaded files, a SQLite file —
  disappears the moment `podman rm` removes that container.
  `stop`/`start` keeps it; `rm` does not.
- Fine for stateless apps. A data-loss bug waiting to happen for anything
  that needs to survive a restart or a redeploy.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-blue-500/10 text-sm">
If data needs to outlive the container, it can't live in the writable
layer. That's what volumes are for.
</div>

---
---

# Three ways to attach storage

| | Named volume | Bind mount | tmpfs |
|---|---|---|---|
| Managed by | Podman/Docker | You (host path) | Podman/Docker, in memory |
| Survives container removal | Yes | Yes (it's a host directory) | No |
| Good for | App data, databases | Config, source during dev | Secrets, caches |
| Example | `-v mydata:/var/lib/db` | `-v ./src:/app/src:Z` | `--tmpfs /run` |

<div v-click class="mt-6 text-sm opacity-70">
Default choice for anything you'd call "the database's data": a named
volume. Podman/Docker manages where it actually lives on disk — you don't
need to know or care.
</div>

---
---

# `podman volume create/ls/inspect`

```bash{1|2|3|4}
podman volume create mydata
podman volume ls
podman volume inspect mydata
podman run -d --name db -v mydata:/var/lib/postgresql/data postgres:16
```

<div v-click class="mt-4 text-sm opacity-70">
(docker: identical — <code>docker volume create</code>,
<code>docker volume ls</code>, <code>docker volume inspect</code>)
</div>

<div v-click class="mt-4 text-lg font-bold">
A named volume outlives any single container. Remove and recreate the
"db" container with the same <code>-v mydata:...</code>, and the data is
still there.
</div>

---
---

# `:Z` and `:z` — SELinux labels on bind mounts

<v-clicks>

- On SELinux-enforcing hosts (Fedora, RHEL — what DO188 runs on), a
  container can't read/write a bind-mounted host directory until it's
  relabeled for container access.
- `:z` — **shared** label: multiple containers can use this mount.
- `:Z` — **private** label: only this container can use it. Prefer `:Z`
  unless you specifically need to share the mount.

</v-clicks>

```bash
podman run -d -v ./src:/app/src:Z myapp
```

<div v-click class="mt-4 p-4 rounded bg-red-500/10 text-sm">
Forgetting <code>:Z</code>/<code>:z</code> on an SELinux host is the most
common cause of a mysterious "permission denied" on a bind mount that a
plain <code>ls</code> shows you clearly have access to.
</div>

---
---

# Persistence across container removal

<v-clicks>

- A named volume or bind mount's data exists independently of the
  container using it — `podman rm` the container, the data stays.
- An **anonymous volume** (created implicitly, e.g. by an image's
  `VOLUME` instruction with no `-v` naming it) also survives `rm` by
  default — but nothing refers to it by name, so it quietly accumulates
  until you `podman volume prune` it.
- Bottom line: know whether your data is in a *named* volume before you
  tear a container down.

</v-clicks>

---
---

# `down` vs `down -v` — the dangerous flag

<v-clicks>

- `podman-compose down` (or `docker compose down`) stops and removes
  containers and the default network — named volumes are left alone.
- `podman-compose down -v` additionally **deletes every named volume**
  the compose file defines. That's your database's data, gone.
- There is no confirmation prompt.

</v-clicks>

<div v-click class="mt-8 p-4 rounded bg-red-500/10 text-lg font-bold">
`-v` on `down` is the single most common way to accidentally delete a
week's worth of test data. Read the flag before you type it.
</div>

---
layout: center
class: text-center
---

# Quick check

<div class="text-xl mt-4 max-w-2xl mx-auto text-left">
You stop and remove a container with <code>podman rm</code>. Its data was
in a named volume. Is the data gone?
</div>

<div v-click class="mt-8 text-lg opacity-70">
No — a named volume exists independently of any container. The data is
still there; nothing currently references it until you start a new
container with the same <code>-v</code>.
</div>

---
layout: section
transition: slide-left
---

# Block 4
## Wiring it together with Compose

---
---

# `compose.yaml` — services, networks, volumes

<v-clicks>

- `services:` — one entry per container: image, ports, environment,
  volumes — same options as `podman run`.
- `networks:` — user-defined networks Compose creates for you; every
  service on the same network gets DNS by name automatically.
- `volumes:` — named volumes declared once, referenced by any service.
- `depends_on:` — controls start order (not "wait until ready" — just
  "start after").
- `environment:` — env vars, same as `-e` on `podman run`.

</v-clicks>

<div v-click class="mt-6 text-sm opacity-70">
<code>podman-compose</code> reads this same <code>compose.yaml</code>;
<code>docker compose</code> is the Docker equivalent. Both use the same
file format.
</div>

---
---

# Worked example — app + Redis

```yaml
services:
  app:
    build: ./app
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis
    networks:
      - app-net

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    networks:
      - app-net
    # no "ports:" here — redis is never published to the host

networks:
  app-net:

volumes:
  redis-data:
```

---
---

# Why this shape matters

<v-clicks>

- `redis` isn't published — nothing outside this compose project can
  reach it directly. The "don't publish a database port" rule from Block
  1, applied to Compose.
- `app` reaches `redis` at `redis:6379` — by name, over `app-net` — the
  same user-defined-network DNS from Block 1.
- `redis-data` is a named volume — `podman-compose down` (no `-v`) leaves
  the data intact across restarts.
- `depends_on` starts Redis's *container* before the app's — it does not
  wait for Redis to finish booting. A real app should still retry its
  connection on startup.

</v-clicks>

<div v-click class="mt-6 p-4 rounded bg-blue-500/10 text-sm">
This exact shape — app + Redis, Redis internal-only, named volume — is
today's practice.
</div>

---
layout: section
transition: slide-left
---

# Block 5
## Debugging checklist

---
---

# When it doesn't work

| Symptom | Likely cause |
|---|---|
| Can't reach another container by name | Both on the default bridge, not a user-defined network |
| `permission denied` on a bind mount | Missing `:Z`/`:z` on SELinux, or a UID mismatch |
| Data gone after a rebuild | It was in the writable layer or an anonymous volume, not a named volume |
| Data gone after `down` | Someone ran `down -v` |

<div v-click class="mt-6 text-sm opacity-70">
Networking-shaped bug → check <code>podman network inspect &lt;name&gt;</code>
for who's actually attached. Storage-shaped bug → check
<code>podman volume inspect &lt;name&gt;</code> for where the data
actually lives.
</div>

---
layout: section
transition: slide-left
---

# Block 6
## Today's practice

---
---

# Today's practice — wire app + Redis with Compose

<div class="grid grid-cols-2 gap-6 text-sm">
<div>

Practice 05 gives you `app/` (a small counter API) and a starter
`compose.yaml`. You fill in:

1. A user-defined network shared by both services
2. Redis on a named volume, **not** published to the host
3. `depends_on` so the app starts after Redis
4. An environment variable pointing the app at `redis` by name

</div>
<div>

```bash
podman-compose up -d
curl localhost:8080/count
curl localhost:8080/count
podman-compose down
podman-compose up -d
curl localhost:8080/count
# → count kept increasing — the restart didn't reset it
```

<div v-click class="mt-4 opacity-70">
Graded automatically: Redis not published, named volume used, counter
survives a <code>down</code> + <code>up -d</code> cycle (not
<code>down -v</code> — that's a different result).
</div>

</div>
</div>

---
---

# By the end of this lesson, you should be able to

<v-clicks>

- [ ] Explain why two containers can't reach each other by name on the
      default bridge network
- [ ] Publish a port correctly, and explain when to bind to `127.0.0.1`
      instead of `0.0.0.0`
- [ ] Choose a named volume, bind mount, or tmpfs for a given storage need
- [ ] Write a `compose.yaml` that keeps a database off the host network
      and its data in a named volume

</v-clicks>

---
layout: default
---

# Before next lecture

- [ ] Finish today's practice if you didn't wrap it up in session — same
      Maru submission flow as last time
- [ ] Keep the RHA DO188 lab moving — due next week (Week 6)

<div class="mt-8 text-sm opacity-60">
Week 6 is lab work only — this is your last lecture before that lab is
due.
</div>

---
layout: end
---

# Next lecture

Container security & module wrap-up — locking down what we've built, and
closing out the containers module.
