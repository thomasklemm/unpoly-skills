# Fragments

## Table of contents
- [Targeting basics](#targeting-basics)
- [Special selectors](#special-selectors)
- [Append / prepend / replace children](#append-prepend-replace-children)
- [Multiple fragments](#multiple-fragments)
- [Optional and hungry targets](#optional-and-hungry-targets)
- [Keeping elements across updates](#keeping-elements-across-updates)
- [Client-side templates](#client-side-templates)
- [Handling all links and forms](#handling-all-links-and-forms)
- [Faux interactive elements](#faux-interactive-elements)
- [Emitting events from HTML](#emitting-events-from-html)
- [up.render() options](#uprender-options)

---

## Targeting basics

Use `[up-target]` with a CSS selector to swap only that fragment:

```html
<a href="/posts/5" up-target=".content">Read post</a>
```

The server response may be a full HTML document — only the element matching `.content` is used.

From JS:
```js
up.render({ target: '.content', url: '/posts/5' })
```

When `[up-target]` is omitted, Unpoly updates the **main target** (usually `<main>`).

---

## Special selectors

| Selector | Meaning |
|----------|---------|
| `:main` | The page's primary content area (`<main>` by default) |
| `:layer` | The entire layer (topmost swappable element) |
| `:none` | Make a request with no DOM change |
| `:origin` | Replaced with derived selector of the origin element |
| `.foo:maybe` | Target `.foo` only if present in both page and response |
| `.foo:after` | Append children to `.foo` |
| `.foo:before` | Prepend children to `.foo` |
| `.foo:content` | Replace children of `.foo`, keep the element itself |

Configure main targets:
```js
up.fragment.config.mainTargets = ['main', '.content', '[up-main]']
```

---

## Append / prepend / replace children

```html
<!-- Infinite scroll: append items to list, update pagination link -->
<a href="/page/2" up-target=".tasks:after, .next-page">Load more</a>

<!-- Replace children while keeping the wrapper element -->
<a href="/cards/5" up-target=".card:content">Show card #5</a>
```

---

## Multiple fragments

Comma-separate selectors to update multiple fragments at once:

```html
<a href="/posts/5" up-target=".content, .unread-count">Read post</a>
```

When one target is an ancestor of another, only the ancestor is requested.

---

## Optional and hungry targets

**`:maybe`** — target is optional; no error if missing:
```html
<a href="/cards/5" up-target=".content, .unread-count:maybe">Read post</a>
```

**`[up-hungry]`** — element opts into being updated whenever it appears in any response:
```html
<div class="unread-count" up-hungry>12</div>
```
Common for notification badges, nav bars, flash messages that live outside the targeted fragment.

**`[up-hungry]` attributes:**

| Attribute | Default | Description |
|-----------|---------|-------------|
| `[up-if-layer]` | `'current'` | Layer filter: `'current'`, `'any'`, `'subtree'`, etc. |
| `[up-transition]` | — | Transition animation when updating |
| `[up-on-hungry]` | — | Callback code; call `event.preventDefault()` to skip update |

```html
<!-- Skip update when overlay is open -->
<div class="notifications" up-hungry up-on-hungry="if (up.layer.isOverlay()) event.preventDefault()">
```

By default, `[up-hungry]` only fires when the element's own layer is targeted. Use
`[up-if-layer="any"]` to update the element even when requests target a different layer
(e.g. an overlay):

```html
<!-- Update server-code links even when an overlay handles the response -->
<div class="debug-bar" up-hungry up-if-layer="any">
  <a href="...">Controller</a> / <a href="...">View</a>
</div>
```

This is useful for debugging panels, analytics banners, or any element that should always
reflect the most recent server action regardless of which layer was targeted.

---

## Keeping elements across updates

**`[up-keep]`** — preserve an element across fragment updates (e.g. playing video, open <details>):

```html
<video src="screencast.mp4" controls up-keep></video>
```

Unpoly keeps the old element in place when a matching `[up-keep]` element is found in the response.
Match is by element type + `[id]` by default.

**`[up-keep]` values:**

| Value | Behavior |
|-------|---------|
| `true` (default) | Keep element; match by type + `[id]` |
| CSS selector | Keep element; match if new element matches selector |
| `same-html` | Keep only if new element has same HTML content |
| `same-data` | Keep only if new element has same `[up-data]` |
| `false` | Always replace (overrides existing `[up-keep]`) |

**`[up-on-keep]`** — run code when an element is kept (receives `event`, `newFragment`, `newData`):
```html
<audio src="music.mp3" up-keep up-on-keep="if (newFragment.src !== element.src) event.preventDefault()"></audio>
```

**Force update on a link/form** with `[up-use-keep="false"]` to override kept elements:
```html
<a href="/reset" up-use-keep="false">Hard reset</a>
```

Or from JS: `up.render({ url: '/reset', keep: false })`

**`[up-id]`** — unique identifier for target derivation (alternative to `[id]` for keeping elements with predictable selectors):
```html
<div class="chart" up-id="revenue-chart" up-keep></div>
```

**`[up-expand]`** — make a non-link area clickable by delegating to a child link:

```html
<div class="card" up-expand>
  <h3>Title</h3>
  <a href="/cards/5">Read more</a> <!-- This link's href/attributes are used -->
</div>
```

---

## Client-side templates

Use `<template>` elements to render HTML fragments without a server request.
Useful for placeholders, small overlays, or optimistic rendering.

```html
<!-- Reference a template by ID -->
<a up-fragment="#my-template">Click me</a>

<template id="my-template">
  <div class="target">New content</div>
</template>
```

Templates also work with `[up-content]`, `[up-document]`, and `[up-placeholder]` attributes:
```html
<!-- Open overlay with local template (no request) -->
<a up-layer="new modal" up-document="#help-template">Help</a>

<template id="help-template">
  <div class="content"><p>Help text</p></div>
</template>
```

Pass data to a template via `[up-use-data]` (available in compilers as `data`):
```html
<a up-fragment="#task-template" up-use-data='{ "description": "Buy toast" }'>Add task</a>

<template id="task-template">
  <div class="task"><!-- compiler fills description --></div>
</template>
```

From JS:
```js
up.render({ target: '.content', fragment: '#my-template' })
```

**Custom template engines via `up:template:clone`**

Intercept `up.template.clone()` to implement your own template syntax. Register a listener
on a custom `type` attribute to handle only your template elements:

```html
<!-- Use a non-standard type so the browser doesn't execute it -->
<script id="task-row" type="text/minimustache">
  <div class="task">
    <span class="task-text">{{text}}</span>
  </div>
</script>
```

```js
// Handle up.template.clone() calls for [type="text/minimustache"] elements
up.on('up:template:clone', '[type="text/minimustache"]', function(event) {
  let filled = event.target.innerHTML.replace(
    /\{\{(\w+)\}\}/g,
    (_match, variable) => up.util.escapeHTML(event.data[variable] ?? '')
  )
  event.nodes = up.element.createNodesFromHTML(filled)
})
```

Use from a preview to optimistically insert new rows:

```js
up.preview('add-task', function(preview) {
  let text = preview.params.get('task[text]')
  let taskList = preview.fragment.querySelector('.task-list')
  let newRow = up.template.clone('#task-row', { text })
  preview.insert(taskList, 'beforeend', newRow)
})
```

The cloned node is inserted into the DOM immediately as an optimistic preview, then replaced
by the real server response when it arrives. `up.util.escapeHTML` prevents XSS when rendering
user-supplied values into templates.

---

## Handling all links and forms

To handle all links on a page without adding `[up-follow]` everywhere:

```js
// Handle all links that don't opt out
up.link.config.followSelectors.push('a[href]')

// Exclude links you don't want to handle
up.link.config.noFollowSelectors.push('[up-no-follow]', '.downloads a')
```

Or use `[up-dash]` as a shorthand that sets `[up-follow]`, `[up-target]`, `[up-transition]`, etc.:
```html
<a href="/users" up-dash=".content">Users</a>
<!-- Equivalent to: <a href="/users" up-follow up-target=".content"> -->
```

### Handling everything in a Rails app

```erb
<!-- app/views/layouts/application.html.erb -->
<body>
  <%= link_to "Users", users_path, "up-follow": true %>
</body>
```

```js
// application.js
up.link.config.followSelectors.push('a[href]')
up.form.config.submitSelectors.push('form')
```

---

## Faux interactive elements

Use `[up-follow]` with `[up-href]` to make any non-interactive element behave like a link:

```html
<span up-follow up-href="/details">Read more</span>
<div class="card" up-follow up-href="/cards/5" up-target=".content">...</div>
```

The element gets proper keyboard navigation and ARIA roles automatically.
Prefer real `<a>` tags when possible — they work without JS, support new-tab, and are indexed by crawlers.

**`[up-clickable]`** — make an element keyboard-navigable without following a link:
```html
<div up-clickable>...</div>
```

---

## Emitting events from HTML

Emit a custom DOM event when an element is clicked, without writing JS:

```html
<button up-emit="user:activate" up-emit-props='{ "id": 5 }'>Activate</button>
```

The event bubbles up the DOM and can be handled by a compiler or listener:
```js
up.on('user:activate', function(event) {
  console.log('Activating user', event.id)
})
```

Combine with `[up-follow]` to also navigate after emitting:
```html
<a href="/users/5" up-follow up-emit="user:selected">Select user</a>
```

---

## Fragment utility functions

```js
// Element lookup (layer-aware, ignores .up-destroying elements)
up.fragment.get('.content')           // First match in current layer
up.fragment.get(root, '.item', { layer: 'root' })  // From root, specific layer
up.fragment.all('.item')              // All matches in current layer
up.fragment.closest(element, '.card') // Nearest ancestor within same layer
up.fragment.subtree(element, '.item') // Descendants + self matching selector
up.fragment.contains(root, selector)  // Containment check within layer

// Source/ETag/time of a fragment
up.fragment.source(element)           // URL fragment was loaded from ([up-source])
up.fragment.etag(element)             // ETag from [up-etag]
up.fragment.time(element)             // Date from [up-time]

// Target derivation
up.fragment.toTarget(element)         // Derives CSS selector; throws up.CannotTarget if impossible
up.fragment.isTargetable(element)     // Boolean version of toTarget
up.fragment.isAlive(element)          // True if connected and not in exit animation

// Aborting
up.fragment.abort(element)            // Abort requests for element; always emits up:fragment:aborted
up.fragment.abort({ layer: 'root' })  // Abort by layer
up.fragment.onAborted(element, fn)    // Callback when element (or ancestor) is aborted; auto-cleanup on destroy
```

**Note on `up.fragment.get` vs `document.querySelector`:**
- Only searches the current layer unless `{ layer }` is explicit
- Ignores elements with `.up-destroying` (in exit animation)
- Supports Unpoly-specific selectors (`:main`, `:layer`)
- With `{ origin }`, prefers region-aware matching first (nearest to origin element)

---

## Fragment config (`up.fragment.config`)

```js
up.fragment.config.mainTargets        // ['[up-main]', 'main', ':layer'] — main element selectors
up.fragment.config.runScripts         // false — whether to execute <script> in updated fragments
up.fragment.config.autoRevalidate     // (response) => response.expired — revalidation rule
up.fragment.config.skipResponse       // Function — skips empty/unchanged responses
up.fragment.config.match              // 'region' — prefer region-aware matching; or 'first'
up.fragment.config.badTargetClasses   // CSS classes ignored during target derivation (e.g. 'active', 'selected')
up.fragment.config.targetDerivers     // Array of patterns/functions for automatic target derivation

// Applied to all render passes:
up.fragment.config.renderOptions      // { hungry: true, keep: true, saveScroll: true, ... }

// Applied additionally when { navigate: true }:
up.fragment.config.navigateOptions    // { cache: 'auto', revalidate: 'auto', history: 'auto', ... }
```

---

## up.render() options

Key options for `up.render()`, `up.follow()`, `up.navigate()`:

| Option | HTML attribute | Description |
|--------|---------------|-------------|
| `target` | `[up-target]` | CSS selector to update |
| `url` | `[href]` | URL to fetch |
| `method` | `[up-method]` | HTTP method (GET/POST/PUT/PATCH/DELETE) |
| `layer` | `[up-layer]` | Layer to render into (`'current'`, `'new'`, `'root'`, layer mode) |
| `failTarget` | `[up-fail-target]` | Target for non-2xx responses |
| `fallback` | `[up-fallback]` | Fallback target if main target missing |
| `content` | `[up-content]` | Render local HTML string without server request |
| `fragment` | `[up-fragment]` | Render local HTML fragment |
| `document` | `[up-document]` | Render from full HTML string |
| `transition` | `[up-transition]` | Animation name (e.g. `'cross-fade'`, `'move-left'`) |
| `cache` | `[up-cache]` | `true`, `false`, or `'auto'` |
| `revalidate` | `[up-revalidate]` | Revalidate cached content |
| `scroll` | `[up-scroll]` | Scroll behavior (`'target'`, `'top'`, `'restore'`, `'hash'`, CSS selector) |
| `focus` | `[up-focus]` | Focus behavior after update |
| `history` | `[up-history]` | Whether to update browser history (`true`/`false`) |
| `confirm` | `[up-confirm]` | Show browser confirm dialog before following |
| `params` | — | Additional request parameters (object or `up.Params`) |
| `headers` | `[up-headers]` | Additional request headers |
| `origin` | — | Origin element for proximity-based ambiguous selector resolution |

### Rendering local content without a server request

```js
// { content } — replace children of target element
up.render({ target: '.content', content: '<p>Hello</p>' })

// { fragment } — target derived from root element; no explicit target needed
up.render({ fragment: '<div class="content"><p>Hello</p></div>' })

// { document } — extract fragment from larger HTML string
up.render({ target: '.content', document: fullHTML })

// { response } — render from a previously fetched up.Response object
let response = await up.request('/path')
up.render({ target: '.content', response })
```

```html
<!-- Open overlay with local content (no request) -->
<a up-layer="new popup" up-content="<p>Help text</p>">Help</a>
```
