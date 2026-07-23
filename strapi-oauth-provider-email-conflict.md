---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-strapi-users-permissions-oauth
keywords: [strapi, oauth, google-sign-in, users-permissions, email-conflict, account-linking, provider]
related: []
---

# Strapi OAuth callback rejects an email already registered locally

## Symptom

A "Sign in with Google" flow reaches Strapi's `/api/auth/google/callback` successfully — the Google token is valid, the endpoint is correct — and Strapi answers `400`:

```json
{"data":null,"error":{"status":400,"name":"ApplicationError","message":"Email is already taken.","details":{}}}
```

It reads like a client bug: wrong token, wrong scope, wrong endpoint, a missing field. Hours disappear into the client integration. The integration is fine.

The tell is that the *same* flow with a never-before-seen Google account succeeds. The failure depends on the email, not the request.

## Root cause

Strapi's bundled `users-permissions` plugin does **not** auto-link an OAuth login to an existing user when that email already exists with `provider: "local"`. When someone registered with email + password and later signs in with Google using the same email, the callback validates the Google token, retrieves the email, finds it already taken by a local account, and refuses — which is correct, conservative server behaviour, not a defect.

This is not specific to Google. Every bundled provider — Facebook, GitHub, Discord — shares the callback logic and the same conflict. Strapi's "Sign in with X" documentation describes registration but skips the cross-provider conflict case, so nothing points you at the real cause.

The same wall blocks a later "add Google login to my existing account" feature: it is the identical conflict from the other direction.

## Fix

This needs a product decision before it needs code, because the framework deliberately refuses to guess:

- **Decide the linking policy.** If a Google login should attach to an existing local account with the same email, that is an explicit account-merge decision, and it needs a backend customization that links by email instead of rejecting.
- **Or replace the bundled callback** with a custom endpoint that owns the linking policy — the general answer, since the bundled callback has no policy hook.
- **Or constrain the lifecycle** to one provider per user, so the conflict cannot arise.

What does not work is treating it on the client. The client did everything right; the decision about what "same email, different provider" means lives on the server.

## Where this is likely to occur

- Any Flutter app on Strapi's `users-permissions` plugin offering both local (email/password) and social login
- Apps that add social login after already having email/password users — the existing accounts are exactly the ones that collide
- A planned "link another provider to my account" feature, which hits the same conflict deliberately
- Testing where the same email was used for local signup earlier in the session, then reused for OAuth

## How to detect early

- A `400 "Email is already taken"` from an OAuth callback — not from registration — is this pitfall almost by definition. Registration says it about a form; a callback says it about a cross-provider clash.
- If the same OAuth flow works for a fresh account and fails for a known one, stop looking at the client. The discriminator is prior existence of the email, which is a server-state fact.
- Before shipping social login onto an app that already has local accounts, decide the linking policy up front. The conflict is guaranteed for every pre-existing email, so it is a launch-path question, not an edge case.
