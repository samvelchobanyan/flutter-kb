---
type: pattern
severity: recommended
status: stable
applies_to: flutter-projects-with-playback
keywords: [audio-service, media-item, art-uri, lock-screen, artwork, fallback, asset, media-session]
related: [[two-layer-audio-stack]], [[adr-002-just-audio-plus-audio-service]]
---

# System player artwork fallback

## Problem

An app has a proper image-fallback story in Flutter: a shared network-image widget that collapses loading, error, and null-url into one bundled placeholder. Every list, hero, and player screen shows something sensible when a cover is missing.

The lock screen and notification show nothing.

The reason is that `MediaItem.artUri` leaves Flutter entirely. It is handed to `MediaSession` on Android and `MPNowPlayingInfoCenter` on iOS, and those load it themselves — from `http(s)` or `file` URIs only. A Flutter asset path is not a URI they can read. Worse, the failure is silent and one-way: the art fails **after** the URI has been handed over, in a process that never calls back, so none of the in-app fallback machinery is even reachable. Whatever the app does with `errorWidget` is irrelevant here.

The practical result is that queues sourced from anywhere without a network cover — search results, favorites, custom playlists whose cover is a bundled asset — play with a blank system player, while the in-app UI looks fine. Two surfaces, one track, different outcomes.

## Solution

Give the system a URI it can actually read, and make sure the fallback chain reaches it.

**Materialize the asset to a file, once.** The native side cannot read assets, but it can read `file://`. Lazily copy the bundled placeholder into the app support directory on first use and keep the resulting `Uri.file` — subsequent queue loads reuse it:

```dart
Uri? _noImageArtUri;

Future<void> _ensureNoImageArt() async {
  if (_noImageArtUri != null) return;
  final bytes = await rootBundle.load(_noImageAsset);
  final file = File('${dir.path}/$_noImageFileName');
  await file.writeAsBytes(bytes.buffer.asUint8List());
  _noImageArtUri = Uri.file(file.path);
}
```

If materialization fails, leave it null and let `artUri` be absent — a missing cover is the same outcome as today, not a crash.

**Guard the chain by scheme.** This is the half that is easy to miss and the reason the fallback silently never fires:

```dart
Uri? _networkUri(String? url) =>
    url != null && url.startsWith('http') ? Uri.parse(url) : null;

artUri: _networkUri(track.imageUrl)
     ?? _networkUri(fallbackArtUrl)   // the collection's cover
     ?? _noImageArtUri;               // the materialized placeholder
```

Without the scheme check, a non-network value — an asset path used as a playlist cover, an empty string from a search result — is **non-null**, so it terminates the `??` chain and is handed to the native side as a dead URI. The placeholder is never reached. The guard converts "present but unusable" into "absent", which is what the chain needs in order to fall through.

**Chain from most to least specific.** The track's own cover, then the cover of the collection the queue came from, then the bundled placeholder. The middle link is what makes a track with no artwork of its own still show its album or playlist art on the lock screen.

Keep all of this inside the audio handler. It is infrastructure concerned with what the platform can consume, and it fits the handler's role in [[two-layer-audio-stack]] without the notifier or any screen knowing about it.

## When to apply

- Any app publishing `MediaItem`s to `audio_service` where some tracks lack a network cover
- Apps whose queues can originate from sources with no artwork at all — search, favorites, user-created playlists
- Apps that use bundled assets as covers anywhere, since those paths cannot reach the system player

## When NOT to apply

- Every track is guaranteed a valid network cover by the backend — then the chain is one link and the machinery is unnecessary
- No background playback or system integration, so no `MediaItem` is ever published
- The placeholder is not worth the copy: showing no artwork is an acceptable outcome, and skipping this keeps one fewer file on disk

## Trade-offs

- (+) The lock screen and notification stop being blank for whole categories of queue
- (+) The fallback lives in one place; call sites keep passing whatever url they have
- (+) One file write for the lifetime of an install, on the first queue load
- (-) A file materialized outside the asset bundle is a small piece of state to reason about — it must be regenerated if the placeholder asset changes, which a version-suffixed filename handles
- (-) The scheme guard hard-codes an assumption that only `http`-prefixed values are usable; a project serving covers over another readable scheme has to widen it
- (-) It cannot help with URLs that are syntactically valid and dead

## Common pitfalls

- Assuming the in-app image fallback covers the system player. It cannot — the failure happens after the handoff, in another process, with no callback
- Building `artUri` straight from `imageUrl` without a scheme check, so empty strings and asset paths become dead URIs that block the chain
- Materializing the asset on every queue load instead of once, turning a one-time copy into per-play disk churn
- Expecting a 404 cover to fall through. A syntactically valid URL cannot be validated client-side without a preflight request; a dead link that the backend serves is a data problem and should be fixed there
