---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-go-router-custom-transitions
keywords: [go-router, custom-transition-page, ios, swipe-back, edge-gesture, material-page, cupertino]
related: [[adr-004-go-router-stateful-shell]]
---

# CustomTransitionPage silently drops the iOS swipe-back gesture

## Symptom

On iOS, the edge-swipe-back gesture works on some screens and does nothing on others, with no error and no visual feedback. The user reports it as "swipe back is broken on the news detail screen" — naming one screen, because that is where they happened to notice it.

Investigation naturally starts on the named screen, looking at its scroll views, gesture detectors, and nested `Navigator`s. Nothing there is wrong. Meanwhile the same gesture works on visually similar screens in the same app, which makes the failure look arbitrary.

## Root cause

The discriminator is the **page type used to build the route**, not anything about the screen.

Flutter wires the Cupertino edge-back gesture through `CupertinoRouteTransitionMixin`, which is mixed into `MaterialPageRoute` and `CupertinoPageRoute`. A route only gets the gesture detector if its `Route` carries that mixin. `MaterialPage` and `CupertinoPage` produce such routes; `CustomTransitionPage` does not — it builds a plain `PageRoute` with the supplied `transitionsBuilder` and no gesture support.

In go_router this maps onto a syntactic distinction that is easy to overlook when scanning a route table:

- `builder:` — go_router wraps the widget in a `MaterialPage` (or `CupertinoPage` on iOS) → gesture present
- `pageBuilder:` returning a `CustomTransitionPage` → gesture absent

So every screen routed through a custom transition helper loses swipe-back at once, and the affected set is exactly "screens whose route entry uses `pageBuilder:`". Since custom transitions are typically added to detail screens only, the result is a scattered, screen-looking pattern of breakage that has a single structural cause.

The trade is real and unavoidable in this form: `CustomTransitionPage` buys a custom animation and costs the platform back gesture.

## Fix

Decide which of the two matters more for the affected routes.

**Prefer the platform gesture (recommended default).** Route through `builder:` and accept the platform transition. On iOS the Cupertino push/pop animation is what users expect anyway, and the gesture is a stronger affordance than a bespoke fade. This is usually the right call for detail screens reached from lists.

**Keep the custom transition.** Then the gesture must be reintroduced deliberately — either by returning a `CupertinoPage` with a custom `transitionsBuilder` where the desired effect allows it, or by using a page implementation that mixes in `CupertinoRouteTransitionMixin` so the route participates in edge-gesture handling. A `GestureDetector` bolted onto the screen body is not equivalent: it will not coordinate with the route's transition animation, so the interactive drag-to-dismiss feel cannot be reproduced that way.

Whichever is chosen, apply it uniformly. The worst outcome is a mixed route table where the gesture's availability depends on which screen the user is looking at.

## Where this is likely to occur

- go_router projects with a shared transition helper (`fadeSlidePage`, `slideUpPage`) applied to a subset of routes
- Apps where custom transitions were added late, screen by screen, so the route table drifted into two styles
- Android-first teams — the gesture is iOS-only, so the regression is invisible in day-to-day development and surfaces in device testing or user reports
- Route tables long enough that `builder:` and `pageBuilder:` entries are not visible on one screen

## How to detect early

- Bisect by page type, not by screen. Group routes into `builder:` versus `pageBuilder:` and check whether that split predicts the working set exactly — it usually does, immediately.
- Treat "gesture broken on exactly one screen" as suspicious framing. Verify the full inventory before accepting that the failure is screen-specific.
- When introducing a custom transition helper, note the gesture cost at the definition site. The helper is where the decision is actually made, even though the consequence appears at every call site.
