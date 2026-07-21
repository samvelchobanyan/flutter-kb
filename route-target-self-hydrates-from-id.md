---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [go-router, navigation, deep-link, extra, detail-screen, hydration, route-parameter]
related: [[adr-004-go-router-stateful-shell]], [[domain-vs-screen-state]]
code_refs:
  - go_sport/lib/features/artists/presentation/artist/artist_screen.dart
  - go_sport/lib/features/artists/presentation/artist/artist_controller.dart
  - go_sport/lib/core/navigation/app_router.dart
---

# Route target self-hydrates from an id

## Problem

A detail screen is first built for one caller — a list that already holds the full entity — so it is declared as requiring that entity, passed through go_router's `extra`. This is convenient and gives an instant first paint with no loading state.

Then a second caller appears that has only an id. The music player knows a track's `artistId` and `artistName` but not the `Artist` object. A push notification carries an id in its payload. A deep link is just a URL. None of them can satisfy the screen's contract, and the options all look bad: fetch the entity at the call site purely to hand it over (leaking the detail screen's data needs into the player), duplicate the screen, or pass a half-built entity with empty fields and hope nothing reads them.

The root problem is that `extra` is an in-memory object handed between widgets. It cannot survive a cold start from a deep link, cannot be reconstructed from a URL, and is unavailable to any caller that did not already load the entity. A screen that *requires* it is reachable only from the places that happen to hold it.

## Solution

Invert the contract: the **id** is required, the entity is an optional hint.

```dart
class ArtistScreen extends ConsumerStatefulWidget {
  final String artistId;
  final Artist? artistHint;

  const ArtistScreen({super.key, required this.artistId, this.artistHint});
}
```

The route takes the id from a path parameter and the hint from `extra`, which is null-tolerant by construction:

```dart
GoRoute(
  path: 'artist/:id',
  builder: (context, state) => ArtistScreen(
    artistId: state.pathParameters['id']!,
    artistHint: state.extra as Artist?,
  ),
)
```

The screen watches an id-keyed controller that fetches the entity, and renders from `loadedEntity ?? hint`, falling back to a placeholder or skeleton only when both are absent. Three cases fall out of one expression:

- **Hint present** — instant first paint from the hint, then a seamless swap to loaded data. Callers that already hold the entity behave exactly as before.
- **Hint absent** — skeleton while the fetch runs. Deep links, notifications, and partial-data callers work with no special handling.
- **Both absent (fetch failed)** — the screen's existing error branch, which it needed regardless.

The fetch is usually free. A detail screen already calls the backend for its secondary content — an artist's albums, an album's tracks — and that endpoint typically returns the parent entity in the same response, previously parsed and discarded. Widening the repository method from `getArtistAlbums` to `getArtistDetails` returning `(artist, albums)` adds no network round-trip.

## When to apply

- A detail screen has, or plausibly will have, more than one caller
- The app supports deep links or notification taps that target detail screens
- Some callers legitimately hold only an id — players, search results, embedded references in text
- The detail screen already fetches something from an endpoint that also carries the parent entity

## When NOT to apply

- The screen is genuinely reachable from exactly one place and always will be — a wizard step, a confirmation screen
- There is no endpoint that can fetch the entity by id, so the hint is the only possible source of data
- The entity is not server-backed and exists only in memory (an unsaved draft, a transient selection)

## Trade-offs

- (+) The screen becomes reachable from anywhere with just an id; deep linking and notification routing need no per-screen special-casing
- (+) Existing list navigations are unchanged — they keep passing `extra` and keep their instant paint
- (+) Removes the pressure on callers to load data they do not otherwise need
- (+) The route URL becomes self-sufficient, which is what makes cold-start deep links work at all
- (-) The screen must handle a loading state it previously did not have, including a skeleton for the header region
- (-) Two data paths (hint and loaded) coexist and can briefly disagree if the hint is stale
- (-) A fetch fires even when the hint is present, unless deliberately skipped — usually the right trade, since it refreshes possibly-stale list data

## Common pitfalls

- Reading the hint once into local state instead of preferring loaded data on each build — the screen then keeps rendering the stale hint after the fetch completes
- Making the placeholder structurally different from the loaded layout, so the screen visibly jumps when data arrives; a skeleton matching the real layout avoids the shift
- Casting `extra` without a null-safe cast (`state.extra as Artist` rather than `as Artist?`) — this crashes precisely on the id-only entries the pattern exists to support
- Keeping the hint type as the route contract in spirit — if any code path still assumes the hint is non-null, the screen has not actually become id-addressable
