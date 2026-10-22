# Flutter-kb Log

Append-only chronicle of events in the wiki. Three purposes:
- Historical record of when wiki changes happened
- Accumulator of candidates from project work (deposited by project agents during continuous capture)
- Audit trail for lint and re-ingest operations

## Entry format

```
## [YYYY-MM-DD] <event-type> | <context>
<details over one or more lines>
```

## Event types

- `bootstrap` — initial bootstrap completion or partial bootstrap pass
- `ingest` — routine ingest of new knowledge
- `re-ingest` — update of existing pages from changed sources
- `lint` — lint pass with summary of findings
- `create` — targeted page creation
- `candidate` — deposited by a project agent (continuous capture); pending ingest

## Append rules

- Always append at the bottom; never edit existing entries
- Always use the consistent prefix format for parseability
- Always include date in YYYY-MM-DD format

---

## [2026-05-11] init | wiki repository created
Repository initialized at C:\knowledge\flutter-kb\. CLAUDE.md, four templates in _templates/, empty index.md and log.md in place. Bootstrap pending source materials.

## [2026-05-12] candidate | claude-code-desktop | worktree-vs-main-checkout-invisibility
Type: pitfall
Target applies_to: workflows-using-claude-code-desktop
What: Claude Code Desktop creates session worktrees in `.claude/worktrees/` by default. For workflows that require direct file visibility outside the Claude Code session (e.g. wiki bootstrap where the curator needs to review files in Obsidian), this causes invisible writes — files exist on disk but not where the user expects them.
Why interesting: General pitfall of Claude Code Desktop architecture, not project-specific. Applies to any workflow combining Claude Code Desktop with external file viewers/editors (Obsidian vault, IDE indexer, file watcher).
Context: Encountered during wiki bootstrap; 4 calibration pages were written to the worktree path `C:\knowledge\flutter-kb\.claude\worktrees\competent-ritchie-500b53\` and were invisible in the Obsidian vault rooted at `C:\knowledge\flutter-kb\`. Fixed by explicitly requesting writes to the main checkout path.
Source: Claude Code session 2026-05-12, wiki bootstrap calibration step.
Suggested action: Create new pitfall page during future ingest. Workaround: at session start, explicitly request "write files to `<absolute-path>`" rather than relying on the default working directory; or detect worktree mode at session start and request a relocation.

## [2026-05-12] candidate | flutter-kb-system | canonical-branch-for-wiki-operations
Type: principle (methodology)
Target applies_to: flutter-kb-system (curation methodology, not Flutter engineering)
What: Canonical branch for wiki operations is `dev` (or project-specified default). All bootstrap / ingest / re-ingest / lint verifications are against this branch. Continuous capture during in-project work operates on whatever branch the developer is currently on; candidates are canonicalized at next ingest by checking against the canonical branch.
Why interesting: Resolves ambiguity between "wiki describes current code" (which "current" — `dev` or my current feature branch?) and prevents wiki from being polluted by in-flight feature work that may not ship.
Context: Surfaced during 2026-05-12 bootstrap when a calibration pitfall page was written based on code that existed on a non-`dev` branch but was rolled back / never merged to `dev`. Without a canonical-branch rule, the curator agent has no anchor for verification.
Source: Claude Code session 2026-05-12, after Sam pointed out that `branchNavigatorKeyResolver` (referenced in the calibration `modal-in-stateful-shell-route.md`) does not exist on `dev`.
Suggested action: At next CLAUDE.md update, add an explicit section about (a) canonical branch for wiki operations and (b) the relationship between in-project working-branch continuous capture and canonical-branch verification at ingest. Mirror the rule into go_sport's project CLAUDE.md.

## [2026-05-12] candidate | go_sport | modal-in-stateful-shell-route (deferred investigation)
Type: pitfall (open investigation)
Target applies_to: flutter-projects-with-stateful-shell-route
What: Modal opened via `showModalBottomSheet` inside `StatefulShellRoute` resolves to the root navigator, breaking active branch back-stack. A fix using a branch-navigator-key resolver + `useRootNavigator: false` was previously implemented in go_sport but was rolled back during refactoring (timing and reasoning not recorded). Current state on `dev` branch: `MainShell` no longer accepts a `BranchNavigatorKeyResolver`, `FullPlayerScreen.show()` uses `useRootNavigator: true`. Branch navigator keys remain in `app_router.dart` but are not used to resolve the active branch context.
Why interesting: General Flutter pitfall, not project-specific. Cannot be documented as resolved in wiki until investigation determines whether the problem still manifests in current code and whether the original fix should be restored or a different approach taken. Tabling it preserves honesty of the wiki (no contradiction with code) until the underlying state is clarified.
Context: Caught during 2026-05-12 bootstrap calibration. Calibration page `modal-in-stateful-shell-route.md` was written based on the previous (fixed) state and contradicted current `dev`. Page was deleted from main checkout before being canonicalized; this candidate preserves the knowledge for re-evaluation.
Source: Claude Code session 2026-05-12, comparison of `MainShell` / `FullPlayerScreen.show()` between first-pass reading and `dev`-branch re-reading.
Suggested action: Defer to future ingest after Sam investigates current behavior. Open questions: (1) does the back-stack / swipe-back issue still happen on `dev`? (2) is there a different fix that was applied (e.g. the new `RadioFullPlayerScreen` split changes the navigation model)? (3) was the rollback intentional and the current behavior considered acceptable? Worktree copy of the previously-written page remains at `C:\knowledge\flutter-kb\.claude\worktrees\competent-ritchie-500b53\modal-in-stateful-shell-route.md` and can serve as a starting draft when the investigation concludes.

## [2026-05-12] candidate | go_sport | stateful-shell-with-modal-overlays-pattern (paired investigation)
Type: pattern (paired with modal-in-stateful-shell-route pitfall — same investigation)
Target applies_to: flutter-projects-with-stateful-shell-route
What: stateful-shell-with-modal-overlays pattern — describes the branch-navigator-key resolver + useRootNavigator: false approach for opening modals from inside StatefulShellRoute. Was implemented in go_sport earlier, rolled back during refactoring. Pattern is described in conceptual form but should not be canonicalized in wiki until Sam investigates current dev state.
Why interesting: General pattern, paired with the modal-pitfall it solves. Both are deferred together.
Suggested action: Defer to future ingest after Sam investigates. Reconnect with the parallel modal-pitfall candidate (also recorded 2026-05-12) when canonicalized.

## [2026-05-12] candidate | go_sport | spec-vs-code-divergence-statenotifier
Type: divergence (spec vs code)
Target applies_to: project: go_sport
What: ARCHITECTURE_SPEC.md §8.5 shows `StateNotifier<NewsState>` as the canonical state pattern. Current code uses `Notifier` (Riverpod 2 modern API). The two are functionally equivalent here but stylistically diverge.
Why interesting: SPEC documents need to align with current code to prevent confusion for new team members reading SPEC after onboarding.
Suggested action: At next ingest or methodology review, decide: (1) update ARCHITECTURE_SPEC.md to reflect `Notifier` as canonical, OR (2) wiki canonicalizes `Notifier` (already implicitly does via existing pages) and SPEC is treated as historical. Recommend option (1) — keep SPEC and wiki aligned, update SPEC.

## [2026-05-12] bootstrap | initial wiki population
Created 24 pages from `bootstrap-sources/`, all verified against `go_sport` on branch `dev` (canonical branch for wiki operations).

By type: 6 principles, 10 patterns, 7 cross-project ADRs, 1 pitfall.

Sources used for verification:
- `bootstrap-sources/claude-memory.md` — hypotheses-level distilled memory from Claude.ai sessions
- `bootstrap-sources/discussions/discussion-001-program-reminders.md` — full engineering transcript of the program-reminders feature implementation (rich source for the notifications stack)
- `C:\flutter-projects\go_sport\docs\ARCHITECTURE_SPEC.md` and `DOMAIN_SPEC.md` — project architecture specs (close to code, occasionally lag)
- `C:\flutter-projects\go_sport\lib\` on branch `dev` — canonical source of truth
- `C:\flutter-projects\go_sport\android\`, `ios\`, `pubspec.yaml` — platform configs and dependencies

Significant divergences caught during bootstrap (wiki reflects code in all cases):
- `claude-memory.md` claimed a three-layer Player (Notifier + Handler + PlaybackService); code has two layers (`PlayerNotifier` + `AppAudioHandler`). Documented as [[two-layer-audio-stack]].
- `claude-memory.md` claimed split `PlayerQueueState` / `PlaybackState`; code has a single `PlayerState` with `PlaybackMode` discriminator. Documented as [[dual-mode-player-state]].
- `claude-memory.md` claimed `AuthNotifier` was deliberately deferred; it is in fact implemented as a sealed `AuthState` with a `Notifier`. No standalone page; the implementation is illustrated in [[freezed-immutable-state]] and consumed implicitly by [[domain-vs-screen-state]].
- `claude-memory.md` mentioned `GuestSessionNotifier`; the actual implementation is a plain `StatefulWidget` (`GuestTimerBar`) with a local `Timer.periodic`. No page — the simplest viable choice for this concern, and it was already covered by [[avoid-over-engineering]]-style judgment (which we deliberately did not turn into a page).
- `claude-memory.md` mentioned `pinput` for OTP; not in `pubspec.yaml`. Abandoned.
- `claude-memory.md` described slot-based `TrackTileBase` composition; code uses separate tile classes per context (`TrackTile`, `AlbumTrackTile`, `EpisodeTile`, `ScheduleTile`, etc.). Captured correctly as the first form of [[shared-widgets-stay-dumb]].
- ARCHITECTURE_SPEC.md §8.5 shows `StateNotifier` examples; code uses `Notifier`. Wiki tracks code; spec divergence logged as a separate candidate above.

Deferred candidates from this bootstrap (logged earlier in this file, paired so future ingest processes them together):
- `modal-in-stateful-shell-route` pitfall + `stateful-shell-with-modal-overlays` pattern — paired investigation. A fix using `BranchNavigatorKeyResolver` + `useRootNavigator: false` existed in earlier code but was rolled back on `dev`. Reason unknown; current `dev` uses `useRootNavigator: true`. Both pages are tabled until Sam investigates and decides direction.
- `worktree-vs-main-checkout-invisibility` pitfall — Claude Code Desktop UX, surfaced during the bootstrap calibration when 4 pages were initially written to a worktree path invisible to Obsidian.
- `canonical-branch-for-wiki-operations` principle (methodology) — wiki operations verify against `dev`; in-project continuous capture happens on whatever working branch the developer is on, and candidates canonicalize at next ingest after verification against `dev`. Recommend incorporating into both this CLAUDE.md and `go_sport` project CLAUDE.md at next methodology review.

Calibration four-pack written first and reviewed by Sam: [[dto-to-domain-mapping]], [[local-state-via-storage-class]], [[adr-005-secure-storage-for-tokens]], `modal-in-stateful-shell-route`. The fourth was deleted from main checkout after the post-calibration verification pass revealed code on `dev` did not match the documented fix; that knowledge moved to a deferred candidate. The verification methodology — full sweep against canonical branch before canonicalizing remaining pages — also caught the paired `stateful-shell-with-modal-overlays` pattern before it was written. Both catches demonstrate that the verification step is doing real work.

`projects/go_sport/` remains empty by design; first project-scoped ADRs will arrive via continuous capture during in-project work.

## [2026-05-12] candidate | go_sport | domain-vs-screen-state-criterion-refinement
Type: principle (refinement of existing page)
Target applies_to: any-flutter-project
What: Refine the criterion in [[domain-vs-screen-state]] from categorical ("business data vs UI-only state") to operational ("number of consumers"): **global state** = used by ≥2 screens → lives in `domain/state/` as long-lived `Notifier`; **local state** = used by exactly 1 screen → lives in the feature's screen-controller (`features/<feature>/presentation/<screen>/<screen>_controller.dart`), even if it contains domain entities like `List<Album>` or `User`.
Why interesting: The current wiki criterion is subjective and routinely misclassified. A list-of-domain-entities screen (`my_albums`, `album`, `artist`, `playlist`, `program_details`, `schedule`, `music_dashboard`, `my_favorites`, `my_artists`, `my_playlists`, `my_programs`, `new_episodes`) in go_sport keeps its `List<DomainEntity>` inside the screen controller — and that's correct under the operational rule, because each list is consumed by exactly one screen. Moving them to `domain/state/` would force long-lived caching of data that is only meaningful while the screen is mounted. Under Sam's operational rule the existing project layout is consistent, and the few legitimate violations (e.g. `ProfileState.user` — read by both ProfileScreen and EditProfileScreen) become unambiguously identifiable. The categorical rule produced two failure modes: false positives (flagging legitimate local list-state) and false negatives (excusing shared data because it didn't "feel like business data").
Context: Code review of `user_profile` branch in go_sport. Initial flag was `ProfileState.user` as a "domain entity in screen controller" violation. Audit of all `features/*/presentation/*_controller.dart` revealed ≥12 controllers with `List<DomainEntity>` in their state. Sam clarified the project's actual rule: 1 screen → local; 2+ screens → global. Under that rule, `ProfileState.user` is still a violation (EditProfileScreen reads `profileControllerProvider` to prefill fields and re-invokes `getUser()` after update — explicit cross-screen consumption), but the other controllers are not. The current wiki page [[domain-vs-screen-state]] also gives `MiniPlayerWidget._isMusicMode` as an example of "screen state" — that example fits the operational rule cleanly (only the mini-player consumes it).
Source: Claude Code session 2026-05-12, files: `lib/features/user_profile/profile/presentation/profile/profile_controller.dart`, `lib/features/user_profile/edit_profile/presentation/edit_profile/edit_profile_screen.dart` (cross-screen consumer of `User`), plus audit of `lib/features/{albums,artists,playlists,program_details,schedule,music,favorites/*}/presentation/*_controller.dart`.
Suggested action: update [[domain-vs-screen-state]] — replace the categorical framing ("business data vs UI-only") with the operational consumer-count rule. Preserve the existing examples but reframe their justification (`PlayerState` is shared by mini + full + queue screens → global; `MyAlbumsState` is consumed by my_albums_screen alone → local). Keep the secondary guidance about screen controllers not *proxying* domain state (a screen that needs shared state reads the domain provider directly, not via its own controller). Consider adding `User` / profile data as the canonical worked example of "started local, turned out to be shared, needs to be promoted."


## [2026-05-17] candidate | go_sport | non-exhaustive-loading-guard-on-single-object-screen
Type: pitfall
Target applies_to: any-flutter-project
What: A render guard of the form `if (isLoading && data == null) return spinner;` followed by a body that force-unwraps the data (`data!`) crashes whenever `data == null` while `isLoading == false` — the initial pre-fetch frame and the post-error state. The guard is non-exhaustive over `data == null`.
Why interesting: The project's list screens (my_albums, my_playlists, my_artists, news_list, home, music) all gate the spinner with `isLoading && list.isEmpty`, and that is safe there because an empty list still renders — the uncaught initial/error frame harmlessly falls through to an empty list. Copying that same gate shape to a single-object screen is a trap: a null object cannot fall through to the render branch. Rule: single-object loading screens must gate exhaustively over `data == null` (spinner + error branches must cover every null case); list screens need not. General Flutter pitfall for any project using the manual isLoading/error/data state pattern (vs AsyncValue, which is not used in go_sport).
Context: Code review of user_profile branch. ProfileScreen guarded with `isLoading && user == null` then rendered with `user!` (profile_screen.dart lines 80/86/92/99). Crashes on cold start (Future.microtask getUser hasn't flipped isLoading yet → default ProfileState()) and on any failed profile load (catch sets isLoading:false, leaves user:null). Fix applied: three-way gate mirroring news_list_screen._buildBody — `error != null && user == null` → error widget with Retry; `user == null` → spinner; else render with a null-promoted local `user`.
Source: Claude Code session 2026-05-17, files lib/features/user_profile/profile/presentation/profile/profile_screen.dart, profile_controller.dart; convention compared against lib/features/news/presentation/news_list/news_list_screen.dart and favorites/*/presentation/*_screen.dart.
Suggested action: create new pitfall page. Relates loosely to [[domain-vs-screen-state]] (same screen, different concern). Optionally pair with a small pattern page documenting the project's three-way render gate (loading&&empty → spinner; error&&empty → error; else data) if the curator judges it worth a standalone pattern.

## [2026-05-26] candidate | go_sport | strapi-google-oauth-email-conflict
Type: pitfall
Target applies_to: flutter-projects-with-strapi-users-permissions-oauth
What: Strapi's default `/api/auth/google/callback` endpoint (from the `users-permissions` plugin) does NOT auto-link to an existing user when that user's email already exists with `provider: "local"`. It returns `400 ApplicationError "Email is already taken"`. The same applies to all bundled OAuth providers (Facebook, GitHub, Discord, etc.).
Why interesting: Manifests as a 400 that looks like a client bug (wrong token, wrong endpoint, missing scope) but is actually correct server behavior + a missing product decision. Easy to lose hours assuming the client integration is broken. Also worth recording because Strapi's docs do not surface this clearly — the "Sign in with X" provider docs describe registration but skip the cross-provider conflict case. Resolution requires either (a) a product decision to merge accounts and a backend customization that links by email, (b) using only one provider per user lifecycle, or (c) replacing the bundled callback with a custom endpoint that decides linking policy.
Context: Implementing Google Sign-In in go_sport. Client side reached the Strapi callback successfully (Google picker → access_token via google_sign_in 7.x → GET /api/auth/google/callback?access_token=...). Strapi validated the token against Google, retrieved email, then rejected because the same email was already registered via local password (the developer had logged in with email/password earlier in the same session). Identical flow with a never-before-seen Google account would have succeeded. Same issue would block a "user adds Google login to existing account" UX later.
Source: Claude Code session 2026-05-26, files: `lib/data/repositories/auth_repository_impl.dart` (calls `/api/auth/google/callback`), `lib/core/auth/google_auth_service.dart` (obtains access_token), `lib/features/auth/login/presentation/login/login_controller.dart` (orchestration). Response body received: `{"data":null,"error":{"status":400,"name":"ApplicationError","message":"Email is already taken.","details":{}}}`.
Suggested action: create new pitfall page. Scope is broader than Google — title should reflect the OAuth-provider-vs-local conflict in Strapi generally. Pair-reference any future `[[strapi-mobile-oauth-flow]]` pattern page (not yet logged) that documents the recommended custom-endpoint-with-idToken approach for mobile, since that pattern naturally implements an explicit linking policy and sidesteps this pitfall.

## [2026-05-29] candidate | go_sport | like-state-central-registry
Type: decision (ADR) + pattern
Target applies_to: project: go_sport (decision); any-flutter-project with cross-screen entity-property sync needs (pattern, with caveats)
What: Architecture for keeping like-state consistent across screens. A central LikeRegistry holds the per-user truth as  (no entity payloads). All toggles go through  which owns optimistic update, backend POST, and rollback. List-holding controllers (, ,  for albums; analogous controllers for tracks/programs/artists/playlists) subscribe to the registry via  and patch the  field on entities in their  lists when the registry changes. Data layer seeds the registry from every backend response: for each entity in any response, the registry is brought into sync with the response's  (added/updated for liked, removed for unliked) - the rule is match the backend, not just merge-add. / stay on the entities (no DTO refactor); the entity field becomes a projection of the registry's truth, propagated by the subscribing controllers. Detail screens (e.g., ) keep their existing local  for instant tap feedback and additionally call ; the local copy is transient (dies with the widget) and re-initializes from the now-updated parent list on next entry.
Why interesting: Solves a real, observed bug class in the project - currently like toggles only update one local screen state + backend, so , dashboard carousels, artist tabs, and other detail-screen visits show stale data until restart. Three alternatives were explored and rejected during the session:
  (a) Naive sync-all (child screen directly updates parent controllers) - breaks because the child would have to know its parent.
  (b) Sync-all via events bus (repository emits, each controller subscribes) - works but N->M coupling, gonky concurrency, and each controller still holds an authoritative copy of the truth.
  (c) Registry + widget-reads-registry directly - cleanest by some metrics but Sam pushed back on the ontological split (one entity, two read locations).
The chosen variant (registry + controller-level subscription) preserves entity wholeness (UI still reads  via its controller), centralizes the toggle logic, and uses Riverpod's built-in pub-sub for propagation. The controller-level subscription insight was Sam's - it decouples where the truth lives (one place: registry) from where the subscription happens (at each list controller, not at each widget), which addresses the split-entity discomfort without losing centralization.
Trade-offs accepted: per-list-controller boilerplate (subscribe + scan + patch), discipline requirement that any new list of like-able entities must subscribe, dual writes in detail screens (local setState + registry.toggle) - acceptable because the local copy is transient. Optimistic semantics: registry handles its own rollback; detail screens chain  on 's returned Future to roll back their local .
Context: Sam started by asking for an audit of the like functionality across the project (tracks, albums, programs, playlists, artists, episodes) - what works, what's a dead heart icon, where state is misaligned. Audit found: dead  button in  bottom sheet (debugPrint only), and four parallel sync gaps (hero like -> stale in  and other lists; player like -> stale in source controllers like ; favorites lists not refreshed; AlbumScreen  snapshot stale on return). Discussion converged over ~15 turns from naive solutions through events to the final shape. Existing controllers that hold album lists: ,  (autoDispose family),  (long-lived).
Source: Claude Code session 2026-05-29, files: , , , , , , . No code written - design discussion only.
Suggested action: split into two candidate pages.
  (1) Project-scoped ADR for go_sport - title like  - records the specific decision with the rejected alternatives and the rationale. Should reference the audit findings as evidence for the problem. Sam noted: ?? ???? ??? ? ?????? ??????? ??????? ??? ?? ?? ??? ??????? ??????? ???? ????? ????????????? - so frame it as a project decision, not a recommendation.
  (2) Optional pattern page for any-flutter-project - title like  - extracts the transferable structure: when you need cross-screen sync of a property that's part of a server-returned entity, a small registry + controller-level subscription (not widget-level) preserves entity wholeness while centralizing the write path. Caveats: requires per-list-controller boilerplate, requires discipline at data layer for seeding, transient dual-writes acceptable at detail screens. Could note alternative variants (widget-reads-registry, sync-all events) with their trade-offs as recorded in the discussion.

## [2026-05-29] candidate | go_sport | like-state-central-registry
Type: decision (ADR) + pattern
Target applies_to: project: go_sport (decision); any-flutter-project with cross-screen entity-property sync needs (pattern, with caveats)
What: Architecture for keeping like-state consistent across screens. A central LikeRegistry holds the per-user truth as (entityType, id) -> likeId (no entity payloads). All toggles go through registry.toggle(...) which owns optimistic update, backend POST, and rollback. List-holding controllers (MusicDashboardController, ArtistController, MyAlbumsController for albums; analogous controllers for tracks/programs/artists/playlists) subscribe to the registry via ref.listen and patch the isLiked field on entities in their state lists when the registry changes. Data layer seeds the registry from every backend response: for each entity in any response, the registry is brought into sync with the response's isLiked (added/updated for liked, removed for unliked) - the rule is "match the backend, not just merge-add". Album.isLiked/Track.isLiked stay on the entities (no DTO refactor); the entity field becomes a projection of the registry's truth, propagated by the subscribing controllers. Detail screens (e.g., AlbumScreen) keep their existing local _isLiked for instant tap feedback and additionally call registry.toggle(); the local copy is transient (dies with the widget) and re-initializes from the now-updated parent list on next entry.
Why interesting: Solves a real, observed bug class in the project - currently like toggles only update one local screen state + backend, so MyAlbums, dashboard carousels, artist tabs, and other detail-screen visits show stale data until restart. Three alternatives were explored and rejected during the session:
  (a) Naive sync-all (child screen directly updates parent controllers) - breaks because the child would have to know its parent.
  (b) Sync-all via events bus (repository emits, each controller subscribes) - works but N-to-M coupling, gnarly concurrency, and each controller still holds an authoritative copy of the truth.
  (c) Registry + widget-reads-registry directly - cleanest by some metrics but Sam pushed back on the ontological split (one entity, two read locations).
The chosen variant (registry + controller-level subscription) preserves entity wholeness (UI still reads album.isLiked via its controller), centralizes the toggle logic, and uses Riverpod's built-in pub-sub for propagation. The "controller-level subscription" insight was Sam's - it decouples "where the truth lives" (one place: registry) from "where the subscription happens" (at each list controller, not at each widget), which addresses the split-entity discomfort without losing centralization.
Trade-offs accepted: per-list-controller boilerplate (subscribe + scan + patch), discipline requirement that any new list of like-able entities must subscribe, dual writes in detail screens (local setState + registry.toggle) - acceptable because the local copy is transient. Optimistic semantics: registry handles its own rollback; detail screens chain .catchError on registry.toggle()'s returned Future to roll back their local _isLiked.
Context: Sam started by asking for an audit of the like functionality across the project (tracks, albums, programs, playlists, artists, episodes) - what works, what's a dead heart icon, where state is misaligned. Audit found: dead "Like" button in track_options.dart bottom sheet (debugPrint only), and four parallel sync gaps (hero like -> stale in MyAlbumsController and other lists; player like -> stale in source controllers like PlaylistController; favorites lists not refreshed; AlbumScreen widget.album snapshot stale on return). Discussion converged over ~15 turns from naive solutions through events to the final shape. Existing controllers that hold album lists: MusicDashboardController.featuredAlbums, ArtistController.albums (autoDispose family), MyAlbumsController.albums (long-lived).
Source: Claude Code session 2026-05-29, files: lib/features/albums/presentation/album/album_screen.dart, lib/features/albums/presentation/album/album_controller.dart, lib/features/favorites/presentation/my_albums/my_albums_controller.dart, lib/features/music/presentation/music/music_dashboard_controller.dart, lib/features/artists/presentation/artist/artist_controller.dart, lib/domain/state/player_state.dart, lib/features/shared_widgets/bottom_pop_ups/track_options.dart. No code written - design discussion only.
Suggested action: split into two candidate pages.
  (1) Project-scoped ADR for go_sport - title like adr-like-state-central-registry - records the specific decision with the rejected alternatives and the rationale. Should reference the audit findings as evidence for the problem. Sam noted: "ne fact chto v drugom proekte sdelaem tak zhe no kak reshenie kotoroe bylo mozhno zafiksirovat" - so frame it as a project decision, not a recommendation.
  (2) Optional pattern page for any-flutter-project - title like central-registry-with-controller-subscription - extracts the transferable structure: when you need cross-screen sync of a property that's part of a server-returned entity, a small registry + controller-level subscription (not widget-level) preserves entity wholeness while centralizing the write path. Caveats: requires per-list-controller boilerplate, requires discipline at data layer for seeding, transient dual-writes acceptable at detail screens. Could note alternative variants (widget-reads-registry, sync-all events) with their trade-offs as recorded in the discussion.

## [2026-06-01] candidate | go_sport | central-like-registry-pattern
Type: pattern
Target applies_to: any-flutter-project (Riverpod + multiple screens that share like/favorite state across detail, list, player, and dashboard surfaces)
What: A central `LikeRegistry` Notifier in `domain/state/` owns the canonical lists of liked entities (one `List<X>` per entity type), and every screen reads from it via `ref.watch(likeRegistryProvider.select(...))`. The registry is eagerly populated on auth (`initSession()` loads all pages of every `/api/user-X` endpoint in parallel via a `_loadAllPages` while-loop helper) and cleared on logout. Toggle is a single method per entity type — `toggleAlbumLike(Album)`, `toggleTrackLike(Track)`, etc. — that takes the full entity, looks up by id, and branches: lookup hit → optimistic remove + repo DELETE + rollback on error; lookup miss → optimistic add + repo POST + `copyWith(likeId: newLikeId)` to update the in-list entity + rollback on error. List-of-favorites screens ("My X") become thin views (`ref.watch(registry.likedX)`); dashboards read counts via `registry.likedX.length`; detail screens read the per-entity `isLiked` via `s.likedX.any((x) => x.id == id)`. Repositories stay pure (HTTP → data → return) — the registry calls them and merges via private `_mergeX` helpers, not the other way around.
Why interesting: Cross-screen like state is a recurring problem (player heart vs source-list heart desync; detail-screen toggle invisible on dashboard; pull-to-refresh races; first-load shows empty hearts for already-liked items). go_sport previously had a half-solution per entity type — each `MyXController` owned its list + paginated + listened for unlikes — plus an even older partial workaround for playlists via `FeaturedPlaylistsState.updateLike`. Centralizing collapses 6 controllers' worth of fetch/sync/listen logic into one Notifier, removes the `data → domain state` layer-bend (repos no longer push to a state Notifier as a side effect), eliminates duplicate HTTP on My-X visit, and makes counts reactive for free. The pattern is general: any "small bounded user-scoped collection that is referenced across many screens" (likes, follows, bookmarks, subscriptions, opt-ins) fits the same shape.
Context: Two-day architectural discussion + refactor on `like-refactor` branch of go_sport, May 30 — June 1 2026. Started from a code review of an earlier attempt (single `LikeRegistry` with id-only maps + side-effect syncX in repos) that surfaced four issues: maps-as-truth requires duplicate `copyWith(isLiked: ...)` wrapping in hero widgets, repos that mutate state are hard to test/reason about, the dup HTTP on My-X visit was wasteful, and the partial `updateLike` mechanism on `FeaturedPlaylistsState` was dead. The user proposed enriching at the repo level; we walked through the trade-offs and converged on registry-owns-entities. Built across 7 sequenced steps (dead code → pagination → registry extension → UI toggle migration → My-X as views → counters from registry → final cleanup) with build_runner regenerations between Freezed shape changes. Touched ~30 files; removed ~700 lines net (deleted `MusicRepository` and its mock, 5 unused mock repos, 6 old `toggleX(String)` methods, 12 `isXLiked`/`xLikeId` helpers, 6 parallel id-only maps, `FeaturedPlaylistsState.updateLike`, isLiked field across 5 entities).
Source: Claude Code sessions 2026-05-30, 2026-05-31, 2026-06-01. Reference files: `lib/domain/state/like_registry.dart` (the registry), `lib/core/auth/auth_state.dart` (initSession orchestration), `lib/data/repositories/*_impl.dart` (pure repos), `lib/features/favorites/presentation/my_*/*_controller.dart` (thin view notifiers), `lib/features/music/presentation/music/music_screen.dart` (counters via `select`).
Suggested action: create new pattern page. Naming candidates: `[[central-user-collection-registry]]` or `[[like-registry-pattern]]`. Pair with related pages [[domain-vs-screen-state]] (the registry is the canonical domain state; My-X notifiers become thin views — illustrates the operational consumer-count rule from the 2026-05-12 candidate, since the registry has 10+ consumers) and [[selector-providers-for-rebuilds]] (per-screen reads use narrow `.select` slices: `likedX.length` for counters, `likedX.any((x) => x.id == id)` for per-entity isLiked checks).

## [2026-06-01] candidate | go_sport | repo-side-effects-on-domain-state-anti-pattern
Type: pitfall
Target applies_to: any-flutter-project with layered architecture and Riverpod state
What: A repository method that, after returning data, also pushes that data into a domain-state Notifier as a side effect — for example, `getFavoriteAlbums()` internally calling `_registry.syncAlbumsLikes(albumList)` before returning — creates a layer-bend (data → domain state) and a hidden contract violation. The method signature claims a pure return, but the call mutates app-wide state. Symptoms: testing the repo requires mocking the registry; readers don't see the state mutation from the call site; same repo method called from two places mutates the registry twice; "kольцо" — registry calls repo which calls registry. Cure: the repo returns data only; the caller (the registry's own init method, or a controller) does the merge.
Why interesting: This is the natural shortcut when "I have two callers that need the registry populated, easiest to bundle the side effect into the shared repo method." Tempting because it removes one line from each caller. Cost: every future reader has to inspect repo internals to know which providers mutate when. In go_sport the side-effect approach lived for ~2 weeks before being torn out — readers including the user himself were surprised by what was actually going on when the structure was walked through line by line. The pitfall is independent of the specific registry — it generalizes to any case where a data-layer object reaches into a state Notifier ("the repo writes to the cache as a side effect" is the same shape).
Context: go_sport `like-refactor` branch initial implementation. The repos took a `LikeRegistry` constructor dependency and called `_registry.syncXLikes(items)` inline before returning the list. Initially defended as "centralization" — two callers (initSession and MyXController.loadFavorites) both wanted the registry populated, so we did it in the repo. Then during architectural review the inconsistency became visible: `toggleLike` on the same repo did NOT push to registry (it returned the likeId, registry processed it itself), while `getFavoriteAlbums` did. Same class, two contracts. Refactor: repo became pure (HTTP → entity list); initSession loops pages and calls a private `_mergeX(items)` helper inside the registry; MyXController no longer needs to call the repo at all (subscribes to registry). The "kольцо" disappeared cleanly.
Source: Claude Code session 2026-06-01, refactor across `lib/data/repositories/*_impl.dart` (removed `LikeRegistry _registry` field + `_registry.syncXLikes(...)` calls), `lib/domain/state/like_registry.dart` (sync methods renamed to private `_mergeX`, called from `initSession`'s `_loadAllPages` callback).
Suggested action: create new pitfall page. Relates to [[layered-architecture]] (data layer depending on domain-state is the layering violation) and [[domain-vs-screen-state]] (compounds the operational rule — if a repo mutates a domain state Notifier, the Notifier's consumer set effectively includes "everything the repo touches", which makes the consumer-count argument meaningless). Could also be paired with [[central-user-collection-registry]] above as a contrast: "this is how NOT to keep the registry populated; do this instead."

## [2026-06-01] candidate | go_sport | modal-in-stateful-shell-route (RESOLVED — root cause found)
Type: pitfall (resolution of deferred 2026-05-12 investigation) + decision (dependency upgrade)
Target applies_to: flutter-projects-with-stateful-shell-route (pitfall) + project: go_sport (version decision)
What: The long-deferred `modal-in-stateful-shell-route` pitfall is a back-dispatch bug in **go_router 14.8.1**, not a problem in app code. Opening a bottom sheet via `showModalBottomSheet(useRootNavigator: true)` over a `StatefulShellRoute.indexedStack` corrupts system-back routing: the sheet itself dismisses fine, but afterwards the Android back gesture exits the app at ANY branch depth instead of popping within the active branch. Cause: go_router 14.8.1's `GoRouterDelegate.popRoute` uses `_findCurrentNavigator()` (singular) — returns ONE navigator and gives up if its `maybePop` returns false, so back permanently resolves to the root navigator once an imperative (go_router-untracked) root modal has been involved. Fixed by upgrading go_router 14.8.1 -> 17.2.3, where the logic was rewritten as `_findCurrentNavigators()` (plural, prioritized [branch...->root] list that `popRoute` tries in order). Confirmed fixed by Sam on device.
Why interesting: Resolves the paired deferred candidates (`modal-in-stateful-shell-route` pitfall + `stateful-shell-with-modal-overlays` pattern). Key correction to the old hypothesis: the original `BranchNavigatorKeyResolver` + `useRootNavigator: false` workaround was a patch for an OLD go_router bug; rolling it back was correct, and the right long-term fix is the framework upgrade, not restoring the resolver. The `stateful-shell-with-modal-overlays` resolver pattern should NOT be canonicalized — it is obsolete.
Migration cost (go_router 14.8.1 -> 17.2.3 in go_sport): near-zero. Breaking changes on the path: 15.0.0 case-sensitive URLs (no uppercase paths in routes.dart -> no impact; `caseSensitive: false` available as safety), 16.0.0 GoRouteData (go_router_builder not used -> no impact), 17.0.0 ShellRoute notifies GoRouter observers by default (no custom observers -> no impact). `flutter analyze` clean of API errors after bump; zero call-site changes. Project had been pinned to major 14 by `^14.2.0` while go_router was already at 16-17.
Context: Sam recalled the back-gesture bug had returned. Investigation traced go_router source in pub cache (14.8.1 vs 16.3.0 delegate.dart), confirmed the rewritten popRoute logic, checked changelog breaking changes 14->17, then upgraded.
Source: Claude Code session 2026-06-01, files: pubspec.yaml (go_router ^14.2.0 -> ^17.0.0), lib/core/navigation/main_shell.dart, lib/core/navigation/app_router.dart, lib/features/**/bottom_sheets/*, lib/features/shared_widgets/bottom_pop_ups/*, lib/features/player/presentation/player/full_player_screen.dart.
Suggested action: At next ingest — (1) canonicalize `modal-in-stateful-shell-route` as a RESOLVED pitfall page (symptom, root cause = go_router <=14.8.1 single-navigator popRoute, fix = upgrade to >=15, ideally 17.x). (2) DROP the deferred `stateful-shell-with-modal-overlays` resolver pattern as obsolete. (3) Update [[adr-004-go-router-stateful-shell]]: version moved to 17.2.3; the "branch navigator keys for modal context resolution" caveat no longer needed.

## [2026-06-09] candidate | go_sport | auth-screen-layout
Type: pattern
Target applies_to: flutter-projects-with-image-top-card-bottom-auth
What: Responsive auth/onboarding screen shell — LayoutBuilder + SingleChildScrollView + ConstrainedBox(minHeight: maxHeight) + IntrinsicHeight + Column, with Expanded background image (Stack/Clip.none, Positioned bottom:-20 to reveal image behind the card's rounded top corners) on top and a white form card below. Two card-height variants observed: (a) content-driven — card shrink-wraps content (mainAxisSize.min), image takes the rest (login_screen); (b) fixed-height — card has an absolute design height, content laid out top + Spacer + buttons-at-bottom, image takes the rest (registration_phone_screen).
Why interesting: canonical "fill viewport but scroll when keyboard/short screen" idiom, plus the explicit fork between content-driven vs fixed-height cards is a recurring layout decision in onboarding flows; not project-specific.
Context: refactoring login (content-driven) and registration_phone (fixed 371px card, top/Spacer/bottom layout) to be responsive and keyboard-safe.
Source: Claude Code session 2026-06-09, files login_screen.dart and registration_phone_screen.dart
Suggested action: create new pattern page

## [2026-06-10] candidate | go_sport | collection-play-pause-toggle
Type: pattern
Target applies_to: flutter-projects-with-playback
What: Collection-level play/pause button (on album/playlist/program hero) that reflects whether THIS collection is the currently-playing queue, not whether a track it contains happens to play. Determined by comparing the player state's active queue source id to the screen's own id: `s.source?.id == thisId && s.isPlaying && !s.isRadioMode`. Read via a narrow `ref.watch(playerStateProvider.select(... -> bool))` so the hero/app-bar rebuilds only when that bool flips, not on ~5/sec position ticks. Tap behavior is a fork: if this collection is the active source -> togglePlayPause(); else -> playQueue(...) (start this collection as a new queue). The hero stays dumb — receives `isPlaying` as a prop, exposes `onPlayTap` callback; the screen owns the watch + the fork.
Why interesting: composes two existing principles ([[selector-providers-for-rebuilds]] + [[shared-widgets-stay-dumb]]) into a recognizable, transferable UI idiom (the Spotify/Apple-Music "collection play button reflects active context" behavior). The non-obvious bit is using queue-source identity (not track membership) as the "is this playing" criterion, which correctly handles the same track appearing in multiple collections.
Context: adding play/pause toggle to album/playlist/program hero buttons; replicated identically across three screens with the same `.select`-bool + tap-fork shape.
Source: Claude Code session 2026-06-10, files album_hero.dart/playlist_hero.dart/program_hero.dart (isPlaying prop) and album_screen.dart/playlist_screen.dart/program_details_screen.dart (.select bool + _onPlayTap fork)
Suggested action: create new pattern page; relates to [[dual-mode-player-state]], [[selector-providers-for-rebuilds]], [[shared-widgets-stay-dumb]]

## [2026-06-10] candidate | go_sport | collection-play-pause-toggle (correction)
Type: pattern
Target applies_to: flutter-projects-with-playback
What: Correction/refinement to the same-day `collection-play-pause-toggle` candidate above. The toggle must distinguish TWO separate bools, not one: (1) `isThisActiveSource` = `s.source?.id == thisId && !s.isRadioMode` — does this collection own the active queue, REGARDLESS of play/pause; (2) `isThisPlaying` = same + `s.isPlaying` — drives the play/pause icon only. Tap behavior keys on (1), icon keys on (2). Pitfall if you use a single `isPlaying`-gated bool for both: pausing flips the bool false, so the next tap falls through to `playQueue(...)` and RESTARTS the collection from track 1 instead of resuming the paused track. Pause preserves `source`/`tracks`/`currentIndex` (only stop() clears them), so source-identity remains the correct, pause-independent criterion for "resume vs restart".
Why interesting: the one-bool version looks correct and even pauses correctly, but silently restarts on resume — a subtle correctness bug worth recording alongside the pattern.
Context: Sam tested on device; non-first track resumed from track 1 after pause→play. Split into two .select bools across album/playlist/program screens.
Source: Claude Code session 2026-06-10, files album_screen.dart/playlist_screen.dart/program_details_screen.dart
Suggested action: fold into the collection-play-pause-toggle pattern page as the core implementation detail (two bools: active-source vs playing); optionally note as an embedded pitfall.

## [2026-06-12] candidate | go_sport | design-system
Type: decision
Target applies_to: project: go_sport
What: Extending the DS dimension token scales to cover canonical design values, with a per-scale naming strategy chosen by value count: DSSpacing keeps t-shirt names + adds numeric "sN" fillers (s6/s8/s10/s12/s14/s20) because the design uses ~11 spacing values; DSRadius stays pure t-shirt via a one-step shift (xl 28→24's slot: new xl=24, old 28 renamed to xxl=28) because only one value was missing; DSIconSize is a brand-new numeric scale (s16/s20/s24/s28/s32/s48). One-off/bespoke values stay as documented literal exceptions.
Why interesting: Concrete rule-of-thumb — t-shirt (xs/s/m/l) sizing stays clean up to ~5-6 values but breaks past that (naming forces xsm/sm/ml or shifts the meaning of existing names); when a design's palette is dense, switch that scale to numeric value-named tokens. The choice is per-scale, not global, and is driven by an audit of literal-value frequency. Also: the centralized DS made this cheap to reason about — a future responsive-scale layer (flutter_screenutil vs a self-made DSScale) plugs in at the token layer only.
Context: Sam asked whether to adopt flutter_screenutil (phones-only app). Audit showed colors/typography are well-tokenized but ~950 dimension literals bypass DS. Decided to first fill the token scales (spacing/radius/icons) before any scaling layer. Discovered DSRadius.circular(100) token is dead, and a heart-icon size drift (38 vs the consistent 32/now-28) that was an actual bug.
Source: Claude Code session 2026-06-12, files ds_spacing.dart / ds_radius.dart / ds_icon_size.dart (new); related principle [[design-system-tokens-mandatory]]
Suggested action: update [[design-system-tokens-mandatory]] with the per-scale t-shirt-vs-numeric naming heuristic, OR create a new project ADR page for go_sport's token scale structure.

## [2026-06-12] candidate | go_sport | refactoring-safety
Type: pitfall
Target applies_to: any-flutter-project
What: Renaming a design-token constant by reassigning its value in place is a silent-bug trap. Inserting a value into a t-shirt scale (e.g. making DSRadius.xl mean 24 instead of 28) must be done by first replacing ALL existing usages (xl→xxl) and only then redefining the constant. If you redefine first (or keep the old name alive with a new value), every existing call site silently renders the new value with no compile error, because the token still resolves.
Why interesting: General Flutter/Dart refactoring pitfall — the compiler cannot catch it since the identifier remains valid; only the numeric meaning changed. Safe sequence: (1) global usage rename to the new name, (2) redefine the old name's value, (3) migrate literals. Verify with a grep that zero old-name usages remain.
Context: Shifting DSRadius so 24 gets a token: old xl=28 had 22 usages; reassigning xl=24 would have turned all 22 into 24. Did global DSRadius.xl→DSRadius.xxl first, then added xl=24.
Source: Claude Code session 2026-06-12, file ds_radius.dart + 18 feature files
Suggested action: create new pitfall page (token-rename-silent-value-drift) or fold into design-system token guidance.

## [2026-06-12] candidate | go_sport | design-system
Type: pattern
Target applies_to: any-flutter-project
What: Separate "functional layout metrics" from the spacing rhythm scale. Values like bottom-bar clearance (space so list content clears a mini-player + nav stack) or hero-content bottom offset (clears an overlapping rounded content sheet) are NOT spacing-scale tokens — they are purpose-specific, often derived from other dimensions (bar height, sheet overlap). Put them in a dedicated DSLayout class (DSLayout.bottomBarClearance, DSLayout.heroContentBottom) with semantic names, not in DSSpacing. Rule of thumb for what belongs in the spacing scale: a value is a "rhythm" token only if it's a general value reused across UNRELATED contexts; a value that is frequent but tied to ONE component/purpose (even if it appears more often than scale values) belongs in a named const, because (a) the name documents intent, (b) it doesn't dilute the scale's constraining job, (c) for responsive scaling it's a clean single chokepoint and ideally derived from its dependencies.
Why interesting: Gives a concrete test to resist "just add s44/s100 to the scale" — frequency is not the criterion, generality-of-context is. Prevents false coupling (changing a rhythm token would wrongly move unrelated functional offsets). DSLayout also fits projects whose DS already holds app-specific values (brand colors, gradients), and becomes the responsive chokepoint alongside DSSpacing/DSRadius/DSIconSize.
Context: go_sport spacing audit found bottom:100 (×12 list clearance under mini-player+nav stack) and bottom:44 (×4 identical hero content bottom over a 24px rounded sheet). Both functional, not rhythm. Created DSLayout in design_system/foundations. Also confirmed a same-value trap: SizedBox(height:100) elsewhere was a plain spacer, NOT clearance — same number, different meaning, must not be mapped to the clearance const.
Source: Claude Code session 2026-06-12, files ds_layout.dart (new) + 16 feature files; related [[design-system-tokens-mandatory]]
Suggested action: create pattern page (functional-layout-metrics-vs-spacing-scale) or add as a section to design-system token guidance.

## [2026-06-12] candidate | go_sport | refactoring-safety
Type: pitfall
Target applies_to: any-flutter-project
What: Mass literal→token migration of spacing has three concrete traps. (1) Float forms: a regex like `\b16\b` matches the "16" inside `16.0`, producing `DSSpacing.m.0` (compile error). Must match `16(\.0)?` or post-fix `DSSpacing\.\w+\.0` → strip. (2) EdgeInsets has multiple shapes — named (`symmetric`/`only`: `key: N`) AND positional (`.all(N)`, `.fromLTRB(a,b,c,d)`); a key-anchored regex silently misses `.all`/`.fromLTRB`. (3) SizedBox is ambiguous: `SizedBox(height: 16)` (single-dim, closed) is a GAP→spacing, but `SizedBox(width: 32, height: 32, child: …)` (two-dim, usually multi-line, has child) is a SIZE→must NOT be tokenized as spacing. Safe SizedBox-gap regex: `SizedBox\(\s*(width|height):\s*N\s*\)` (immediately closed, single dimension) — naturally excludes two-dim/sizing boxes. Always restrict spacing replacements to lines containing EdgeInsets/SizedBox so raw `width:`/`height:` on Container/Image (dimensions) aren't caught.
Why interesting: These make a "simple find-replace" silently wrong in two directions — broken code (float .0) and semantic corruption (sizing boxes turned into spacing tokens, creating false coupling). Verify each batch with `flutter analyze` (errors=0) and a leftover grep; do NOT trust the replace count alone.
Context: go_sport spacing migration (~534 sites). The `16.0`→`DSSpacing.m.0` bug produced 53 analyze errors caught immediately; fixed by stripping trailing `.0`. Also encoding: read/write Dart via .NET ReadAllText/WriteAllText(UTF8 no-BOM), NOT PowerShell Get-Content -Raw (reads as Windows-1252, corrupts Cyrillic comments to mojibake).
Source: Claude Code session 2026-06-12, ~100 feature files; tokens DSSpacing/DSRadius/DSIconSize/DSLayout
Suggested action: create pitfall page (mass-token-migration-gotchas) covering float forms, EdgeInsets shapes, SizedBox gap-vs-size, and the PS encoding trap.

## [2026-06-13] candidate | go_sport | external-links
Type: decision
Target applies_to: project: go_sport
What: url_launcher chosen for opening external URLs (YouTube channel link on program details banner), launched inline in the screen's onTap with LaunchMode.externalApplication
Why interesting: first external-link capability in the project; establishes url_launcher as the package and "dumb banner takes onTap, screen owns the launch" as the wiring; no core helper introduced (single call site, avoid premature abstraction)
Context: program details screen — YoutubeBanner must open the program's YouTube channel; URL comes from a per-program "Youtube" field on the Strapi program API mapped onto the Program entity/DTO
Source: Claude Code session 2026-06-13, files youtube_banner.dart, program_details_screen.dart, program.dart, program_dto.dart, pubspec.yaml
Suggested action: create new project-scoped ADR page

## [2026-06-15] candidate | go_sport | navigation
Type: pitfall
Target applies_to: flutter-projects-with-go-router-custom-transitions
What: go_router CustomTransitionPage (e.g. fadeSlidePage helper) does not provide the iOS edge-swipe-back gesture; only MaterialPage/CupertinoPage routes wire up the Cupertino back-gesture detector, so any screen pushed via a CustomTransitionPage silently loses swipe-to-go-back on iOS
Why interesting: general Flutter/go_router trap; symptom looks screen-specific ("swipe broken only on news detail") but the real discriminator is page type — screens using plain builder: (MaterialPage) keep the gesture, screens using pageBuilder: fadeSlidePage lose it (news, profile, playlist, album, artist, program all affected)
Context: Sam reported iOS back-swipe not working on news_detail screen while other screens were fine; those other screens use builder:/MaterialPage, news_detail uses fadeSlidePage CustomTransitionPage
Source: Claude Code session 2026-06-15, files page_transitions.dart (fadeSlidePage), app_router.dart, news_detail_screen.dart
Suggested action: create new pitfall page; relates to [[adr-004-go-router-stateful-shell]]

## [2026-06-15] candidate | go_sport | layout
Type: pitfall
Target applies_to: any-flutter-project
What: A ListView/scrollable with padding == null auto-injects MediaQuery.padding (safe-area insets) as its own padding, producing an unexpected large top (or bottom) gap; an explicit padding (even EdgeInsets.only(bottom: ...)) suppresses it.
Why interesting: General Flutter behavior, easy to miss. Bites specifically when the list is a sibling of a SafeArea (e.g. a header wrapped in SafeArea) inside a Column — the SafeArea consumes the inset only for its own subtree, so the sibling list still sees the full MediaQuery.padding and silently eats the top safe-area inset. Two screens with identical structure differed only because some lists had explicit padding and others didn't.
Context: Unifying first-item top spacing across the 6 "My *" library screens (my_albums/artists/programs/playlists/favorites, new_episodes). my_artists and new_episodes showed a much larger top gap than the rest; cause was their ListView having no padding param while the others had EdgeInsets.only(bottom: ...). Header is MyCategoriesHeader wrapped in SafeArea(bottom: false), list lives in a sibling Expanded.
Source: Claude Code session 2026-06-15, files my_artists_screen.dart, new_episodes_screen.dart, my_categories_top.dart
Suggested action: create new pitfall page

## [2026-06-16] candidate | go_sport | reactive-derived-view-on-shared-composition
Type: pattern
Target applies_to: any-flutter-project (Riverpod; a per-id detail screen materializes heavy data whose lightweight "composition" is owned by a global single-source-of-truth that other screens mutate)
What: A per-id screen controller holds a heavy materialized view (full Track objects of one opened custom playlist) while a global Notifier owns the lightweight composition of that same resource (the list of member ids + count). The local view must stay consistent with external mutations to the composition. Fix is pull-based, not push: in the local controller's build(arg), `ref.listen` the global state selecting THIS id's composition (its member ids); when the selected composition changes, compare it as a Set against the ids already materialized locally — if they differ, refetch the heavy data (and re-read the updated lightweight object from global to keep count consistent); if they are equal, do nothing. The Set-content comparison is the load-bearing guard: it (a) filters out the controller's own edits echoing back through the global owner (self-trigger loop — local edit → writes global → listener fires → would refetch redundantly), and (b) makes correctness independent of Riverpod's List equality (selecting a List<String> fires on every global rebuild because List identity changes; the explicit Set diff decides whether a refetch is actually warranted).
Why interesting: This is the residual sync problem that survives even after composition ownership is centralized. go_sport had two independent caches of the same backend resource: global myPlaylistsState (custom playlists with trackDocIds + trackCount, loaded via a documentId-only populate) and per-id customPlaylistControllerProvider (full tracks, loaded via its own getCustomPlaylistTracks call). "Add to playlist" from a track's options menu on Search/Album/Favorites (scenario A) mutated backend + global composition but never touched the local controller, which is non-autoDispose and early-returns on re-entry (`if (state is PlaylistDetailsData) return`). Result: add a track from Search, re-open the already-visited playlist, track is missing until app restart; the card count (global) and the in-screen track list (local) drift. autoDispose was considered and rejected: its trigger is "no listeners", which only fires when the detail screen is fully torn down (popped), NOT when the user jumps to another tab/route to perform scenario A while the detail screen stays mounted in its shell branch or under the top route — the common trigger path. So the reliable refresh trigger is "the shared composition changed", not "the screen was re-entered". Generalizes to any heavy-view/light-composition split: a cart screen vs a global cart-item-id set, a thread view vs a global unread-id set, etc. Pairs with the operational consumer-count rule (2026-05-12 candidate): full tracks are consumed by one screen → legitimately local; composition is consumed by many → global; the bridge between them is a reactive derived view, not a second owner.
Context: Architectural discussion in go_sport on how songs are added to custom playlists. Two add-paths exist: scenario A (track options → showAddToPlaylistBottomSheet, a Track-family controller that diffs selected vs initial playlist membership and PUTs each affected playlist, then updates global via myPlaylistsState.updatePlaylist) and scenario B (inside the playlist screen → showAddTracksBottomSheet → customPlaylistController.addTracks, which appends full tracks locally and saves, updating global in save()). B stays consistent both ways because it owns the local view and pushes up; A updates only global, so the local view goes stale. Converged on: keep global as sole composition owner, make the local controller subscribe to its own id's composition and refetch-on-diff. Refetch (not in-memory reconcile) chosen deliberately — data is small, external mutations rare, and additions would need the full Track payload anyway, so in-memory diffing buys nothing.
Source: Claude Code session 2026-06-16, files lib/features/playlists/presentation/playlist/custom_playlist_controller.dart (the per-id local view), lib/domain/state/my_playlists_state.dart (global composition owner), lib/features/playlists/presentation/bottom_sheets/add_to_playlist_controller.dart (scenario A, the external mutator), lib/data/repositories/custom_playlist_repository_impl.dart (the two independent reads: getCustomPlaylists documentId-only vs getCustomPlaylistTracks full populate).
Suggested action: create new pattern page (naming: [[reactive-derived-view]] or [[derived-view-refetch-on-composition-change]]). Relates to [[domain-vs-screen-state]] (the operational consumer-count rule — this is the wiring between a legitimately-local heavy view and a legitimately-global light composition), [[central-user-collection-registry]] (the global owner is exactly that kind of registry; this is what a heavy per-id detail view does on top of it), and [[selector-providers-for-rebuilds]] (the listen uses a narrow per-id `.select`). Could double as a pitfall ("two independent caches of one backend resource; mutating one leaves the other stale; autoDispose does not fix it because the stale view stays mounted").

## [2026-06-16] candidate | go_sport | player / state
Type: pattern
Target applies_to: any-flutter-project
What: Screen-state UI toggle (MiniPlayerWidget._isMusicMode — which panel is expanded) reactively follows a domain-state discriminator (PlayerState.mode) via ref.listen, NOT ref.watch: on a mode transition, compare local flag to desired and flip only on mismatch, so the two stay independent between transitions but auto-align when new playback starts in a different mode.
Why interesting: Sharpens the existing "they are independent and may legitimately disagree" guidance ([[domain-vs-screen-state]], [[dual-mode-player-state]]) with the missing nuance — independence does NOT mean "never reacts". The right tool is ref.listen (side-effect on change), not ref.watch (deriving the panel from mode glues them — manual toggle becomes impossible, and you can't setState/animate inside build anyway). Guard "if (local != desired) toggle()" works precisely because the flag is boolean (flip-on-mismatch == set-to-target); a non-boolean toggle would need an explicit setter.
Context: User wanted: radio panel open in mini-player, music starts from another screen → mini-player should auto-switch to the music panel. Settled on listening to PlayerState.mode and calling existing _toggleMode() only when local _isMusicMode disagrees.
Source: Claude Code session 2026-06-16, file lib/features/player/presentation/player/mini_player_widget.dart
Suggested action: update [[domain-vs-screen-state]] (add "independent but may reactively sync via ref.listen on transition") and/or [[dual-mode-player-state]] pitfalls; possibly small pattern page "screen-toggle-follows-domain-discriminator"

## [2026-06-16] candidate | go_sport | data-layer
Type: pitfall
Target applies_to: flutter-projects-with-strapi (or any-shared-DTO-with-per-call-populate)
What: A shared DTO (TrackDto) parsed from Strapi responses silently yields empty fields when a query forgets a `populate` relation — no error, just an empty string. Featured-playlist tracks query omitted `populate[Tracks][populate][Artists]`, so artistName became '' while favorites and custom-playlist queries (which include it) showed the artist.
Why interesting: general trap — when several endpoints feed the same DTO but each declares its own populate params inline, missing one relation degrades silently into a falsy default instead of failing. Hard to spot because UI just renders blank. Transferable to any GraphQL/Strapi sparse-fieldset setup.
Context: investigating why artist name showed on Favorites and Custom playlist screens but not on Featured playlist; all three render via the same TrackTile and TrackDto.toDomain().
Source: Claude Code session 2026-06-16, files featured_playlist_repository_impl.dart (getPlaylistTracks), track_dto.dart, custom_playlist_repository_impl.dart
Suggested action: create new pitfall page (relates to [[dto-to-domain-mapping]])

## [2026-06-17] candidate | go_sport | story-viewer
Type: pattern
Target applies_to: any-flutter-project
What: Instagram-style auto-advancing story/onboarding viewer built WITHOUT a Riverpod controller — a ConsumerStatefulWidget holds the ephemeral viewing session (current index + a single AnimationController that BOTH times the per-item auto-advance and fills the active segment of a segmented progress bar); it reads the item list from the existing domain Notifier once on open and pushes "viewed" back to that notifier (mirrors how the player artwork carousel reads playerStateProvider and pushes .next()/.previous()). Tap zones (left third = prev, rest = next) via one HitTestBehavior.opaque GestureDetector under the on-top close/CTA buttons; hold-to-pause via onLongPressStart/End → controller.stop()/forward(). Segmented bar is a dumb StatelessWidget (count + currentIndex + progress 0..1).
Why interesting: Concrete worked example of the dogma "screen controllers (Riverpod Notifier) are for data/async; vsync/ticker + ephemeral session state live in the widget State" ([[domain-vs-screen-state]]) — even the most functional screen here (full player) keeps AnimationController/PageController in the widget, not in a Notifier. Reusable insight: a single AnimationController is both the timer and the progress source (controller.value), so timer and bar animation are one mechanism, not two. Boundary handling (last→close, first→no-op) and the "don't put the AnimationController in Riverpod (needs vsync)" rationale are the transferable bits.
Context: implementing tap-through + 7s auto-advance + segmented progress + hold-to-pause for home-screen stories; chose ConsumerStatefulWidget over a new StoryController after surveying album/search controllers (data) vs mini_player/artwork_carousel (local AnimationController/PageController).
Source: Claude Code session 2026-06-17, files story_overlay.dart, widgets/story_progress_bar.dart, home_screen.dart (_openStoryOverlay)
Suggested action: create new pattern page (relates to [[domain-vs-screen-state]]); could cross-link from [[shared-widgets-stay-dumb]] re: feature-local widgets legitimately being Consumers

## [2026-06-18] candidate | go_sport | navigation
Type: pitfall
Target applies_to: flutter-projects-with-stateful-shell-route
What: Hero tag collision across simultaneously-alive StatefulShellRoute.indexedStack branches — the same Hero tag rendered in two tab branches (both kept alive by IndexedStack) lives in the shell route subtree at once; any PageRoute→PageRoute transition on the ROOT navigator (e.g. pushing a top-level route like /profile) makes the root HeroController scan the shell subtree via Hero._allHeroesFor and assert "multiple heroes that share the same tag within a subtree", crashing on the next frame.
Why interesting: General StatefulShellRoute trap, not project-specific. Counter-intuitive points: (1) the duplicate can be latent for a long time and surface only when a root-level transition starts triggering the hero scan; (2) Hero._allHeroesFor scans the SOURCE route unconditionally on every page transition, so a matching destination hero is NOT required to crash — the duplicate in the source alone is enough; (3) the fix "make the tag unique per placement" must also forward that tag to the destination, otherwise the shared-element transition breaks.
Context: go_sport crash report — same featuredPlaylistsStateProvider rendered via FeaturedPlaylistsSection in BOTH Home and Music tabs, each PlaylistCard wrapping its image in Hero(tag: 'playlist-image-$id'); both tabs alive in StatefulShellRoute.indexedStack. Crash surfaced around recent route changes switching detail/profile routes from CustomTransitionPage (fadeSlidePage) to default builder pages (commits efa695b, 245ace0). Trigger not 100% confirmed at code level; structural duplicate-tag cause is certain.
Source: Claude Code session 2026-06-18, files playlist_card.dart, playlist_hero.dart, featured_playlists.dart, home_screen.dart, music_screen.dart, app_router.dart
Suggested action: create new pitfall page (relates to [[adr-004-go-router-stateful-shell]])

## [2026-06-18] candidate | go_sport | layout
Type: pattern
Target applies_to: any-flutter-project
What: "background image + bottom sheet-card" form layout that survives keyboard — Stack with Positioned image (height = screenH - cardHeight + overlap), bottomCenter Container(fixed cardHeight) wrapping SingleChildScrollView > ConstrainedBox(minHeight) > IntrinsicHeight > Column(...fields, Spacer(), button); resizeToAvoidBottomInset:true lets the card ride up over the image on keyboard, content scrolls, Spacer keeps button pinned at card bottom. IntrinsicHeight is what makes Spacer legal inside a scroll view.
Why interesting: recurring, consistent structure across ~11 auth screens; the broken variant (fixed screenHeight*0.6 + extendBodyBehindAppBar + Column without scroll) overflows and slides under AppBar when keyboard opens — concrete anti-pattern worth contrasting. Also: image height is derived from card height (the two fill the screen with overlap), and BoxFit.cover crop side is controlled via Image/DecorationImage alignment (Alignment.topCenter = crop from bottom).
Context: fixing ProfileChangePasswordScreen keyboard overflow by aligning it to the existing auth-screen pattern; then propagated alignment:topCenter crop control to all background-image screens.
Source: Claude Code session 2026-06-18, files lib/features/user_profile/change_password/.../change_password_screen.dart and lib/features/auth/**/*screen.dart
Suggested action: create new pattern page (e.g. [[background-image-bottom-card-form]]); cross-link a pitfall note about Spacer-in-scroll needing IntrinsicHeight and the fixed-percent-height keyboard overflow

