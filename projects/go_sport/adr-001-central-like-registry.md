---
type: decision
severity: required
status: stable
date: 2026-06-01
applies_to: project: go_sport
keywords: [likes, favorites, registry, cross-screen-sync, riverpod, notifier, optimistic-update, domain-state]
supersedes:
superseded_by:
related: [[domain-vs-screen-state]], [[repo-side-effects-on-domain-state]], [[reactive-derived-view]], [[selector-providers-for-rebuilds]]
code_refs:
  - go_sport/lib/domain/state/like_registry.dart
  - go_sport/lib/core/auth/auth_state.dart
  - go_sport/lib/features/favorites/presentation/
  - go_sport/lib/features/music/presentation/music/music_screen.dart
---

# ADR-001: Central like registry owning full entity lists

## Context

Likeable entities in go_sport span six types — albums, tracks, programs, artists, playlists, episodes — and the heart icon for any of them appears on many surfaces: detail screens, "My X" library lists, dashboard carousels, artist tabs, the player, and track option sheets.

Each of those surfaces had grown its own copy of the truth. A toggle updated the local screen's state and the backend, and nothing else. An audit of the like functionality turned up a consistent class of bug rather than isolated defects: liking from a detail screen left the corresponding library list stale until app restart; liking from the player left the source list that started the queue stale; favorites lists did not refresh after changes made elsewhere; a detail screen holding an entity snapshot showed the pre-toggle value on return; and one heart button in a track options sheet was wired to nothing but a `debugPrint`. Every "My X" screen additionally re-fetched its own list on visit, duplicating data the app had already loaded.

The underlying shape was six parallel half-solutions: each `MyXController` owned a list, paginated it, and listened for unlike events, with an older partial mechanism bolted onto featured playlists on top. The question was where the truth about "what this user has liked" should live.

## Decision

A single `LikeRegistry` `Notifier` in `domain/state/` owns the canonical liked collections, holding **full entities** rather than identifiers:

```dart
@freezed
class LikeRegistryState with _$LikeRegistryState {
  const factory LikeRegistryState({
    @Default([]) List<Album> likedAlbums,
    @Default([]) List<Track> likedTracks,
    @Default([]) List<Program> likedPrograms,
    @Default([]) List<Artist> likedArtists,
    @Default([]) List<Playlist> likedPlaylists,
    @Default([]) List<Track> likedEpisodes,
  }) = _LikeRegistryState;
}
```

**Population.** `initSession()` runs on authentication and loads every page of each user-collection endpoint, fanned out in parallel, merging results through private `_mergeX` helpers. A `_loadAllPages` helper wraps the paging loop. `clear()` empties everything on logout.

**Toggle.** One method per entity type, taking the whole entity — `toggleAlbumLike(Album)`, `toggleTrackLike(Track)`. It looks the entity up by id and branches: a hit means unlike (optimistic removal, repository `DELETE`, restore the previous list on error); a miss means like (optimistic insert, repository `POST`, then `copyWith(likeId: newLikeId)` on the in-list entity, restore on error).

**Reads.** Surfaces read narrow slices via `select`. A per-entity heart is `likedX.any((x) => x.id == id)`; a dashboard counter is `likedX.length`; a "My X" library screen is the list itself. Because the registry holds entities and not ids, the library screens need no fetch of their own.

**Repository purity.** Repositories perform HTTP and return data. The registry calls them and merges the results; nothing in the data layer holds a reference to the registry. See [[repo-side-effects-on-domain-state]] for why — an earlier iteration of this work did the opposite and it was torn out.

**Entity shape.** The `isLiked` boolean was removed from the five affected entities. Membership in the registry *is* the like state, so a second representation on the entity could only disagree with it.

**Scope boundary.** News articles are deliberately outside the registry and keep their own `isLiked` field. News likes have no library surface, no counter, and no cross-screen consumer — the registry would add machinery with nothing to synchronize.

## Alternatives considered

- **Naive sync-all** — the screen that toggles reaches into the controllers that hold affected lists — rejected: requires a child screen to know its parents, which is both impossible in general and unmaintainable where it is possible.
- **Event bus** — a repository emits like events, every interested controller subscribes — rejected: works, but reintroduces N-to-M coupling, makes concurrent toggles awkward to reason about, and leaves every controller holding its own authoritative copy of the truth. The sync problem is redistributed, not solved.
- **Registry of id-only maps, entities patched by subscribing controllers** — a registry holds `(entityType, id) -> likeId`; list-holding controllers subscribe and patch the `isLiked` field on their entities — rejected after roughly two weeks in the codebase. Ids-as-truth forces every hero widget to wrap its entity in `copyWith(isLiked: ...)` before rendering, keeps the six `MyXController` fetch-and-paginate implementations alive, and — in the form it was built — populated the registry through side effects inside repository methods. Holding entities collapses all three problems at once: the library screens become views, the duplicate fetches disappear, and there is nothing left to patch.
- **Registry holds entities, widgets read the registry directly** — cleanest by dependency count — rejected: splits one entity across two read locations, so a widget rendering an album reads the album from one provider and its like state from another. Reading the entity whole, from one place, was judged worth the extra indirection.

## Consequences

- (+) One source of truth for six entity types; a toggle anywhere is visible everywhere on the next frame
- (+) Six `MyXController` implementations lost their fetch, pagination, and unlike-listener logic and became views over registry slices
- (+) Counters are reactive for free — `likedX.length` through a `select` needs no separate query or invalidation
- (+) No duplicate HTTP when entering a library screen; the data is already loaded
- (+) The `data → domain state` layering inversion is gone; repositories are pure and independently testable
- (+) Roughly 700 lines net removed across ~30 files: a dead `MusicRepository` and its mock, five unused mock repositories, six legacy `toggleX(String)` methods, twelve `isXLiked` / `xLikeId` helpers, six parallel id-maps, and a dead partial like mechanism on featured playlists
- (-) `initSession()` eagerly loads every page of every liked collection at login — bounded here, but it is a startup cost that grows with the user's library
- (-) All six collections live in one state class, so an unrelated toggle rebuilds anything watching the state without a `select`. Narrow selectors are mandatory rather than an optimization
- (-) The registry is a single large file concentrating six near-identical toggle implementations; the repetition is visible and deliberate, since a generic version would need to erase the entity types
- (-) Any new likeable entity type means touching the state class, the toggle set, and `initSession` — the registry is a place that must be edited, not extended from outside

## Notes for future projects

This is recorded as a decision for go_sport, not a recommendation to repeat. It suits this app because the liked collections are small, bounded, user-scoped, and referenced from an unusual number of surfaces — enough that centralizing paid for itself several times over in deleted code alone.

Re-evaluate if any of those stop holding. A library that can reach thousands of entries makes eager full loading untenable and points toward paged queries with an id-set for membership checks. A single likeable entity type, or like state that appears on one screen, does not justify the machinery — news in this same app is the standing example of a case correctly left out.

The two generalizable lessons were extracted separately and are not project-specific: repositories must not write to domain state ([[repo-side-effects-on-domain-state]]), and a per-id heavy view synchronizes with a global light composition by listening and diffing rather than by being pushed to ([[reactive-derived-view]]).
