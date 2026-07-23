# Flutter-kb Index

This is the navigation backbone of the wiki. Agents read this file first when working with the knowledge base. Maintained automatically by Claude Code; do not edit by hand.

Each entry has the format:

```
- [[page-name]] — severity: <value>, applies_to: <value> — <one-line description>
```

For project-scoped ADRs, entries are grouped under "Project-specific knowledge" with subheadings per project.

---

## Principles

- [[design-system-tokens-mandatory]] — severity: required, applies_to: any-flutter-project — All visual values come from DSColors / DSSpacing / DSRadius / DSTypography; no hardcoded colors, sizes, or font styles in features
- [[domain-vs-screen-state]] — severity: required, applies_to: any-flutter-project — Domain state in domain/state/ (long-lived Notifier); screen state in feature controller (short-lived); screens read domain state directly
- [[dto-to-domain-mapping]] — severity: required, applies_to: any-flutter-project — All API responses parsed via DTOs with fromJson + toDomain(); DTOs never leave the data layer
- [[layered-architecture]] — severity: required, applies_to: any-flutter-project — Five layers (core, design_system, domain, data, features) with strict downward dependency direction
- [[semantic-vs-navigational-ownership]] — severity: required, applies_to: any-flutter-project — Navigation follows user flow, not content category; news under Home, library screens under Music, schedule under Radio
- [[shared-widgets-stay-dumb]] — severity: required, applies_to: any-flutter-project — Shared tiles hold no state and no provider subscriptions; separate classes over conditional flags; state via props, actions via callbacks

## Patterns

- [[api-client-with-config-and-interceptors]] — severity: recommended, applies_to: any-flutter-project — ApiClient wraps Dio, takes AppConfig + interceptor list at construction; env selection via --dart-define=ENV
- [[async-init-provider-override]] — severity: recommended, applies_to: any-flutter-project — Provider<T>(throw) declaration + ProviderScope.overrides for infrastructure that needs await init()
- [[background-image-bottom-card-form]] — severity: recommended, applies_to: any-flutter-project — Full-bleed image over a bottom form card that survives keyboard and short screens; IntrinsicHeight makes Spacer legal in a scroll view
- [[collection-play-button-reflects-active-source]] — severity: recommended, applies_to: flutter-projects-with-playback — Collection header play button keys on queue-source identity, not track membership; two booleans, not one
- [[dual-mode-player-state]] — severity: recommended, applies_to: flutter-projects-with-playback — Single PlayerState with PlaybackMode discriminator; switching modes preserves the other mode's queue
- [[event-stream-vs-state-for-discrete-events]] — severity: recommended, applies_to: any-flutter-project — Discrete events (notification taps, deep links) as Stream + StreamProvider, not StateProvider — avoids stale-state and dedup pitfalls
- [[freezed-immutable-state]] — severity: recommended, applies_to: any-flutter-project — Freezed @freezed for product types + sealed Freezed for sum types; copyWith / == / hashCode generated, Dart 3 patterns supported
- [[local-notifications-stack]] — severity: recommended, applies_to: flutter-projects-with-local-notifications — NotificationService + ReminderStorage + Notifier orchestrator + dumb tile; four-layer separation for scheduled local notifications
- [[local-state-via-storage-class]] — severity: recommended, applies_to: any-flutter-project — Concrete storage class (in-memory cache + sync getters + async init) for local non-API state; no abstract repository
- [[parallel-data-loading-with-records]] — severity: flexible, applies_to: any-flutter-project — Dart 3 record .wait to fan out independent repository calls in parallel
- [[persist-and-restore-playback-session]] — severity: recommended, applies_to: flutter-projects-with-playback — Snapshot plus separate position key; periodic position saves; restore-or-prime paused on cold start
- [[reactive-derived-view]] — severity: recommended, applies_to: any-flutter-project — Per-id heavy view listens to a global light composition and refetches on Set-diff; autoDispose does not solve this
- [[system-player-artwork-fallback]] — severity: recommended, applies_to: flutter-projects-with-playback — MediaItem.artUri reads only http/file; materialize the asset to a file and guard the chain by scheme
- [[route-target-self-hydrates-from-id]] — severity: recommended, applies_to: any-flutter-project — Detail screen requires an id and fetches itself; go_router extra is an optional first-paint hint, not a contract
- [[selector-providers-for-rebuilds]] — severity: recommended, applies_to: any-flutter-project — Narrow Provider<T> selectors over a large Notifier; isolate high-frequency from low-frequency updates
- [[two-layer-audio-stack]] — severity: recommended, applies_to: flutter-projects-with-playback — PlayerNotifier (state + user API) + AppAudioHandler (just_audio + audio_service infrastructure); two layers, not three

## Decisions (cross-project)

