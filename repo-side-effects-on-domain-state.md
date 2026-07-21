---
type: pitfall
severity: required
status: stable
applies_to: any-flutter-project
keywords: [repository, data-layer, domain-state, side-effect, riverpod, notifier, layering, purity]
related: [[layered-architecture]], [[domain-vs-screen-state]]
---

# Repository writes to domain state as a side effect

## Symptom

A repository method has a pure-looking signature — `Future<List<Album>> getFavoriteAlbums()` — and, on its way to returning, also pushes what it fetched into a domain-state `Notifier`. Nothing at the call site says so.

The consequences accumulate quietly:

- Testing the repository requires mocking a state object it has no business knowing about
- Reading a call site tells you nothing about what app-wide state just changed
- Calling the same method from two places mutates the shared state twice, which is either wasteful or wrong depending on whether the merge is idempotent
- The dependency graph develops a cycle: the state Notifier calls the repository, which calls back into the Notifier

The last one is the tell. When a state object and a repository each hold a reference to the other, a data-flow question ("who updates this list?") has no local answer — it has to be traced through both.

## Root cause

The data layer reaches upward into domain state, inverting the dependency direction that [[layered-architecture]] establishes. Data may be consumed by domain; it must not write to it.

The motivation is always reasonable-sounding, which is why the shape recurs: two different callers both need the shared state populated after fetching, so bundling the write into the shared method removes one line from each of them. It reads as centralization. What it actually centralizes is a *side effect*, in the one place that has the least context about whether it is wanted.

The real damage is to the method's contract. `Future<List<Album>> getFavoriteAlbums()` promises to return albums. It does — and also does something else, invisibly, to state the caller never mentioned. Every future reader must now open the repository to find out which providers a call mutates, and that cost is paid on every read forever, to save one line written once.

**The diagnostic signal is inconsistency inside a single class.** The side effect is never applied uniformly, because there is no principle behind it — only convenience at the moment each method was written. One method pushes its result into the state; a sibling on the same repository returns its result and lets the caller merge. Same class, two contracts, no way to know which is which without reading each body. When a repository shows this split, the side-effecting half is the part to remove.

## Fix

The repository returns data and nothing else. Whoever wanted the state populated does the merging, explicitly:

- The state Notifier's own init method calls the repository and merges the result through a private helper
- Or a controller calls the repository and passes the result where it belongs

Both make the mutation visible at the place that decided to make it, and the cycle disappears — dependencies point one way, from domain state down to the repository.

If two callers really do need the same fetch-and-merge sequence, that sequence is a method **on the state Notifier**, not on the repository. It belongs to whoever owns the state.

## Where this is likely to occur

- Repositories that take a `Notifier`, registry, or cache as a constructor dependency — this is the structural signature; the layering is already inverted at that point
- Any "the repository writes to the cache as a side effect" arrangement, which is the same shape with different vocabulary
- Codebases where a shared collection (likes, follows, permissions) must be populated from several entry points, and one of them is a fetch that several screens already perform
- Refactors that centralized "duplicated" code by moving it downward into a shared layer, without checking whether the direction of that move was legal

## How to detect early

- Look at repository constructor parameters. A repository should depend on an API client, a storage class, and mappers. A state object among them is the pitfall, before any method has been read.
- Grep the data layer for imports from `domain/state/`. There should be none; domain state imports repositories, not the reverse.
- Read a repository's methods for contract consistency. If some mutate shared state and others do not, that asymmetry is the bug — not a style inconsistency to be smoothed over by making the rest match.
- Treat "I'll do the merge in the repo so both callers get it" as the moment to stop. That is the pitfall being introduced, and it looks like a simplification at exactly that moment.
