# Contributing — how six people ship this without colliding

## 1. Branching

- `main` — always deployable, always demo-able
- `feat/<track>-<thing>` — e.g. `feat/frontend-cascade-overlay`
- `fix/<thing>`

Rules:

- **No direct pushes to `main`.**
- One reviewer, and the reviewer must be from a **different track** — that is how contract drift gets caught.
- Squash merge. Keep history readable.

## 2. The contract rule

`docs/PROTOCOL.md` is frozen. A change to it requires:

1. Team lead sign-off
2. A message in the group naming exactly which fields changed
3. Updated test vectors in the same PR

Everything else in this repo can be rewritten freely. This one file cannot, because a silent change here costs four people a day each and you will only find out at integration.

## 3. Definition of done

A task is not done until:

- [ ] Tests pass, including the shared vectors where applicable
- [ ] It runs on someone else's machine from a clean clone
- [ ] Its track README's task list is ticked
- [ ] Anything that surprised you is written into that README's Gotchas section

That last one is the difference between a team that learns and a team that repeats mistakes across three tracks.

## 4. Shared code

Two things exist in more than one place and must never diverge:

| Artefact | Lives in | Used by |
|----------|----------|---------|
| `decide()` | `backend/app/arbiter.py` (Python), `gateway/src/arbiter.c` (C) | backend, gateway, firmware |
| Test vectors | `docs/vectors/` | all of the above |

CI runs both implementations against every vector. If they disagree, the build fails. Do not add a third, convenient, slightly-different copy.

## 5. Weekly rhythm

| Day | What |
|-----|------|
| Mon | 20-minute standup, each track states its blocker |
| Wed | Integration check — does `main` still demo end to end? |
| Fri | Demo whatever moved, to each other, out loud |

The Friday run-through is not ceremony. Explaining your track to a teammate is how you find out your explanation does not hold up, which is exactly what happens in front of judges.

## 6. Demo-day checklist

Print this.

- [ ] Demo URL loads on phone hotspot, not just campus wifi
- [ ] Backend warmed (hit `/healthz` 10 minutes before)
- [ ] 90-second screen recording on a local drive as fallback
- [ ] Scaled hardware rig packed: pole mock, conductor, cut mechanism, gateway, three nodes, relay, spare batteries, spare everything
- [ ] Field-test video on the laptop, not in the cloud
- [ ] Characterisation plots printed as a handout
- [ ] BOM with per-unit and 1,000-unit cost
- [ ] Laptop charged, HDMI adapter, extension cord

## 7. Questions the panel will ask

Assign an owner to each. Rehearse the answers.

| Question | Owner |
|----------|-------|
| "What stops it tripping in heavy rain?" | TBD — answer with the characterisation data and a live scenario button |
| "What if the radio fails?" | TBD — fail-safe rule 1, comms loss never trips |
| "Why not just use a smart meter?" | TBD — cost per point, no outage to install, works on unmetered spans |
| "Who is liable if it trips wrongly?" | TBD — default ALERT_ONLY, utility opts into auto-isolation per feeder |
| "What does one node cost at scale?" | TBD — BOM.csv |
| "Has any utility actually seen this?" | TBD — be honest: none has. Self-proposed under Open Innovation; describe the pilot you would propose |

The last one is a trap for teams that overclaim. Answer it straight.