- [[adr-001-riverpod-for-state-management]] — severity: required, applies_to: any-flutter-project — Riverpod 2.x chosen as the single state-management and DI mechanism
- [[adr-002-just-audio-plus-audio-service]] — severity: required, applies_to: flutter-projects-with-playback — just_audio + audio_service stack for playback with background and lock-screen support
- [[adr-003-freezed-for-models]] — severity: required, applies_to: any-flutter-project — Freezed + json_serializable for entities, state classes, and DTOs via build_runner
- [[adr-004-go-router-stateful-shell]] — severity: required, applies_to: any-flutter-project — go_router with StatefulShellRoute.indexedStack for tab state preservation and deep-link resolution
- [[adr-005-secure-storage-for-tokens]] — severity: required, applies_to: any-flutter-project — FlutterSecureStorage for auth tokens, fronted by TokenStorage with in-memory cache and synchronous getters
- [[adr-006-flutter-local-notifications-plus-timezone]] — severity: required, applies_to: flutter-projects-with-local-notifications — flutter_local_notifications + timezone for locally-scheduled reminders; FCM alternative rejected
- [[adr-007-inexact-alarm-as-default]] — severity: recommended, applies_to: flutter-projects-with-local-notifications — AndroidScheduleMode.inexactAllowWhileIdle as default to avoid Play Store policy risk and Android 14 UX detour

## Pitfalls

- [[audio-error-recovery]] — severity: required, applies_to: flutter-projects-with-just-audio — Reentrancy, duplicate error events, and native auto-advance make naive skip-on-error harmful; branch on the failed index
- [[constant-value-drift-on-rename]] — severity: required, applies_to: any-flutter-project — Reassigning a named constant's value in place silently retargets every call site; rename usages first, redefine second
- [[custom-transition-page-breaks-ios-swipe-back]] — severity: required, applies_to: flutter-projects-with-go-router-custom-transitions — CustomTransitionPage has no Cupertino back-gesture; the discriminator is page type, not screen
- [[dual-source-of-truth-state]] — severity: required, applies_to: any-flutter-project — One concept mirrored in a Notifier and a backing store; a write to only one drifts, disguised as a platform/timing bug
- [[full-screen-modal-loses-safe-area-insets]] — severity: required, applies_to: any-flutter-project — Only MediaQueryData.fromView survives the sheet's removePadding; copying parent MediaQuery and reading viewPadding both fail
- [[hero-tag-collision-across-shell-branches]] — severity: required, applies_to: flutter-projects-with-stateful-shell-route — Same Hero tag in two indexedStack branches asserts on any root-navigator transition
- [[image-cache-dims-distort-aspect-ratio]] — severity: required, applies_to: any-flutter-project — Both cacheWidth and cacheHeight decode to exact pixels and drop aspect ratio; BoxFit.cover cannot fix an already-distorted bitmap
- [[listview-null-padding-consumes-safe-area]] — severity: required, applies_to: any-flutter-project — A ListView with null padding injects MediaQuery.padding; a sibling of a SafeArea eats the top inset twice
- [[local-notifications-platform-gotchas]] — severity: required, applies_to: flutter-projects-with-local-notifications — Android core library desugaring requirement (FLN 17+); iOS UNUserNotificationCenter delegate must be registered; vendor battery optimization kills scheduled notifications on Xiaomi/Huawei/Oppo
- [[modal-in-stateful-shell-route]] — severity: required, applies_to: flutter-projects-with-stateful-shell-route — go_router <=14.8.1 singular popRoute navigator lookup breaks system back after a root modal; fix is the upgrade
- [[player-fast-path-masks-dead-player]] — severity: required, applies_to: flutter-projects-with-playback — "Same source, just seek" assumes a live player and a matching queue; optimistic UI hides both failures
- [[repo-side-effects-on-domain-state]] — severity: required, applies_to: any-flutter-project — A repository that pushes results into a state Notifier inverts the layering and hides the mutation from every call site
- [[shared-dto-partial-populate]] — severity: required, applies_to: flutter-projects-with-strapi — One DTO hydrated by endpoints with divergent populate sets under-hydrates silently into falsy defaults; keep populate in sync
- [[strapi-oauth-provider-email-conflict]] — severity: required, applies_to: flutter-projects-with-strapi-users-permissions-oauth — Bundled OAuth callback rejects an email already registered as provider=local with 400; needs a linking policy, not a client fix

## Project-specific knowledge

### go_sport

- [[projects/go_sport/adr-001-central-like-registry]] — Central LikeRegistry owning full entity lists for six likeable types; library screens become views, repositories stay pure
- [[projects/go_sport/adr-002-persist-player-session]] — Music session persisted over two SharedPreferences keys; restore-or-prime lands paused, radio never persisted