## [2026-06-19] candidate | go_sport | guest-session-limit
Type: pitfall
Target applies_to: any-flutter-project
What: Time-based access gating (guest session limit) implemented as ephemeral widget state — a Timer.periodic living in a StatefulWidget (GuestTimerBar) in the shell's bottomNavigationBar — and expiry done via context.go('/expired-guest') with no persisted "expired" flag. NOTE: reset-on-app-kill is intended here (in-memory, no persistence), and because all guest-reachable content lives inside StatefulShellRoute.indexedStack the shell stays mounted, so the timer does NOT reset during normal browsing or tab switches — that part is correct. The residual trap is that expiry is not held as state: it relies on the expired screen being a navigation dead-end (works only because go() replaced the stack — Android back exits, iOS has nothing to pop; no PopScope). So programmatic navigation back into the shell while expired (push-notification tap, deep link) — or leaving to an auth top-level route and returning — remounts the widget and restarts the countdown without a kill, which the "reset only on kill" design did not intend.
Why interesting: general trap — once gating is a navigation concern, enforcement should be a router redirect backed by long-lived state (domain/state Notifier or in-memory storage class exposing remaining + an expired flag), not a UI dead-end that only holds for the happy path. The per-widget timer happens to behave correctly for in-session counting *because* it sits in a persistent shell, which is fragile/incidental rather than designed.
Context: reviewing go_sport guest session timer ("плашка таймер выход") for correctness and Android/iOS parity at Sam's request; Sam clarified reset-on-kill is by design.
Source: Claude Code session 2026-06-19, files lib/features/auth/guest_timeout_bar/widgets/guest_timer_bar.dart, lib/features/auth/expired_guest/presentation/expired_guest.dart, lib/core/navigation/main_shell.dart, lib/core/navigation/app_router.dart
Suggested action: create new pitfall page; cross-link [[domain-vs-screen-state]] and [[event-stream-vs-state-for-discrete-events]]

