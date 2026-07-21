---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-playback
keywords: [playback, optimistic-update, fast-path, seek, queue, pagination, player-status]
related: [[dual-mode-player-state]], [[audio-error-recovery]], [[collection-play-button-reflects-active-source]]
---

# A play fast-path masks a dead or mismatched player

## Symptom

Tapping tracks inside the collection that is currently open does nothing audible — but the UI accepts every tap. The tile's playing indicator moves, the mini-player updates its title, and there is silence. Tapping into a *different* collection works normally.

It reads as "tap handling is broken on this screen". It is not: the taps are handled, the state updates, and the command sent to the player is a no-op. The player usually died some time earlier — often on a broken track (see [[audio-error-recovery]]) — and nothing since then has forced it back to life.

A second, quieter variant: in a paginated list, tapping a track near the bottom plays the wrong track, or nothing.

## Root cause

Play-queue logic almost always carries an optimization along the lines of "same source — just skip to the track instead of reloading the queue". It is a good optimization: re-tapping a track in the album already playing should not tear down and rebuild the native queue.

The problem is that it rests on assumptions about the native player that the Dart-side state does not actually check.

**Assumption one: the player still holds the queue.** After a playback error, or after an explicit `stop()`, the native player is idle. A bare `seek` into an idle player silently does nothing — no exception, no state change. Meanwhile the optimistic state update has already run, so the UI reflects a track that is not going to play. Optimism is the right default for responsiveness, and here it is exactly what hides the corpse: without it the bug would be obvious, because nothing would happen at all.

**Assumption two: the loaded queue matches what the user is looking at.** With paginated sources — an artist's tracks, a favorites list — the on-screen list grows past the queue that was loaded when playback began. The tapped index may not exist in the loaded queue, or may exist and hold a different track. "Same source" is true; "same list" is not.

Both are the same failure: the fast-path's precondition is stated as source identity, and source identity is weaker than what the shortcut actually requires.

## Fix

Check the two real preconditions before taking the shortcut, and fall through to a full queue reload when either fails:

```dart
final playerAlive = state.status != PlayerStatus.error &&
                    state.status != PlayerStatus.idle;
final queueMatches = startIndex < state.tracks.length &&
                     state.tracks[startIndex].id == tracks[startIndex].id;

if (state.source?.id == source.id && playerAlive && queueMatches) {
  // safe to just skip
}
// otherwise: full reload
```

Falling through is cheap and always correct — the reload is the path that runs for every other tap in the app. There is no case worth optimizing where a reload would be wrong, so when in doubt, reload.

Note the deliberate asymmetry with pause state: a paused player is **alive**, and pausing must not push the tap into the reload path — that would restart the collection from the top instead of resuming. Pause preserves the queue, the index, and the source; only `stop()` clears them. Source identity remains the right criterion for "resume versus restart" — see [[collection-play-button-reflects-active-source]].

## Where this is likely to occur

- Any player with a "same source, just skip" shortcut, which is nearly every queue-based player
- Apps with error recovery that can leave the native player idle without clearing the Dart-side queue state
- Screens backed by paginated or incrementally-loaded lists, where the visible list can outgrow the loaded queue
- Anywhere the UI is updated optimistically before the player confirms — the optimism is what converts a silent failure into a convincing one

## How to detect early

- Any conditional guarding a player shortcut deserves the question "what does this assume about the native player, and is that assumption checked?" Source id is a claim about *which* queue, not about whether one is loaded
- Reproduce deliberately: force a playback error, then tap another track in the same collection. If the UI accepts the tap and nothing plays, the guard is incomplete
- With pagination, scroll past the initially loaded page and tap there. That exercises the index-and-identity half specifically
- Treat "works in other collections but not this one" as a state-of-the-player signal rather than a screen-specific one. The difference between the two paths is the shortcut
