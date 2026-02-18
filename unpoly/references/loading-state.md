# Loading State

## Table of contents
- [CSS loading classes](#css-loading-classes)
- [Placeholders](#placeholders)
- [Previews (arbitrary loading state)](#previews-arbitrary-loading-state)
- [Optimistic rendering](#optimistic-rendering)
- [Disabling forms while submitting](#disabling-forms-while-submitting)
- [Global progress bar](#global-progress-bar)
- [Network issues and offline](#network-issues-and-offline)

---

## CSS loading classes

When Unpoly is loading, it automatically adds CSS classes to elements:

| Selector | When active |
|----------|------------|
| `.up-active` | Added to the link/button being followed |
| `.up-loading` | Added to the fragment being updated |
| `[aria-busy]` | Added to the fragment being updated |
| `up-progress-bar` | Global progress bar element (custom element) |

**Style loading state:**
```css
/* Dim the fragment being replaced */
.up-loading {
  opacity: 0.6;
  transition: opacity 0.2s;
}

/* Show spinner on active link */
a.up-active::after {
  content: ' ⟳';
}
```

---

## Placeholders

Show a placeholder element while a fragment loads:

```html
<div id="content" up-defer="/slow-endpoint">
  <!-- This placeholder is shown while loading -->
  <div class="skeleton">
    <div class="skeleton-line"></div>
    <div class="skeleton-line"></div>
  </div>
</div>
```

**`[up-placeholder]`** — show a placeholder when a link is followed:
```html
<a href="/posts" up-target=".content" up-placeholder="<div class='spinner'>Loading…</div>">
  Posts
</a>
```

**Named placeholder elements:**
```html
<!-- The placeholder element is shown when .content is loading -->
<div class="content-placeholder" up-placeholder="true">
  <div class="skeleton">...</div>
</div>

<div class="content">Current content</div>
```

**Pattern: reusable skeleton screens in `<template>` elements**

Define skeleton screens once as named `<template>` elements and reference them by ID in
`[up-placeholder]`. Only inject the templates on the initial full-page load — on Unpoly
fragment updates the `<template>` elements already exist in the DOM from the initial load
and don't need to be re-sent:

```erb
<%# Only render template elements on full-page loads — Unpoly requests don't need them %>
<%= render 'placeholders/templates' unless up? %>
```

```erb
<%# placeholders/_templates.erb %>
<template id="table-placeholder">
  <%= render 'placeholders/table_skeleton' %>
</template>

<template id="form-placeholder">
  <%= render 'placeholders/form_skeleton' %>
</template>
```

Reference by ID and optionally pass data to a compiler that adjusts the skeleton:

```erb
<%# Show a 5-row skeleton table while the list loads %>
<%= link_to 'Companies', companies_path,
  'up-follow': true,
  'up-placeholder': '#table-placeholder { rows: 5 }' %>

<%# up.reload with placeholder — used after accepting an overlay %>
'up-on-accepted': "up.reload('#companies', { placeholder: '#table-placeholder { rows: 5 }' })"
```

The `{ rows: 5 }` part is parsed as `[up-data]` on the cloned placeholder element and passed
to a compiler as the `data` argument:

```js
// Trim skeleton rows to the expected count
up.compiler('.skeleton-table', function(element, { rows = 10 }) {
  let trs = element.querySelectorAll('tr')
  // Remove excess rows so the skeleton matches the expected result set size
  Array.from(trs).slice(0, trs.length - rows).forEach(tr => tr.remove())
})
```

---

## Previews (arbitrary loading state)

Previews let you apply arbitrary DOM changes while waiting for a response,
then revert them when the response arrives:

**HTML preview attribute:**
```html
<a href="/inbox" up-target=".content" up-preview="show-spinner">
  Open inbox
</a>
```

**Register a named preview:**
```js
up.preview('show-spinner', function(preview) {
  // preview.fragment is the element being updated
  preview.insert(preview.fragment, 'afterbegin', '<div class="spinner">Loading…</div>')
  // Changes are automatically reverted when response arrives
})
```

**Inline preview with up.Preview:**
```js
up.compiler('.load-btn', function(button) {
  button.addEventListener('click', function() {
    up.render({
      url: button.dataset.url,
      target: '.content',
      preview: function(preview) {
        preview.addClass(document.body, 'is-loading')
        preview.setStyle(preview.fragment, { opacity: 0.5 })
      }
    })
  })
})
```

**`up.Preview` API:**
```js
up.preview('my-preview', function(preview) {
  preview.fragment               // element being updated
  preview.origin                 // element that triggered the update
  preview.params                 // form params from the pending request

  // All changes are reverted automatically:
  preview.addClass(element, 'loading')
  preview.removeClass(element, 'loaded')
  preview.setStyle(element, { opacity: 0.5 })
  preview.insert(element, 'beforeend', '<p>Loading…</p>')
  preview.swapContent(element, '<span class="spinner"></span>')  // replace content, revert on response
  preview.hide(element)          // hide element, restore on response
  preview.disable(formElement)
  preview.showPlaceholder('<div class="skeleton">…</div>')  // fill overlay placeholder area

  // Or return a cleanup function:
  let timer = setInterval(() => { }, 1000)
  return () => clearInterval(timer)
})
```

**Composed previews** — apply multiple named previews with a comma-separated string:

```html
<a href="/tasks/clear" up-preview="btn-spinner, clear-tasks">Clear done</a>
```

Useful for combining a generic "show spinner on button" preview with a domain-specific
optimistic update. Both previews run independently and are both reverted when the response arrives.

In Rails/ERB you can select the preview dynamically:

```erb
<% preview = task.new_record? ? 'add-task, btn-spinner' : 'btn-spinner' %>
<%= form_for task, html: { 'up-preview': preview } do |form| %>
```

**Pattern: button spinner that preserves layout dimensions**

When swapping button text for a spinner, lock the button's size first with `preview.setStyle()`
to prevent layout shift:

```js
up.preview('btn-spinner', function(preview) {
  let button = preview.origin.matches('.btn')
    ? preview.origin
    : preview.origin.closest('form')?.querySelector('button[type=submit]')

  if (!button) return

  // Lock dimensions so the layout doesn't shift when content is replaced
  preview.setStyle(button, {
    height: button.offsetHeight + 'px',
    width: button.offsetWidth + 'px'
  })
  preview.swapContent(button, '<span class="spinner"></span>')
})
```

**Pattern: dim and greyscale content behind a loading spinner using CSS `:has()`**

Insert a spinner at the top of the main area; CSS `:has()` automatically dims everything else:

```js
up.preview('main-spinner', function(preview) {
  // Find the :main ancestor of the fragment being updated
  let main = up.fragment.closest(preview.fragment, ':main')
  preview.insert(main, 'afterbegin', '<div class="main-spinner"></div>')
})
```

```css
.main-spinner {
  /* your spinner styles */
  position: absolute; top: 50%; left: 50%;
  transform: translate(-50%, -50%);
}

/* Dim siblings via :has() — no JS needed */
main:has(.main-spinner) {
  position: relative;
}
main:has(.main-spinner) > *:not(.main-spinner) {
  transition: opacity 0.3s ease-out, filter 0.3s ease-out;
  opacity: 0.4;
  filter: grayscale(100%);
}
```

The spinner and all style changes are reverted automatically when the response arrives.

---

## Optimistic rendering

Render a predicted response immediately, then replace with the real response:

```js
up.compiler('.like-button', function(button) {
  button.addEventListener('click', function() {
    up.render({
      url: button.dataset.url,
      method: 'POST',
      target: '.like-count',
      preview: function(preview) {
        // Optimistically show +1
        let count = preview.fragment
        count.textContent = parseInt(count.textContent) + 1
      }
    })
  })
})
```

**With client-side templates:**
```html
<template id="optimistic-like">
  <span class="like-count">{{ count + 1 }}</span>
</template>
```

```js
up.preview('optimistic-like', function(preview) {
  let template = document.getElementById('optimistic-like')
  let count = parseInt(preview.fragment.textContent)
  // Render template with data
  preview.insert(preview.fragment, 'outerHTML',
    template.innerHTML.replace('{{ count + 1 }}', count + 1))
})
```

---

## Disabling forms while submitting

`[up-disable]` disables form fields and buttons while a request is in flight:

```html
<!-- Disable entire form (fields + submit button) -->
<form action="/checkout" up-submit up-disable>

<!-- Disable only the submit button -->
<form action="/checkout" up-submit up-disable="button[type=submit]">

<!-- Disable a fieldset while watching -->
<select name="country" up-watch up-target=".state-field" up-watch-disable=".state-field">
```

From JS:
```js
up.submit(form, { disable: true })
```

---

## Global progress bar

Unpoly shows a progress bar at the top of the page for requests taking more than 300ms:

```css
/* The progress bar is a custom element */
up-progress-bar {
  background: blue;
  height: 3px;
}
```

**Configure:**
```js
up.network.config.badResponseTime = 400  // ms before showing (default 400)
```

**Disable:**
```js
up.network.config.progressBar = false
```

---

## Network issues and offline

Unpoly handles disconnects and slow connections:

**Offline cache:** When offline, Unpoly can serve cached responses.
The cache is kept for up to 90 minutes by default.

**Detect offline state:**
```js
up.on('up:network:offline', function(event) {
  // Show offline indicator
})

up.on('up:network:recover', function(event) {
  // Hide offline indicator
})
```

**Configure:**
```js
up.network.config.badResponseTime = 400     // ms before showing progress bar
up.network.config.cacheEvictAge = 90 * 60 * 1000  // 90 min offline cache
```

**`[up-offline-feedback]`** on flash/notification elements to show only when offline:
```html
<div class="offline-banner" up-hide-for=":online">You are offline</div>
```

**Retry failed requests:**
```js
up.on('up:fragment:aborted', function(event) {
  if (confirm('Request failed. Retry?')) {
    event.request.load()
  }
})
```