## [2026-06-19] candidate | go_sport | image-decode-distortion
Type: pitfall
Target applies_to: any-flutter-project
What: Passing BOTH cacheWidth and cacheHeight to Image.asset (or ResizeImage) decodes the bitmap to exactly those pixel dimensions and does NOT preserve the source aspect ratio (default ResizeImagePolicy.exact). If the cacheWidth:cacheHeight ratio differs from the asset's native ratio, the decoded image is already stretched — and a subsequent fit: BoxFit.cover cannot fix it, because the layout box has the same (wrong) ratio, so the distorted bitmap is shown as-is.
Why interesting: counterintuitive — devs add cacheWidth/cacheHeight purely for memory and assume BoxFit.cover still governs aspect ratio. Fix: omit cache dims (let BoxFit.cover + alignment handle crop), or specify only ONE of cacheWidth/cacheHeight to keep proportions. Concrete contrast in-repo: login screen renders the same login_bg.png correctly via DecorationImage(fit: cover, alignment: topCenter) with no cache dims, while the expired-guest screen distorted it by setting both cache dims to screenWidth × screenHeight*0.7.
Context: fixing visible background-image distortion on the guest-expired screen; aligned it to the existing auth-screen image approach.
Source: Claude Code session 2026-06-19, files lib/features/auth/expired_guest/presentation/expired_guest.dart and lib/features/auth/login/presentation/login/login_screen.dart
Suggested action: create new pitfall page; cross-link the [[background-image-bottom-card-form]] candidate (alignment-controls-crop note) from 2026-06-18

