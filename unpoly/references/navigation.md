# Navigation

## Table of contents
- [History management](#history-management)
- [Scrolling](#scrolling)
- [Focus management](#focus-management)
- [URL patterns](#url-patterns)
- [Restoring history](#restoring-history)

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

**Configure defaults:**
```js
up.fragment.config.navigateOptions.history = true  // default
up.history.config.updateMetaTags = true  // sync <title>, <meta> on navigation
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

**Anchored headers** — prevent content from being obscured by fixed headers:
```html
<nav class="header" up-fixed="top">...</nav>
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

## Restoring history

**Back/forward navigation:** Unpoly intercepts the browser's popstate event and restores
fragment content from cache (or re-fetches if expired).

**Scroll restoration:** Scroll positions are automatically restored when navigating back.
```js
up.history.config.restoreTargets = ['.content']  // Default restore targets
```

**`[up-restore-scroll]`** — restore scroll position to a specific element:
```html
<div class="content" up-restore-scroll>...</div>
```

**Custom restore behavior:**
```js
up.on('up:location:restore', function(event) {
  // event.location is the URL being restored
  // event.defaultPrevented to skip default restore
})
```
