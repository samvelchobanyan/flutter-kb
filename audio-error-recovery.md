---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-just-audio
keywords: [just-audio, error-handling, broken-track, seek-to-next, reentrancy, platform-exception, playlist]
related: [[two-layer-audio-stack]], [[adr-002-just-audio-plus-audio-service]], [[player-fast-path-masks-dead-player]]
---

# Recovering from a broken track in just_audio 0.9.x

## Symptom

One unplayable track in a playlist — a dead file url, a 404 — and the obvious fix is to skip it. The obvious implementation produces four different failures, in escalating order of subtlety:

1. **The app hangs with an unhandled exception**: `Cannot fire new event. Controller is already firing an event`.
2. **Two tracks are skipped instead of one.** The broken track is passed, and so is the good one after it.
3. **A perfectly good track is skipped (Android), or the currently playing track is killed mid-play (iOS).** On iOS this is the strangest form: audio is playing fine and suddenly jumps forward.
4. **A broken track at the *start* of a queue makes the whole queue unplayable**, even though the same track sitting in the middle of the queue would be survived.

Symptom 3 is the one that makes "skip on error" code get deleted from projects. It looks correct, it passes a superficial test with one broken track, and then it breaks normal playback in ways that are hard to attribute.

## Root cause

Four distinct mechanisms, all downstream of one thing: just_audio 0.9.x reports errors as **errors on a stream**, which is the wrong shape for something the app is expected to recover from.

**Reentrancy.** `playbackEventStream`'s `onError` fires while the underlying rxdart subject is mid-dispatch. Calling `seek()` or `seekToNext()` synchronously from that handler writes back into the same subject that is currently firing — hence the "controller is already firing an event" crash.

**Multiple events per failure.** A single failed source can emit more than one error event. Each one independently schedules its own recovery, and the skips compound.

**The native plugins already recover.** This is the mechanism that makes the naive fix actively harmful, and it is invisible from Dart:

- **Android** re-prepares onto `failed + 1` by itself, up to five errors per load. By the time the Dart handler runs, the player has usually already moved on — so an unconditional `seekToNext()` skips the *next, good* track.
- **iOS** `AVQueuePlayer` skips failed items on its own. It also broadcasts errors for **upcoming** items it failed to preload, while the current track is still playing normally — so an unconditional skip kills healthy playback in response to a problem with a track further down the queue.

**Cold-start load failures are different in kind.** When the starting track fails, `setAudioSource` throws; and when the native player was created for that particular load, just_audio disposes it — discarding even a successful native auto-advance. Nothing survives to recover from, which is why a broken first track behaves worse than a broken middle track.

## Fix

**Defer, then deduplicate.** Never act synchronously inside `onError`. Schedule the recovery with `Future(...)` so it runs after the current dispatch completes, and guard it with an in-flight boolean so multiple error events for one failure produce one recovery.

**Ask where the player actually is before moving it.** The error payload carries the failed index — `PlatformException.details` is a map containing `{index: N}` on both platforms. Read it, compare against `player.currentIndex`, and branch on what the native side has already done:

| State | Action |
|---|---|
| Moved past the failed item, still playing | Do nothing — the plugin handled it |
| Moved past the failed item, but stalled | `play()` only — do not move the index |
| Still sitting on the failed item | `seekToNext()` then `play()` |
| Failed item is last, nothing to advance to | `stop()` |

The current index is only a fallback for reading the failed index, not a substitute: by the time the handler runs, the player may already have moved.

**Separate load-time errors from playback-time errors.** A failing load emits the same error events as a failing playback, but during a load they belong to whatever retry logic the queue-loading path owns. Without a second flag marking "a load is in progress", two mechanisms command the player simultaneously and fight. Mark the load, and have the `onError` recovery stand down while it is set.

**Step over broken starting tracks explicitly.** Because a failed initial load can take the native player down with it, the queue-loading path needs its own retry: try `initialIndex`, and on failure advance and try again until one loads, rethrowing only when nothing in the queue is loadable.

## Where this is likely to occur

- Any just_audio 0.9.x app playing a backend-supplied queue, where some entries can be dead
- Apps whose "skip on error" logic was written from the Dart side alone, without reading either native plugin's source — the auto-advance behaviour is not documented at the Dart API level
- Apps that restore or prime a queue on startup, which makes a broken first track a launch-path problem rather than an edge case
- Codebases where such recovery code was written, caused strange skipping, and was removed — the removal is usually a response to symptom 3

## How to detect early

- Treat any synchronous player command inside a stream's `onError` as suspect on sight; the reentrancy crash is structural, not incidental
- Log the failed index alongside `player.currentIndex` in the handler before doing anything. If those two disagree, the native side has already moved and any unconditional skip is a bug
- Test with a broken track that is not first, a broken track that is first, and two consecutive broken tracks. The four failure modes above separate cleanly under those three cases

## Notes on package versions

Everything above is scaffolding around just_audio 0.9.x's error model, and it is worth knowing that the model was redesigned rather than patched. In 0.10.x:

- `AudioPlayer(maxSkipsOnError: N)` replaces hand-written recovery, and — importantly — counts **consecutive** errors, resetting on every successfully loaded track. The 0.9.x native counter is cumulative per load, so a queue that accumulates enough scattered errors stops auto-advancing permanently and strands the player, with no way to reset short of a full reload.
- `errorStream` replaces `playbackEventStream`'s error channel: errors arrive as **data**, which removes the reentrancy hazard as a class rather than requiring every handler to defer.
- The dispose-on-failed-initial-load behaviour is gone upstream, which is what made the broken-first-track case need its own retry.

If a project is on 0.9.x and this recovery logic is being written from scratch, weigh the upgrade first — most of this page becomes unnecessary on the newer error model.
