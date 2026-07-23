---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [image, cache-width, cache-height, resize-image, aspect-ratio, boxfit, decode, distortion]
related: [[background-image-bottom-card-form]]
---

# Both cache dimensions distort the image aspect ratio

## Symptom

An image renders visibly stretched or squashed even though `fit: BoxFit.cover` is set — the very thing that should have preserved its proportions. The same asset renders correctly elsewhere in the app with no `cacheWidth`/`cacheHeight`.

## Root cause

Passing **both** `cacheWidth` and `cacheHeight` to `Image.asset` (or a `ResizeImage`) decodes the bitmap to exactly those pixel dimensions and does **not** preserve the source aspect ratio — the default is `ResizeImagePolicy.exact`. If the `cacheWidth : cacheHeight` ratio differs from the asset's native ratio, the decoded bitmap is already distorted before layout.

`BoxFit.cover` cannot undo it. `cover` governs how the bitmap maps into the layout box, but the box here has the same wrong ratio the cache dimensions imposed, so the already-distorted bitmap is shown as-is. Developers add `cacheWidth`/`cacheHeight` purely for memory and assume `BoxFit.cover` still owns proportions — it does not, once both dimensions are pinned.

## Fix

Do not pin both dimensions:

- **Omit the cache dimensions** and let `BoxFit.cover` + alignment handle the crop. Correct for any image that is not large enough to be a memory concern.
- **Specify only one** of `cacheWidth` / `cacheHeight`. With a single dimension, the decoder scales the other proportionally, so the aspect ratio survives and you still cap memory.

The in-repo contrast: the same background asset renders correctly via `DecorationImage(fit: cover, alignment: topCenter)` with no cache dimensions, and distorted on another screen that set both cache dimensions to `screenWidth × screenHeight * 0.7`.

## Where this is likely to occur

- Full-screen or large background images where a developer added cache dimensions for memory and pinned both
- Any `Image.asset` / `Image.network` / `ResizeImage` given both `cacheWidth` and `cacheHeight` from layout-derived numbers that do not match the asset's native ratio
- Screens copied from a correct one but with cache dimensions "added for performance" during the copy

## How to detect early

- Both `cacheWidth` and `cacheHeight` present on one image is the signature — check whether their ratio matches the asset's native ratio; if not, it is already distorted.
- Stretched output despite `BoxFit.cover` means the distortion happened at decode, before layout. Look at the cache dimensions, not the fit.
- When adding cache dimensions for memory, add only one. Two is the trap; one is safe.
