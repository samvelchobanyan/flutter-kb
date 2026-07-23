---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [riverpod, lazy-init, global-notifier, ref-read, emptiness-guard, partial-snapshot, load-timing]
related: [[domain-vs-screen-state]]
---

# Emptiness guard on a lazily-merged global read

## Symptom

A short-lived screen or sheet shows a partial list that never fills in. In one reproduction: an "Add to playlist" sheet shows no custom playlists for a user who has liked playlists but has not yet visited the My Playlists tab — and it stays empty for the life of the sheet, even though the data does load elsewhere.

## Root cause

A short-lived controller does a one-shot `ref.read` of a lazily-initialized global `Notifier` in its `_init()`, and decides whether it needs to trigger a load by testing `.isEmpty` on a list the global exposes.

That works only if the list has a single source. When the global merges **several** sources with different load timings, the guard breaks. Consider a global that returns featured playlists synchronously in `build()` and schedules the custom playlists via `Future.microtask(loadFavorites)`. At the moment the controller reads it, the featured entries are already present, so the merged list is non-empty, so `.isEmpty` is false — and the controller concludes nothing needs loading. The custom playlists never get fetched, and because the sheet watches only its own controller's snapshot, the later microtask completion never propagates to it. The controller holds a partial snapshot forever.

The bug is combining two different questions into one test: "is the global initialized?" and "is the subset I need present?" `.isEmpty` on the merged list answers neither reliably once the global blends sources that arrive at different times.

## Fix

Do not gate on `.isEmpty` of the merged list. Either:

- **Filter to the subset you actually need before the emptiness check** — test whether the *custom* playlists are empty, not whether the combined list is, so a full featured subset cannot mask an unloaded custom subset; or
- **Track an explicit `loaded` flag on the global** for each source, and gate on that flag rather than inferring load-state from list contents.

Both replace "the list has something in it" (which conflates sources) with "the thing I need has actually loaded" (which does not).

## Where this is likely to occur

- A short-lived controller reading a long-lived global `Notifier` with a one-shot `ref.read`, deciding load-need from list contents — the interaction [[domain-vs-screen-state]] describes, seen from the failure side
- Globals that merge multiple sources with different load timings — some synchronous in `build()`, some via microtask/async
- Bottom sheets and dialogs that snapshot a provider once on open and never re-watch it
- Any "load if empty" guard over a list that aggregates more than one independently-loaded source

## How to detect early

- An `isEmpty`-gated load over a list that combines multiple sources is the signature. Ask: can one source be present while another is still absent? If yes, the guard can wrongly conclude "loaded".
- If a short-lived reader snapshots a global once, check that the thing it needs is guaranteed loaded at read time — a lazily-initialized global rarely guarantees that.
- Reproduce by exercising the state where one source is populated and the other is not (here: liked playlists present, My Playlists tab never visited). That asymmetric state is exactly what the merged-emptiness guard misreads.
