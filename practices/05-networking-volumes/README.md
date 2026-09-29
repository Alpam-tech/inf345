# Practice 05 — Container Networking & Volumes

**Objective:** wire two containers together with Compose — a
user-defined network so they can reach each other by service name, and
a named volume so data survives a restart — the skill from today's
lecture.

**Timebox:** ~30 min of actual work within today's session.

> **How to submit:** this practice runs on **Maru**, not the fork+PR flow
> from earlier practices. Go to the Maru link posted in the course
> channel, sign in with your invited Google account, link your GitHub
> account (once, the first time), then click **Accept** on "Practice 05."
> Maru creates your own **private** repo under `weeebdev-edu` and invites
> you as a collaborator — accept the GitHub invite (check your email, or
> your GitHub notifications), clone it, and work there. Nobody else can
> see your repo. There's no PR: just commit and push to `main` —
> GitHub Actions grades every push automatically, in about a minute.

## Task

Your repo has two things already done, which you do not need to touch:

- `app/` — a tiny Go HTTP server that serves a visit counter it keeps in
  Redis (`GET /`, reading `REDIS_ADDR`, default `redis:6379`) and a
  liveness probe (`GET /healthz`).
- `app/Containerfile` — already builds and runs correctly (last week's
  skill).

Your job is `compose.yaml` at the repo root, which already has a stub
with `TODO` comments. Fill it in so that:

1. There are two services: `app` (built from `./app`) and `redis`
   (`image: redis:7-alpine` — keep the tag pinned).
2. Both services join a **user-defined network** you declare (e.g.
   `backend`). `app` must reach `redis` **by service name** — the
   hostname `redis`, not `localhost`, not an IP, not left to the default
   network.
3. Only `app` publishes a port to the host (`8080:8080`). `redis`
   publishes **nothing** — nothing outside the compose stack should be
   able to reach it directly.
4. Redis's data lives on a **named volume** mounted at `/data`, and
   persistence is turned on (`redis-server --appendonly yes`) — so the
   counter survives `docker compose down` (without `-v`) followed by
   `docker compose up` again. A bind mount to a host folder does not
   count — it has to be a named volume declared under `volumes:`.
5. `app` declares `depends_on: redis`.

Test it locally before you push:

```bash
docker compose config                # validate + see the resolved config
docker compose up -d --build         # or: podman-compose up -d --build
curl localhost:8080                  # -> INF345 Practice 05 OK visits=1
curl localhost:8080                  # -> visits=2

docker compose down                  # stops + removes containers, KEEPS volumes
docker compose up -d                 # counter should keep counting, not reset
curl localhost:8080                  # -> visits=3

docker compose down -v               # NOW the volume is gone — counter resets
```

## Definition of done

- [ ] `compose.yaml` exists at the repo root and `docker compose config`
      validates it
- [ ] `docker compose up -d --build` brings up both containers and
      `GET /healthz` responds on `:8080`
- [ ] Two requests to `GET /` show the counter increasing
- [ ] `docker compose down` (no `-v`) then `docker compose up` again —
      the counter keeps counting instead of resetting to 1
- [ ] `redis` has no `ports:` — it isn't reachable from the host
- [ ] A user-defined network is declared and both `app` and `redis` are
      attached to it
- [ ] Redis's `/data` is a named volume, not a bind mount, not anonymous
- [ ] Pushed to `main` before the session ends (this is also your
      attendance signal)

## Common mistakes

- **App can't reach `redis` by name.** This almost always means one of
  the two services isn't actually on your user-defined network — check
  both services list the same network under `networks:`, and that you
  didn't leave one of them to join only the (implicit) default network.
- **Counter resets after every restart.** Two separate things have to
  both be true: persistence has to be turned on
  (`redis-server --appendonly yes`), *and* `/data` has to be a named
  volume (declared under the top-level `volumes:` and referenced by
  name) rather than an anonymous one. Either alone isn't enough.
- **Data really is gone.** If you ran `docker compose down -v` at any
  point, the volume is deleted along with the containers — that's
  expected, not a bug. Only plain `docker compose down` (no `-v`) is
  supposed to preserve it.
- **Publishing redis "just to check it works."** Don't add a `ports:`
  entry under `redis` to poke it with `redis-cli` from your host — it
  costs you points, and it isn't necessary: `docker compose exec redis
  redis-cli` gets you a shell inside the network without publishing
  anything.

## Grading

Real autograding, same shape as Practice 04 — GitHub Actions actually
brings the stack up, hits it over HTTP, and restarts it, out of 100
points total:

| Check | Points |
|---|---|
| `compose.yaml` exists and `docker compose config` validates | 10 |
| Stack comes up and `GET /healthz` responds | 10 |
| Counter increments across two requests | 10 |
| Counter survives `docker compose down` (no `-v`) + `up` | 25 |
| `redis` publishes no host port | 15 |
| User-defined network declared, both services attached | 20 |
| Named volume (not bind, not anonymous) mounted at `/data` for redis | 10 |

Check the **Actions** tab in your repo for the run, or the workflow
run's **Summary** for the exact score. Workflow:
`.github/workflows/classroom.yml` (in your repo, not this one).

This score is this session's grade within the **Weekly practice
sessions** category (10% of the final course grade, split evenly across
all ~14 practice sessions — see `SYLLABUS.md`), not 10% on its own.
