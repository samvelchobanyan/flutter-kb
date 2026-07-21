---
type: pattern
severity: recommended
status: stable
applies_to: flutter-projects-with-playback
keywords: [audio, playback, just-audio, audio-service, player, notifier, background-playback]
related: [[adr-002-just-audio-plus-audio-service]], [[dual-mode-player-state]], [[selector-providers-for-rebuilds]], [[persist-and-restore-playback-session]], [[audio-error-recovery]], [[system-player-artwork-fallback]]
code_refs:
  - go_sport/lib/domain/state/player_state.dart
  - go_sport/lib/core/audio/app_audio_handler.dart
---

# Two-layer audio stack

## Problem

An audio app needs to integrate with platform audio infrastructure (lock-screen controls, background playback, notification, headset buttons) while exposing a clean, Riverpod-native API to the UI. The platform side (audio_service) is event-driven and AudioHandler-shaped; the UI side wants `PlayerState`, `playTrack(...)`, `pause()`. Trying to merge them into a single class produces a class that's both a `BaseAudioHandler` (with stream callbacks for queue items, playback state, media item) and a Notifier (with `state` and explicit user API) — confusing and fragile.

## Solution

Two layers, not three:

1. **`AppAudioHandler`** — infrastructure layer. Extends `BaseAudioHandler with QueueHandler, SeekHandler` from `audio_service`. Wraps a `just_audio.AudioPlayer` and bridges its streams (`playbackEventStream`, `currentIndexStream`, `durationStream`, `icyMetadataStream`) to the system. Exposes raw position/duration/buffered streams and queue-management methods (`setQueue`, `playRadioStream`, `skipToQueueItem`, `setLoopMode`, `setShuffleEnabled`). Knows nothing about Riverpod.

2. **`PlayerNotifier`** — domain layer. Extends `Notifier<PlayerState>`, lives in `domain/state/player_state.dart`. Subscribes to the audio handler's streams in `build()`, maps them into a single Freezed `PlayerState`, and exposes the user-facing API (`playQueue`, `togglePlayPause`, `seek`, `next`, `previous`, `playRadio`, `cycleRepeatMode`, `toggleShuffle`). It is the only thing UI talks to.

```
UI widgets
   │  ref.watch(playerInfoProvider) / etc
   ▼
Selectors  ──watches──▶  PlayerNotifier (Notifier<PlayerState>)
                              │  user API:
                              │  • playQueue(...)
                              │  • play() / pause() / next() / seek(...)
                              │  • playRadio() / resumeMusic()
                              │  listens to streams:
                              ▼
                       AppAudioHandler (BaseAudioHandler)
                              │  • setQueue / playRadioStream / skipToQueueItem
                              │  • positionStream / durationStream / icyMetadataStream
                              ▼
                       just_audio + audio_service
                              │
                              ▼
                       OS audio system
```

UI is split into two full-screen player screens (`FullPlayerScreen` for music, `RadioFullPlayerScreen` for radio) — that's a UI concern. Both screens read the same `PlayerState` via [[selector-providers-for-rebuilds]]; the underlying state is single. The split doesn't create a third layer.

**The handler loads; the notifier decides to play.** `setQueue(tracks, initialIndex, initialPosition)` only calls `setAudioSource(...)` — load and seek — and never calls `play()` itself. Starting playback is a decision one level up: `playQueue` and `resumeMusic` call `setQueue(...)` then `play()`, both inside a single try/catch so the pair is atomic. A separate `loadPausedQueue(...)` calls `setQueue(...)` and stops there, leaving a player that is `ready` but not playing — which maps to `PlayerStatus.paused`, so "has active playback" is true and the mini-player shows the track silently.

This mirrors just_audio's own API, where `setAudioSource` and `play` are already separate calls. Bundling them inside the handler was the anomaly, not the split. A `setQueue` that also starts playback is a lying name, and the lie is load-bearing: it makes "a loaded but silent queue" inexpressible.

That state turns out to be worth expressing. It is exactly what a cold-start session restore needs, and what priming the mini-player on first launch needs — both want the queue loaded and the track named, with no sound (see [[persist-and-restore-playback-session]]). Removing one `play()` call and adding an `initialPosition` parameter is the whole cost. It also matches iOS, which will not honour autoplay on a cold start regardless.

## When to apply

- Apps with background audio playback (music, podcasts, radio)
- Apps that need lock-screen / notification controls
- When playback state has to be reflected in UI through Riverpod
- When the same playback session may be presented through multiple screens (mini-player + full-screen music player + full-screen radio player) — they all read the same state

## When NOT to apply

- Sound-effect playback only (no background, no controls) — `just_audio` directly inside a feature is enough; no need for audio_service or a Notifier
- Plain video playback with no background mode — `video_player` + a feature controller, no infrastructure layer needed

## Trade-offs

- (+) Each layer has one job; the bridge between them is one file (the Notifier's stream subscriptions)
- (+) UI can rebuild on any state slice independently via selectors; the handler doesn't know about UI rebuilds
- (+) Testing the handler with a fake `AudioPlayer` doesn't require Riverpod; testing the Notifier with a fake `AppAudioHandler` doesn't require `audio_service`
- (+) Keeping "load" and "play" as separate operations lets the notifier compose them freely — the same handler method serves normal playback, session restore, and silent priming
- (-) Two places to look when tracking down a playback bug: was the state miscomputed (Notifier), or did the handler not emit (audio_service)?
- (-) Stream subscriptions in the Notifier must be cancelled in `ref.onDispose`; forgetting leaks listeners and produces ghost state updates after navigation
- (-) The Notifier needs optimistic UI updates (e.g. `state.copyWith(currentIndex: startIndex)` before `_audioHandler.setQueue(...)` resolves) — the handler's stream is the eventual source of truth, but UI doesn't want to wait for it

## Common pitfalls

- Adding a "PlaybackService" middle layer between the two — the Notifier already orchestrates; an extra layer just hides the bridge
- Letting the handler hold any Riverpod imports — it should be a pure audio_service subclass usable in tests without ProviderScope
- Forgetting that `_audioHandler.playbackState` is a `BehaviorSubject` — calling `.value` before it's emitted throws. Listen to the stream; don't read synchronously at construction.
- Mismatching system-side and UI-side notion of "current track" — when `mediaItem` emits, sync `state.currentIndex` to match (see `_onMediaItemChanged` in the Notifier)
- Calling `play()` inside the handler's queue-loading method — it fuses "loaded" and "playing" into one operation, and every later feature that needs a silent loaded queue has to work around it