## [2026-06-19] candidate | go_sport | dual-source-of-truth-guest-state
Type: pitfall
Target applies_to: any-flutter-project
What: The same logical concept (guest mode) had TWO sources of truth read by different layers — a persisted flag (TokenStorage.choseGuest) read synchronously by the go_router redirect, and a Riverpod Notifier state (authProvider → AuthGuest) read by widgets. The "Continue as guest" button updated only the persisted flag (tokenStorage.setChoseGuest()) and bypassed the notifier's continueAsGuest() (which sets BOTH the flag and state=AuthGuest). Result: the router treated the user as guest (content browsable) but authProvider never became AuthGuest, so the guest-only UI (timer bar, gated on `ref.watch(authProvider) is AuthGuest`) never rendered. The notifier build() only reflects the flag if it happens to (re)build after the flag is set, so the symptom was a build-timing race — appeared on Android, not on iOS — masquerading as a platform/safe-area layout bug.
Why interesting: classic dual-source-of-truth drift. When a Notifier owns a piece of state that ALSO has a persisted/cached mirror read elsewhere, every mutation must go through the notifier (which updates both), never through the backing store directly. A correct continueAsGuest() existed but had zero call sites — a smell worth grepping for. Also a debugging lesson: a "platform-specific" UI-visibility bug that's actually a provider-not-rebuilt race, not layout.
Context: guest session timer bar not showing on iOS; chased a wrong safe-area/MediaQuery.removePadding hypothesis first (which regressed Android), then found the login button bypassed authProvider.notifier.continueAsGuest().
Source: Claude Code session 2026-06-19, files lib/features/auth/login/presentation/login/login_screen.dart, lib/core/auth/auth_state.dart, lib/core/navigation/main_shell.dart, lib/core/navigation/app_router.dart
Suggested action: create new pitfall page; relates to the planned guest-expiry-enforcement work (expired flag must also be a single source the router + widgets agree on)

