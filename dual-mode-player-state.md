---
type: pattern
severity: recommended
status: stable
applies_to: flutter-projects-with-playback
keywords: [player-state, playback-mode, music, radio, freezed, sealed, discriminator, queue, mode-switch]
related: [[two-layer-audio-stack]], [[freezed-immutable-state]], [[domain-vs-screen-state]], [[collection-play-button-reflects-active-source]], [[player-fast-path-masks-dead-player]]
code_refs:
  - go_sport/lib/domain/state/player_state.dart
---

# Dual-mode player state

## Problem

A player that supports both queue-based playback (music, podcast episodes) and live streams (radio) has different state shapes per mode: music needs `tracks`, `currentIndex`, `shuffleEnabled`, `repeatMode`; radio needs `radioStreamUrl`, `radioTitle`, `radioNowPlaying` (from ICY metadata). Splitting these into two separate Notifiers means the UI has to know which one to watch and switch carefully when modes change; coordinating queue-preservation across the split is awkward (resuming music after radio needs the music queue to still exist).

## Solution

A single `PlayerState` Freezed class with a `PlaybackMode { music, radio }` discriminator field. All mode-specific fields coexist; only the relevant ones are populated in each mode. Switching mode preserves the other mode's data so resuming is trivial.

```dart
enum PlaybackMode { music, radio }

@freezed
class PlayerState with _$PlayerState {
  const factory PlayerState({
    // Mode discriminator
    @Default(PlaybackMode.music) PlaybackMode mode,

    // Queue (music) — preserved when mode switches to radio
    @Default([]) List<Track> tracks,
    @Default(0) int currentIndex,
    QueueSource? source,

    // Shared playback state
    @Default(PlayerStatus.idle) PlayerStatus status,
    @Default(Duration.zero) Duration position,
    @Default(Duration.zero) Duration bufferedPosition,
    @Default(Duration.zero) Duration totalDuration,

    // Music-specific
    @Default(false) bool shuffleEnabled,
    @Default(RepeatMode.off) RepeatMode repeatMode,
    List<int>? shuffleIndices,

    // Radio-specific
    String? radioTitle,
    String? radioStreamUrl,
    String? radioImageUrl,
    String? radioNowPlaying, // from ICY metadata

    String? errorMessage,
  }) = _PlayerState;
}
```

The Notifier exposes mode-aware methods:

- `playQueue(tracks, source: ...)` switches to `PlaybackMode.music` and starts playback; preserves nothing radio-side
- `playRadio()` switches to `PlaybackMode.radio`; **keeps `tracks` and `currentIndex`** so the music queue is intact
- `resumeMusic()` switches back to music with the preserved queue at the same index

A `QueueSource` (sealed Freezed class with variants `album` / `playlist` / `program` / `favorites` / `episodes`) tags where the music queue came from, used for "same source — just skip" optimization (re-tapping a track in the same album doesn't reload the queue).

## When to apply

- Audio apps with two or more playback modalities that share a single output
- When user expectations require seamless mode switching (resume music after radio, return to live after pausing music)
- When mode-specific fields are few enough to coexist in one state class (single-digit count)

## When NOT to apply

- Apps with one playback mode only — the discriminator and mode-switching methods are dead weight
- Many fully-separate playback domains (e.g., 5+ types of media each with its own state shape) — split into separate states or use sealed `PlayerState.music(...) | PlayerState.radio(...) | ...` variants instead
- When mode switching is rare and the modes don't need to preserve each other's state — independent states are simpler

## Trade-offs

- (+) One state object, one Notifier, one provider; UI subscribes uniformly regardless of mode
- (+) Mode switching is `state = state.copyWith(mode: ...)` plus side-effect on the audio handler — no state migration
- (+) Preserving the other mode's data is free (fields just stay populated)
- (+) UI can be split into per-mode full-screen players (`FullPlayerScreen`, `RadioFullPlayerScreen`) — each reads the same state through selectors filtering mode-relevant fields. The split is a UI choice, not a state-layer choice.
- (-) State object is wider than either mode alone needs — some fields are `null` in some modes
- (-) Computed properties (`currentTrack`, `displayImageUrl`) must handle both modes, with mode-conditional branches inside
- (-) Selector design (see [[selector-providers-for-rebuilds]]) has to bundle "mode-relevant fields" by mode, not by single field

## Common pitfalls

- Clearing the music queue on `playRadio()` — breaks `resumeMusic()`. Keep `tracks`, `currentIndex`, `source` intact; only swap `mode`, `radioTitle`, `radioStreamUrl`.
- Confusing UI-mode toggle (`MiniPlayerWidget._isMusicMode` — which panel is animated open) with `PlayerState.mode` (what is actually playing). They are independent and may legitimately disagree at moments — see [[domain-vs-screen-state]].
- Reading that independence as "the UI toggle never reacts to the mode". It should react, on **transitions**: when playback starts in the other mode — the user is on the radio panel and music begins from another screen — the panel should follow. The right tool is `ref.listen` on the discriminator, comparing the local flag to the desired one and flipping only on mismatch (`if (local != desired) toggle()`). Not `ref.watch`: deriving the panel from the mode glues them permanently, so manual toggling becomes impossible, and a `setState` or an animation cannot be triggered during `build` anyway. The flip-on-mismatch guard works precisely because the flag is boolean — for a non-boolean toggle it must become an explicit setter.
- Letting selectors read across the mode boundary uncritically (`isPlaying && mode == music && tracks.isNotEmpty`) — these compound conditions are best computed as named getters on the state class so the UI doesn't repeat the logic
