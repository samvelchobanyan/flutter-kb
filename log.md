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

## [2026-06-13] candidate | go_sport | external-links
Type: decision
Target applies_to: project: go_sport
What: url_launcher chosen for opening external URLs (YouTube channel link on program details banner), launched inline in the screen's onTap with LaunchMode.externalApplication
Why interesting: first external-link capability in the project; establishes url_launcher as the package and "dumb banner takes onTap, screen owns the launch" as the wiring; no core helper introduced (single call site, avoid premature abstraction)
Context: program details screen — YoutubeBanner must open the program's YouTube channel; URL comes from a per-program "Youtube" field on the Strapi program API mapped onto the Program entity/DTO
Source: Claude Code session 2026-06-13, files youtube_banner.dart, program_details_screen.dart, program.dart, program_dto.dart, pubspec.yaml
Suggested action: create new project-scoped ADR page

## [2026-06-16] candidate | go_sport | reactive-derived-view-on-shared-composition
Type: pattern
Target applies_to: any-flutter-project (Riverpod; a per-id detail screen materializes heavy data whose lightweight "composition" is owned by a global single-source-of-truth that other screens mutate)
What: A per-id screen controller holds a heavy materialized view (full Track objects of one opened custom playlist) while a global Notifier owns the lightweight composition of that same resource (the list of member ids + count). The local view must stay consistent with external mutations to the composition. Fix is pull-based, not push: in the local controller's build(arg), `ref.listen` the global state selecting THIS id's composition (its member ids); when the selected composition changes, compare it as a Set against the ids already materialized locally — if they differ, refetch the heavy data (and re-read the updated lightweight object from global to keep count consistent); if they are equal, do nothing. The Set-content comparison is the load-bearing guard: it (a) filters out the controller's own edits echoing back through the global owner (self-trigger loop — local edit → writes global → listener fires → would refetch redundantly), and (b) makes correctness independent of Riverpod's List equality (selecting a List<String> fires on every global rebuild because List identity changes; the explicit Set diff decides whether a refetch is actually warranted).
Why interesting: This is the residual sync problem that survives even after composition ownership is centralized. go_sport had two independent caches of the same backend resource: global myPlaylistsState (custom playlists with trackDocIds + trackCount, loaded via a documentId-only populate) and per-id customPlaylistControllerProvider (full tracks, loaded via its own getCustomPlaylistTracks call). "Add to playlist" from a track's options menu on Search/Album/Favorites (scenario A) mutated backend + global composition but never touched the local controller, which is non-autoDispose and early-returns on re-entry (`if (state is PlaylistDetailsData) return`). Result: add a track from Search, re-open the already-visited playlist, track is missing until app restart; the card count (global) and the in-screen track list (local) drift. autoDispose was considered and rejected: its trigger is "no listeners", which only fires when the detail screen is fully torn down (popped), NOT when the user jumps to another tab/route to perform scenario A while the detail screen stays mounted in its shell branch or under the top route — the common trigger path. So the reliable refresh trigger is "the shared composition changed", not "the screen was re-entered". Generalizes to any heavy-view/light-composition split: a cart screen vs a global cart-item-id set, a thread view vs a global unread-id set, etc. Pairs with the operational consumer-count rule (2026-05-12 candidate): full tracks are consumed by one screen → legitimately local; composition is consumed by many → global; the bridge between them is a reactive derived view, not a second owner.
Context: Architectural discussion in go_sport on how songs are added to custom playlists. Two add-paths exist: scenario A (track options → showAddToPlaylistBottomSheet, a Track-family controller that diffs selected vs initial playlist membership and PUTs each affected playlist, then updates global via myPlaylistsState.updatePlaylist) and scenario B (inside the playlist screen → showAddTracksBottomSheet → customPlaylistController.addTracks, which appends full tracks locally and saves, updating global in save()). B stays consistent both ways because it owns the local view and pushes up; A updates only global, so the local view goes stale. Converged on: keep global as sole composition owner, make the local controller subscribe to its own id's composition and refetch-on-diff. Refetch (not in-memory reconcile) chosen deliberately — data is small, external mutations rare, and additions would need the full Track payload anyway, so in-memory diffing buys nothing.
Source: Claude Code session 2026-06-16, files lib/features/playlists/presentation/playlist/custom_playlist_controller.dart (the per-id local view), lib/domain/state/my_playlists_state.dart (global composition owner), lib/features/playlists/presentation/bottom_sheets/add_to_playlist_controller.dart (scenario A, the external mutator), lib/data/repositories/custom_playlist_repository_impl.dart (the two independent reads: getCustomPlaylists documentId-only vs getCustomPlaylistTracks full populate).
Suggested action: create new pattern page (naming: [[reactive-derived-view]] or [[derived-view-refetch-on-composition-change]]). Relates to [[domain-vs-screen-state]] (the operational consumer-count rule — this is the wiring between a legitimately-local heavy view and a legitimately-global light composition), [[central-user-collection-registry]] (the global owner is exactly that kind of registry; this is what a heavy per-id detail view does on top of it), and [[selector-providers-for-rebuilds]] (the listen uses a narrow per-id `.select`). Could double as a pitfall ("two independent caches of one backend resource; mutating one leaves the other stale; autoDispose does not fix it because the stale view stays mounted").

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
