# Practice 03 — Write a Containerfile

**Objective:** write a Containerfile from scratch for a small app — base
image, dependency install, non-root user, in the right order — the skill
from today's lecture.

**Timebox:** ~30 min of actual work within today's session.

> **How to submit:** this practice runs on **Maru**, not the fork+PR flow
> from earlier practices. Go to the Maru link posted in the course
> channel, sign in with your invited Google account, link your GitHub
> account (once, the first time), then click **Accept** on "Practice 03."
> Maru creates your own **private** repo under `weeebdev-edu` and invites
> you as a collaborator — accept the GitHub invite (check your email, or
> your GitHub notifications), clone it, and work there. Nobody else can
> see your repo. There's no PR: just commit and push to `main` —
> GitHub Actions grades every push automatically, in about a minute.

## Task

Your repo has one folder: `app/` — a tiny Flask app (`app.py`) and its
`requirements.txt`. You did not write this app and don't need to touch
it. Your job is to containerize it.

Add a file named exactly `Containerfile` at the **root** of the repo
(not inside `app/`) that:

1. Starts `FROM` a slim Python base image (e.g. `python:3.12-slim`).
2. Creates a non-root user.
3. Sets a working directory.
4. Copies `app/requirements.txt` in and installs it with `pip install`
   — **before** copying the rest of the app code (this is the
   layer-caching order from today's lecture).
5. Copies the rest of `app/` in.
6. `EXPOSE`s port `8080`.
7. Switches to the non-root user with `USER` **before** the final `CMD`.
8. `CMD`s the app so it listens on `0.0.0.0:8080`.

This is the exact shape of the Containerfile from today's slides — same
five ideas, applied to this app.

Test it locally before you push:

```bash
podman build -t practice03 .   # or: docker build -t practice03 .
podman run --rm -p 8080:8080 practice03
curl localhost:8080            # should print: INF345 Practice 03 OK
```

## Definition of done

- [ ] `Containerfile` exists at the repo root
- [ ] It builds successfully
- [ ] The container runs and responds on port `8080`
- [ ] It does **not** run as root
- [ ] `requirements.txt` is copied and installed before the rest of the
      app code
- [ ] Pushed to `main` before the session ends (this is also your
      attendance signal)

## Grading

Real autograding this time — GitHub Actions actually builds and runs
your container, out of 100 points total:

| Check | Points |
|---|---|
| Containerfile builds | 25 |
| Container runs and responds correctly on `:8080` | 30 |
| Runs as a non-root user | 25 |
| `requirements.txt` copied+installed before the rest of the app | 20 |

Check the **Actions** tab in your repo for the run, or the workflow
run's **Summary** for the exact score. Workflow:
`.github/workflows/classroom.yml` (in your repo, not this one).

This score is this session's grade within the **Weekly practice
sessions** category (10% of the final course grade, split evenly across
all ~14 practice sessions — see `SYLLABUS.md`), not 10% on its own.
