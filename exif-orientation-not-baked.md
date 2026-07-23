---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [exif, orientation, camera, image-upload, image-picker, rotation, bake-orientation]
related: [[projects/go_sport/adr-003-image-orientation-package]]
---

# Camera EXIF orientation is not baked into pixels

## Symptom

A photo captured with the camera looks upright in the app's own preview, and comes back rotated 90° after it is saved to the server. The tell is precise: **preview correct, saved image rotated**. Because the preview is fine, the investigation goes to the display layer — the widget showing the uploaded image, its `fit`, its rotation — which is the wrong place entirely.

## Root cause

Camera photos store rotation as an EXIF metadata tag, not in the pixel data. The sensor writes pixels in its native orientation and records "rotate this on display" as a tag.

Flutter's `Image.file` / `Image.network` honour that tag, so the local preview looks upright. But many backends strip or drop EXIF when they re-encode an upload — so the bytes that survive are the sensor's raw, unrotated pixels with the "rotate on display" instruction gone. The uploaded image is therefore sideways, while the preview that read the intact EXIF looked fine. The divergence is the whole trap: the two views disagree because one had the tag and one did not, and it points at the display layer precisely when the fix belongs at upload.

`image_picker`'s `maxWidth` / `imageQuality` do not reliably bake orientation on Android — reaching for those parameters looks like a fix and is not one.

## Fix

Normalize the pixels client-side before upload, rather than trusting metadata to survive:

1. **Decode** the picked file to a bitmap.
2. **Bake the orientation** — apply the EXIF rotation to the actual pixels (`bakeOrientation` in the pure-Dart `image` package).
3. **Re-encode** to JPEG. The JPEG re-encode drops the orientation tag, which is now correct: the rotation lives in the pixels, so no tag is needed.
4. **Upload the normalized bytes.**

After baking, the image is upright regardless of whether the backend keeps or strips EXIF, because there is no longer any orientation metadata to strip — the pixels already carry it.

## Where this is likely to occur

- Any camera-capture-then-upload flow: profile avatars, document photos, posts
- Backends that re-encode images on receipt (resize, format-normalize, strip metadata) — most do
- Android especially, where `image_picker`'s resize/quality options do not bake orientation
- Anywhere the local preview and the round-tripped image are rendered by different code, so only one of them ever reveals the problem

## How to detect early

- "Preview correct, saved image rotated" is close to a signature. When those two disagree, suspect metadata that survived locally and was stripped in transit — do not start at the display widget.
- Check whether the upload path bakes orientation at all. If it uploads the picked file's bytes directly, EXIF is the only thing keeping it upright, and it is at the backend's mercy.
- Test with a photo taken in landscape or with the phone rotated — portrait-only captures often have a trivial EXIF orientation that hides the bug.

## Notes

go_sport's choice of package for the decode-bake-re-encode step, and why it took the pure-Dart route over a native one, is recorded in [[projects/go_sport/adr-003-image-orientation-package]] — that ADR is the chosen solution to this pitfall.