## [2026-06-19] candidate | go_sport | guest-choice-is-per-session
Type: decision (ADR)
Target applies_to: project: go_sport
What: The guest-mode choice is per-session and in-memory only — TokenStorage.choseGuest is no longer persisted to FlutterSecureStorage (removed the chose_guest key write/read). Each app launch starts at the login screen until the user taps "Continue as guest" again. Chosen over persisting the flag.
Why interesting: aligns the guest-choice lifecycle with the already-in-memory guest session timer (previously inconsistent: the choseGuest flag survived restart but the timer reset on kill, so a relaunched user was silently a guest again with no path back to login). Per-session guest also reinforces the freemium nudge — every launch faces login first. Trade-off accepted: a returning guest must re-tap "Continue as guest" each launch.
Context: after fixing the guest bar visibility, restart landed users straight on the guest home with no way to reach login; Sam chose "login on every launch" over persisting guest.
Source: Claude Code session 2026-06-19, files lib/core/auth/token_storage.dart, lib/core/auth/auth_state.dart, lib/features/auth/login/presentation/login/login_screen.dart
Suggested action: create project-scoped ADR under projects/go_sport; cross-link the dual-source-of-truth-guest-state pitfall (same session) and adr-005-secure-storage-for-tokens (what does/doesn't belong in secure storage)

## [2026-06-23] candidate | go_sport | unified-network-image-fallback
Type: pattern
Target applies_to: any-flutter-project
What: Single dumb presentational widget (DSNetworkImage) wrapping CachedNetworkImage that collapses the loading, load-error, and null/empty-url states into one shared "noimage" fallback (a bundled asset). Replaces per-call-site errorWidget/errorBuilder + null-ternary boilerplate that had drifted into ~5 inconsistent placeholders (gray20/gray40/divider, Icons.error/music_note/radio/person/broken_image). Exposes width/height/fit/borderRadius/memCacheWidth/memCacheHeight so it can absorb every call site, including AspectRatio/Positioned.fill/Hero-wrapped ones; lives in design_system/components/ next to skeleton/.
Why interesting: image-fallback handling is a textbook case of duplicated UI logic that silently diverges across a codebase (24 call sites here, two APIs — CachedNetworkImage vs raw Image.network). Centralizing it is a clean application of [[shared-widgets-stay-dumb]] + [[design-system-tokens-mandatory]]. Notable boundaries worth recording: (a) avatars kept their own SVG placeholder (different semantics, not folded in); (b) local Image.asset call sites (custom-playlist covers, notification icons) deliberately excluded — the widget is network-only; (c) BoxDecoration/DecorationImage call sites (mini-player background) can't migrate without layout restructuring since the image is a decoration, not a child — a real limitation of the child-widget approach.
Context: "how do we handle images that didn't arrive and show noimage" — designer supplied a single noimage.png; decided one visual covers loading+error+null (no separate shimmer). Standardized the app onto CachedNetworkImage and dropped raw Image.network.
Source: Claude Code session 2026-06-23, files lib/design_system/components/network_image/ds_network_image.dart + 24 migrated call sites across features/shared_widgets, features/music, features/radio, features/home, features/news, features/player, features/playlists
Suggested action: create new pattern page; cross-link [[shared-widgets-stay-dumb]] and [[design-system-tokens-mandatory]]

## [2026-07-01] candidate | go_sport | data-layer
Type: pitfall
Target applies_to: flutter-projects-with-strapi (or any REST API with per-endpoint field expansion)
What: The same domain entity (Track) is hydrated by a single DTO, but different endpoints request different `populate` sets. The album-tracks endpoint populated only `Tracks.File`, so `Album.Cover` and `Artists` were absent → TrackDto produced Track with imageUrl=null and artistName='' for album-sourced tracks, while the same track fetched from a playlist/favorites endpoint (which populate Album.Cover + Artists) came back fully hydrated. Fix: align the album query's populate with the playlist query (`populate[Tracks][populate][Album][populate]=Cover` + `populate[Tracks][populate][Artists][fields][0]=Name`).
Why interesting: a general trap of partial-response APIs — one entity, N endpoints, divergent expansion → the entity is silently under-hydrated on some code paths. It presents as a UI bug in a distant layer, not where the query lives. Compounded here by a second smell: a UI-level fallback in one widget (mini-player: `track.imageUrl ?? source.imageUrl`) MASKED the missing data, while a sibling widget without the fallback (full-player artwork: raw `track.imageUrl`) EXPOSED it — so identical playback of "the same track" showed an image from a playlist but a blank from an album, which looks like a player bug but is a data-population bug. Lesson: when one entity is fetched by multiple endpoints, keep their populate/expansion in sync; and prefer a single shared fallback (the state already had an unused `displayImageUrl` getter doing track→source) over per-widget ad-hoc fallbacks that diverge.
Context: user noticed album tracks showed no artist name (mini) and no artwork (full), while the same tracks from a playlist/favorites showed both. Traced through player → PlayerState → TrackDto → per-endpoint populate.
Source: Claude Code session 2026-07-01, files lib/data/repositories/albums_repository_impl.dart (getAlbumTracks populate), lib/data/dto/track_dto.dart, lib/data/repositories/featured_playlist_repository_impl.dart (reference populate), lib/domain/state/player_state.dart (displayImageUrl), lib/features/player/presentation/player/{mini_player_widget,widgets/player_artwork_carousel}.dart
Suggested action: create new pitfall page; cross-link [[dto-to-domain-mapping]]

## [2026-07-02] candidate | go_sport | riverpod-lazy-global-read
Type: pitfall
Target applies_to: any-flutter-project
What: A short-lived screen controller reads a lazily-initialized global Notifier with a one-shot ref.read in its _init(), and gates the "need to load" decision on the combined list being .isEmpty; when the global exposes a merged list (e.g. liked-featured + custom) and only one subset is loaded synchronously while the rest arrives via a microtask, the non-empty featured entries make .isEmpty false, the load never fires, and the controller keeps a stale/partial snapshot (the sheet watches only its own notifier, so later global updates never propagate).
Why interesting: general Riverpod trap — combining "is the global initialized?" with "is the list empty?" breaks whenever the global merges multiple sources with different load timings; fix is to filter to the actually-needed subset BEFORE the emptiness guard, or track an explicit loaded flag on the global.
Context: "Add to playlist" bottom sheet showed an empty custom-playlist list when the user had liked (featured) playlists but hadn't visited the My Playlists tab yet; myPlaylistsState returns featured synchronously in build() and schedules loadFavorites (customs) via Future.microtask.
Source: Claude Code session 2026-07-02, files add_to_playlist_controller.dart (_init), my_playlists_state.dart (build/loadFavorites)
Suggested action: create new pitfall page (relates to [[domain-vs-screen-state]] — short-lived controller reading long-lived domain state)

## [2026-07-02] candidate | go_sport | image-orientation-package
Type: decision
Target applies_to: project: go_sport
What: Chose the pure-Dart `image` package (bakeOrientation + encodeJpg) over the native `flutter_image_compress` to bake EXIF orientation into pixels before uploading a profile avatar.
Why interesting: package-choice ADR with explicit tradeoff. flutter_image_compress is faster (native) and does resize+rotation in one call, but adds a native plugin (pod install, per-platform build surface) and duplicates the resize image_picker already does. For a single infrequent avatar the perf gain is irrelevant, so the decisive factors were: zero native surface (pure Dart, no pod, reliable across the 2-Windows + Mac dev setup) and no duplication. Revisit if image processing broadens (posts/galleries/bulk upload) — then native wins.
Context: fixing camera-captured avatar appearing rotated 90° after save on Android; wired camera/gallery chooser on edit profile screen.
Source: Claude Code session 2026-07-02, files edit_profile_screen.dart (_pickImage/_bakeOrientation), pubspec.yaml (image: ^4.3.0)
Suggested action: create project-scoped ADR under projects/go_sport

## [2026-07-02] candidate | go_sport | exif-orientation-not-baked
Type: pitfall
Target applies_to: any-flutter-project
What: Camera photos store rotation in EXIF metadata, not in the pixels. Flutter's Image.file/Image.network honor the EXIF tag so the local preview looks upright, but many backends strip EXIF on re-encode — so the *uploaded* image ends up sideways while the preview was fine. The tell is "preview correct, saved image rotated."
Why interesting: the divergence (preview fine, saved rotated) misleads you into looking at the display layer when the real fix is client-side normalization before upload — decode → bakeOrientation → re-encode (JPEG re-encode drops the orientation tag), then upload the normalized bytes. image_picker's maxWidth/imageQuality does NOT reliably bake orientation on Android.
Context: profile avatar captured via image_picker camera source; correct in the Image.file preview, rotated 90° after Save round-tripped through the server.
Source: Claude Code session 2026-07-02, files edit_profile_screen.dart, profile_repository_impl.dart (updateUser multipart)
Suggested action: create new pitfall page

## [2026-07-03] candidate | go_sport | load-queue-without-autoplay
Type: pattern (refinement of existing page)
Target applies_to: flutter-projects-with-playback
What: Split the audio handler's "load queue" from "start playback". In [[two-layer-audio-stack]], `AppAudioHandler.setQueue(tracks, initialIndex, initialPosition)` should ONLY `setAudioSource(...)` (load + seek), never call `_player.play()` internally. The decision to play moves up to the notifier: `playQueue`/`resumeMusic` do `setQueue(...)` then `play()` (both inside one try/catch to keep it atomic); a new `loadPausedQueue(...)` does `setQueue(...)` and stops — leaving a `ready`-but-not-playing player that maps to `PlayerStatus.paused`, so `hasActivePlayback` is true and the mini-player shows the track silently. This mirrors just_audio's own API where `setAudioSource` and `play` are already separate; bundling them was the anomaly.
Why interesting: A single, small change (remove one `play()` call, add an `initialPosition` param) unlocks two features that both need "a loaded-but-silent queue": cold-start session restore and first-launch mini-player priming. Generalizes to any playback app that wants to show "up next / last played" without auto-blasting audio on launch (also aligns with iOS not allowing autoplay on cold start). The naming honesty angle: a method called `setQueue` that also plays is a lying name.
Context: Mini-player was empty on first app open and after kill. Refactored setQueue to not auto-play, added `loadPausedQueue` as a twin of `playQueue` minus the final play() plus a position. Screens were unaffected — they all call the notifier's `playQueue`, never the handler's `setQueue` directly (the handler's setQueue has only two callers, both inside PlayerNotifier).
Source: Claude Code session 2026-07-03, files lib/core/audio/app_audio_handler.dart (setQueue), lib/domain/state/player_state.dart (playQueue/resumeMusic/loadPausedQueue).
Suggested action: update [[two-layer-audio-stack]] — add the "handler loads, notifier decides to play" split as a trade-off/guidance point, with `loadPausedQueue` as the paused-prime entry point. Relates to [[dual-mode-player-state]].

## [2026-07-03] candidate | go_sport | persist-and-restore-player-session
Type: decision (ADR) + pattern
Target applies_to: project: go_sport (decision); flutter-projects-with-playback (pattern)
What: Persist the music side of the player across app kill and prime the mini-player on cold start. A `PlayerSessionStorage` (SharedPreferences, applies [[local-state-via-storage-class]]) holds a `PlayerSession` snapshot — `tracks`, `currentIndex`, `position`, and `source` flattened to primitive fields (`sourceType/id/title/imageUrl`). The flattening is deliberate: keeping `source` as a `QueueSource` would make the storage file import player_state.dart while player_state.dart imports the storage → circular import; the notifier owns the `QueueSource ↔ fields` mapping instead. Manual (de)serialization lives in the storage class so the domain `Track`/`QueueSource` stay free of json (respects [[dto-to-domain-mapping]] — Track is freezed WITHOUT json_serializable). On startup the notifier runs `_restoreOrPrime()` (scheduled from `build()` via `Future(...)`, never mutating state during build, guarded to act only on an idle/empty player): restore the saved session if present, else fetch the first featured playlist and prime it — both via `loadPausedQueue` (paused, see the load-queue-without-autoplay candidate). Saving is event-driven, never per-position-tick: on queue start (`playQueue`), on track change (inside the index-change branch of `_onMediaItemChanged`, which naturally skips the priming/restore emit since the index already matches), on `resumeMusic`, and on `AppLifecycleListener.onPause/onDetach` — backgrounding is the reliable capture point before an OS kill. Radio is never persisted; `_saveSession` early-returns unless mode==music with a non-empty queue and a source, so a radio-time background doesn't clobber the saved music session.
Why interesting: "closed → reopened, everything's where I left it" is baseline user expectation for any media app (Spotify/YT Music/podcasts), and the empty-mini-player-on-first-launch is a real UX gap. The design reuses an existing move (`resumeMusic` already reloads a queue-by-index from in-memory state; restore is the same move sourced from disk instead) rather than inventing new machinery. Notable decisions: (1) three-object chain live `PlayerState` / serializable `PlayerSession` / `PlayerSessionStorage` — the middle exists because `PlayerState` is stream-backed and not serializable; (2) restore/prime lands PAUSED, never autoplays (Sam's explicit call; also iOS-cold-start-safe); (3) featured-prime is NOT saved (loadPausedQueue doesn't save), so an untouched primed featured re-primes next launch instead of freezing as a "session"; (4) init() drops corrupt/outdated snapshots (try/catch on fromJson) to never crash startup on a schema change.
Context: Sam reported the mini-player was empty (a) on the very first launch ever and (b) after kill-then-reopen. Multi-turn design discussion; implemented in two phases — Phase 1 pure refactor (split setQueue's load-vs-play, verified playback unchanged on device), Phase 2 storage + restore + prime. A related in-session fix: `resumeMusic` never preserved the music position across a radio detour (pre-existing, restarted the track at 0); now stashed in a `_musicPosition` field on playRadio and restored via setQueue's new initialPosition.
Source: Claude Code session 2026-07-03, files lib/core/audio/player_session_storage.dart (new), lib/domain/state/player_state.dart, lib/main.dart, lib/domain/repositories/featured_playlist_repository.dart (getFeaturedPlaylists/getPlaylistTracks used for prime).
Suggested action: (1) create project-scoped ADR for go_sport (persist-player-session) recording the snapshot shape, event-driven save triggers, restore-vs-prime startup priority, and the paused-not-autoplay decision. (2) optional pattern page (flutter-projects-with-playback) for the general "persist + restore playback session, prime paused on cold start" structure, referencing [[local-state-via-storage-class]] and the load-queue-without-autoplay refinement. Note the flatten-source-to-avoid-circular-import detail as a concrete gotcha.

## [2026-07-03] candidate | go_sport | persist-player-session (correction — position needs its own key + periodic save)
Type: decision (ADR) + pitfall (refines the same-day persist-and-restore-player-session candidate above)
Target applies_to: project: go_sport (decision); flutter-projects-with-playback (pattern/pitfall)
What: Correction to the save strategy in the earlier persist-and-restore candidate. Relying ONLY on event-driven saves (queue start, track change, `AppLifecycleListener.onPause`) meant the up-to-date PLAYBACK POSITION was effectively persisted only at the onPause moment — and that single last-moment async SharedPreferences write is unreliable: between "user backgrounds" and "OS kills" the fire-and-forget `save()` often doesn't flush to disk, so on restore the queue came back but position was stale (≈0, the value from the last calm save at track/queue change). Sam observed this on device ("position didn't save"). Fix has two parts: (1) SPLIT persistence into two SharedPreferences keys in the same storage class — `player_session` = heavy queue snapshot (tracks/index/source), written rarely on queue/track change; `player_position` = a single int (ms), written often. Position is REMOVED from the PlayerSession snapshot; the two are merged on restore (`loadPausedQueue(..., position: storage.position)`). (2) Save position PERIODICALLY while playing via `Timer.periodic(10s)` gated on `state.isPlaying && mode==music`, plus onPause/onDetach — so a hard kill loses at most ~10s regardless of lifecycle-callback timing. The periodic write is a single int, so re-serializing the whole track list every tick is avoided (that was Sam's load concern and the reason for the key split — high-frequency position vs low-frequency queue, same rationale as [[selector-providers-for-rebuilds]] isolating high- from low-frequency updates). `_savePosition` guards mode==music so a radio-time tick never overwrites the saved music position on disk.
Why interesting: General playback-persistence pitfall — "save on background/onPause" looks sufficient but silently loses the most volatile field (position) precisely because the kill races the async disk write. The robust answer is periodic throttled position persistence, not lifecycle-only. The key-split is the clean way to make frequent position writes cheap (one int) while keeping the queue snapshot untouched. Both generalize to any media app.
Context: Continuation of the same 2026-07-03 session, after Sam tested kill→reopen: queue restored, position did not. Diagnosed as onPause-only position save + unreliable last-moment async flush. Implemented two-key storage + 10s periodic position timer.
Source: Claude Code session 2026-07-03, files lib/core/audio/player_session_storage.dart (two keys: saveSession/savePosition/session/position), lib/domain/state/player_state.dart (_positionSaveTimer, _savePosition, _persistNow, _saveSession without position, _restoreOrPrime merges storage.position).
Suggested action: fold into the persist-player-session ADR/pattern pages as the position-persistence detail (two keys + periodic save while playing); optionally record the "lifecycle-onPause-only loses position on kill" as an embedded pitfall.

## [2026-07-03] candidate | go_sport | navigation
Type: pattern
Target applies_to: any-flutter-project
What: A route-target screen self-hydrates from an id; the full entity passed via go_router `extra` is an optional first-paint hint, not a requirement. Screen watches an id-keyed controller that fetches the entity; renders from `loadedEntity ?? extraHint`, showing a placeholder only when both are absent (deep-link / id-only entry).
Why interesting: Decouples a detail screen from callers that don't hold the full entity (e.g. the music player knows only a track's artist id/name, not the Artist object). Keeps existing list navigations unchanged (they still pass `extra` → instant hero) while making the screen usable from anywhere with just an id. General Flutter/go_router pattern.
Context: Full-player "title + artist" tap should open the artist page. Track carried only artistName; ArtistScreen required a full Artist in `extra`. Evolved ArtistScreen to ArtistScreen({required artistId, Artist? artistHint}); the existing single-caller repo method getArtistAlbums → getArtistDetails now returns (artist, albums) from the same /api/artists/:id response (artist data was previously fetched then discarded), so no extra network call. Added Track.artistId parsed from Artists relation documentId.
Source: Claude Code session 2026-07-03, files artist_screen.dart, artist_controller.dart, artists_repository_impl.dart, track_dto.dart, track.dart, app_router.dart, player_control_panel.dart
Suggested action: create new pattern page (consider cross-link from a go_router extra / deep-linking page if one exists)

## [2026-07-04] candidate | go_sport | system-player-artwork-fallback
Type: pattern
Target applies_to: flutter-projects-with-playback
What: MediaItem.artUri (audio_service) cannot reference Flutter assets - the native side (MediaSession / MPNowPlayingInfoCenter) only loads http(s)/file URIs. To give the system player a bundled "noimage" fallback: lazily copy the asset to the app support directory once (rootBundle.load -> File.writeAsBytes) and use Uri.file as the last link of the artUri chain; guard the chain with a scheme check (url.startsWith('http')) so non-network imageUrl values (asset paths used as playlist covers, empty strings from search/favorites sources) never reach the system as dead URIs - without the guard they are non-null, stop the ??-chain and the fallback never fires.
Why interesting: two-part transferable knowledge: (1) pitfall - the in-app image fallback chain (errorWidget etc.) does not protect the system player because system art fails AFTER the URI is handed over, with no error callback; artUri built from "whatever imageUrl holds" silently shows nothing for asset paths, empty strings and dead URLs; (2) remedy pattern - materialize-asset-to-file + network-scheme guard, fully contained in the AudioHandler (fits [[two-layer-audio-stack]]). Known boundary: syntactically valid but dead (404) URLs cannot be caught client-side without preflight requests - that class is a data problem, not a code problem.
Context: playlist tracks without covers showed fallback art in-app but nothing on lock screen; queues from search/favorites/custom playlists have no network source cover at all (empty string or asset path). Also found tracks with non-null but dead S3 cover URLs, which bypass all ??-fallbacks by design.
Source: Claude Code session 2026-07-04, files app_audio_handler.dart, player_state.dart, pubspec.yaml (path_provider)
Suggested action: create new pattern page (reference [[two-layer-audio-stack]] and [[adr-002-just-audio-plus-audio-service]])

## [2026-07-04] candidate | go_sport | audio error recovery
Type: pitfall
Target applies_to: any-flutter-project-with-just-audio
What: Recovering from a broken playlist track (seekToNext/stop) synchronously inside playbackEventStream's onError crashes with "Cannot fire new event. Controller is already firing an event" — seek() writes into the same rxdart subject that is mid-dispatch; recovery must be deferred via Future(...) AND deduplicated with an in-flight flag because just_audio can emit multiple error events per failed source (otherwise two tracks get skipped).
Why interesting: general just_audio 0.9.x trap; the naive "skip on error" code works most of the time and then hangs the app with an unhandled reentrancy exception; the crash likely explains why such recovery code gets deleted from projects.
Context: broken track in a playlist made the whole playlist unplayable; restored old skip-on-error logic, hit the reentrancy crash, fixed with deferred + guarded recovery in AppAudioHandler.
Source: Claude Code session 2026-07-04, lib/core/audio/app_audio_handler.dart
Suggested action: create new pitfall page (related: [[two-layer-audio-stack]], [[adr-002-just-audio-plus-audio-service]])

## [2026-07-04] candidate | go_sport | player fast-path
Type: pitfall
Target applies_to: any-flutter-project-with-playback
What: A "same source — just seek" fast-path in play-queue logic must verify the native player actually has the queue loaded (status not idle/error); after a playback error the seek is a silent no-op while the optimistic state update still changes the UI (tile icon, mini-player), so taps look accepted but nothing plays.
Why interesting: the optimistic UI update masks the dead player, making the bug look like "tap is broken" instead of "player died earlier"; the fix is falling through to a full queue reload when the player is not alive.
Context: after a broken-track error killed the player, taps inside the current playlist changed UI but played nothing, while other playlists (full reload path) worked.
Source: Claude Code session 2026-07-04, lib/domain/state/player_state.dart (playQueue)
Suggested action: create new pitfall page (related: [[dual-mode-player-state]])

## [2026-07-04] candidate | go_sport | audio error recovery (refinement)
Type: pitfall
Target applies_to: any-flutter-project-with-just-audio
What: "Skip broken track on error" must NOT blindly seekToNext: both native plugins already auto-advance past a failed playlist item (Android re-prepares and seekTo(failed+1), up to 5 consecutive errors; iOS AVQueuePlayer skips failed items itself), and iOS additionally broadcasts preload errors for UPCOMING items while the current track is still playing. Blind seekToNext skips a good track (Android) or kills a playing track (iOS). Correct recovery: read the failed index from the error payload ({index: N} in PlatformException.details on both platforms), then compare with player.currentIndex — moved on & playing: do nothing; moved on & stalled: play() only; still on failed item: seekToNext+play; no next: stop.
Why interesting: the naive fix looks correct and passes a superficial test; the failure modes are platform-specific and only visible by reading both native plugin sources (AudioPlayer.java onPlayerError, AudioPlayer.m sendError/currentItem observer).
Context: follow-up to today's earlier "audio error recovery" candidate — debug logging showed the double-skip was caused by Android's native auto-advance racing our seekToNext, not by duplicate error events.
Source: Claude Code session 2026-07-04, lib/core/audio/app_audio_handler.dart; just_audio 0.9.46 sources
Suggested action: merge with today's earlier "audio error recovery" candidate into one pitfall page

## [2026-07-04] candidate | go_sport | safe-area / modal overlay
Type: pitfall
Target applies_to: any-flutter-project
What: For a full-screen showModalBottomSheet, the ONLY reliable inset source is MediaQueryData.fromView(View.of(context)) captured before the sheet opens and reinjected via builder: (_) => MediaQuery(data: mq, child: overlay). Two tempting alternatives both fail: (a) copying the parent's MediaQuery (MediaQuery(data: MediaQuery.of(context))) breaks when the source context sits under an AppBar/SafeArea — padding.top is already 0 there; (b) reading MediaQuery.viewPaddingOf inside the sheet also returns 0, because MediaQueryData.removePadding (which ModalBottomSheetRoute applies with removeTop:true when useSafeArea:false) subtracts the removed padding from viewPadding too (see media_query.dart removePadding impl) — viewPadding is NOT immune, contrary to intuition. fromView reads the window directly and bypasses every tree-level removal.
Why interesting: two independently plausible fixes are both wrong for non-obvious reasons; (b) was even "verified" by reading ModalBottomSheetRoute source (removePadding looks padding-only) — the trap is inside MediaQueryData.removePadding itself. Symptom either way: overlay controls (progress bars, close button) slide under the status bar, and a purely-unrelated host-screen refactor (swapping an in-scroll SliverAppBar for Scaffold.appBar) is enough to trigger it.
Context: story viewer opened via showModalBottomSheet from Home; regressed when Home's SliverAppBar was moved to Scaffold(appBar:) in a prior commit. Fixed by adopting the pattern already used by the project's full-player/search sheets (MediaQueryData.fromView + reinject), confirmed by on-device debugPrint showing viewPadding.top=0 inside the sheet.
Source: Claude Code session 2026-07-04, files lib/features/home/presentation/home/home_screen.dart and lib/features/home/presentation/story/story_overlay.dart
Suggested action: create new pitfall page

## [2026-07-04] candidate | go_sport | network error notification
Type: pattern
Target applies_to: any-flutter-project
What: Global network-error notification via a Dio onError interceptor feeding a broadcast event stream, consumed by a single ScaffoldMessengerKey listener at the app root. ErrorInterceptor maps DioException → an app-level enum (NetworkErrorKind: noConnection/timeout/server/unknown), and — for non-GET requests only, unless extra['silent']==true — pushes it into a NetworkErrorService (StreamController.broadcast). A StreamProvider exposes the stream; MainApp's ref.listen shows one SnackBar via a GlobalKey<ScaffoldMessengerState> on MaterialApp.router. Interceptor never swallows (always handler.next(err)), so call-site control-flow (optimistic rollback in LikeRegistry) still runs; with the global SnackBar in place the per-call rethrow becomes pure debug-break noise and is dropped.
Why interesting: cleanly separates the two concerns that get conflated in "add try/catch everywhere" — control-flow/recovery (rollback, error-state) stays LOCAL per call site because only it knows how to recover; user NOTIFICATION is cross-cutting and identical everywhere, so it belongs in ONE place. Method-based filter (non-GET → notify) exploits that load-GETs already surface their own error (loading→error→Retry) while mutations surface nothing, avoiding double-messaging without per-request tagging. Notification modelled as an event stream (not an error field in state) specifically to dodge the same-value-dedup and manual-reset pitfalls — two identical 500s in a row would silently drop the second SnackBar if kept in state.
Context: user got a raw DioException debug-break when a program-like POST hit a transient 500; wanted failures surfaced to the user "in one place" across all requests. Discussion established 3 error-handling categories in the codebase (caught+surfaced / caught+rollback+rethrow-to-nowhere / no try-catch at all e.g. custom playlist save-delete) and that only notification centralizes.
Source: Claude Code session 2026-07-04, files lib/core/network/network_error_service.dart, lib/core/network/interceptors/error_interceptor.dart, lib/core/di/network_providers.dart, lib/main.dart, lib/domain/state/like_registry.dart
Suggested action: create new pattern page; cross-reference (extends) [[api-client-with-config-and-interceptors]] and [[event-stream-vs-state-for-discrete-events]]

## [2026-07-06] candidate | go_sport | freezed
Type: pitfall
Target applies_to: any-flutter-project-using-freezed-codegen
What: Adding a field to a generated *.freezed.dart by hand (getter + copyWith) without declaring it on the source @freezed class — code compiles and callers use the field, but the next `dart run build_runner build` regenerates the file and silently deletes the field, breaking the build; and because toDomain()/the constructor can never populate the phantom field, it stays at its default (null) at runtime, silently breaking logic that depends on it.
Why interesting: two-headed trap. (1) Build-time: works until anyone regenerates, then compile error far from the edit. (2) Runtime NOW: field is unreachable from the real constructor, so it's permanently the default — here NewsArticle.likeId was always null on API load, so the unlike branch (likeId==null → treated as "like") posted a duplicate like instead of deleting. General signal: a *.freezed.dart changed in a diff while its source *.dart is NOT changed is always a red flag.
Context: reviewing AnaElez merge "removed news mock" (real Strapi news repo replacing mock). news_state.dart referenced article.likeId / copyWith(likeId:) but news_article.dart has no likeId field; only the generated freezed file was hand-patched. Violates go_sport rule "codegen only via dart run build_runner".
Source: Claude Code session 2026-07-06, files lib/domain/entities/news_article.dart, lib/domain/entities/news_article.freezed.dart, lib/domain/state/news_state.dart, lib/data/dto/news_dto.dart
Suggested action: create new pitfall page; cross-reference the freezed/codegen convention page if one exists

## [2026-07-21] ingest | Navigation & Modal — processed 7 candidates
Thematic ingest covering navigation and modal candidates from 2026-05-12 through 2026-07-04. Verified against `go_sport` branch `dev`. Note: the project lives at `C:\flutter\go_sport`, not the `C:\flutter-projects\go_sport` path assumed in earlier entries.

Created 4 pitfall pages and 1 pattern page:
- [[modal-in-stateful-shell-route]] — resolves the deferred 2026-05-12 investigation using the 2026-06-01 root-cause finding. Documented as a go_router bug (<=14.8.1 singular `_findCurrentNavigator` in `popRoute`), fixed by upgrading to >=15 / 17.x, not as an app-architecture problem.
- [[custom-transition-page-breaks-ios-swipe-back]] — from the 2026-06-15 candidate.
- [[hero-tag-collision-across-shell-branches]] — from the 2026-06-18 candidate.
- [[full-screen-modal-loses-safe-area-insets]] — from the 2026-07-04 safe-area/modal-overlay candidate.
- [[route-target-self-hydrates-from-id]] — from the 2026-07-03 navigation candidate, added to this ingest during planning as thematically adjacent.

Updated [[adr-004-go-router-stateful-shell]]: `go_router: ^14.2.0` -> `^17.0.0` (lock resolves 17.2.3 on `dev`). Replaced the "branch navigator keys for modal context resolution / see open investigation" consequence — that investigation is now closed — with consequences linking the three new shell pitfalls. Extended the `state.extra` consequence with a pointer to the id-addressable route pattern. Added a "stay current on the major version" note recording the near-zero 14 -> 17 migration cost and the three breaking changes on that path. `date` left at 2026-05-12: the version bump is a consequence of the original decision, not a new one.

Processed without producing a page:
- `stateful-shell-with-modal-overlays` (candidate 2026-05-12, paired investigation) — REJECTED as obsolete. The `BranchNavigatorKeyResolver` + `useRootNavigator: false` approach it describes was a workaround for the go_router bug now identified, not an architectural pattern. Confirmed absent from `dev`; all 11 modal call sites use `useRootNavigator: true`. Canonicalizing it would have prescribed a workaround for a bug that no longer exists. The reasoning is preserved in the Fix section of [[modal-in-stateful-shell-route]] as an explicit "do not do this".
- A go_router upgrade ADR under `projects/go_sport/` was considered and declined — the bump is a forced fix for a framework bug, not a choice between alternatives, and is fully covered by the pitfall page plus the adr-004 update.

Verification findings on `dev`: `BranchNavigatorKeyResolver` absent; `MediaQueryData.fromView` reinjection used at all 4 full-screen sheet entry points; `fadeSlidePage` defined in `page_transitions.dart` but with zero call sites (all routes reverted to `builder:`/MaterialPage); no Flutter `Hero` widgets remain (the `*Hero` classes are the project's own header-section widgets, not shared-element heroes). Per Principle 7 the pitfall pages capture the traps and their canonical fixes regardless of current implementation state.

Navigation-adjacent candidates deliberately left for later thematic ingests: 2026-06-19 `guest-session-limit` (gating belongs in a router redirect backed by long-lived state — better ingested together with the two other guest candidates as an auth/guest-session theme) and 2026-06-15 `layout` (ListView with `padding == null` auto-injecting `MediaQuery.padding` — safe-area-adjacent but a layout concern).

## [2026-07-21] ingest | Like-state & cross-screen sync — processed 5 candidates
Thematic ingest covering the like-registry work of 2026-05-12 through 2026-06-16. Verified against `go_sport` branch `dev` at `C:\flutter\go_sport`.

Refined [[domain-vs-screen-state]]: replaced the categorical criterion ("business data vs UI-only state") with the operational consumer-count rule — two or more consuming screens means `domain/state/`, exactly one means the screen's controller, even when the state holds domain entities. Rewrote the first failure mode in Rationale around drift between copies rather than data category, and added a paragraph on why the count is decidable where the category is not (false positives on legitimately local entity lists, false negatives on shared data that "doesn't feel like business data"). Reframed the `PlayerState` example under the new criterion, added `ProfileState.user` as the worked example of promotion when a second consumer appears, and added a note that consumers are counted as distinct reading sites, not route entries. Source: 2026-05-12 candidate.

Created 2 cross-project pages:
- [[repo-side-effects-on-domain-state]] (pitfall) — from the 2026-06-01 anti-pattern candidate. Records the inconsistency-within-one-class diagnostic (one method pushes to state, a sibling returns and lets the caller merge) and the constructor-dependency signature as the early detection points.
- [[reactive-derived-view]] (pattern) — from the 2026-06-16 candidate. Records the Set-diff guard as load-bearing for two distinct reasons (self-trigger suppression and independence from Riverpod list equality) and why `autoDispose` does not solve the problem.

Created the first project-scoped page: [[projects/go_sport/adr-001-central-like-registry]]. `projects/go_sport/` was previously empty. Numbering note: `adr-001` matches both ingest order and chronology — 2026-06-01 is the earliest pending go_sport ADR candidate (external-links 06-13, guest-choice 06-19, image-orientation 07-02, persist-player-session 07-03 follow). Where a later ingest surfaces an older decision, assign the next free number rather than renumbering.

Scope decision by Sam during planning: the central-registry architecture is treated as **go_sport-specific**, so no portable pattern page was created for it. The full structure (entity lists rather than id maps, `initSession` + `_loadAllPages` population, toggle-by-entity with optimistic update and rollback, library screens as views, pure repositories) is recorded in the project ADR instead. Consequence for future lint: the log entries of 2026-06-01 and 2026-06-16 contain `[[central-user-collection-registry]]` references to a page that was deliberately not created — these are historical candidate text, not page content, and are not dangling references to fix.

Processed without producing a page:
- `like-state-central-registry` (2026-05-29, recorded twice — once with mojibake Cyrillic, once clean; one candidate) — noted as historical context only. This is the draft architecture: id-only maps as truth, list controllers patching `isLiked` on their entities, registry seeded through repository side effects. All three traits were superseded by the 2026-06-01 finale and none survive on `dev`. Its value is as the path to the final design; its central mistake is canonicalized separately as [[repo-side-effects-on-domain-state]].

Also raised and closed during planning: five `MyXController` classes on `dev` (albums, tracks, artists, programs, episodes) are 1:1 wrappers that watch a registry slice and expose `refresh()`, which reads as proxying under the existing "screens MUST NOT proxy domain data through their controller" clause. Sam's call: the local/global principle stands as written, the arrangement is a project specific, and no exception clause or deviation record goes into the wiki. The clause was left untouched. (`my_playlists` is a different shape — it reads `myPlaylistsState`, which owns playlist composition, not the registry.)

Two thematically adjacent candidates were proposed during planning and NOT processed, so they remain pending: 2026-07-02 `riverpod-lazy-global-read` (short-lived controller reading a lazily-initialized global via one-shot `ref.read`, gating on `.isEmpty` of a merged list) and the 2026-06-16 `player / state` candidate (screen toggle following a domain discriminator via `ref.listen`, which targets [[domain-vs-screen-state]] and [[dual-mode-player-state]]).

## [2026-07-21] ingest | Player session & audio — processed 9 candidates
Thematic ingest covering the player-persistence and audio-error work of 2026-06-10 through 2026-07-04. Verified against `go_sport` branch `dev` at `C:\flutter\go_sport`.

Refined [[two-layer-audio-stack]] (candidate 2026-07-03 `load-queue-without-autoplay`): added the load-versus-play split to Solution — the handler's queue method only calls `setAudioSource` (load + seek) and never `play()`; `playQueue`/`resumeMusic` compose the two inside one try/catch, `loadPausedQueue` stops after the load. Recorded the rationale (this mirrors just_audio's own separate `setAudioSource`/`play`; bundling was the anomaly; a `setQueue` that plays is a lying name that makes "loaded but silent queue" inexpressible) and what the split unlocks (cold-start restore, mini-player priming, iOS cold-start autoplay restrictions). Added one Trade-offs bullet and one Common-pitfalls bullet.

Refined [[dual-mode-player-state]]: folded the second half of the 2026-06-16 `player / state` candidate into Common pitfalls — independence between a UI panel toggle and the playback-mode discriminator does not mean the toggle never reacts. On transitions it should follow, via `ref.listen` with a flip-on-mismatch guard, not `ref.watch` (which glues them, making manual toggling impossible, and cannot drive `setState`/animation during build). Noted that the guard works because the flag is boolean.

Created 4 cross-project pages:
- [[persist-and-restore-playback-session]] (pattern) — from candidate 2026-07-03 `persist-and-restore-player-session`, with the same-day correction candidate folded in as the core of the Solution rather than as a separate page. The two-key split (heavy snapshot written rarely, position as a single int written on a ~10s timer) and the reason lifecycle-only saves lose position (the kill races the async flush) are recorded as the load-bearing detail, along with the flatten-source-to-avoid-circular-import gotcha and restore-or-prime-always-paused.
- [[system-player-artwork-fallback]] (pattern) — from candidate 2026-07-04. Both halves kept: the in-app fallback chain cannot protect the system player (art fails after the URI is handed to another process, with no callback), and the remedy (materialize the asset to a file once, guard the `??` chain by url scheme so non-network values do not terminate it).
- [[audio-error-recovery]] (pitfall) — merged from the 2026-07-04 candidate and its same-day refinement, plus findings from `dev`. Single page covering four symptoms of one cause.
- [[player-fast-path-masks-dead-player]] (pitfall) — from candidate 2026-07-04 `player fast-path`, extended with the second precondition found on `dev`.
- [[collection-play-button-reflects-active-source]] (pattern) — from the 2026-06-10 candidate and its same-day correction, proposed during planning and approved for inclusion. Named more descriptively than the candidate's `collection-play-pause-toggle` because the transferable idea is the criterion (queue-source identity, not track membership), not the toggle. The correction is folded in as the core: two booleans, since a single `isPlaying`-gated one restarts the collection from track one on resume.

Created [[projects/go_sport/adr-002-persist-player-session]]. Number 002 is the next free number per the agreed rule — project ADR numbering follows wiki ingest order, not the chronology of the decisions. Records go_sport's specific choices (snapshot shape, 10s interval, restore-over-prime priority, featured-playlist prime that is deliberately not saved, paused-never-autoplay, radio excluded from persistence, corrupt snapshots dropped on init) and five rejected alternatives. The portable structure lives in the pattern page; the ADR does not repeat it.

Three findings on `dev` that were NOT in any candidate and are folded into the pages above:
- The play fast-path carries a second precondition beyond "player is alive": the loaded queue must still match the on-screen list. With paginated sources the visible list outgrows the loaded queue, so the tapped index must exist and hold the same track id. Folded into [[player-fast-path-masks-dead-player]].
- A second in-flight flag separates load-time errors from playback-time errors: a failing load emits the same events as a failing playback, but during a load they belong to the queue-loading retry path, and without the flag two mechanisms command the player simultaneously. There is also a retry loop stepping over broken *starting* indices, needed because a failed initial load can take the native player down with it. Both folded into [[audio-error-recovery]].
- A `KNOWN UNFIXED` limitation recorded in the handler on 2026-07-05: the Android plugin stops auto-advancing after five cumulative errors per load, stranding the player in permanent buffering, deliberately not worked around because the planned just_audio upgrade makes the fix obsolete.

Scope decision by Sam on that last finding: split it. The transferable half — just_audio 0.9.x reports errors as stream errors, which is the wrong shape for recoverable failures, and 0.10.x redesigns this (`maxSkipsOnError` counting consecutive rather than cumulative errors and resetting on success; `errorStream` making errors data and removing the reentrancy hazard) — went into the closing section of [[audio-error-recovery]]. The go_sport-specific upgrade plan and its "delete this on upgrade" checklist did NOT go into the wiki: it is pending work, which §1 excludes, and the code comment beside the code being deleted is the right place for it. A project ADR recording the accepted limitation plus migration plan was considered and declined as premature until the upgrade happens.

Candidate status notes:
- 2026-06-16 `player / state` is now **fully processed**. Its first half (the operational consumer-count criterion) was folded into [[domain-vs-screen-state]] during the Like-state ingest and logged there as partial; its second half (reactive sync via `ref.listen` on transitions) is folded into [[dual-mode-player-state]] here.
- 2026-07-03 `persist-player-session (correction)` and 2026-07-04 `audio error recovery (refinement)` produced no pages of their own by design — both were folded into the pages their parent candidates produced.
- 2026-06-10 `collection-play-pause-toggle (correction)` likewise folded into the pattern page rather than becoming a separate embedded pitfall.

Still pending from earlier proposals: 2026-07-02 `riverpod-lazy-global-read`.

## [2026-07-21] ingest | Design system & tokens — processed 4 candidates
Thematic ingest of the four 2026-06-12 go_sport design-system candidates. Verified against `dev` at `C:\flutter\go_sport` (DSIconSize numeric scale, DSRadius.xxl=28, DSLayout.bottomBarClearance/heroContentBottom all present and in use). Scope was deliberately kept to transferable principles only — the wiki does not document the design system itself or the project's specific scales, which are design-driven project specifics.

All four 2026-06-12 candidates are now processed:
- `design-system (per-scale naming)` — folded into [[design-system-tokens-mandatory]] as one Implications bullet: the transferable heuristic only (t-shirt names stay legible to ~5–6 values, switch a denser scale to numeric value-named tokens, chosen per scale). The project's concrete scales, values, and renames were deliberately left out.
- `design-system (functional layout metrics)` — folded into [[design-system-tokens-mandatory]] as one Implications bullet: separate the rhythm scale from functional layout metrics, the criterion being generality of context rather than frequency. The concrete DSLayout class and its values were left out.
- `refactoring-safety (token rename)` — created [[constant-value-drift-on-rename]], reframed wider than "token rename" to any named-constant refactor (reassigning a constant's value in place silently retargets every call site; rename usages first, redefine second, grep to confirm). Design-token scales appear as the leading example inside the page, not as the page's frame.
- `refactoring-safety (mass migration gotchas)` — REJECTED, no page. Recorded here as a precedent for future similar candidates: **operational details of one specific migration vs transferable architectural knowledge.** The candidate's content (regex forms like `\b16\b` matching inside `16.0`, EdgeInsets named-vs-positional shapes, SizedBox gap-vs-size disambiguation, PowerShell/Windows-1252 encoding traps) is procedure for executing one literal→token migration on one codebase. It is real and was useful, but it is not knowledge that transfers as a principle to the next project — its scope is a single task, not a class of decisions. When a candidate is a how-to for one migration run rather than a rule that outlives it, it does not earn a page.

One update ([[design-system-tokens-mandatory]], two Implications bullets) and one new page ([[constant-value-drift-on-rename]]). No project ADR, no pattern page — the compact filter for this theme.

Checked for adjacent candidates and deliberately left them for later themes: 2026-06-23 `unified-network-image-fallback` (a DS component, not a token/naming concern) and 2026-06-18 `background-image-bottom-card-form` (a layout idiom). Neither is in scope for design-system tokens.

## [2026-07-21] ingest | Auth + Data + Layout — processed 6 candidates, 2 folds
Thematic ingest across auth, data-hydration, and layout candidates from 2026-05-26 through 2026-07-01. Verified against `dev` at `C:\flutter\go_sport`. Several fixes are already applied on `dev` (album populate synced with playlist populate; `my_artists`/`new_episodes` lists now carry explicit padding; cache-dims removed from expired_guest) — per Principle 7 the pitfall pages capture the trap and its fix regardless of current implementation state.

Created 5 pitfall pages and 1 pattern page:
- [[strapi-oauth-provider-email-conflict]] (pitfall) — from 2026-05-26. Generalized across all bundled Strapi `users-permissions` OAuth providers, not just Google; framed as a server-side product/linking decision rather than a client bug.
- [[dual-source-of-truth-state]] (pitfall) — from 2026-06-19 `dual-source-of-truth-guest-state`, deliberately raised to a general architectural pitfall: one concept mirrored in a Notifier and a backing store, a write to only one drifts, mutations must go through the owner. Guest mode is the worked example inside the page, including the diagnostic signal (a correct owner-method with zero call sites).
- [[shared-dto-partial-populate]] (pitfall, FOLD) — merged 2026-06-16 and 2026-07-01. Kept compact: symptom + cause + fix, with the per-widget-fallback-masking factor as a single aggravating sentence rather than a second section.
- [[background-image-bottom-card-form]] (pattern, FOLD) — merged 2026-06-09 and 2026-06-18. The 06-09 candidate gave the formula and the content-driven vs fixed-height fork; 06-18 added keyboard survival (resizeToAvoidBottomInset + overlap, IntrinsicHeight making Spacer legal in a scroll view) and alignment-controls-crop, plus the broken fixed-percent variant as a contrast pitfall.
- [[listview-null-padding-consumes-safe-area]] (pitfall) — from 2026-06-15.
- [[image-cache-dims-distort-aspect-ratio]] (pitfall) — from 2026-06-19.

No project ADR and no updates to existing pages, by design — the theme was scoped to new transferable pages only.

Rejected (recorded with explicit reasons):
- 2026-06-19 `guest-session-limit` — too narrow. A transferable kernel exists ("do not gate time-based access through ephemeral widget state; gating belongs in a router redirect backed by long-lived state"), but the candidate's substance is tied to go_sport's specific StatefulShellRoute structure and does not carry to a standalone page on its own.
- 2026-06-19 `guest-choice-is-per-session` — a product decision (go_sport's freemium logic), not an architectural one. Another project could choose the opposite based on its business model, so it is project-scoped precedent at most, not cross-project knowledge. If captured at all it would be a go_sport ADR, not a wiki principle.

Deferred to the next ingest (not rejected):
- 2026-07-02 `exif-orientation-not-baked` — considered for this theme's "images" scope and held back to keep it paired with the 2026-07-02 `image-orientation-package` ADR (the `image` vs `flutter_image_compress` package choice) in Ingest 6. Splitting the pair across sessions would lose the natural "pitfall → chosen fix" link and the cross-page linking between them. The images angle here is already covered by [[background-image-bottom-card-form]] and [[image-cache-dims-distort-aspect-ratio]], neither of which is about EXIF.

Still pending from earlier proposals: 2026-07-02 `riverpod-lazy-global-read`.

## [2026-07-21] ingest | Images + State + Architecture — final ingest of the series
Final thematic ingest. Verified against `dev` at `C:\flutter\go_sport`. After this pass the log has no unprocessed candidates — every candidate is either turned into a page, folded into one, or explicitly rejected with a reason. A full audit of all candidate entries was performed as part of this ingest (see the closeout below).

Created 6 pages:
- [[unified-network-image-fallback]] (pattern) — from 2026-06-23. Boundaries kept explicit: avatars (own SVG-placeholder semantics), local `Image.asset` (network-only widget), and `BoxDecoration`/`DecorationImage` (decoration, not child) are deliberately excluded.
- [[centralized-network-error-notification]] (pattern) — from 2026-07-04 `network error notification`. Records the control-flow-vs-notification split, the never-swallow (`handler.next`) rule, the non-GET method filter, and the event-stream-not-state choice that dodges same-value dedup.
- [[exif-orientation-not-baked]] (pitfall) — from 2026-07-02. Deferred from Ingest 5 specifically to keep it paired with the ADR below; the pairing is now in place with two-way cross-links.
- [[non-exhaustive-loading-guard]] (pitfall) — from 2026-05-17.
- [[riverpod-lazy-global-read]] (pitfall) — from 2026-07-02, pending since Ingest 2, now closed.
- [[projects/go_sport/adr-003-image-orientation-package]] (project ADR) — from 2026-07-02 `image-orientation-package`. Number is adr-003, the next free number in `projects/go_sport/` (adr-001 like registry, adr-002 persist player); the task's suggested "adr-004" was corrected after verifying the directory. Records the pure-Dart `image` choice, the rejected native `flutter_image_compress` alternative, and the revisit-if-image-processing-broadens condition. Paired two-way with [[exif-orientation-not-baked]] (ADR = the chosen solution, pitfall = the problem).

Processed without creating a page (3):
- 2026-05-12 `canonical-branch-for-wiki-operations` — a curation-methodology candidate whose suggested action was to add a canonical-branch section to CLAUDE.md. Already done: CLAUDE.md §6.0 "Canonical project branch" exists. No wiki page — it is curation methodology, not Flutter engineering knowledge.
- 2026-05-12 `worktree-vs-main-checkout-invisibility` — out of scope. A pitfall about Claude Code Desktop worktree behaviour (`applies_to: workflows-using-claude-code-desktop`); it is neither Flutter engineering knowledge nor wiki-curation methodology, so it does not belong in this knowledge base.
- 2026-05-12 `spec-vs-code-divergence-statenotifier` — a project-doc concern (align go_sport's ARCHITECTURE_SPEC.md, which shows `StateNotifier`, with the code, which uses `Notifier`), not wiki content. The wiki already canonicalizes `Notifier` explicitly in [[domain-vs-screen-state]], so the knowledge is captured; the remaining action is on the project's spec.

Rejected during the final audit (3) — these were never logged as processed but had been discussed earlier in series planning; recording the rejections here closes them:
- 2026-06-13 `external-links` — no ADR. It records the choice of a standard package (`url_launcher`) with no alternatives weighed and no trade-off; that is documentation of state ("we use url_launcher"), not an architectural decision. The "no abstraction for a single call site" judgment it also carries is already implicit in the wiki's avoid-over-engineering stance.
- 2026-06-17 `story-viewer` — no pattern. Its transferable core (screen controllers are for data/async; vsync/ticker + ephemeral session state live in widget `State`) is already canonicalized as principle in [[domain-vs-screen-state]]. The concrete Instagram-style story viewer is go_sport UI specifics, not transferable architecture.
- 2026-07-06 `freezed` (hand-edited generated file) — no pitfall. It is a developer mistake (someone bypassed the codegen workflow), not an architectural lesson. "Do not hand-edit generated files" is baseline Flutter/Freezed knowledge and does not warrant a page.

Series closeout: all candidates from 2026-05-12 through 2026-07-06 are now accounted for across Ingests 1–6. Nothing remains deferred or pending. The log is a clean audit trail with no open items.

## [2026-07-27] candidate | go_sport | text-scaling
Type: pattern
Target applies_to: any-flutter-project
What: Lock (or clamp) OS font-size accessibility scaling at the app root via MaterialApp `builder` using MediaQuery.withNoTextScaling / MediaQuery.withClampedTextScaling, not the deprecated textScaleFactor.
Why interesting: General Flutter technique with a non-obvious gotcha — an app can have multiple MaterialApp roots (main router, offline screen, init-error fallback), and EACH needs the builder or scaling leaks through on those screens. Also carries the accessibility trade-off: full noTextScaling is a regression vs clamping; record both so the choice is explicit.
Context: System font enlargement was breaking go_sport's layout. Chose full lock (withNoTextScaling) as the first step via a shared _lockTextScale helper applied to all three MaterialApp instances; may switch to clamped scaling later once a safe max is measured.
Source: Claude Code session 2026-07-27, file lib/main.dart
Suggested action: create new pattern page

## [2026-08-03] candidate | go_sport | playback
Type: pitfall
Target applies_to: flutter-projects-with-playback
What: A track-tile tap handler that calls playQueue unconditionally restarts the current track instead of pausing it — the "same source, just skip" fast path resolves to seek(Duration.zero, index: sameIndex), which is a rewind, not a toggle.
Why interesting: General trap, and it hides behind a correct-looking implementation. The tile already renders a play/pause equalizer indicator, so users read it as a play/pause control; the handler has no matching fork. The failure is silent — no error, audible playback continues, only the position jumps to 0 — so it survives casual testing. Related page [[collection-play-button-reflects-active-source]] covers the same fork for the collection HEADER button but explicitly scopes its criterion to queue-source identity; the tile needs the weaker per-item criterion (currentTrack.id == track.id), so the existing page does not cover this case and its "two booleans" reasoning does not transfer directly.
Context: Reported in go_sport search results — second tap on a playing track restarts it. Diagnosis showed the gap is not search-specific: seven track-list screens (album, playlist, my_favorites, new_episodes, artist, program_details, radio_page) all call playQueue unconditionally from the tile tap; only add_tracks_bottom_sheet has the toggle fork. Header play buttons on the same screens DO have the fork, which is what makes the tile omission easy to miss.
Source: Claude Code session 2026-08-03, files lib/features/search/presentation/search/search_screen.dart (_TrackRow), lib/domain/state/player_state.dart (playQueue), lib/core/audio/app_audio_handler.dart (skipToQueueItem)
Suggested action: create new pitfall page, cross-linked with [[collection-play-button-reflects-active-source]] (header vs tile: same fork, different criterion) and [[player-fast-path-masks-dead-player]] (the fast path is what converts the missing fork into a rewind)

## [2026-10-21] candidate | GastroSuit | wiki-versions
Type: principle
Target applies_to: wiki-wide
What: Principle and pattern pages should not pin library versions or version-specific APIs; versions belong to project pubspecs and to ADRs only.
Why interesting: Version-pinned content goes stale silently and then misleads. Examples found: [[domain-vs-screen-state]] and [[adr-001-riverpod-for-state-management]] name `AutoDisposeNotifier` (removed in Riverpod 3, now a provider modifier); [[freezed-immutable-state]] shows `@freezed class X with _$X`, which does not compile in Freezed 3 (classes must be `abstract` or `sealed`). The same staleness already hit go_sport's ARCHITECTURE_SPEC (StateNotifier template while code used Notifier).
Context: Writing GastroSuit's ARCHITECTURE_SPEC from scratch; decided the spec carries no code and no versions, and the user asked for the same discipline in the wiki.
Source: Claude Code session 2026-10-21
Suggested action: update [[adr-001-riverpod-for-state-management]] (Riverpod 3 is current; re-evaluation condition met), [[domain-vs-screen-state]], [[freezed-immutable-state]] — strip version-specific API names or mark examples as version-bound

## [2026-10-21] candidate | GastroSuit | shared-widgets-placement
Type: principle
Target applies_to: any-flutter-project
What: Reusable widgets live in two places, not three: `design_system/components/` (domain-agnostic, plain-value props) and `features/shared_widgets/` (domain-aware, dumb). Domain-agnostic helpers like `EqualizerIndicator` go to the design system; `shared/widgets/` is dropped.
Why interesting: In go_sport `shared/widgets/` held a single file and its boundary with `design_system/components/` was undefined; the deciding axis is "knows the domain or not", which yields exactly two places. Also: placement follows consumers (screen → feature → shared_widgets), moving up only when a new consumer appears; the design handoff's component list does not decide placement.
Context: Defining reusable-widget rules for GastroSuit's spec.
Source: Claude Code session 2026-10-21, C:\flutter\gastro_suit\docs\ARCHITECTURE_SPEC.md §8
Suggested action: update [[layered-architecture]]; cross-link [[shared-widgets-stay-dumb]]

## [2026-10-21] candidate | GastroSuit | dto-without-freezed
Type: decision
Target applies_to: any-flutter-project
What: DTOs use json_serializable on plain classes, without Freezed; Freezed stays for entities and state classes.
Why interesting: A DTO lives for one parse-and-map; Freezed's `==`/`copyWith` are never used there and double the generated code. The value of codegen for DTOs is generated `fromJson` (no key typos, typed failures at parse time, nested objects, `fieldRename`). go_sport ignored the ADR entirely and hand-wrote all DTOs (likely because of Strapi nesting), so the ADR did not reflect practice either.
Context: GastroSuit spec discussion; final confirmation pending a look at the Swagger response shapes.
Source: Claude Code session 2026-10-21
Suggested action: update [[adr-003-freezed-for-models]] and [[dto-to-domain-mapping]]

## [2026-10-21] candidate | GastroSuit | state-ownership-refinements
Type: principle
Target applies_to: any-flutter-project
What: Three refinements to the consumer-count rule: (1) the real prohibition is a stored copy of domain data in a controller, not data passing through it — a controller may derive (filter, sort, combine) from domain state via watch; (2) data obviously shared from the start (cart, user, session) goes to domain state immediately, consumer counting is for unclear cases; (3) lifetime is a separate axis from placement — "don't refetch on re-entry" is solved by provider lifetime (autoDispose or not), not by promoting to domain state.
Why interesting: The current blanket "MUST NOT proxy" forbids legitimate derived views; go_sport's favorites controllers show the harmless-but-pointless pass-through case, while the harmful case is a ref.read copy that drifts ([[dual-source-of-truth-state]]).
Context: Rewriting the state section of GastroSuit's spec.
Source: Claude Code session 2026-10-21; go_sport lib/features/favorites/presentation/my_albums/my_albums_controller.dart
Suggested action: update [[domain-vs-screen-state]]; cross-link [[dual-source-of-truth-state]], [[reactive-derived-view]]

## [2026-10-21] candidate | GastroSuit | error-presentation
Type: pattern
Target applies_to: any-flutter-project
What: Separate the architectural part of error handling (one app-level `AppException`, created only at the network boundary; the receiver decides recovery) from presentation (SnackBar / dialog / inline), which is a per-case design decision and not part of the architecture.
Why interesting: go_sport had no app error type: `DioException` leaked into domain state and controllers, and Strapi error-body parsing was duplicated in three places. Conversely, adopting [[centralized-network-error-notification]] by default pre-decides UX before the design is known.
Context: GastroSuit spec errors section.
Source: Claude Code session 2026-10-21; go_sport lib/domain/state/notifications_state.dart, change_password_controller.dart, confirm_delete_controller.dart
Suggested action: create new pattern page (app-level exception at the network boundary); add a "not by default — presentation is a design decision" note to [[centralized-network-error-notification]]

## [2026-10-21] candidate | GastroSuit | feature-folder-template
Type: pattern
Target applies_to: any-flutter-project
What: One feature folder template: `features/<feature>/presentation/<screen>/{<screen>_screen, <screen>_controller (optional), widgets/}` plus `<feature>/presentation/widgets/`; multi-step flows are one feature with one folder per step (no sub-features); a complex widget may have its own controller next to it; sheets and dialogs follow the same placement.
Why interesting: Without an explicit template go_sport drifted into four layouts (~20 screens on the template, 8 with an extra sub-feature level, 12 flat files, 1 without presentation), mostly in multi-screen flows like auth. go_sport's `home` feature is the clean reference.
Context: GastroSuit spec features section.
Source: Claude Code session 2026-10-21; go_sport lib/features/
Suggested action: create new pattern page or merge into [[layered-architecture]]

## [2026-10-21] candidate | GastroSuit | provider-placement
Type: principle
Target applies_to: any-flutter-project
What: A provider is declared next to the class it provides; only interface→implementation bindings (repositories) are collected in `core/di/`. The resulting domain → core/di → data import chain is accepted rather than removed with override ceremony.
Why interesting: "All infrastructure providers in core/di" (go_sport spec) was too broad and was not followed in code (6 providers next to their classes), for good reason. The remaining transitive import is compile-level only; removing it costs a list of overrides in main for no behavioural gain.
Context: GastroSuit spec DI section.
Source: Claude Code session 2026-10-21
Suggested action: update [[layered-architecture]] (Implications) and/or [[async-init-provider-override]]

## [2026-10-21] candidate | GastroSuit | string-route-literals
Type: pitfall
Target applies_to: flutter-projects-with-go-router
What: Route paths written as string literals in features drift from the router: go_sport has ~98 `context.push` / `context.go` literals vs 14 uses of route constants, and Home pushes `/album/$id` while the route only exists under `/music/album/:id`.
Why interesting: Nested shell routes make full paths non-obvious; a literal with a wrong prefix compiles and fails only at runtime on that tap. Referencing routes through constants or builder functions prevents it.
Context: go_sport code survey while writing GastroSuit's navigation rules.
Source: Claude Code session 2026-10-21; go_sport lib/features/home/presentation/home/home_screen.dart:42-48, lib/core/navigation/routes.dart
Suggested action: create new pitfall page

## [2026-10-21] candidate | GastroSuit | localization-from-day-one
Type: principle
Target applies_to: flutter-projects-with-multiple-languages
What: For multi-language apps, localization is structural from the first screen: all UI strings from ARB (gen-l10n), selected language is persisted app state, backend content language is sent by an interceptor rather than per repository.
Why interesting: Retrofitting means extracting literals from every screen; the locale header is a cross-cutting request concern that belongs with the other interceptors. Not covered by the wiki at all.
Context: GastroSuit has several languages.
Source: Claude Code session 2026-10-21
Suggested action: create new principle page

## [2026-10-21] candidate | GastroSuit | spec-documents
Type: principle
Target applies_to: any-flutter-project
What: A project architecture spec states structure and rules only — no code templates, no library versions, rules phrased positively and kept only if they prevent a concrete problem; a "deviations from the knowledge base" table records where and why the project departs from the wiki.
Why interesting: go_sport's spec matched its own code only ~60% after a few months; the stale parts were exactly the code templates and implementation details, while the layer structure held up. Earlier entry `spec-vs-code-divergence-statenotifier` was closed as a project-doc concern; this generalizes it.
Context: Rewriting the spec for GastroSuit based on go_sport's.
Source: Claude Code session 2026-10-21, C:\flutter\gastro_suit\docs\ARCHITECTURE_SPEC.md
Suggested action: create new principle page (or a curation note if out of scope)

## [2026-10-21] candidate | GastroSuit | Claude Design handoff carries stale values next to live ones
Type: pitfall
Target applies_to: any-flutter-project
What: A Claude Design "send to Claude Code" bundle contains several older snapshots of the same tokens/components/screens (export/, handover/*, design_handoff_*/, components/*.json) plus CSS `var(--token, fallback)` fallbacks from earlier palettes; only the files imported by the live screens (`project/_ds/*` + `*.dc.html`) are current.
Why interesting: Reading the wrong copy or trusting a fallback silently produces wrong values (e.g. `var(--radius-md, 16px)` where the token is 12px; indigo hex fallbacks in a red-branded system); token files across copies differed in 8 of 12 files.
Context: First analysis of the GastroSuit (Coffee House) handoff to derive the design system and judge screen readability.
Source: Claude Code session 2026-10-21, coffee-house-ui/project/
Suggested action: create new page (reading a Claude Design handoff: identify the live source by the primary screen's imports, ignore CSS fallbacks)
