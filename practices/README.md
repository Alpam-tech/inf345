# Practice sessions — design

Every lesson has a **1-hour in-class practice session** in addition to the
lecture (see `SYLLABUS.md` — 10% of the final grade, separate from the 5%
attendance line). Practice sessions are graded differently from labs:

| | Labs | Practice sessions |
|---|---|---|
| When | Take-home, days to finish | In-class, ~1 hour, synchronous |
| Where | RHA cloud labs (01/02) or a fork of this repo (03/Capstone) | Fork of this repo + pull request |
| Grading | RHA completion or a full build/run autograder | A fast, lightweight autograder — checks git history and file content, not builds |
| Purpose | Depth on one module's tool | Reinforce that lesson's specific skill, immediately |

## Why the autograder is deliberately lightweight

A practice session's autograder has to return a result **while the student
is still in the room** — a 5-minute Docker build is useless feedback if
class ends in 10 minutes. Every practice autograder in this repo:

- Never builds a container image or spins up infrastructure.
- Only checks git history (commit count, file diffs) and final file
  content — this runs in seconds, not minutes.
- Uses `pull_request` triggers with `concurrency: cancel-in-progress:
  true` (same minute-budget rules as labs — see `labs/README.md`).

## Attendance signal

A practice session's pull request, opened within the class window,
doubles as the attendance record for that lesson — a student who didn't
open a PR didn't attend, from a grading standpoint. This is a *signal*,
not strict enforcement: if a student has a real reason for a late
submission, that's a normal instructor judgment call, not something the
autograder polices.

## `_template/`

Starting point for a new practice session. Copy it, don't start from
scratch.

## Practice 03 onward: Maru, not fork+PR

Starting with Practice 03, submissions move to **Maru** (a private-repo
platform built for this course) instead of the fork+PR flow above. Each
student gets their own private repo per practice, created from a
template on Accept — no PR, no "lightweight autograder" constraint,
since there's no shared public fork and no time pressure to grade
*during* the session (the repo is the student's own, async). Practices
built on Maru can do real builds — see `practices/03-containers-101/`
and its template repo's `.github/workflows/classroom.yml` for the
pattern. Practice 02 stays on the old fork+PR flow; it isn't being
migrated.
