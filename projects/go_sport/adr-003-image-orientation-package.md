---
type: decision
severity: required
status: stable
date: 2026-07-02
applies_to: project: go_sport
keywords: [image, exif, orientation, package-choice, flutter-image-compress, pure-dart, avatar]
supersedes:
superseded_by:
related: [[exif-orientation-not-baked]], [[dto-to-domain-mapping]]
code_refs:
  - go_sport/lib/features/user_profile/edit_profile/presentation/edit_profile/edit_profile_screen.dart
  - go_sport/pubspec.yaml
---

# ADR-003: `image` package for baking EXIF orientation

## Context

Camera-captured avatars came back rotated 90° after being saved, because the rotation lived in EXIF metadata that the backend stripped on re-encode (the general pitfall is [[exif-orientation-not-baked]]). The fix is to bake the orientation into the pixels client-side before upload: decode → `bakeOrientation` → re-encode. That requires an image-processing package, and there were two realistic options with a genuine trade-off between them.

This is a single, infrequent operation — one avatar, chosen occasionally — which is what tilts the decision.

## Decision

Use the pure-Dart **`image`** package (`bakeOrientation` + `encodeJpg`) to normalize the picked photo before upload. The bake happens inline in the edit-profile flow's image-pick handler; the normalized bytes are what get sent to the server.

Manual decode/bake/encode lives at the call site rather than behind a core helper — it is a single call site, and introducing an abstraction for it would be premature.

## Alternatives considered

- **`flutter_image_compress` (native)** — rejected. It is faster (native code) and does resize + rotation in a single call, but it adds a native plugin: `pod install`, a per-platform build surface, and another dependency that has to build across every dev machine. It also duplicates the resize that `image_picker` already performs. For a single infrequent avatar the performance advantage is irrelevant, and the native surface is a standing cost.
- **`image` (pure Dart)** — accepted. Slower, and it does not resize in the same call, but it adds zero native surface: no pod, no per-platform build step, reliable across the project's mixed dev setup (two Windows machines + a Mac). No duplication of what `image_picker` already did.

The decisive factors were therefore not speed — which does not matter here — but zero native build surface and no duplicated work.

## Consequences

- (+) No native plugin added; the app builds unchanged across the Windows + Mac dev setup
- (+) Orientation is baked into pixels before upload, so the saved avatar is upright regardless of backend EXIF handling
- (+) No premature abstraction — the decode/bake/encode sits at its one call site
- (-) Baking runs in Dart on the UI isolate; acceptable for one small avatar, but it would need an isolate if it scaled up
- (-) Resize is not folded into the same call, so if resizing becomes necessary it is a separate step
- (-) The choice is tuned to "one infrequent image" and does not generalize to heavier image workloads

## Notes for future projects

Revisit if image processing broadens — user posts, galleries, bulk upload, anything high-volume or on a hot path. At that point the native package's speed and single-call resize+rotate stop being irrelevant and start winning, and the native build surface is justified by the workload. The pure-Dart choice is correct precisely because this is one small, rare operation; change the workload and the trade-off inverts.
