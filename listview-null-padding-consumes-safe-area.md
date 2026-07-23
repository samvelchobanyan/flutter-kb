---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [listview, padding, safe-area, media-query, insets, scroll-view, top-gap]
related: [[full-screen-modal-loses-safe-area-insets]]
---

# ListView with null padding consumes the safe-area inset

## Symptom

A scrollable list shows an unexpected large gap at the top (or bottom) — bigger than any padding in the code accounts for. Two screens with the same structure differ: one has the gap, one does not, and the only difference is that one list passed a `padding` and the other did not.

## Root cause

A `ListView` (or any `ScrollView`) with `padding == null` does not mean "no padding". It falls back to injecting `MediaQuery.padding` — the safe-area insets — as its own padding. Passing any explicit `padding`, even `EdgeInsets.only(bottom: ...)`, suppresses that fallback entirely.

This bites specifically when the list is a **sibling** of a `SafeArea` rather than a child of it. A common shape: a header wrapped in `SafeArea(bottom: false)` and a list in a sibling `Expanded`, both inside a `Column`. The `SafeArea` consumes the top inset for *its own subtree only*, so the sibling list still sees the full `MediaQuery.padding` and silently eats the top safe-area inset a second time. The inset is effectively applied twice — once by the SafeArea for the header, once by the null-padding list for itself.

## Fix

Give the list an explicit `padding`. Even a padding that only sets another edge — `EdgeInsets.only(bottom: bottomClearance)` — turns off the automatic inset injection, because the fallback triggers only on `null`, not on "padding that happens to be zero on top".

If the list genuinely needs no padding, pass `EdgeInsets.zero` rather than leaving it null. `null` and `EdgeInsets.zero` are not the same here: the first injects the safe-area inset, the second does not.

## Where this is likely to occur

- A list that is a sibling (not a descendant) of a `SafeArea` inside a `Column`, so it sees insets the SafeArea already handled
- Screens unified from a template where some lists were given padding and others were left null — the null ones develop the extra gap
- Any `ListView`/`CustomScrollView`/`GridView` left without an explicit `padding` under a status bar or notch
- Top-of-screen lists especially; the top inset is the largest and most visible

## How to detect early

- An unexplained top gap on a scrollable that scales with the device's safe area (bigger on a notched phone) points straight at this — the gap *is* the inset.
- When two structurally identical screens differ only in a top gap, diff the list's `padding` argument. A `null` on the gapped one is the cause.
- Treat a `ListView` with no `padding` as a decision, not a default. If the intent is "no padding", write `EdgeInsets.zero` so the intent is explicit and the inset injection is off.
