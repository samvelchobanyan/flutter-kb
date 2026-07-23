---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-strapi
keywords: [strapi, dto, populate, partial-response, sparse-fieldset, silent-default, hydration]
related: [[dto-to-domain-mapping]]
---

# Shared DTO under-hydrated by a partial populate

## Symptom

The same entity looks fully populated on one screen and blank in a field on another. An artist name, a cover image — present when the entity came from a playlist or favorites, empty when the same entity came from an album. The UI just renders blank; there is no error.

## Root cause

One entity is hydrated by a single DTO, but different endpoints request different `populate` sets, each declared inline at the query. When an endpoint omits a relation the DTO expects, the missing data does not fail — it degrades into the field's falsy default (`''` for a string, `null` for a nullable), because the DTO reads a key that is not there and falls back. So the album-tracks endpoint that populated only `Tracks.File` yields tracks with `artistName: ''` and `imageUrl: null`, while the same track from an endpoint that populates `Album.Cover` and `Artists` comes back whole. The bug surfaces in the UI, far from the query that caused it.

A per-widget fallback makes this worse by hiding it unevenly: one widget's `track.imageUrl ?? source.imageUrl` masks the gap while a sibling widget without that fallback exposes it, so "the same track" shows an image from one screen and a blank from another — which looks like a widget bug, not a data bug.

## Fix

Keep the `populate` sets in sync across every endpoint that hydrates the same entity. If the playlist query populates `Album.Cover` and `Artists`, the album query must populate them too — the DTO's expectations are one contract, and every endpoint feeding it has to meet the same one.

Prefer a single shared fallback over per-widget ones. If a track without its own cover should fall back to its source's cover, put that in one place (a `displayImageUrl` getter on the state) so every widget behaves identically, instead of each widget inventing its own fallback that masks the underlying gap on some screens and not others.

## Where this is likely to occur

- Strapi or any REST/GraphQL API with per-endpoint field expansion, where several endpoints feed one shared DTO
- Entities reached from many places — a track fetched from albums, playlists, favorites, search — each query written independently
- Sparse-fieldset setups where `populate`/`fields` are declared inline per call rather than centralized for the entity

## How to detect early

- When one DTO is hydrated by multiple endpoints, diff their `populate` declarations. Any relation present in one and absent in another is a silent under-hydration waiting to happen.
- A field that is populated on some screens and empty on others, for the same entity, points at the query, not the widget — trace back through the DTO to each endpoint's populate.
- Be suspicious of per-widget fallbacks: if two widgets rendering the same entity disagree, one is probably masking missing data the other reveals.
