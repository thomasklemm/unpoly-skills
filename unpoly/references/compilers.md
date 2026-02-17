# Compilers

## Table of contents
- [Registering compilers](#registering-compilers)
- [Destructors (cleanup)](#destructors-cleanup)
- [Passing data to compilers](#passing-data-to-compilers)
- [Accessing render pass metadata](#accessing-render-pass-metadata)
- [Macros](#macros)
- [Integrating third-party libraries](#integrating-third-party-libraries)
- [up.hello() — compile dynamic content](#uphello--compile-dynamic-content)
- [Layer context](#layer-context)
- [Legacy scripts](#legacy-scripts)

---

## Registering compilers

A compiler is a JS function that runs for every matching element inserted into the DOM —
whether on initial page load or after a fragment update:

```js
// Replaces DOMContentLoaded — works for every render pass
up.compiler('.current-time', function(element) {
  element.textContent = new Date().toLocaleTimeString()
})
```

**Avoid `DOMContentLoaded`** — it only fires once. Compilers fire for every matching element:
```js
// ❌ Only runs on initial load
document.addEventListener('DOMContentLoaded', function() {
  for (let el of document.querySelectorAll('.widget')) { init(el) }
})

// ✅ Runs on initial load AND after every fragment update
up.compiler('.widget', function(element) {
  init(element)
})
```

**Compilers receive their element as the first argument:**
```js
up.compiler('input[autofocus]', function(input) {
  input.focus()
})
```

**Multiple compilers** can target the same elements — all run.

**Compiler priority** — run before/after others:
```js
up.compiler('.widget', { priority: 10 }, function(element) { })
// Higher priority runs first (default: 0)
```

---

## Destructors (cleanup)

When a compiled element is removed from the DOM, Unpoly calls any registered destructors.
This prevents memory leaks in long-running sessions.

**Event listeners on the element itself** don't need cleanup — they're GC'd automatically:
```js
// ✅ No cleanup needed
up.compiler('.click-to-hide', function(element) {
  element.addEventListener('click', () => element.remove())
})
```

**Global event listeners / intervals need cleanup:**
```js
// ❌ Memory leak — window listener survives element removal
up.compiler('.scroll-watch', function(element) {
  window.addEventListener('scroll', handler)
})

// ✅ Return a destructor function
up.compiler('.scroll-watch', function(element) {
  let handler = () => console.log('scrolled')
  window.addEventListener('scroll', handler)
  return () => window.removeEventListener('scroll', handler)
})
```

**Multiple cleanup functions — return an array:**
```js
up.compiler('.widget', function(element) {
  let timer = setInterval(tick, 1000)
  window.addEventListener('resize', onResize)
  return [
    () => clearInterval(timer),
    () => window.removeEventListener('resize', onResize)
  ]
})
```

**Alternative: `up.destructor()`** — register cleanup inline:
```js
up.compiler('.widget', function(element) {
  let timer = setInterval(tick, 1000)
  up.destructor(element, () => clearInterval(timer))

  let onResize = () => { }
  window.addEventListener('resize', onResize)
  up.destructor(element, () => window.removeEventListener('resize', onResize))
})
```

---

## Passing data to compilers

Attach data to elements via HTML5 data attributes or `[up-data]` (relaxed JSON):

```html
<!-- HTML5 data attributes -->
<div class="user" data-name="Alice" data-age="31">Alice</div>

<!-- up-data: relaxed JSON (unquoted keys, trailing commas OK) -->
<span class="user" up-data="{ age: 31, name: 'Alice' }">Alice</span>
```

The compiler receives an object with all merged data:
```js
up.compiler('.user', function(element, data) {
  console.log(data.name)  // "Alice"
  console.log(data.age)   // 31
})
```

**Set defaults:**
```js
up.compiler('.chart', { defaults: { color: 'blue', size: 100 } }, function(element, data) {
  console.log(data.color)  // 'blue' unless overridden
})
```

---

## Accessing render pass metadata

The compiler's third argument has info about the current render pass:

```js
up.compiler('.widget', function(element, data, meta) {
  meta.layer          // The up.Layer this element belongs to
  meta.layer.mode     // 'root', 'modal', 'drawer', etc.
  meta.revalidating   // true if this is a cache revalidation render pass
  meta.fromCache      // true if the content came from cache
})
```

**Common use case — skip analytics on revalidation:**
```js
up.compiler('[track-page-view]', function(element, data, meta) {
  if (!meta.revalidating) {
    analytics.trackPageView(meta.layer.location)
  }
})
```

---

## Macros

Macros are like compilers but run **before** any other compilers.
Use them to transform custom HTML attributes into standard Unpoly attributes:

```js
up.macro('[dashboard-link]', function(element) {
  // Transform custom attribute to Unpoly attributes
  element.setAttribute('up-follow', '')
  element.setAttribute('up-target', '.dashboard-content')
  element.setAttribute('up-transition', 'cross-fade')
})
```

```html
<!-- Now this works: -->
<a href="/dashboard" dashboard-link>Dashboard</a>
```

---

## Integrating third-party libraries

**Map library (Leaflet, Google Maps):**
```js
up.compiler('.map', function(element, data) {
  let map = L.map(element).setView([data.lat, data.lng], data.zoom)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map)
  return () => map.remove()  // Cleanup
})
```

**Date picker:**
```js
up.compiler('input.date-picker', function(input) {
  let picker = flatpickr(input, { dateFormat: 'Y-m-d' })
  return () => picker.destroy()
})
```

**Lightbox:**
```js
up.compiler('a.lightbox', function(element) {
  lightboxify(element)
  // No cleanup needed if lightboxify cleans itself up
})
```

---

## up.hello() — compile dynamic content

When you insert HTML into the DOM yourself (bypassing Unpoly), call `up.hello()` to run compilers:

```js
let element = document.createElement('div')
element.className = 'widget'
element.setAttribute('up-data', '{ value: 42 }')
document.body.appendChild(element)
up.hello(element)  // Runs matching compilers on element and its children
```

---

## Layer context

Access the current layer's context from a compiler:

```js
up.compiler('.admin-panel', function(element, data, meta) {
  let context = meta.layer.context
  if (!context.isAdmin) {
    element.remove()
  }
})
```

Set context values from JS:
```js
up.layer.current.context.key = 'value'
```

---

## Legacy scripts

If you have legacy `$(document).ready()` or `DOMContentLoaded` handlers,
Unpoly can re-run them after fragment updates:

```js
// Re-run jQuery document-ready handlers after each fragment update
up.compiler(document, { batch: true }, function() {
  // No-op — just trigger jQuery's ready handlers
})
```

Or use `up.script.config.nonceableAttributes` to control CSP-compatible inline scripts.

For gradual migration, configure which selectors Unpoly should handle:
```js
// Opt into Unpoly for specific links only
up.link.config.followSelectors = ['a[up-follow]', 'a.ajax-link']
```
