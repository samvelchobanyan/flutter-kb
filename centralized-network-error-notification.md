---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [dio, interceptor, error-handling, snackbar, event-stream, scaffold-messenger, notification, network-error]
related: [[api-client-with-config-and-interceptors]], [[event-stream-vs-state-for-discrete-events]]
---

# Centralized network-error notification

## Problem

"Add a try/catch everywhere" conflates two concerns that do not belong together. One is control flow and recovery — rolling back an optimistic update, moving a screen into an error state — which only the call site knows how to do, because only it knows what it was attempting. The other is telling the user something failed, which is identical everywhere: the same "no connection" or "server error" message, regardless of which request produced it. Handled per call site, the notification half is either duplicated into every catch block or, more often, forgotten — so mutations fail silently while the user waits.

## Solution

Split the two. Keep recovery local; centralize notification.

A Dio `ErrorInterceptor` maps every `DioException` to an app-level enum — `NetworkErrorKind { noConnection, timeout, server, unknown }` — and, for requests that warrant a user-facing message, pushes that enum into a `NetworkErrorService` built on a broadcast `StreamController`. A `StreamProvider` exposes the stream, and a single listener at the app root shows one SnackBar through a `GlobalKey<ScaffoldMessengerState>` on `MaterialApp.router`.

Three decisions make it work:

**The interceptor never swallows.** It always calls `handler.next(err)`, so the exception still propagates and call-site control flow — the optimistic rollback in a registry, a screen's error branch — runs exactly as before. Notification is added alongside recovery, not in place of it.

**Filter by method, not by tagging every request.** Notify on non-GET requests only (unless the request opts out via `extra['silent'] == true`). This exploits an existing asymmetry: load-GETs already surface their own failure through a loading→error→Retry UI, while mutations surface nothing. Method is a free discriminator that avoids double-messaging without per-request bookkeeping.

**Model the notification as an event, not as state.** A broadcast stream, not an error field on some state object. This dodges the same-value-dedup trap: two identical 500s in a row are two distinct events, but two identical *values* in state look like no change, so a state-based approach would silently drop the second SnackBar — see [[event-stream-vs-state-for-discrete-events]].

## When to apply

- Any app with a Dio (or interceptor-capable) client where failures should be surfaced uniformly
- When mutations currently fail silently while loads have their own error UI
- When the same user-facing error message is wanted app-wide, decided in one place

## When NOT to apply

- Apps small enough that a handful of call sites surface their own errors adequately — a global channel is overhead
- When every failure genuinely needs a bespoke, context-specific message — there is no cross-cutting notification to centralize
- Non-Dio stacks without an interceptor seam — the pattern needs one choke point where all requests pass

## Trade-offs

- (+) Recovery stays where the knowledge is (the call site); notification lives where it is identical (one root listener)
- (+) Mutations get user feedback without touching any mutation code
- (+) The event-stream model is immune to the consecutive-identical-error dedup bug
- (+) `handler.next(err)` keeps the interceptor transparent to existing control flow
- (-) A global SnackBar means a global look; per-request customization needs the `silent` escape hatch plus a local message
- (-) Two mechanisms now touch every failure (interceptor + call site); understanding one error path means knowing both are involved
- (-) The `GlobalKey<ScaffoldMessengerState>` is app-wide singleton state, with the coupling that implies

## Common pitfalls

- Letting the interceptor swallow the error (returning without `handler.next`), which silences the call-site rollback the app still depends on
- Putting the error in state instead of an event stream, reintroducing the same-value-dedup drop on repeated failures
- Notifying on GETs too, so a failed load shows both its own error UI and a SnackBar
- Forgetting the `silent` opt-out, leaving no way for a call site that handles its own failure to suppress the global message
