# Snap robustness: which inference wins, and how long it stays

> Status: **OPEN** - nothing in this document is implemented yet. It is the plan
> for the *behaviour* work, and it is deliberately a separate document from
> `docs/snap-pipeline-plan.md`, which records the *structural* refactor now in
> review. That refactor is this plan's prerequisite: every rule of the snap
> search lives in `core/inference.py::InferenceEngine` behind one call site, so
> the policy changes below have a single place to live and a single place to
> test.
>
> Written against issue #368, from a reported case (below), not from theory.
>
> Line numbers are hints and will drift; every claim is anchored to a function
> or attribute name. Trust the names, re-grep the numbers.

## The reported case

A plane is drawn and push/pulled into a parallelepiped whose edges are square
to the axes. The user selects the top-right edge and grabs the endpoint nearest
the camera, to drag that edge leftwards along X.

**Expected**: X engages immediately, and no other
inference appears for as long as the drag stays on X. Another inference shows
up only if the user deliberately goes looking for it.

**Observed**: inference points light up "all over the place" as the cursor
moves, and the axis competes with them instead of silencing them.

Three symptoms, kept separate because they may not share one cause:

- **S1 - engagement.** The axis does not take hold at the start of the drag,
  or does not hold once taken.
- **S2 - authority.** While the axis is engaged, other cues are still drawn.
- **S3 - stability.** Cues appear and disappear as the cursor moves, so the cue
  the user aims at is not the cue that fires.

## What the code does today

Four facts read from the code, not inferred from behaviour:

1. **Candidates are ranked by screen distance, in pixel buckets.**
   `compute_snap` (`core/snap.py:1173`) hands each candidate to the nested
   `_consider` (`core/snap.py:1337`), which measures its distance from the
   cursor in pixels and drops anything beyond `threshold_px`. The nested
   `_resolve` (`core/snap.py:1352`) sorts by `floor(d / _TIE_PX)` and takes the
   first candidate that is not occluded. `_TIE_PX` is thus already a
   retention-like idea: inside one bucket, sub-pixel distance stops mattering
   and the order in the list decides.
2. **Within a bucket, append order decides, plus one contextual rule.**
   `_resolve` sorts so that a candidate with no `context` comes first - "a
   derived point never beats the point it derives from" (the 0.4.4 rule, from
   issue #36: a loose vertex beats a component's copy at the same spot). The
   rest is why `compute_snap` considers the named points (`center`,
   `component_origin`, `arc_midpoint`, `core/snap.py:1392`-`1411`) before the
   plain endpoints.
3. **An engaged axis already outranks every discrete point.** The axis/lock
   branch returns a `SnapResult` before the discrete-point ranking is ever
   reached (`core/snap.py:1324`, `return SnapResult(locked, "axis", ...)`, and
   the crossing case above it). So S2 is not, in itself, a missing priority:
   when that branch fires, nothing else is considered at all.
4. **`SnapResult` carries no priority field** (`core/snap.py:60`): `kind`,
   `color`, `axis`, `guide`, `guide_color`, `guides`, `context`, `label`.
   Priority is implicit in the order of consideration and in the bucket sort,
   not stored as a number.

A dwell mechanism already exists in the engine: `InferenceEngine._dwell_on`
(`core/inference.py:260`), `_encourage_dwelt` (`:273`) and `encourage_point`
(`:277`). Its exit conditions (`Viewport.reset()`, geometry deleted) were part
of the refactor's contract, and it is the natural candidate for "the point the
user deliberately returned to".

## Diagnosis (leading hypothesis, to be confirmed first)

The axis has authority (fact 3). What it does not obviously have is
**engagement**: if the axis/lock branch's entry condition does not fire during
a drag along X, then `compute_snap` falls through to the distance ranking - and
the distance ranking is precisely what produces the cloud, because every
vertex, midpoint and origin within `threshold_px` of the cursor is a legitimate
candidate and the nearest one keeps changing as the cursor moves. S1 would then
be feeding S2 and S3.

The first thing to establish is therefore **why the axis branch does not fire
in the reported case**: whether it is the branch's entry condition (the head of
`compute_snap`, `core/snap.py:1173`-`1324`, `_lock_line_snaps` at `:1098`,
`_detect_axis_alignment` at `:185`, `_detect_axis_on_screen` at `:220`), the
reference point the direction is measured from while the mouse is dragging, or
the threshold it is measured against.

**This is not yet verified.** It is recorded as a hypothesis so that the first
work item tests it instead of assuming it.

## The rule the fix has to implement

Stated once, because everything below follows from it:

