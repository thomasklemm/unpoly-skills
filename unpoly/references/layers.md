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

> **Gotcha:** When no `[up-target]` is given, Unpoly looks for `:main` in the overlay response.
> `:main` matches (in order): an `[up-main]` attribute, a `<main>` element, or the layer's
> topmost swappable element. Without any of these, Unpoly throws:
> `up.CannotMatch: Could not find common target`.
>
> Add `[up-main]` to the outer wrapper of views that can be opened as overlays:
>
> ```erb
> <% if up.layer.overlay? %>
>   <div class="p-6" up-main>  <%# up-main makes this the overlay's :main target %>
>     …modal content…
>   </div>
> <% else %>
>   <main>…full page layout…</main>
> <% end %>
> ```

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

Layer values: `'current'`, `'root'`, `'parent'`, `'closest'`, `'origin'`, `'front'`, `'new'`, `'swap'`, `'shatter'`, overlay mode name, or a number (0 = root).

**`[up-layer="shatter"]`** — dismiss all overlays, then open from root:
```html
<a href="/home" up-layer="shatter">Start over</a>
```

**`[up-layer="swap"]`** — replace the current overlay with a new one instead of stacking:

```html
<!-- In a project overlay: navigate to the company overlay, replacing this one -->
<a href="/companies/1" up-layer="swap">View company</a>
```

Useful for "lateral" navigation within an overlay context — the user drills into a related record
without adding an extra layer to the stack. Combined with `[up-dismiss-location]` to auto-close
when the user navigates back to an index:

```html
<a href="/companies/1"
   up-layer="swap"
   up-dismiss-location="/projects">
  View company
</a>
```

**`[up-dismiss-location]`** — dismiss the overlay when navigation inside it reaches a URL pattern:

```html
<!-- Overlay auto-dismisses when the user navigates to /companies -->
<a href="/companies/new"
   up-layer="new modal"
   up-dismiss-location="/companies">
  New company
</a>
```

Unlike `[up-accept-location]`, `[up-dismiss-location]` dismisses the layer (no accepted value),
which triggers `[up-on-dismissed]` on the opener.

**`[up-peel]`** — close overlays above the target layer before updating:
```html
<a href="/home" up-target=".content" up-layer="root" up-peel>Go home</a>
```

**`[up-main]` — declare per-mode main targets on page elements:**

Mark a page element as the main target for a specific overlay mode. Overlays use the
`:main` selector as their default target. You can declare additional selectors per mode:

```html
<!-- This column is the main target when a drawer overlay renders content -->
<div up-main="drawer" class="content-column">
  ...
</div>
```

Or configure globally in JS for all overlays / a specific mode:
```js
up.layer.config.overlay.mainTargets.push('.layout--content')  // all overlays
up.layer.config.drawer.mainTargets.push('.layout--content')   // drawers only
up.layer.config.popup.mainTargets.push('.layout--content')    // popups only
```

This lets drawer/popup responses target `.layout--content` without needing `[up-target]` on
every link. The full layout is still rendered on direct page loads; only the content column
is swapped when the page is opened inside an overlay.

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
| `[up-accept-location]` | URL pattern that closes and accepts the overlay (`$id` and `:id` captures become `value`) |
| `[up-dismiss-location]` | URL pattern that closes and dismisses the overlay |
| `[up-accept-event]` | Event name that closes and accepts the overlay (with event payload as value) |
| `[up-dismiss-event]` | Event name that closes and dismisses the overlay |
| `[up-on-accepted]` | JS to run when overlay is accepted (has `value`, `layer`) |
| `[up-on-dismissed]` | JS to run when overlay is dismissed |

**Location-based acceptance (navigation flow):**

The `:id` and `$id` wildcards both capture a URL segment and pass it as `value.id` in
`[up-on-accepted]`. `$id` is handy when using Rails route helpers to generate the pattern:

```erb
<%# Rails: $id works with route helpers — company_path('$id') → '/companies/$id' %>
<%= link_to 'New company', new_company_path,
  'up-layer': 'new',
  'up-accept-location': company_path('$id'),
  'up-on-accepted': "up.reload('#companies')" %>
```

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

**Event-based acceptance (emit from server controller):**

Use `[up-accept-event]` when the overlay should close upon an event emitted by the server — useful
when the standard redirect flow stays in the overlay (e.g., `redirect_to @new_record`), but you
want the parent layer to react to the newly created record's ID.

```html
<!-- Link opens a drawer to create a tag; closes when server emits 'tag:created' -->
<a href="/tags/new"
   up-layer="new"
   up-mode="drawer"
   up-accept-event="tag:created"
   up-on-accepted="up.reload('.tag-list')">
  New tag
</a>
```

The `value` in `[up-on-accepted]` is the emitted DOM event object — the keyword args passed to
`up.layer.emit` are merged as event properties, so `value.id` returns `42`.

**Event-based dismissal:**
```html
<!-- Overlay watching for a destructive event -->
<a href="/company/1"
   up-layer="new"
   up-dismiss-event="company:destroyed"
   up-on-dismissed="up.reload('.company-list')">
  View company
</a>
```

**Create related record in overlay, then validate parent form with new ID:**

A common pattern when a parent form needs a foreign key for a record that doesn't exist yet.
Open a drawer to create the record, emit an event with its ID, and inject the ID as a
validation param so the parent form re-validates with the new value.

```html
<!-- In the parent form: "New Contact" button next to a contact_id select -->
<a href="/contacts/new"
   up-layer="new"
   up-mode="drawer"
   up-size="grow"
   up-position="right"
   up-accept-event="contact:created"
   up-on-accepted="up.validate('form.deal-form', {
     params: { 'deal[contact_id]': value.id }
   })">
  + New Contact
</a>
```

For the Rails controller implementation of these patterns, see
[patterns.md](../../unpoly-rails/references/patterns.md) in the unpoly-rails skill.

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
up.layer.current.location // Current layer URL
up.layer.current.context // Layer context object
up.layer.current.parent  // Parent layer reference

// Layer stack
up.layer.count           // number of open layers
up.layer.root            // root layer
up.layer.front           // topmost layer
up.layer.overlays        // array of all overlay layers
up.layer.get(0)          // layer by index (0 = root)
up.layer.get('front')    // by reference string

// Iterate layers
for (let layer of up.layer.stack) { }

// Check if a layer is open
up.layer.isOverlay()     // true if current is not root
up.layer.isRoot()        // true if current is root
up.layer.isFront()       // true if current is topmost

// Dismiss all overlays at once
up.layer.dismissOverlays()

// Append element to current layer
up.layer.affix('.flash', { text: 'Hello' })

// Containment check
up.layer.contains(element)

// Targeting
up.render({ url: '/menu', layer: 'root' })
```

**Config:**
```js
up.layer.config.mode = 'modal'          // default overlay mode
up.layer.config.foreignOverlaySelectors = ['dialog']  // non-Unpoly overlays to ignore

// Per-mode visual config (all inherit from config.overlay):
up.layer.config.overlay.openAnimation    // default open animation
up.layer.config.overlay.closeAnimation   // default close animation
up.layer.config.overlay.history          // whether overlay updates history
up.layer.config.overlay.trapFocus        // trap keyboard focus in overlay
up.layer.config.modal.backdrop = true    // modal has a backdrop
up.layer.config.modal.size = 'medium'    // overlay size: 'small', 'medium', 'large', 'grow', 'full'
up.layer.config.drawer.position = 'left' // drawer position: 'left' or 'right'
up.layer.config.popup.align = 'left'     // popup alignment: 'left' or 'right'
up.layer.config.popup.trapFocus = false  // popups don't trap focus by default
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
