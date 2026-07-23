---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [network-image, cached-network-image, fallback, placeholder, dumb-widget, design-system, noimage]
related: [[shared-widgets-stay-dumb]], [[design-system-tokens-mandatory]]
---

# Unified network-image fallback

## Problem

Network images fail in three ordinary ways — the load is still in flight, the load errored, or the url was null or empty to begin with — and each call site handles them ad hoc. One passes an `errorWidget`, another an `errorBuilder`, a third wraps the whole thing in a null-ternary. Over a codebase these drift: the placeholders diverge into a gray box here, a broken-image icon there, a music note somewhere else, and the loading state is sometimes a spinner and sometimes nothing. The same missing-image condition looks different on every screen, and there is no single place to change how it looks.

## Solution

One dumb presentational widget — `DSNetworkImage` — wraps the image loader (`CachedNetworkImage`) and collapses all three failure states into a single shared fallback: a bundled "no image" asset. Every screen renders images through it, so loading, error, and null/empty-url all resolve to the same placeholder, defined once.

The widget stays dumb per [[shared-widgets-stay-dumb]] — it holds no state and no provider subscriptions, taking only what it needs to render: `imageUrl`, plus `width` / `height` / `fit` / `borderRadius` / `memCacheWidth` / `memCacheHeight` so it can absorb every call site, including ones wrapped in `AspectRatio`, `Positioned.fill`, or a `Hero`. It lives in `design_system/components/`, which makes it a token at the component level (see [[design-system-tokens-mandatory]]) — the one place the "missing image" visual is decided.

Standardize the whole app onto one image loader while doing this. Mixing `CachedNetworkImage` and raw `Image.network` means two fallback APIs to keep aligned; picking one and dropping the other is part of the consolidation.

## When to apply

- A codebase with many network-image call sites that have grown inconsistent placeholders
- Any project with a design-supplied "no image" asset that should appear uniformly
- When loading, error, and null-url should look identical (one visual, no separate shimmer) — the common case

## When NOT to apply

- **Avatars.** A missing avatar has different semantics (initials, a person-silhouette SVG) and should not fold into the generic image placeholder — keep it its own widget
- **Local assets.** This widget is network-only. `Image.asset` call sites — bundled covers, notification icons — do not belong here; forcing them through a network widget is wrong
- **Decoration images.** A `BoxDecoration`/`DecorationImage` (e.g. a blurred background) is a decoration, not a child widget, so it cannot migrate without restructuring the layout to use a child. This is a real limitation of the child-widget approach, not an oversight — leave those where they are
- Projects with only a handful of image call sites and no drift — the consolidation buys little

## Trade-offs

- (+) One place defines the missing-image visual; changing it changes every screen
- (+) Call sites shrink to `DSNetworkImage(imageUrl: ...)` with no per-site error/placeholder boilerplate
- (+) Loading, error, and null-url can never again diverge, because they are the same code path
- (-) The widget's parameter surface grows to cover every call site's needs (sizing, radius, mem-cache), which is a wide but shallow API
- (-) The boundaries above (avatars, local assets, decorations) must be held deliberately; the temptation is to funnel everything through one widget and break the semantics
- (-) Standardizing on one loader is a migration cost paid once across all existing call sites

## Common pitfalls

- Folding avatars in because they are "also images" — the placeholder semantics differ and the result looks wrong
- Trying to route `Image.asset` or `DecorationImage` sites through a network-only widget
- Letting the widget acquire state or a provider subscription "just for this one case" — it stops being reusable the moment it does
