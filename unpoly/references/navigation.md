# Navigation

## Table of contents
- [Navigation vs rendering](#navigation-vs-rendering)
- [Active nav links (up-current)](#active-nav-links-up-current)
- [History management](#history-management)
- [Scrolling](#scrolling)
- [Focus management](#focus-management)
- [URL patterns](#url-patterns)
- [Back links and instant navigation](#back-links-and-instant-navigation)
- [Restoring history](#restoring-history)

---

## Navigation vs rendering

**Navigation** is a render pass that also updates the browser history, scroll position, and focus.
Links and form submissions **navigate** by default. `up.render()` from JS does **not** navigate by default.

Key navigation defaults (applied automatically when navigating):
- History is updated (`{ history: true }`)
- Cache is used (`{ cache: 'auto' }`)
- Fallback to main if target missing (`{ fallback: true }`)
- Scroll position restored on back/forward

Use `up.navigate()` to trigger a full navigation from JS:
```js
up.navigate({ url: '/users' })       // navigates (updates history, scroll, etc.)
up.render({ url: '/users' })         // just renders, no history update
up.render({ url: '/users', navigate: true })  // opt into navigation behavior
```

Configure navigation defaults:
```js
up.fragment.config.navigateOptions.cache = false   // disable cache for navigation
up.fragment.config.navigateOptions.history = false // disable history for navigation
```

---

## Active nav links (up-current)

Unpoly automatically adds `.up-current` to links whose `[href]` matches the current browser URL.
This works automatically inside `<nav>` elements. Mark other containers with `[up-nav]`:

```html
<nav>
  <a href="/dashboard">Dashboard</a>  <!-- gets .up-current when at /dashboard -->
  <a href="/users">Users</a>
</nav>

<!-- Explicitly mark a container as nav -->
<div class="sidebar" up-nav>
  <a href="/settings">Settings</a>
</div>
```

Style active links with CSS:
```css
nav a.up-current {
  font-weight: bold;
  color: var(--accent);
}
```

`.up-current` is updated after every fragment update that changes the URL — no JS needed.

**Per-layer:** links in an overlay track the overlay's URL, not the root URL.

**Aliases** — mark a link as current for additional URLs:
```html
<a href="/users" up-alias="/users/*">Users</a>
<!-- Active for /users, /users/1, /users/new, etc. -->
```

**Configure the current class:**
```js
up.history.config.currentClasses = ['active', 'up-current']
```

**`up-nav` config:**
```js
up.status.config.navSelectors      // ['[up-nav]', 'nav'] — containers that track current URL
up.status.config.currentClasses    // ['up-current']
```

---

## History management

Fragment updates change the browser URL by default when **navigating**.

**Control history per link:**
```html
<!-- Update URL (default for navigating links) -->
<a href="/users" up-follow up-history="true">Users</a>

<!-- Don't change URL -->
<a href="/users" up-follow up-history="false">Users</a>

<!-- Custom URL in browser bar -->
<a href="/api/users.json" up-follow up-location="/users">Users</a>
```

**What counts as "navigation":** Following a link or submitting a form is navigation by default.
Background renders (`up.render()` from JS) do not navigate unless you pass `{ navigate: true }`.

**Update history from JS:**
```js
up.navigate({ url: '/users' })         // Navigate (updates history)
up.render({ url: '/users', history: true })  // Explicit history update

// Push state manually
up.history.push('/users')
up.history.replace('/users')
```

**Current URL:**
```js
up.history.location  // Current URL string
```

**Server override the URL:**
```http
X-Up-Location: /canonical-url
```

**Check current URL:**
```js
up.history.location           // current URL string
up.history.isLocation('/foo') // true if current URL matches (normalizes host)
```

**Configure defaults:**
```js
up.fragment.config.navigateOptions.history = true  // default
up.history.config.updateMetaTags = true   // sync <title>, <meta>, canonical etc. on navigation
up.history.config.metaTagSelectors        // what counts as a meta tag (link[rel=canonical], etc.)
up.history.config.restoreTargets = ['body']  // selectors restored on back/forward (default: body)
```

**`[up-meta]`** — mark custom `<head>` elements for synchronization during history changes:
```html
<link rel="license" href="..." up-meta>
<meta charset="utf-8" up-meta="false">  <!-- explicitly exclude -->
```

---

## Scrolling

Unpoly manages scroll position during fragment updates.

**`[up-scroll]` values:**

| Value | Behavior |
|-------|---------|
| `'target'` | Scroll target into view if not visible |
| `'top'` | Scroll layer to top |
| `'hash'` | Scroll to element matching URL hash |
| `'restore'` | Restore previous scroll position (back/forward) |
| `'reset'` | Scroll layer to top (alias) |
| `false` | Don't scroll |
| CSS selector | Scroll to matching element |

```html
<a href="/users" up-follow up-scroll="top">Users (scroll to top)</a>
<a href="/users" up-follow up-scroll=".first-result">Users (scroll to first result)</a>
```

From JS:
```js
up.render({ url: '/users', scroll: 'top' })
up.scroll('.result', { behavior: 'smooth' })  // Scroll element into view
```

**Smooth scrolling:**
```js
up.viewport.config.scrollBehavior = 'smooth'
```

**Viewport selector** — tell Unpoly which element is the scroll container:
```html
<div class="scroll-container" up-viewport>
  <!-- Unpoly will scroll this element -->
</div>
```

**Fixed elements** — tell Unpoly about `position: fixed` elements so it can compensate scroll and overlay width:
```html
<nav class="header" up-fixed="top">...</nav>      <!-- fixed to top -->
<footer up-fixed="bottom">...</footer>             <!-- fixed to bottom -->
<div class="sidebar" up-anchored="right">...</div> <!-- anchored to right edge -->
```

When a scrolling overlay opens, Unpoly adjusts the `right` position of `[up-anchored="right"]` and `[up-fixed]` elements to prevent layout shift. Use the `--up-scrollbar-width` CSS variable if you need to compensate manually.

**Custom viewport** — tell Unpoly which element is the scroll container:
```html
<div class="scroll-container" up-viewport>
  <!-- Unpoly will scroll this element -->
</div>
```

---

## Focus management

Unpoly manages keyboard focus during fragment updates for accessibility.

**`[up-focus]` values:**

| Value | Behavior |
|-------|---------|
| `'target'` | Focus the updated fragment |
| `'autofocus'` | Focus `[autofocus]` element in new content |
| `'main'` | Focus the main element |
| `'hash'` | Focus element matching URL hash |
| `'keep'` | Preserve focus if element survived update |
| `'layer'` | Focus the layer element (overlay root) |
| `false` | Don't change focus |
| CSS selector | Focus matching element |

```html
<a href="/search" up-follow up-focus="autofocus">Search</a>
```

From JS:
```js
up.render({ url: '/search', focus: '.search-input' })
```

**Focus visibility** — Unpoly manages `:focus-visible` to avoid focus rings on mouse clicks:
```js
up.viewport.config.focusVisible = 'auto'  // default: auto-detect
```

**Autofocus in overlays:**
When an overlay opens, focus moves to the first `[autofocus]` element or the overlay itself.

---

## URL patterns

URL patterns match multiple URLs using wildcards. Used with `[up-accept-location]`,
`[up-dismiss-location]`, and in JS APIs.

**Pattern syntax:**

| Pattern | Matches |
|---------|---------|
| `/users/:id` | `/users/1`, `/users/alice` |
| `/users/*` | `/users/anything/including/slashes` |
| `/users/` | `/users/` (exact trailing slash) |
| `/users` | `/users` (no trailing slash) |

```html
<!-- Close overlay when user lands on any /users/:id page -->
<a href="/users/new"
   up-layer="new modal"
   up-accept-location="/users/:id"
   up-on-accepted="console.log('Created user', value)">
  New user
</a>
```

**Named captures are passed as `value`:**
When `up-accept-location="/users/:id"` matches `/users/5`, the accepted value is `{ id: '5' }`.

From JS:
```js
up.layer.open({
  url: '/users/new',
  acceptLocation: '/users/:id',
  onAccepted: ({ value }) => console.log('User ID:', value.id)
})
```

**URL pattern matching in JS:**
```js
let pattern = new up.URLPattern('/users/:id')
pattern.test('/users/5')      // true
pattern.test('/users')        // false
pattern.recognize('/users/5') // { id: '5' }
```

---

## Back links and instant navigation

**`[up-back]`** — a link that navigates to the previous URL (not a browser `history.back()`):
```html
<a href="/default" up-back>Go back</a>
<!-- Transformed to: <a href="/previous-page" up-follow up-scroll="restore"> -->
```

If no previous URL is known, the link falls back to the `[href]` value.

**`up.history.previousLocation`** — the previous URL:
```js
up.history.previousLocation  // URL string or undefined
```

**`[up-instant]`** — follow a link on mousedown instead of click (faster perceived response):
```html
<a href="/users" up-follow up-instant>Users</a>
```

Configure globally:
```js
up.link.config.instantSelectors.push('a[href]')
```

**`[up-preload]` timing values:**

| Value | Behavior |
|-------|---------|
| `hover` (default) | Preload after hovering for `preloadDelay` ms (default: 90ms) |
| `insert` | Preload immediately when link is inserted into DOM |
| `reveal` | Preload when link scrolls into viewport |

```html
<a href="/dashboard" up-preload="reveal">Dashboard</a>
<a href="/dashboard" up-preload up-preload-delay="200">Dashboard</a>
```

Configure preload delay globally:
```js
up.link.config.preloadDelay = 90  // default: 90ms
```

**`up:click`** — normalized click event that honors `[up-instant]`:
```js
// Fires on mousedown for [up-instant] elements, click for others
// Not fired for modifier keys (Shift/Ctrl/Meta), non-primary buttons, or drags
up.on('up:click', 'a.special', function(event) {
  event.preventDefault()
})
```

**Combine instant + preload for maximum perceived speed:**

Stack both optimizations: follow on mousedown (instant) and pre-fetch on hover (preload).
Use `.unshift()` to ensure the selector takes priority over any built-in exclusions:

```js
// Follow all links on mousedown (no click delay)
up.link.config.instantSelectors.unshift('a[href]')

// Pre-fetch all links on hover so the response is ready before mousedown
up.link.config.preloadSelectors.unshift('a[href]')
```

With both set, hovering a link starts the request; the mousedown fires the navigation; by the
time the user lifts the mouse, the response is often already in the cache. The result is
navigation that feels near-instant even on modest connections.

---

## Restoring history

**Back/forward navigation:** Unpoly intercepts the browser's popstate event and restores
fragment content from cache (or re-fetches if expired).

**Scroll restoration:** Scroll positions are automatically restored when navigating back.
```js
up.history.config.restoreTargets = ['body']  // Default restore targets (tries each in order)
```

**`[up-restore-scroll]`** — restore scroll position to a specific element:
```html
<div class="content" up-restore-scroll>...</div>
```

**Custom restore behavior:**
```js
up.on('up:location:restore', function(event) {
  event.location       // URL being restored
  event.renderOptions  // mutable render options for the restoration pass
  event.preventDefault()  // stop Unpoly from restoring (browser still updates URL)
})
```

**`up:location:changed` event properties:**
```js
up.on('up:location:changed', function(event) {
  event.location          // new URL (already in address bar)
  event.previousLocation  // URL before the change
  event.reason            // 'push', 'replace', 'pop', or 'hash'
  event.willHandle        // mutable: set false to suppress Unpoly's handling
})
```
