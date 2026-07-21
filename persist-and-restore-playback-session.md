---
type: pattern
severity: recommended
status: stable
applies_to: flutter-projects-with-playback
keywords: [playback, persistence, shared-preferences, session-restore, cold-start, position, lifecycle]
related: [[two-layer-audio-stack]], [[local-state-via-storage-class]], [[dual-mode-player-state]], [[projects/go_sport/adr-002-persist-player-session]]
---

# Persist and restore a playback session

## Problem

"I closed the app and everything was where I left it" is baseline expectation for a media app — every major music and podcast player behaves that way. Two gaps break it: after a kill-and-reopen the queue is gone, and on the very first launch the mini-player is an empty bar taking up space with nothing in it.

Persisting playback looks trivial until the details bite. The live player state is stream-backed and not serializable. The position changes several times a second, so writing it the same way as the queue is wasteful — but writing it only at natural checkpoints loses it, because the moment that matters is a process kill, which is exactly when an async disk write does not get to finish. And restoring is not simply "replay what was playing": an app that starts blasting audio on launch is a worse experience than one that forgot.

## Solution

**Three objects, not two.** The live state (`PlayerState`) stays as it is — stream-backed, un-serializable. A plain snapshot class (`PlayerSession`) holds only the fields needed to rebuild the queue. A storage class over `SharedPreferences` (see [[local-state-via-storage-class]]) owns the (de)serialization. The middle object exists precisely because the first cannot be serialized; skipping it means putting json concerns into the domain state.

**Two keys, split by write frequency.** This is the core of the pattern:

- `session` — the heavy snapshot (tracks, current index, source). Written rarely: on queue start, on track change, on resume, and on app lifecycle pause/detach.
- `position` — a single integer of milliseconds. Written often: on a `Timer.periodic` (≈10s) gated on "playing, and in the mode we persist", plus the same lifecycle hooks.

Position is deliberately **not** a field of the snapshot. Merging them means re-serializing the whole track list on every position write; splitting them makes the frequent write a single int. The two are recombined on restore. This is the same instinct as [[selector-providers-for-rebuilds]] — isolate the high-frequency thing from the low-frequency thing so the expensive one is not dragged along by the cheap one.

**Periodic position saves, not lifecycle-only.** Saving on `onPause`/`onDetach` alone looks sufficient and silently fails for the most volatile field. Between "user backgrounds the app" and "OS kills it" there is no guarantee that a fire-and-forget async write reaches the disk — so the queue comes back and the position is stale. A periodic write bounds the loss to the interval regardless of whether any lifecycle callback got to run. Keep the lifecycle hooks too; they cover the graceful case exactly.

**Restore or prime, always paused.** Schedule the startup work from `build()` via `Future(...)` — never mutate state during build — and guard it to act only when the player is idle and empty, so it cannot stomp on playback that has already begun. Then: if a snapshot exists, restore it; otherwise prime with a sensible default collection. Both paths go through the load-without-play entry point described in [[two-layer-audio-stack]], so the app comes up with a visible, named, silent queue. Restoring straight into playback is the wrong default — it is startling, and iOS will not honour autoplay on a cold start anyway.

**Flatten the source to primitives.** The snapshot needs to record where the queue came from. Keeping a rich `QueueSource` object in it makes the storage file import the player-state file, which already imports the storage — a circular import. Store `sourceType` / `sourceId` / `sourceTitle` / `sourceImageUrl` as flat fields and let the notifier own the mapping in both directions. The storage class stays a leaf.

**Tolerate a corrupt snapshot.** Wrap deserialization in a try/catch that drops the stored value. A schema change shipped in an update must degrade to "no session restored", never to a crash on startup.

## When to apply

- Any app with queue-based playback that a user expects to survive being closed
- Apps that want a populated mini-player on first launch, before the user has played anything
- Any long-running media session where the position is meaningful and losing it is noticeable

## When NOT to apply

- Short-form or ephemeral audio (sound effects, voice messages, previews) — nothing meaningful to restore
- Live streams, where position has no meaning and "resume" means "join at the current moment" — persist the fact that a stream was playing, if anything, not a position into it
- Apps where playback is strictly a foreground activity inside one screen and the queue is trivially rebuilt from what the screen already loaded

## Trade-offs

- (+) Kill-and-reopen keeps the queue and lands within one interval of the true position
- (+) The mini-player is never empty, including on a first-ever launch
- (+) The frequent write is a single int; the track list is serialized only when the queue actually changes
- (+) Restoring reuses the existing "load a queue by index" path rather than inventing new machinery — it is the same move, sourced from disk instead of memory
- (-) A repeating timer runs for the whole playing session; it is cheap, but it is one more thing to cancel on dispose
- (-) Position is accurate only to the interval; a hard kill still loses up to that much
- (-) Two keys mean two things that can disagree — a restored queue with a position pointing past its current track has to be tolerated
- (-) The snapshot is a second schema over domain entities, and it needs versioning discipline as those entities change

## Common pitfalls

- Persisting only on lifecycle callbacks and concluding the feature works because it survives a graceful background — the failing case is the kill, and it is the one users hit
- Writing position into the queue snapshot, which re-serializes every track on every tick
- Letting a mode that must not be restored overwrite the saved session: if the app has a second playback mode (radio, live), the save path must early-return for it, or backgrounding during that mode clobbers the music session — see [[dual-mode-player-state]]
- Saving the primed default the same way as a real session, which freezes a collection the user never chose into "where they left off"; priming should leave the stored session empty so the next launch primes again
- Restoring during `build()` instead of scheduling it, or restoring without an idle guard — both can overwrite playback the user just started
