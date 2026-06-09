---
oncalls: ['sae_partnerships']
description: Gotchas for the GTM Web ParamBuilder tag template
apply_to_regex: 'fbcode/signals/gtm/Web/Tags/.*'
---
# GTM Web ParamBuilder Tag — Gotchas

## NEVER call both `data.gtmOnSuccess()` and `data.gtmOnFailure()`

GTM expects exactly one terminal callback per fire. Invoking both — or invoking the success callback after a failure path falls through — corrupts tag firing telemetry and can make subsequent triggers misreport. ALWAYS branch with explicit `else` so each code path runs exactly one terminal callback.

```js
// BAD — both callbacks fire when the method is missing
if (typeof clientParamBuilder.processAndCollectAllParams === 'function') {
  clientParamBuilder.processAndCollectAllParams(url);
  data.gtmOnSuccess();
}
data.gtmOnFailure();          // unconditional!

// GOOD — exactly one terminal callback per branch
if (typeof clientParamBuilder.processAndCollectAllParams === 'function') {
  clientParamBuilder.processAndCollectAllParams(url);
  data.gtmOnSuccess();
} else {
  data.gtmOnFailure();
}
```

## NEVER change the injected script URL without updating `___WEB_PERMISSIONS___`

The sandboxed JS injects from `https://capi-automation.s3.us-east-2.amazonaws.com/public/client_js/capiParamBuilder/clientParamBuilder.bundle.js`. The `inject_script` permission entry in `___WEB_PERMISSIONS___` lists URLs as **literal strings** — GTM rejects `injectScript()` calls to a URL not in the allowlist with **no console error** at runtime in production. ALWAYS update both the URL constant and the permission entry in the same diff.

## DO NOT add entries to `___TEMPLATE_PARAMETERS___`

The list is intentionally `[]` — this tag has no user-configurable parameters by design. The sibling Pixel template's `gtm-template-e2e` does **NOT** exercise `Web/Tags/template.tpl`. Adding a parameter without first wiring up an E2E config for this template ships an untested user-facing field directly to the Gallery.

## NEVER edit `___INFO___` `id`, `displayName`, or `thumbnail`

These three fields are persisted by the Gallery listing — `id` is the primary key (changing it forks the template into a duplicate listing), `displayName` is what advertisers see in the picker, and `thumbnail` is a fixed base64 blob owned by the brand. Mutating any of them desyncs from the published listing and confuses advertisers searching the Gallery.

## ALWAYS keep the legacy `processAndCollectParams` fallback

The script probes `processAndCollectAllParams` first and falls back to the older `processAndCollectParams` for compatibility with already-deployed bundles in advertiser browser caches. Removing the fallback breaks tags whose `clientParamBuilder.bundle.js` cache hasn't yet rolled forward — keep both branches.
