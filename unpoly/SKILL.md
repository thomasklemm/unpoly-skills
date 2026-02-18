---
name: unpoly
description: >
  Comprehensive guide for building server-rendered apps with Unpoly — a progressive
  enhancement framework that adds SPA-like behavior through HTML attributes.
  Use when working with Unpoly fragment updates, overlays/layers, forms, compilers,
  caching, lazy loading, animations, transitions, error handling, polling, the render lifecycle, or server integration.
  Covers [up-follow], [up-target], [up-layer], [up-submit], [up-validate], up.compiler(),
  up.render(), up.layer.open(), caching, previews, optimistic rendering, and more.
---

# Unpoly

Unpoly enhances server-rendered HTML so links and forms update page fragments instead
of full pages, without building a SPA. The server still renders HTML — Unpoly handles
the smart patching, caching, overlays, and lifecycle.

**Current version:** 3.12 — [API reference](https://unpoly.com/api) | [Changelog](https://unpoly.com/changes)

## Core mental model

1. **Fragments** — CSS-selector-targeted pieces of a page that get swapped on navigation
2. **Layers** — A stack of contexts (root + any number of overlays)
3. **Compilers** — JS functions that run whenever matching elements are inserted into the DOM
4. **Navigation** — Link/form interactions that change history and trigger fragment updates

## Quick reference: key HTML attributes

| Attribute | Purpose |
|-----------|---------|
| `[up-follow]` | Follow link as fragment update |
| `[up-target]` | CSS selector of fragment to replace |
| `[up-layer="new"]` | Open response in overlay |
| `[up-layer="new drawer\|popup\|modal"]` | Open in specific overlay mode |
| `[up-submit]` | Submit form as fragment update |
| `[up-validate]` | Validate field against server on blur |
| `[up-watch]` | Watch field changes and auto-submit |
| `[up-autosubmit]` | Auto-submit form on field change |
| `[up-hungry]` | Optionally update element whenever it appears in any response |
| `[up-preload]` | Preload link on hover |
| `[up-instant]` | Follow link on mousedown, not click |
| `[up-back]` | Navigate to previous URL |
| `[up-defer]` | Lazy-load a fragment |
| `[up-poll]` | Periodically reload a fragment |
| `[up-keep]` | Preserve element across fragment updates |
| `[up-transition]` | CSS animation between old and new content |
| `[up-switch]` | Show/hide/enable/disable elements based on field value |
| `[up-disable]` | Disable fields/buttons during request |
| `[up-confirm]` | Show confirmation dialog before following |
| `[up-content]` | Open overlay from inline HTML (no request) |
| `[up-accept-event]` | Close+accept overlay when named event fires (payload becomes `value`) |
| `[up-dismiss-event]` | Close+dismiss overlay when named event fires |
| `[up-form-group]` | Mark container as the validation group for `[up-validate]` |
| `[up-main]` | Declare which element is the main target for a specific overlay mode |

## Quick reference: key JS functions

```js
up.render({ target: '.content', url: '/page' })  // Render fragment from URL
up.follow(linkElement)                            // Follow a link
up.submit(formElement)                            // Submit a form
up.reload('.sidebar')                             // Reload a fragment
up.navigate({ url: '/page' })                    // Navigate (updates history)

up.layer.open({ url: '/modal', layer: 'new' })   // Open overlay
up.layer.current                                  // Current layer
up.layer.accept(value)                            // Close overlay with value
up.layer.dismiss(value)                           // Dismiss overlay

up.compiler('.widget', (element, data) => { })   // Register compiler
up.on('up:fragment:inserted', handler)           // Listen to events

up.cache.expire()                                 // Mark all cache stale
up.cache.evict()                                  // Clear all cache entries
```

## Server response headers

| Header | Purpose |
|--------|---------|
| `X-Up-Target` | Override which fragment to update |
| `X-Up-Accept-Layer: value` | Accept overlay from server |
| `X-Up-Dismiss-Layer: value` | Dismiss overlay from server |
| `X-Up-Open-Layer: {}` | Force-open overlay from server |
| `X-Up-Events: [...]` | Emit events on the frontend |
| `X-Up-Context: {}` | Merge JSON into current layer context |
| `X-Up-Location` | Override the URL pushed to history |

## Reference files

Load these when the user's question covers that topic:

- **[fragments.md](references/fragments.md)** — Targeting, rendering options, client-side templates, handling all links/forms, `:main`, `:layer`, `:maybe`, `[up-hungry]`, `[up-keep]`, `[up-expand]`, `up.render()` options
- **[forms.md](references/forms.md)** — Form submission, validation (`[up-validate]`), reactive forms (`[up-watch]`, `[up-autosubmit]`), form state switching (`[up-switch]`), watch options, disabling forms
- **[layers.md](references/layers.md)** — Opening overlays, layer modes, subinteractions, close conditions, accepting/dismissing layers, layer context, targeting layers
- **[performance.md](references/performance.md)** — Caching, revalidation, lazy loading (`[up-defer]`), preloading, optimizing server responses, polling (`[up-poll]`)
- **[loading-state.md](references/loading-state.md)** — CSS loading classes, placeholders, previews, optimistic rendering, disabling forms, progress bar, offline handling
- **[compilers.md](references/compilers.md)** — Registering compilers, destructors, passing data, macros, `up.hello()`, integrating JS libraries, context
- **[navigation.md](references/navigation.md)** — Navigation vs rendering, `.up-current` active links, history, scroll restoration, focus management, URL patterns
- **[animations.md](references/animations.md)** — Built-in animations/transitions, custom animations, duration/easing, disabling animation
- **[error-handling.md](references/error-handling.md)** — Failed responses, `[up-fail-target]`, skipping rendering, conditional requests (ETags), network errors, flash messages
- **[lifecycle.md](references/lifecycle.md)** — Render lifecycle events, controlling rendering, error handling, server integration headers
