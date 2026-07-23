---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [loading-state, null-safety, render-guard, force-unwrap, async, isloading, single-object-screen]
related: [[domain-vs-screen-state]]
---

# Non-exhaustive loading guard on a single-object screen

## Symptom

A screen that loads one object crashes with a null dereference — on cold start, or after a failed load — even though it "handles loading". The guard looks complete:

```dart
if (isLoading && data == null) return spinner;
return renderWith(data!);   // force-unwrap
```

It works when copied from a list screen and breaks when applied to a single object.

## Root cause

The guard is non-exhaustive over `data == null`. It only returns the spinner when `isLoading && data == null`, then force-unwraps. But there are two states where `data == null` while `isLoading == false`, and both fall straight through to the `data!` crash:

- **The initial frame.** Before a `Future.microtask`/`Future(...)` in `initState` has flipped `isLoading` to true, the state is the default: `isLoading == false`, `data == null`. The first build runs the guard, the guard's condition is false, and `data!` throws.
- **After an error.** A `catch` block that sets `isLoading: false` and leaves `data: null` lands in the same hole — not loading, no data, force-unwrap.

This is safe on **list** screens and dangerous on **single-object** screens, which is why the shape gets copied incorrectly. A list screen gates the spinner with `isLoading && list.isEmpty`, and an empty list still renders — the uncaught initial and error frames harmlessly fall through to an empty list widget. A null object has no such fallback: there is nothing to render, and the force-unwrap crashes instead of degrading.

## Fix

Gate exhaustively over every `data == null` case, so the force-unwrap is only ever reached when data is genuinely present:

```dart
if (error != null && data == null) return errorWidget;  // failed load
if (data == null)                 return spinner;        // still loading / initial
final d = data;                                          // promoted non-null
return renderWith(d);
```

The error branch and the null-data branch together cover every way `data` can be null — initial frame, in-flight load, and post-error — leaving the render branch reachable only with data in hand. Promote to a local non-null variable rather than force-unwrapping, so the compiler enforces it.

This applies to the manual `isLoading` / `error` / `data` state pattern. Code built on `AsyncValue` gets exhaustiveness from `when(data:, loading:, error:)` and does not have this hole — the trap is specific to hand-rolled tri-state.

## Where this is likely to occur

- Single-object screens (profile, detail-of-one) built by copying a list screen's guard shape
- Projects using manual `isLoading`/`error`/`data` state rather than `AsyncValue`
- Any screen whose load is kicked off from `initState` via a microtask, so the first build precedes `isLoading` becoming true
- Error paths that clear the loading flag but leave the data null

## How to detect early

- Any `data!` after a guard deserves the question: does the guard cover *every* path where `data` is null, or only the loading one? Enumerate initial, loading, error, loaded — four states, and the guard must handle all three null ones.
- Test cold start (first frame before the fetch resolves) and a forced load failure, not just the happy path. Both are the states the naive guard misses.
- Copying a list screen's `isLoading && isEmpty` gate onto a single-object screen is the moment the bug is introduced — an empty list tolerates the gap, a null object does not.
