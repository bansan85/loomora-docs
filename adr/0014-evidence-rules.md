# ADR 0014: Evidence is stored only when it proves something new, sorted by change and platform, and images need the owner's authorization

- **Status**: accepted
- **Date**: 2026-11-14 (the first two rules date from 2026-11-13, the third from 2026-11-14)
- **Spike / requirement**: FR-016, FR-017, FR-018

## Context

`docs/implementation/validation/evidence/` grew with every trial. The first folders are named for a trial or a topic (`first-trial`, `linux-legacy-xlib`, `xlib-unicode-manylinux`), so they do not say which decision a piece supports, and the same probe lines, vectors and screenshots were stored again after changes that did not alter the behaviour. The owner set the rules below to keep the documentation repository small and reviewable.

## Decision

- **Sorted**: each piece of evidence goes in `validation/evidence/<adr-or-change>/<platform-id>/`, the subfolder named for the ADR or the change it belongs to, then the platform.
- **Only new proof**: evidence is stored only when it proves something new (a new feature or behaviour). Documents already validated are not stored again: the tests (`run_vectors.py`, the selftests, Catch2) are the proof that behaviour is unchanged. A re-run is mentioned in the verdict notes with its result, not stored again.
- **Never edited**: stored evidence is not modified. A later ADR supersedes an older one in text (ADR 0007 supersedes the manylinux2010 evidence) and the old files stay as they were.
- **Images**: screenshots and other images are not evidence by default. Before an image is added to `docs/implementation` the owner authorizes it. An agent does not commit images; it lists the candidates in its report (scenario, why it proves something new, file name) and they are committed only after the owner approves. At most one screenshot per distinct rendering scenario, taken only by `verify.py full` (ADR 0013), never to check that something works.

## Rejected alternatives

- **Store every run**: cost in size and review time, and the new proof is lost among the old.
- **Evidence by trial or topic name** (the first folders): it does not say which decision it supports.
- **Screenshots as the default proof**: the probe line and the vectors prove the behaviour exactly; an image proves only rendering, and only a rendering change needs one.

## Consequences

- The evidence of the first trials (`first-trial`, `xlib-unicode-manylinux`, `linux-legacy-xlib`) keeps its old names; the new rule applies from 2026-11-13.
- A verdict states what was re-run and what the result was; it does not link a copy.
- Reviewing a change means reading the one folder of that change.
