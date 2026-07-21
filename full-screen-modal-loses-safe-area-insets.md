---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [modal-bottom-sheet, safe-area, media-query, view-padding, status-bar, insets, overlay]
related: [[modal-in-stateful-shell-route]]
---

# Full-screen modal loses safe-area insets

## Symptom

A full-screen `showModalBottomSheet` overlay — a story viewer, full player, or search sheet — renders its top controls under the status bar. Progress bars, close buttons, and headers sit beneath the system clock and notch, partly unreadable and partly untappable.

The regression is often triggered by an edit somewhere else entirely. Refactoring the **host** screen — moving an in-scroll `SliverAppBar` into `Scaffold(appBar:)`, wrapping a section in `SafeArea` — is enough to break an overlay whose own code did not change. That decoupling makes the cause hard to locate: the file that broke is not the file that was edited.

## Root cause

`ModalBottomSheetRoute` with `useSafeArea: false` applies `MediaQueryData.removePadding(removeTop: true)` to the sheet's subtree. Inside the sheet, `padding.top` is therefore 0. That much is documented and expected.

The trap is that the two obvious repairs are both wrong, for **different** non-obvious reasons:

**(a) Copy the parent's MediaQuery.** Wrapping the sheet in `MediaQuery(data: MediaQuery.of(context))`, capturing the host screen's data, fails whenever the capturing context sits under an `AppBar` or inside a `SafeArea`. Those widgets consume the inset for their subtree, so `padding.top` at that context is *already* 0 — the copy faithfully propagates a zero. This is exactly why a host-screen refactor breaks the overlay: moving the trigger point under an `AppBar` changes what the copy captures. The fix appears to work until the host layout shifts.

**(b) Read `viewPadding` inside the sheet.** The reasoning is that `padding` reflects consumed insets while `viewPadding` reports the physical ones, so `MediaQuery.viewPaddingOf(context)` should survive the removal. It does not. `MediaQueryData.removePadding` subtracts the removed padding from **`viewPadding` as well**, not only from `padding`. Reading `ModalBottomSheetRoute`'s source appears to confirm approach (b) — the call there looks padding-only — because the subtraction lives one level down, inside `removePadding` itself. The result is a fix that is "verified" by source reading and still returns 0 on device.

## Fix

Read the insets from the view directly, bypassing every tree-level modification:

```dart
final mq = MediaQueryData.fromView(View.of(context));

showModalBottomSheet(
  context: context,
  useRootNavigator: true,
  builder: (_) => MediaQuery(data: mq, child: overlay),
);
```

`MediaQueryData.fromView` constructs the data from the window, so it is immune to `removePadding`, `SafeArea`, `AppBar`, and anything else the widget tree did on the way down. Capture it **before** opening the sheet, at the call site, and reinject it inside `builder:`.

Apply this consistently at every full-screen sheet entry point. Full-screen overlays are exactly the case where the sheet needs the true device insets, because it is drawing to the physical screen edges and no ancestor is managing that space for it.

## Where this is likely to occur

- Full-screen `showModalBottomSheet` overlays: media players, story viewers, search, image galleries
- Any sheet opened with `useSafeArea: false` (required when the overlay must paint into the status-bar area itself)
- Apps where sheets are launched from varied host contexts — some under an `AppBar`, some not — so approach (a) works on some entry points and not others
- Codebases with several full-screen sheets added at different times, likely to have drifted into a mix of the three approaches

## How to detect early

- Instrument rather than reason. `debugPrint` both `padding.top` and `viewPadding.top` from **inside** the sheet on a device with a notch. Intuition about which one survives is unreliable here; the printed values settle it in one run.
- Treat a sheet reading insets from anything other than `fromView` as suspect, even if it currently renders correctly — it may be working only because its host happens not to consume the top inset yet.
- When a full-screen overlay regresses without its own code changing, look at what moved in the host screen's app-bar or `SafeArea` structure.
