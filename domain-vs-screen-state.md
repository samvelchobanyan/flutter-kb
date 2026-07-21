---
type: principle
severity: required
status: stable
applies_to: any-flutter-project
keywords: [riverpod, state-management, notifier, domain-state, screen-state, controller, single-source-of-truth]
related: [[layered-architecture]], [[freezed-immutable-state]], [[selector-providers-for-rebuilds]], [[reactive-derived-view]], [[repo-side-effects-on-domain-state]]
---

# Domain state vs. screen state

## Rule

Distinguish two kinds of state by **how many screens consume them**, and place each in its proper layer:

- **Domain state** — consumed by two or more screens. Lives in `domain/state/` as a `Notifier<T>` (or `AutoDisposeNotifier<T>`) exposing a Freezed state class and business operations (`load`, `toggle`, `refresh`). Long-lived across screen navigation.
- **Screen state** — consumed by exactly one screen. Lives next to the screen in `features/<feature>/presentation/<screen>/<screen>_controller.dart`. Short-lived; dies when the screen unmounts.

The criterion is the **consumer count**, not the category of the data. A `List<Album>` or a `User` read by exactly one screen is screen state and belongs in that screen's controller, despite being made of domain entities. A boolean flag read by three screens is domain state.

Screens watch domain state directly via `ref.watch(domainStateProvider)`. Screens **MUST NOT** proxy domain data through their controller (no `controller.articles` that internally reads `newsState.articles` — the screen reads `newsStateProvider` itself).

## Rationale

Two failure modes this rule prevents:

1. **Putting shared data in a screen controller.** Then a change made on one screen is invisible to the other consumers until they refetch, and the two copies drift. The controller also dies with its screen, taking data that outlives it.
2. **Proxying domain data through the screen controller.** Then every domain change rebuilds the controller, which rebuilds the screen. Selector-based optimization becomes impossible (see [[selector-providers-for-rebuilds]]); the controller becomes a shadow copy of the domain state that has to be kept in sync.

The split also matches a deeper concern: domain state is *what the app knows*; screen state is *what the user is looking at right now*. They have different lifecycles and ownership.

**Why the consumer count and not "is this business data".** The categorical version of this rule — business data goes global, UI-only state stays local — reads well but is not decidable in practice, and it fails in both directions. It produces false positives, flagging a screen's own `List<Album>` as a layering violation merely because albums are domain entities, when that list is meaningful only while the screen is mounted and promoting it would force long-lived caching of throwaway data. It produces false negatives just as easily, excusing genuinely shared data because it "doesn't feel like business data".

The consumer count avoids both because it is a property of the code rather than a judgement about the data: grep the provider, count the screens that read it. It also names the actual harm directly — the reason shared data must be centralized is that multiple readers can disagree, and a value with one reader has nothing to disagree with.

## Implications

- A concept that two or more screens read → new file in `domain/state/` with `<concept>_state.dart` (Freezed state class + `Notifier` + provider). `PlayerState` qualifies because the mini-player, the full player, and the queue screen all read it — not because playback is conceptually "business data"
- A list of domain entities read by exactly one screen stays in that screen's controller. Most "my albums" / "search results" / "artist detail" style lists are of this kind: they are domain entities, they are also screen state
- **Promotion is a real event.** State that starts with one consumer and gains a second must move to `domain/state/`; leaving it in place is the moment the drift starts. The worked example is a profile: `ProfileState.user` began as ProfileScreen's own data, then EditProfileScreen started reading it to prefill its form and re-invoking `getUser()` after saving. Two readers, two lifetimes, one of them stale after every edit. When adding a second consumer, moving the state is part of the change, not a follow-up
- A screen with no UI-only state → no controller file at all; `<screen>_screen.dart` reads `domainStateProvider` directly
- A screen with UI-only state → `<screen>_controller.dart` next to the screen, exposes only the UI state, never the domain data
- Domain Notifiers extend `Notifier<T>` (Riverpod 2 modern API). `AutoDisposeNotifier<T>` is used when state should die with the last listener (e.g., a dashboard that should re-fetch when re-entered). **`StateNotifier` is not the current canonical** — older spec examples may show it, but every Notifier on `dev` uses `Notifier` / `AutoDisposeNotifier`. New state classes should follow the current code, not the older spec snippets.
- UI-only toggles that look domain-like aren't. Example: `MiniPlayerWidget._isMusicMode` (which panel is expanded in the mini-player) is screen state and lives inside the widget's `State`; it is intentionally separate from `PlayerState.mode` (what the player is actually playing). The two can be in opposite values simultaneously without contradiction — the user can toggle which panel is shown without changing playback.

## When this principle does NOT apply

- Trivial widget-local state (a `TextEditingController`, an `AnimationController`, an `_isHovered` flag) belongs in widget `State`, not in a Riverpod controller. The rule targets state that other widgets might care about, not every `setState`.
- Forms with multi-step flows (registration, password reset) may legitimately hold transient cross-screen state in a domain-state Notifier even though the data isn't "business" in the traditional sense — once the flow completes, the Notifier resets. This is the consumer-count rule working as intended: several screens read the flow's state, so it is domain state regardless of how un-businesslike it looks.
- Counting consumers requires judgement when a screen is reached by several routes or rendered in several places. Count **distinct reading sites**, not route entries: one screen pushed from three places still has one consumer.
