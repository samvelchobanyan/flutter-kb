---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-stateful-shell-route
keywords: [go-router, stateful-shell-route, modal, bottom-sheet, back-gesture, pop-route, navigator]
related: [[adr-004-go-router-stateful-shell]]
---

# Modal over StatefulShellRoute corrupts system back (go_router <= 14.8.1)

## Symptom

An app using `StatefulShellRoute.indexedStack` opens a bottom sheet with `showModalBottomSheet(useRootNavigator: true)`. The sheet itself opens and dismisses normally — nothing looks wrong at that moment. Afterwards, the Android system back gesture stops popping within the active branch and exits the app instead, at **any** stack depth. The user is three screens deep inside a tab, presses back, and the app closes.

The symptom outlives the modal: once a root-level imperative route has been involved, back stays broken for the rest of the session. It reads as a bug in the app's own shell wiring, which sends investigation toward branch navigator keys, `PopScope` handlers, and the shell widget — none of which are the cause.

## Root cause

A back-dispatch bug in `go_router` itself, fixed in 15.x and later.

In go_router 14.8.1, `GoRouterDelegate.popRoute` resolves the target through `_findCurrentNavigator()` — singular. It returns exactly **one** navigator and gives up if that navigator's `maybePop` returns false. A modal opened imperatively on the root navigator is not tracked in go_router's route matching, so after it participates, the resolution collapses to the root navigator permanently. The root navigator has nothing to pop, `maybePop` returns false, no fallback is attempted, and the back event propagates to the OS as "exit app".

Later versions rewrote this as `_findCurrentNavigators()` — plural. It builds a prioritized list ordered `[branch navigators... -> root]` and `popRoute` tries each in order, so an unpoppable root navigator no longer terminates the dispatch.

The app code is correct in both versions. `useRootNavigator: true` is the right choice for a full-screen or shell-spanning modal — it is the framework's dispatch that was incomplete.

## Fix

Upgrade `go_router` to `>= 15.0.0`; 17.x is the recommended target. No application-code change is required — the fix is entirely inside the delegate.

Breaking changes along the 14 -> 17 path are narrow and easy to audit:

- **15.0.0** — URLs became case-sensitive. Only affects projects with uppercase characters in route paths. `caseSensitive: false` is available as an escape hatch.
- **16.0.0** — `GoRouteData` API changes. Only affects projects using `go_router_builder`.
- **17.0.0** — `ShellRoute` notifies `GoRouter` observers by default. Only affects projects with custom `NavigatorObserver`s attached.

A project that uses none of these three features migrates with zero call-site changes.

**Do not** reach for the historical workaround: resolving the active branch's navigator key and passing `useRootNavigator: false` to keep the modal inside the branch. That approach was a patch for this framework bug, not an architectural improvement. It adds a resolver dependency threaded through the shell, constrains modals to the branch subtree (so they render under the bottom navigation bar rather than over it), and becomes dead weight the moment the framework is current. If such a resolver exists in a project on a modern go_router, removing it is the correct move.

## Where this is likely to occur

- Projects pinned to a go_router major by a caret constraint (`^14.2.0` silently caps at 14.x while upstream moved to 17.x) — the pin is invisible until someone reads `pubspec.lock`
- Any `StatefulShellRoute.indexedStack` app that opens full-screen modals: players, search overlays, story viewers, option sheets
- Codebases where a branch-navigator-key resolver was added long ago and never revisited — it may be masking the same bug rather than solving anything
- Long-lived projects where dependency bumps are deferred because "navigation works fine"

## How to detect early

- The break is **stateful**: back works correctly on a fresh launch and only degrades after a modal has been opened once. If back misbehaves only in the second half of a session, suspect dispatch, not stack construction.
- It reproduces at any branch depth, and identically across every branch — an app-code bug in shell wiring would normally be depth- or branch-specific.
- Check `pubspec.lock` for the resolved `go_router` version before reading any app code. A resolved version at or below 14.8.1 with this symptom is a near-certain match.
- Reading `popRoute` in the resolved package source in the pub cache confirms it in under a minute: singular `_findCurrentNavigator` means affected, plural `_findCurrentNavigators` means fixed.
