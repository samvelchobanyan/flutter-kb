---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [refactoring, constants, rename, silent-bug, design-tokens, scale, value-drift]
related: [[design-system-tokens-mandatory]]
---

# Silent value drift when renaming a constant

## Symptom

The value behind a named constant is changed, the project compiles cleanly, and call sites silently start producing a different value. There is no error anywhere, because every identifier still resolves — only the number it resolves to has changed.

The classic trigger is inserting a value into an ordered scale. You want `DSRadius.xl` to mean 24 instead of 28, so you reassign it in place. Every one of the existing `xl` call sites — sized correctly for 28 — now renders 24, and nothing flags it. The bug surfaces later as "some corners look slightly off", far from the one-line edit that caused it.

## Root cause

The compiler protects you against a name that no longer exists. It cannot protect you against a name whose meaning changed, because from the type system's point of view nothing changed at all: the identifier is still valid, still the right type, still in scope. Only the literal behind it moved.

Reassigning a constant's value in place, or keeping an old name alive with a new value, therefore mutates every call site at once, invisibly. This is most dangerous with an ordered set of named constants — a scale — because inserting a new value in the middle tempts you to shift what an existing name means rather than adding a genuinely new name.

## Fix

Rename first, redefine second. When inserting a value into an ordered set of named constants:

1. Globally rename every usage of the old name to its new name (`DSRadius.xl` → `DSRadius.xxl`) — a mechanical, compiler-checked rename.
2. Only then redefine or add the value under the old name (`xl = 24`).
3. Grep to confirm zero remaining usages of the old name at its old meaning.

The ordering is what makes it safe: after step 1 there are no call sites left holding the old name, so redefining it in step 2 cannot silently retarget anything. Doing it in the other order — redefine, then migrate — means every existing call site renders the new value for the entire window between the two steps, with no compile error to stop you.

## Where this is likely to occur

- Design-token scales, where inserting a value between existing steps is a routine request (a new radius, a new spacing step) — see [[design-system-tokens-mandatory]]
- Any ordered set of named constants: enum-like level or tier constants, size or breakpoint scales, priority or z-index bands
- Configuration constants whose value is tuned over time (timeouts, thresholds, limits) while the name stays the same
- Any refactor framed as "just change the number" — that framing is itself the warning sign

## How to detect early

- A diff that changes a constant's value while its call sites are untouched is the signature. If the definition moved and nothing that uses it moved, ask whether every existing usage still wants the old value.
- Before reassigning a constant, grep its usages and decide, per usage, whether it wants the old value or the new one. If any want the old value, they need renaming first — the reassignment is not a rename, it is a silent retarget.
- Treat "rename" and "change the value" as two distinct operations that must never be collapsed into one edit. The compiler verifies the first and is blind to the second.
