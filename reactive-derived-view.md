---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [riverpod, ref-listen, derived-state, cache-invalidation, per-id-controller, composition, refetch]
related: [[domain-vs-screen-state]], [[selector-providers-for-rebuilds]], [[repo-side-effects-on-domain-state]]
---

# Reactive derived view on a shared composition

## Problem

Centralizing ownership of shared state solves most cross-screen sync problems, but leaves one behind.

A global `Notifier` owns the lightweight **composition** of a resource — which member ids belong to it, how many there are. A per-id detail screen owns the heavy **materialization** of that same resource — the full entities, fetched by a different endpoint with a fuller expansion. Both are legitimate under [[domain-vs-screen-state]]: the composition has many consumers (cards, counters, pickers) so it is domain state; the full entities are read by one screen so they are screen state.

The two are separate caches of the same backend resource, and they drift. A user opens a playlist, navigates away, and from a completely different screen adds a track to that playlist via a bottom sheet. That flow updates the backend and the global composition — but it has never heard of the detail controller, which is still alive and still holding its stale list. Returning to the playlist shows the old tracks while the card's count already shows the new number. One resource, two numbers, on screen at once.

The naive repairs both fail. Having the mutating screen push into the detail controller reintroduces exactly the N-to-M coupling that centralizing ownership removed. Marking the detail controller `autoDispose` looks like it should work and does not — see below.

## Solution

Make the local view **pull** from the global owner, driven by change rather than by re-entry.

In the local controller's `build(arg)`, listen to the global state narrowly selected down to *this id's* composition:

```dart
ref.listen(
  globalStateProvider.select(
    (s) => s.items.where((i) => i.id == arg).firstOrNull?.memberIds
        ?? const <String>[],
  ),
  (_, next) => _onCompositionChanged(next),
);
```

When it fires, compare the incoming composition as a **`Set`** against the ids already materialized locally. Differ → refetch the heavy data, and re-read the updated lightweight object from the global owner so derived values like counts stay consistent with the list. Equal → do nothing.

**The Set comparison is the load-bearing part**, and it does two jobs that are easy to conflate:

1. **It breaks the self-trigger loop.** The local controller's own edits are written to the global owner, which fires the listener right back at it. Without the guard, every local edit causes a redundant refetch of data the controller just produced.
2. **It makes correctness independent of collection equality.** A `select` returning a `List` fires on every rebuild of the global state, because list identity changes even when contents do not. The explicit Set diff — not the selector — is what decides whether a refetch is warranted. Relying on the selector to filter would make the behavior depend on an equality guarantee Riverpod does not provide here.

**Why not `autoDispose`.** Its trigger is "no listeners remain", which fires only when the screen is fully torn down. The common path for an external mutation is the user leaving the detail screen mounted — switching tabs in a shell, or pushing a route above it — performing the mutation elsewhere, and coming back. The screen never lost its listeners, so `autoDispose` never fires, and the stale view survives. The reliable trigger is "the shared composition changed", not "the screen was entered again".

**Refetch rather than in-memory reconcile.** Diffing locally and patching only what changed sounds more efficient but usually buys nothing: additions need the full entity payload anyway, so the network call happens regardless. Refetch when the data is small and external mutations are rare; reconcile only when profiling says the refetch hurts.

## When to apply

- A global owner holds a light composition and a per-id controller holds the heavy materialization of the same resource
- The resource can be mutated from screens that do not know the detail controller exists
- The detail controller is long-lived, or early-returns on re-entry, so re-entering does not refresh it
- Derived values (counts, badges) are read from the global owner while the list is read locally — they must not disagree

## When NOT to apply

- The heavy data is cheap enough that the detail screen can simply refetch on every entry — do that instead, it is far less machinery
- The detail view is the only writer of the composition; with no external mutations there is nothing to react to
- The global owner already holds the heavy entities, so no second cache exists — the screen just reads the global state

## Trade-offs

- (+) External mutations propagate without any mutator knowing which views are open
- (+) Ownership stays singular: the global owner remains the only source of truth for composition
- (+) The listener is narrow (`select` down to one id), so unrelated global changes cost nothing
- (-) Each such controller carries the listener and the Set-diff guard — real boilerplate, and omitting the guard produces an infinite refetch loop rather than a visible error
- (-) Two caches still exist; this makes them converge rather than collapsing them into one
- (-) A refetch is a network round trip in response to a state change, so a burst of external mutations produces a burst of fetches

## Common pitfalls

- Comparing lists with `==` instead of Set contents — order differences trigger pointless refetches, and Dart list equality is identity-based anyway
- Omitting the "is this my own edit echoing back" guard, which turns every local mutation into a refetch and can loop
- Reacting to the whole global state instead of this id's slice, so unrelated resources' mutations refetch this one
- Refetching the heavy data but not re-reading the light object, leaving the list correct and the count stale — the exact inconsistency the pattern exists to remove
