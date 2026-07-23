---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [single-source-of-truth, riverpod, notifier, persisted-state, race-condition, state-sync, drift]
related: [[domain-vs-screen-state]], [[repo-side-effects-on-domain-state]], [[event-stream-vs-state-for-discrete-events]]
---

# One concept with two sources of truth

## Symptom

A single logical concept is represented in two places that different layers read, and a mutation touches only one of them. The two drift, and the drift wears the disguise of something else — most often a platform-specific or timing bug, because whether the stale copy is observed depends on incidental build or navigation order.

The classic presentation: a feature is gated correctly by one reader (the app lets you in) but the UI that belongs to that state never appears (the widget that reads the other copy sees the old value). It looks like a layout or safe-area problem on one platform, because that is where the timing happened to expose it.

## Root cause

When a `Notifier` owns a piece of state that *also* has a persisted or cached mirror read elsewhere, correctness depends on every mutation going through the Notifier — because only the Notifier updates both copies. A write that goes straight to the backing store updates one copy and leaves the other whatever it was.

The Notifier's `build()` reflects the backing store only if it happens to run *after* the store was written. Nothing guarantees that ordering, so the symptom is a build-timing race: sometimes the rebuild lands and the copies agree, sometimes it does not and they disagree. That non-determinism is exactly what makes it look platform-specific rather than structural.

A reliable diagnostic signal: a correct owner-method exists — the one that updates both copies — and has **zero call sites**. The mutation is happening somewhere else, by writing the backing store directly. A method that should be the only write path but is called nowhere is a smell worth grepping for.

## Fix

Route every mutation through the owner. The `Notifier` that owns the concept is the only thing allowed to write it; it updates its own state *and* the backing store together. Nothing else writes the store directly.

Concretely: if there is a `continueAsGuest()` / `setLoggedIn()` / `selectPlan()` on the Notifier that sets both the in-memory state and the persisted flag, callers must invoke that — never `storage.setX()` on its own. The backing store is an implementation detail of the owner, not a second public API for the same concept.

If two readers genuinely need different shapes of the same truth (a router redirect wants a synchronous flag; widgets want a reactive state), that is fine — they can both derive from one owner. What is not fine is two independent *writers* keeping two independent copies.

### Worked example — guest mode

go_sport represented "is this a guest session" twice: a persisted `TokenStorage.choseGuest` flag read synchronously by the go_router redirect, and an `AuthGuest` state in the auth `Notifier` read by widgets. The "Continue as guest" button called `tokenStorage.setChoseGuest()` directly, bypassing the Notifier's `continueAsGuest()` (which sets *both* the flag and `state = AuthGuest`).

Result: the router saw a guest (content was browsable) but `authProvider` never became `AuthGuest`, so the guest-only UI — a session timer bar gated on `ref.watch(authProvider) is AuthGuest` — never rendered. Whether the Notifier's `build()` happened to re-read the flag after it was set was a race, so the bar appeared on iOS and not Android, and the bug was first chased as a safe-area layout problem. The fix was one line of routing: send the button through `continueAsGuest()`. The `continueAsGuest()` method already existed and had no callers — the smell that would have pointed straight at the cause.

## Where this is likely to occur

- Any concept with both a persisted mirror (SharedPreferences, secure storage) and a live state object, read by different layers
- Router redirects that read a synchronous flag while widgets read a reactive provider for the same fact — two readers invites two writers
- Caches fronted by a state object, where a shortcut writes the cache directly "just this once"
- Auth, onboarding-completed, feature-flags, chosen-plan — small booleans that outlive a screen and get read from more than one layer

## How to detect early

- When a concept has both a stored form and a state form, find the single write path and confirm it is the *only* one. Grep the backing-store setter: every call should be inside the owner.
- An owner-method that updates both copies but has zero call sites is the signature of this bug. It means the real mutations are going around it.
- Treat "works on one platform, not the other" for a state-visibility problem as a possible provider-not-rebuilt race before assuming layout. The platform difference is often just which build order that platform happened to produce.
