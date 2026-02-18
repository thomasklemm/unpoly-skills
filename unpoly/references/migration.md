# Migrating Unpoly with unpoly-migrate.js

## Table of contents
- [How it works](#how-it-works)
- [Installing the shim](#installing-the-shim)
- [Upgrade workflow](#upgrade-workflow)
- [Keeping the shim permanently](#keeping-the-shim-permanently)
- [Using with automated tests](#using-with-automated-tests)
- [Notable manual migration items (3.11)](#notable-manual-migration-items-311)

---

## How it works

`unpoly-migrate.js` is a compatibility shim that polyfills renamed and removed Unpoly APIs.
When your app calls an old API, the shim transparently forwards it to the current equivalent
and logs a deprecation warning to the browser console.

It covers renamed HTML attributes, JS functions, events, options, and packages across the
full 1.x → 2.x → 3.x history — so upgrading directly from v1 to v3 without going through v2
is supported. Changes handled by the shim are **not** considered breaking changes.

> Note: event aliases only work via `up.on()`, not native `addEventListener()`.

---

## Installing the shim

The shim is included in the `unpoly` npm package and available on CDN. Load it **after**
`unpoly.js` and **before** your own application code:

**CDN:**
```html
<script src="unpoly.js"></script>
<script src="unpoly-migrate.js"></script>
<script src="app.js"></script>
```

**npm / bundler:**
```js
import 'unpoly/unpoly.js'
import 'unpoly/unpoly-migrate.js'
import 'unpoly/unpoly.css'
```

**Rails Asset Pipeline (Sprockets):**
```js
//= require unpoly
//= require unpoly-migrate
```

Load order matters — if the shim loads before `unpoly.js`, the polyfills won't attach.

---

## Upgrade workflow

1. Add `unpoly-migrate.js` to your build (see above)
2. Open your app in a browser and use the affected features
3. Follow the deprecation warnings in the console — each one names the old API and the replacement
4. Update each call site to the new API
5. Once the console is clean, remove `unpoly-migrate.js`

> Tip: load the shim for every upgrade, even minor version bumps — renamed APIs are not
> counted as breaking changes and won't appear in the changelog without it.

---

## Keeping the shim permanently

If you prefer to upgrade at your own pace, you can keep `unpoly-migrate.js` in production
(adds ~9 KB gzipped) and silence the console warnings:

```js
up.migrate.config.logLevel = 'none'
```

> Note: polyfills can be dropped from the shim in any major Unpoly release, so plan to
> work through the warnings eventually.

---

## Using with automated tests

Turn deprecation warnings into errors to surface them in your test suite:

```js
up.migrate.config.logLevel = 'error'
up.log.config.format = false  // unformatted text is easier to extract from test output
```

Your E2E tests can detect browser console errors via the Selenium/WebDriver API.

---

## Notable manual migration items (3.11)

These 3.11 changes are **not** polyfilled automatically — they require code changes:

**Scripts in fragments no longer execute by default:**

`<script>` tags inside updated fragments are silently skipped. To restore the old behavior:

```js
up.fragment.config.runScripts = true
```

Prefer moving inline scripts to compilers (`up.compiler()`) — see [compilers.md](compilers.md).

**`[up-scroll='reset']` renamed to `[up-scroll='top']`:**

`unpoly-migrate.js` polyfills this automatically. Manual migration: search for
`up-scroll="reset"` and replace with `up-scroll="top"`.

**`[up-href]` without `[up-follow]` no longer auto-follows:**

Elements with only `[up-href]` are no longer treated as followed links.
`unpoly-migrate.js` polyfills the old behavior. Manual migration: add `[up-follow]`
explicitly, or switch to a standard `<a href>` element.