> An engaged strong inference (an axis, a held lock, an acquired reference)
> silences the weak ones while the cursor stays inside its reach. The reach of
> a direction is a narrow cone around it and is generous; the reach of a point
> is a tight disk and is small. Getting a point while an axis is engaged means
> leaving the cone and landing on the point - that is the "deliberately going
> looking for it" of the report.

Two properties make this safe rather than lossy, and both are worth keeping in
view when tuning:

- Inside the cone, a point that *disagrees* with the axis is not what the user
  asked for, so silencing it costs nothing.
- Inside the cone, a point that *agrees* with the axis lies on the axis line,
  where the axis snap and the point snap produce the same result, so there is
  no conflict to resolve and no cue to flicker.

## Plan

Each phase is independently verifiable and can ship on its own.

### Phase 0 - measure before changing

Confirm or kill the diagnosis with the probe already built and committed
(`scripts/probe_snap_matrix.py`, which drives `Viewport._refresh_snap`):
replay the reported gesture - box, top-right edge, nearest endpoint, drag along
X - and record, per cursor step, which `kind` is returned and how often the
axis branch fires.

Deliverable: a short table of kinds-per-step for that gesture, before any code
changes. This is the baseline every later phase is measured against, and it is
what turns S1 from an impression into a number.

### Phase 1 - engagement

Make the axis engage from the direction of the drag itself, as soon as the drag
commits to a direction, rather than requiring the cursor to satisfy whatever
the current entry condition asks for.

Deliverable: in the probe's replay, the axis fires on the first movement and
stays fired.

Risk to watch: an axis that engages too eagerly breaks drawing *off* the axis
(the user cannot place a point one pixel to the side). The entry condition
needs a clear answer to "when is the user committed to a direction".

### Phase 2 - authority while engaged

Holding the axis engaged is what makes the early return at `core/snap.py:1324`
do the silencing. If the axis keeps dropping out mid-drag, the fall-through to
the distance ranking keeps happening and S2 survives Phase 1.

Deliverable: no non-axis kind in the probe's stream while the gesture stays
inside the cone, and a defined, deliberate way out of the cone.

### Phase 3 - retention when nothing strong is engaged

With no axis in play, the discrete points are still ranked by raw distance,
bucket by bucket, so S3 remains. Apply retention to the weak tier: keep the
last discrete snap until the cursor moves decisively away from it, or until the
dwell mechanism says the user is looking elsewhere. `_TIE_PX` (`core/snap.py`)
is the place where this half-measure already lives.

Deliverable: in the probe's replay of a slow, ambiguous approach to a vertex,
one kind holds instead of alternating.

This is the phase that needs a *time* or *distance* number, and the number is
to be tuned from the probe, not chosen by taste.

### Phase 4 - edges as a continuum

Only now: `on_edge`, `extension`, `perpendicular`, `parallel` and the axis
locks mutually exclude each other over the same pixels, and an edge has no
point to hold. This needs its own retention policy (the line stays, not the
point) *and* an arbitration rule for which linear inference wins. It is
deliberately last: a discrete-point policy that works is a much better
foundation for it than a guess made now.

## Delivery note (ordering, not scope)

This plan refers to a feature (hysteresis) that may change the
`InferenceEngine.snap()` contract only in behaviour, not in signature. Keep it
that way: no new fields on `SnapResult`, no new plugin surface. If a phase
appears to need a signature change, that is a signal the phase is doing too
much at once.

## Open decisions

- **The numbers**: cone angle, point radius, retention distance and time. All
  from Phase 0's baseline, all easy to revise.
- **Is dwell the deliberate search?** The mechanism already exists; either it
  becomes the answer to "the user went looking for a point", or it stays
  independent of snap selection. Decide before Phase 3.
- **Whether S2 is even in scope.** If the reported case turns out to be purely
  S1 (the axis never engages), then Phase 2 is a safety net rather than the
  fix, and the priority is different.

## Verification

- **The probe** (`scripts/probe_snap_matrix.py`) replays the gesture and prints
  the kind per step: the Phase 0 baseline, then the same table after each phase.
- **Tests**: one test per phase, pinning behaviour at the `InferenceEngine.snap`
  level rather than through the GUI, following the two tests the refactor added
  (`tests/test_extension_api.py`, `tests/test_tape_axis_source_and_guide_segment.py`).
- **The suite**: `python -m pytest -m "not slow"`, the same set CI runs.
- **The gesture itself**, by hand, because this is a behaviour change and the
  reports come from a hand.

## Out of scope

- New inference kinds, or feature expectation beyond what this case needs.
- Touch and stylus input.
- The number of cues drawn at once as a *drawing* decision. If "several
  indicators at the same time" survives Phase 2, the residue is about how many
  cues are painted and which one wins the ScreenTip - a presentation question,
  in `views/viewport.py`, not in the engine.
