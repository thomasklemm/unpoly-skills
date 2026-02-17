# Layers (Overlays)

## Table of contents
- [Layer terminology](#layer-terminology)
- [Opening overlays](#opening-overlays)
- [Overlay modes](#overlay-modes)
- [Targeting layers](#targeting-layers)
- [Close conditions and subinteractions](#close-conditions-and-subinteractions)
- [Accepting and dismissing](#accepting-and-dismissing)
- [Layer context](#layer-context)
- [Working with layers from JS](#working-with-layers-from-js)
- [Opening overlays from the server](#opening-overlays-from-the-server)

---

## Layer terminology

Unpoly supports a **stack of layers**:

- **Root layer** — the main page (`up.layer.root`)
- **Overlay** — any layer above root (modal, drawer, popup, cover)
- **Current layer** — the layer the user is interacting with (`up.layer.current`)
- **Front layer** — topmost layer (`up.layer.front`)

Layers are isolated: links and forms in an overlay only update that overlay by default.

---

## Opening overlays

**From a link:**
```html
<a href="/menu" up-layer="new">Open menu</a>
```

**From a form (successful submission opens overlay):**
```html
<form action="/submit" up-submit up-layer="new">...</form>
```

**Failed submission only:**
```html
<form action="/submit" up-submit up-fail-layer="new">...</form>
```

**From local content (no request):**
```html
<a up-layer="new popup" up-content="<p>Helpful instructions</p>">Help</a>

<!-- Reference a template in the current page -->
<a up-layer="new popup" up-content="#help-template">Help</a>
<template id="help-template"><p>Help content</p></template>
```

**From JS:**
```js
// Returns a promise for the new up.Layer
let layer = await up.layer.open({ url: '/users/new' })

// With specific mode
let layer = await up.layer.open({ url: '/menu', mode: 'drawer' })

// Ask returns a promise that resolves with the accepted value
let address = await up.layer.ask({ url: '/address-picker' })
```

---

## Overlay modes

| Mode | Description |
|------|-------------|
| `modal` | Centered dialog (default) |
| `drawer` | Slide-in panel from the side |
| `popup` | Small floating box (anchored to origin) |
| `cover` | Full-page overlay |

**Set mode:**
```html
<a href="/menu" up-layer="new drawer">Open drawer</a>

<!-- Shorthand: append mode to up-layer value -->
<a href="/menu" up-layer="new" up-mode="drawer">Open drawer</a>
```

**From JS:**
```js
up.layer.open({ url: '/menu', mode: 'drawer' })
```

**Change default mode:**
```js
up.layer.config.mode = 'drawer'
```

---

## Targeting layers

By default, interactions in a layer target that layer.
Override with `[up-layer]`:

```html
<!-- Link in overlay that updates the root layer -->
<a href="/dashboard" up-layer="root" up-target=".content">Back to root</a>

<!-- Update parent layer -->
<a href="/items" up-layer="parent" up-target=".item-list">Refresh list</a>
```

Layer values: `'current'`, `'root'`, `'parent'`, `'closest'`, `'new'`, overlay mode name, or a number (0 = root).

**`[up-peel]`** — close overlays above the target layer before updating:
```html
<a href="/home" up-target=".content" up-layer="root" up-peel>Go home</a>
```

---

## Close conditions and subinteractions

When opening an overlay, define when it should automatically close:

```html
<!-- Close when user navigates to an accepted URL pattern -->
<a href="/users/new"
   up-layer="new modal"
   up-accept-location="/users/:id"
   up-on-accepted="location.reload()">
  New user
</a>
```

| Attribute | Description |
|-----------|-------------|
| `[up-accept-location]` | URL pattern that closes and accepts the overlay |
| `[up-dismiss-location]` | URL pattern that closes and dismisses the overlay |
| `[up-on-accepted]` | JS to run when overlay is accepted (has `value`, `layer`) |
| `[up-on-dismissed]` | JS to run when overlay is dismissed |

**Common acceptance callbacks:**

```html
<!-- Reload the opener layer's main target -->
<a href="/items/new"
   up-layer="new"
   up-accept-location="/items/:id"
   up-on-accepted="up.reload('.item-list')">
  Add item
</a>

<!-- Add an option to a select, then set its value -->
<a href="/tags/new"
   up-layer="new"
   up-accept-location="/tags/:id"
   up-on-accepted="
     up.element.affix(up.fragment.get('#tag-select'), 'option', { value: value.id, text: value.name });
     document.querySelector('#tag-select').value = value.id
   ">
  New tag
</a>
```

**Await subinteraction from JS:**
```js
let layer = up.layer.open({ url: '/address-picker' })
let address = await layer  // resolves when accepted, rejects when dismissed
```

---

## Accepting and dismissing

**From a link inside the overlay:**
```html
<a up-accept>Save & close</a>
<a up-dismiss>Cancel</a>

<!-- With a value -->
<a up-accept="{ id: 5, name: 'Alice' }">Select Alice</a>
```

**From the server response header:**
```http
X-Up-Accept-Layer: { "id": 5 }
X-Up-Dismiss-Layer: null
```

**From JS inside the overlay:**
```js
up.layer.accept({ id: 5 })
up.layer.dismiss()

// Access accepted value in parent:
let value = await up.layer.open({ url: '/picker' })
// value is what was passed to up.layer.accept()
```

**Close button:** Unpoly automatically adds a close button to overlays.
Dismiss the layer when clicking the backdrop:
```js
up.layer.config.modal.dismissable = true  // default
up.layer.config.popup.dismissable = ['button', 'key', 'outside-click']
```

---

## Layer context

Each layer has isolated key/value **context** (like a per-layer session):

```html
<!-- Set context when opening overlay -->
<a href="/projects" up-layer="new" up-context='{ "projectId": 5 }'>
  Open project
</a>
```

**Read context in JS:**
```js
up.layer.current.context  // { projectId: 5 }
```

**Set context from server:**
```http
X-Up-Context: { "role": "admin" }
```

**Access context in Rails:**
```ruby
up.context[:project_id]  # requires unpoly-rails gem
```

Context is preserved across fragment updates within that layer.

---

## Working with layers from JS

```js
// Current layer
up.layer.current         // up.Layer object
up.layer.current.mode    // 'root', 'modal', 'drawer', etc.
up.layer.current.element // The layer's container element

// Layer stack
up.layer.count           // number of open layers
up.layer.root            // root layer
up.layer.front           // topmost layer
up.layer.get(0)          // layer by index (0 = root)

// Iterate layers
for (let layer of up.layer.stack) { }

// Check if a layer is open
up.layer.isOverlay()     // true if current is not root

// Targeting
up.render({ url: '/menu', layer: 'root' })
```

---

## Opening overlays from the server

Any response can force-open an overlay with a response header:

```http
Content-Type: text/html
X-Up-Open-Layer: {}
```

With options:
```http
X-Up-Open-Layer: { "target": "#menu", "mode": "drawer", "animation": "move-to-right" }
```

Useful for authentication flows — a protected page can redirect to a login overlay.
