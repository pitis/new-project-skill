# UTM tracking (on by default)

Goal: a sign-up and a purchase can be traced to the campaign that brought the person.
Registering UTMs only as PostHog properties on the landing page stops attribution there;
this carries it end to end.

## packages/core/src/attribution.ts (one copy, used by every surface and the API)

```ts
type UtmKey = 'utm_source' | 'utm_medium' | 'utm_campaign' | 'utm_content' | 'utm_term';
export type Attribution = Partial<Record<UtmKey, string>> & {
  referrer: string | null;   // external referrers only; same-host → null
  landingPath: string;
  capturedAt: string;        // Luxon ISO, UTC
};
export type StoredAttribution = { first: Attribution; last: Attribution };

readAttribution(url, referrer, now = DateTime.utc()): Attribution | null  // null if no UTM and no external referrer
mergeAttribution(existing: StoredAttribution | null, touch): StoredAttribution  // first never overwritten
withUtm(url, { source, medium, campaign, content? }): string  // keeps existing query params
```

Unit tests: reads params/referrer/path; null when nothing to attribute; ignores a same-site
referrer; first touch kept, last replaced; `withUtm` keeps `?q=1`.

## Capture (web + app)

- On landing, `readAttribution(location.href, document.referrer)`; if non-null, merge into a
  first-party cookie `<slug>_attr` on the parent domain (shared by web and app), 90 days.
- **Written only after analytics consent.** Without consent nothing is stored; links from
  web to the app still forward the UTM query params, so the app can read them on arrival.
  Whether the cookie needs consent is a legal question for the owner; this is the cautious
  default — say so in CLAUDE.md.

## PostHog

- Last touch as super properties: `posthog.register({ utm_* })`.
- First touch once: `posthog.register_once({ initial_utm_* })` and
  `setPersonProperties({}, { initial_utm_* })` (`$set_once`).

## Account

- The app sends the stored attribution with sign-up; a Better Auth
  `databaseHooks.user.create.before` validates it with a Zod schema and stores it on
  `users.attribution` (`{ first, last }`). Linking an anonymous account to a real one keeps the
  earliest `first`.
- Integration test: sign-up with an attribution body → `users.attribution.first.utm_source`.

## Stripe

- Checkout sessions carry `metadata: { utm_source, utm_medium, utm_campaign }` from
  `attribution.first`, and the server-side `purchase_completed` event includes them, so
  revenue is attributable per campaign.

## Our own links

Every link in emails, the extension and the app goes through `withUtm`:
`utm_source=email|extension|app`, `utm_medium=<template or surface>`,
`utm_campaign=<event>`.
