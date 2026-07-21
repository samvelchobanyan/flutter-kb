---
type: pattern
severity: recommended
status: stable
applies_to: flutter-projects-with-playback
keywords: [playback, play-button, queue-source, collection, hero, selector, resume, pause]
related: [[dual-mode-player-state]], [[selector-providers-for-rebuilds]], [[shared-widgets-stay-dumb]], [[player-fast-path-masks-dead-player]]
---

# Collection play button reflects the active source

## Problem

An album, playlist, or program screen has a play/pause button on its header. The question it must answer is "is *this collection* the thing currently playing?" — and the tempting way to answer it is "is the currently playing track one of my tracks?"

That is wrong, and it is wrong in a way that shows up immediately in a real library: the same track appears in an album, in a playlist, and in favorites. Playing it from the playlist lights up the album's button too. Every collection containing the current track claims to be playing.

The behaviour users actually expect — the one every major music app implements — keys on which collection *owns the queue*, not on what happens to be in it.

## Solution

Compare the player's active queue source against this screen's own identity:

```dart
final isThisActiveSource = ref.watch(playerStateProvider.select(
  (s) => s.source?.id == thisId && !s.isRadioMode,
));
final isThisPlaying = ref.watch(playerStateProvider.select(
  (s) => s.source?.id == thisId && !s.isRadioMode && s.isPlaying,
));
```

**Two booleans, not one.** This is the load-bearing detail, and the one-boolean version is a subtle correctness bug rather than a simplification:

- `isThisActiveSource` — does this collection own the active queue, **regardless of play/pause**. Drives the tap behaviour.
- `isThisPlaying` — the same, plus actually playing. Drives the icon only.

Tap forks on the first: if this collection is the active source, `togglePlayPause()`; otherwise `playQueue(...)` to start it as a new queue.

With a single `isPlaying`-gated boolean the button still pauses correctly, which is why the bug survives casual testing. But pausing flips the boolean to false, so the *next* tap no longer recognizes the collection as active and falls through to `playQueue(...)` — restarting from the first track instead of resuming the paused one. The user pauses track seven and pressing play sends them to track one.

The reason source identity is the right pause-independent criterion is that pause preserves `source`, `tracks`, and `currentIndex`; only `stop()` clears them. So "is this the active source" remains true across a pause, which is exactly what "resume versus restart" needs to ask. The same fact underlies the guard in [[player-fast-path-masks-dead-player]].

**Read both through narrow selectors.** Each is a `bool` derived via `.select`, so the header rebuilds only when that boolean flips — not on the position ticks the player emits several times a second. See [[selector-providers-for-rebuilds]].

**Keep the header dumb.** It receives `isPlaying` as a prop and exposes an `onPlayTap` callback; the screen owns the watches and the tap fork. This keeps the header reusable across album, playlist, and program screens, per [[shared-widgets-stay-dumb]].

## When to apply

- Any collection-detail screen (album, playlist, program, artist) with a play control on its header
- Apps where the same track legitimately belongs to several collections
- Players with more than one playback mode, where "playing" must exclude the other mode — see [[dual-mode-player-state]]

## When NOT to apply

- Tracks belong to exactly one collection, so membership and source identity coincide and the simpler check is genuinely equivalent
- The screen has no collection-level play control, only per-track buttons
- Playback has no queue concept at all — each play is a standalone item with no owning collection

## Trade-offs

- (+) Correct with shared tracks: exactly one collection ever shows as playing
- (+) Pause and resume behave as users expect, including from a non-first track
- (+) Two narrow selectors keep an expensive header out of the position-tick rebuild path
- (+) The identical shape ports across every collection screen, so the behaviour stays consistent
- (-) Two derived booleans per screen instead of one — visibly redundant, and the redundancy is the point
- (-) Depends on the queue carrying a source identity; a player that does not tag its queue must add that first
- (-) The tap fork lives in each screen; extracting it into a shared helper is possible but has to keep the collection's identity as a parameter

## Common pitfalls

- Deriving both the icon and the tap behaviour from one `isPlaying`-gated boolean — pauses correctly, restarts from track one on resume
- Using track membership (`tracks.any((t) => t.id == currentTrackId)`) as the criterion, which lights up every collection containing the current track
- Forgetting the mode exclusion, so a playing radio stream makes a music collection's button show as playing
- Watching the whole player state instead of two selected booleans, which rebuilds the header on every position tick
