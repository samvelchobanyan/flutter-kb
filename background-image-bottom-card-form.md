---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [layout, auth-screen, keyboard, safe-scroll, intrinsic-height, spacer, background-image, boxfit]
related: [[design-system-tokens-mandatory]], [[image-cache-dims-distort-aspect-ratio]]
---

# Background-image + bottom-card form

## Problem

Auth and onboarding screens commonly want a full-bleed background image on top and a white form card on the bottom. The card must survive two things that break a naive `Column`: a small screen (content taller than the viewport) and an open keyboard (which shrinks the viewport further and must not clip the fields or slide them under an app bar). The layout has to fill the viewport when there is room and scroll when there is not — both, from one structure.

## Solution

Nest the viewport-filling idiom so a scroll view can still fill the screen:

```
LayoutBuilder                                   // gives maxHeight
 └─ SingleChildScrollView                       // scroll when short / keyboard up
     └─ ConstrainedBox(minHeight: maxHeight)     // fill when tall
         └─ IntrinsicHeight                      // makes Spacer legal below
             └─ Column
                 ├─ image region (Expanded or Positioned, with overlap)
                 └─ form card
```

`IntrinsicHeight` is the load-bearing piece: it gives the `Column` a bounded height inside the otherwise-unbounded scroll view, which is what makes `Spacer` (and `Expanded`) legal there. Without it, a `Spacer` inside a scroll view throws.

Set `resizeToAvoidBottomInset: true`. When the keyboard opens, the card rides up over the image, the content scrolls, and a `Spacer()` inside the card keeps the primary button pinned to the card's bottom. The image height is derived from the card height — the two fill the screen with a deliberate overlap (the image sits a little behind the card's rounded top corners, e.g. `Positioned(bottom: -20)`), so no seam shows.

Crop side is controlled by the image's alignment, not by cropping the asset: with `BoxFit.cover`, `alignment: Alignment.topCenter` crops from the bottom, keeping the top of the image visible. This is a property of `Image`/`DecorationImage` alignment, independent of `BoxFit`.

**Two card-height variants**, chosen by whether the form's height is content-driven or fixed by design:

- **Content-driven** — the card shrink-wraps its content (`mainAxisSize.min`); the image takes whatever height is left. Use when the form's natural height is fine (a login screen).
- **Fixed-height** — the card has an absolute design height with content laid out top, then `Spacer()`, then buttons at the bottom; the image takes the rest. Use when the design specifies the card height regardless of content.

## When to apply

- Auth, onboarding, and similar full-screen forms with a decorative image and a form card
- Any screen that must fill the viewport when tall enough and scroll when the keyboard or a short screen makes it not
- Screens replicated across a flow (login, registration steps, password reset) that should share one consistent shell

## When NOT to apply

- Plain scrolling content with no fill-the-viewport requirement — a bare `SingleChildScrollView` is simpler
- Screens with no keyboard and no image — the `IntrinsicHeight` + `ConstrainedBox` scaffolding buys nothing
- A form that genuinely exceeds the viewport in normal use — then it is a scrolling form, not a card-pinned-to-bottom layout, and `Spacer` pinning is the wrong model

## Trade-offs

- (+) One structure handles tall screens, short screens, and open keyboards
- (+) The primary button stays pinned to the card bottom without absolute positioning
- (+) Consistent shell across a whole auth flow; per-screen difference is just content and the card variant
- (-) The nesting (`LayoutBuilder` → scroll → `ConstrainedBox` → `IntrinsicHeight`) is non-obvious and easy to get wrong or "simplify" into breakage
- (-) `IntrinsicHeight` does an extra layout pass; negligible for a form, but it is not free
- (-) Image-height-from-card-height coupling means changing one requires rechecking the overlap

## Common pitfalls

- A `Spacer` (or `Expanded`) inside the scroll view without `IntrinsicHeight` — throws, because the scroll view is unbounded and the flex child has nothing to divide
- The broken variant that looks equivalent: fixed `screenHeight * 0.6` + `extendBodyBehindAppBar` + a plain `Column` with no scroll — it overflows and slides under the app bar the moment the keyboard opens
- Forgetting `resizeToAvoidBottomInset: true`, so the keyboard covers the fields instead of the card riding up
- Cropping the asset to control which part shows, instead of setting `alignment` on a `BoxFit.cover` image
