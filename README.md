# setPageLayout props lost on same-path (query-only) navigation

issue: https://github.com/nuxt/nuxt/issues/34845

Minimal Nuxt 4 reproduction for a bug in the `setPageLayout(name, props)`
feature added in [nuxt/nuxt#33805](https://github.com/nuxt/nuxt/pull/33805)
(merged 2025-12-23).

## Versions

Resolved from `pnpm-lock.yaml`:

| Package    | Version |
| ---------- | ------- |
| nuxt       | 4.4.2   |
| vue        | 3.5.32  |
| vue-router | 5.0.4   |

## Steps to reproduce

1. `pnpm install && pnpm dev`
2. Open `/`. The header reads **"Initial title set by setPageLayout"**.
3. Click **Tab B**. URL becomes `/?tab=b`.
4. The header changes to **"(no title)"**.

## Expected

The header keeps showing the title set by `setPageLayout` across same-path
navigations.

## Actual

The title is lost on the first same-path navigation. Only a full page reload
restores it.

## Root cause (hypothesis)

- `setPageLayout(name, props)` mutates `useRoute().meta.layoutProps = props`
  directly when called outside middleware (see
  `nuxt/dist/app/composables/router.js` `setPageLayout`, and
  [nuxt/nuxt#33805](https://github.com/nuxt/nuxt/pull/33805)).
- `<NuxtLayout>` reads `route.meta.layoutProps` reactively and forwards it
  to the layout component (see `nuxt/dist/app/components/nuxt-layout.js`).
- `vue-router` produces a fresh `route.meta` (merged from the static route
  record) on every navigation, so the imperative mutation made during the
  first render is not carried over to the next route object.
- `<script setup>` is not re-run for same-path navigations (the page
  component instance is reused), so `setPageLayout` is never called again
  to repopulate `layoutProps` on the new route.
- Result: `route.meta.layoutProps` is empty on the second navigation, and
  `<NuxtLayout>` re-renders the layout with no props.

## Workaround

A global route middleware can paper over the symptom by copying
`layoutProps` from the previous route's meta into the incoming route's meta
whenever the path is unchanged:

```ts
// app/middleware/preserve-layout-props.global.ts
export default defineNuxtRouteMiddleware((to, from) => {
  // Preserve layoutProps across same-path navigations (e.g. query-only
  // changes like tab switching). router.replace / router.push to the same
  // path causes vue-router to rebuild route.meta from the route record,
  // which drops layoutProps set via setPageLayout in the page's script
  // setup (since script setup is not re-run for same-path navigation).
  // Only carry over when the incoming route has lost its layoutProps.
  if (to.path !== from.path) return;
  if (to.meta.layoutProps) return;
  if (!from.meta.layoutProps) return;

  to.meta.layoutProps = from.meta.layoutProps;
});
```

This is a symptomatic workaround rather than a real fix. `layoutProps` is
being treated as per-navigation ephemeral meta, when from the page author's
point of view it should live with the page component instance. The
workaround inherits several awkward properties of that mismatch:

- It only works for same-path navigations. Any flow that expects
  `setPageLayout` to "stick" across genuine navigations still breaks.
- It relies on `from.meta.layoutProps` still being populated at middleware
  time, which couples the fix to vue-router's current meta lifecycle.
- It does nothing for pages that never call `setPageLayout` with props, yet
  runs on every navigation.
- Nothing in the page source hints that this middleware is load-bearing,
  so it is easy to delete during cleanup and silently regress.

A proper fix likely belongs in Nuxt itself — either by persisting
`layoutProps` alongside the page component instance (so it survives
same-path re-renders without needing meta), or by re-invoking the page's
layout declaration on each resolved navigation.
