---
type: decision
severity: required
status: stable
date: 2026-07-03
applies_to: project: go_sport
keywords: [player, session, persistence, shared-preferences, restore, prime, position, radio]
supersedes:
superseded_by:
related: [[persist-and-restore-playback-session]], [[two-layer-audio-stack]], [[local-state-via-storage-class]], [[dual-mode-player-state]]
code_refs:
  - go_sport/lib/core/audio/player_session_storage.dart
  - go_sport/lib/domain/state/player_state.dart
  - go_sport/lib/main.dart
---

# ADR-002: Persist and restore the music player session

## Context

Two gaps in the player were reported from device use. On a first-ever launch the mini-player was an empty bar — it occupies permanent space at the bottom of the shell, so an empty one is conspicuous. And after killing the app and reopening it, the queue was gone: the user had to navigate back to whatever they had been listening to and start it again.

The player already had the pieces needed to fix both. `resumeMusic()` reloads a queue by index from in-memory state; restoring from disk is the same move with a different source. What was missing was somewhere to keep the queue between launches, and a way to load a queue without starting playback — the latter came from splitting load from play in the audio handler (see [[two-layer-audio-stack]]).

The general structure of the solution is recorded as [[persist-and-restore-playback-session]]. This ADR records what go_sport specifically chose.

## Decision

Persist the **music** side of the player and prime the mini-player on cold start.

**Storage.** `PlayerSessionStorage` over `SharedPreferences`, following [[local-state-via-storage-class]]. Two keys:

- `player_session` — a `PlayerSession` snapshot: `tracks`, `currentIndex`, and the queue source flattened to `sourceType` / `sourceId` / `sourceTitle` / `sourceImageUrl`
- `player_position` — a single int of milliseconds

The source is flattened rather than stored as a `QueueSource` because keeping the rich type would make the storage file import `player_state.dart`, which imports the storage — a circular import. The notifier owns the mapping in both directions. Serialization is hand-written inside the storage class so that domain `Track` and `QueueSource` stay free of json concerns, consistent with [[dto-to-domain-mapping]] — `Track` is Freezed **without** json_serializable.

**Save triggers.** The queue snapshot is written on queue start (`playQueue`), on track change, on `resumeMusic`, and on `AppLifecycleListener` pause/detach. The position is written by a `Timer.periodic(10s)` gated on `state.isPlaying && mode == music`, plus the same lifecycle hooks.

**Startup.** `_restoreOrPrime()` is scheduled from `build()` via `Future(...)` and guarded to act only on an idle, empty player. If a snapshot exists it is restored, merged with the separately stored position. Otherwise the first featured playlist is fetched and used to prime. Both paths go through `loadPausedQueue(...)`.

**Paused, never autoplay.** Restore and prime both land paused. This was an explicit product call, and it is also the only safe option on iOS, which does not honour autoplay on a cold start.

**The prime is not saved.** `loadPausedQueue` does not write a session, so a primed featured playlist the user never touched re-primes on the next launch rather than freezing into a "session" they never chose.

**Radio is never persisted.** `_saveSession` early-returns unless the mode is music with a non-empty queue and a source. Without that guard, backgrounding the app during radio playback would overwrite the saved music session with nothing. `_savePosition` carries the same mode guard so a radio-time tick cannot overwrite the stored music position.

**Corrupt snapshots are dropped.** `init()` wraps deserialization in a try/catch and discards anything it cannot read, so a schema change can never crash startup.

## Alternatives considered

- **Position inside the session snapshot** — one key, one write path — rejected: position changes constantly, so every write would re-serialize the entire track list. Splitting it into its own key makes the frequent write a single int and leaves the queue snapshot untouched.
- **Lifecycle-only saves (no periodic timer)** — save on pause/detach and nothing else — rejected after device testing. The queue restored correctly and the position did not: between backgrounding and the OS kill, the fire-and-forget async write to `SharedPreferences` frequently does not reach disk, so the restored position was whatever the last calm save at a track change had left — effectively zero. The periodic write bounds the loss to ~10s regardless of lifecycle-callback timing.
- **Autoplay on restore** — resume exactly where the user left off, including playing — rejected: startling on launch, and iOS will not honour it on a cold start anyway.
- **Persisting radio alongside music** — rejected: a live stream has no meaningful position to restore, and sharing one snapshot between the modes means whichever mode was last active clobbers the other.
- **Making `PlayerState` itself serializable** — rejected: it is stream-backed and carries live playback fields; a separate snapshot class keeps json out of the domain state and lets the two shapes evolve independently.

## Consequences

- (+) Kill-and-reopen restores the queue and lands within ~10s of the true position
- (+) The mini-player is populated on a first-ever launch instead of showing an empty bar
- (+) Restore reuses the existing load-queue-by-index path rather than introducing new machinery
- (+) Splitting load from play in the handler — the prerequisite for this work — also made `resumeMusic` able to preserve the music position across a radio detour, which it previously lost
- (-) A repeating timer runs for the duration of every playing music session, and must be cancelled on dispose
- (-) The position is accurate only to the save interval; a hard kill still loses up to ~10s
- (-) `PlayerSession` is a second schema over `Track` that has to be maintained by hand as the entity changes, with no compiler link between them
- (-) Radio sessions are not restored at all — accepted, but it means the two modes behave differently on launch

## Notes for future projects

The 10-second interval is a guess that has not been tuned; it was chosen as "small enough that the loss is unnoticeable, large enough that the write is irrelevant". Revisit if profiling ever shows the write costing anything, or if users report losing more than they tolerate.

Re-evaluate the whole shape if radio gains something worth restoring (a station choice rather than a position), or if queues grow large enough that serializing the track list at every track change becomes a visible cost — at which point storing track ids and rehydrating from a repository on restore becomes the better trade.
