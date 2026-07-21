---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-stateful-shell-route
keywords: [hero, hero-tag, stateful-shell-route, indexed-stack, shared-element, transition, assertion]
related: [[adr-004-go-router-stateful-shell]]
---

# Hero tag collision across simultaneously-alive shell branches

## Symptom

The app crashes on the frame after a route transition starts, with an assertion from the Hero machinery:

> There are multiple heroes that share the same tag within a subtree.

The crash fires when pushing a **top-level** route on the root navigator — opening `/profile`, for example — not when navigating inside a tab. The destination screen has nothing to do with the duplicated tag, which makes the stack trace point somewhere unrelated to the actual problem.

The failure is typically **latent for a long time**: the duplicate tags sit harmlessly in the widget tree for weeks, and the crash only appears when some unrelated change causes a root-level transition to start running the hero scan. That timing makes the recent commit look guilty when it merely exposed a pre-existing structural fault.

## Root cause

`StatefulShellRoute.indexedStack` keeps **every** branch mounted simultaneously — that is the whole point, it is how each tab preserves its own stack. The consequence is that widgets from all branches coexist in one subtree at the same time.

If a shared widget rendering `Hero(tag: 'playlist-image-$id')` appears in two branches at once — the same carousel section reused on both a Home tab and a Music tab, say — then two heroes with an identical tag are alive in the shell route's subtree concurrently. Within a single branch this could never happen, which is why the duplication is easy to introduce without noticing.

Two mechanics make this worse than it first appears:

**The scan is unconditional on the source route.** `Hero._allHeroesFor` walks the *source* route's subtree on every `PageRoute` -> `PageRoute` transition, regardless of whether the destination has any matching hero at all. A destination hero is **not** required to trigger the crash — the duplicate in the source alone is enough. So a transition that involves no shared-element animation whatsoever still asserts.

**The trigger is the root navigator.** When the shell is the source route and a root-level push begins, the root `HeroController` scans the shell subtree — which contains all branches at once. Navigating within one branch scans a narrower subtree where only that branch's copy is present, so in-tab navigation appears fine indefinitely.

## Fix

Make the tag unique per **placement**, not per entity. If the entity id alone is the tag, any second placement of that entity collides; the tag must also encode where it is rendered:

```dart
Hero(tag: 'playlist-image-$id-$placement', ...)
```

where `placement` distinguishes the surfaces (`'home'`, `'music'`). Take the placement as a required parameter on the shared widget so a new call site cannot forget it.

**The destination must receive the same tag.** This is the step that is easy to miss: uniquifying only the source stops the crash but silently breaks the shared-element animation, because the destination's tag no longer matches any source hero. The transition degrades to a plain push with no visible error — a regression that trades a loud crash for a quiet one. Thread the placement through the navigation call (via `extra`, a path parameter, or a constructor argument) so the detail screen builds its hero with the identical tag.

If the shared-element transition is not actually valuable on a given surface, the simpler resolution is to render a `Hero` on only one of the branches, or to drop the hero from that widget entirely. A shared-element animation nobody notices is not worth a uniqueness invariant that every future call site must uphold.

## Where this is likely to occur

- A reusable card or section widget containing a `Hero`, rendered on more than one tab of an `indexedStack` shell
- Shared "featured" / "recommended" carousels driven by one provider and deliberately surfaced in several places
- Apps that mix shell-branch routes with top-level root routes (profile, settings, auth) — the root transition is the trigger
- Any codebase where hero tags are built from an entity id alone, with no placement component

## How to detect early

- Grep every `Hero(tag:` construction and ask whether the tag's inputs are unique across **placements**, not just across entities. A tag interpolating only an id is a standing risk.
- When a shared widget containing a `Hero` gains a second call site, treat that as the moment to introduce a placement parameter — the duplicate is created there, even though the crash arrives much later.
- If a hero-tag assertion appears right after a routing or transition change, look for a pre-existing duplicate rather than auditing the change itself. The change altered *when* the scan runs; it rarely created the duplicate.
