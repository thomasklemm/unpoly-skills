---
name: unpoly-rails
description: >
  Ruby on Rails integration for Unpoly. Use when working with the unpoly-rails gem or building
  Unpoly-powered Rails apps. Covers server-side helpers (up?, up.target, up.layer.accept,
  up.layer.dismiss, up.layer.open, up.validate?, up.cache.expire, up.context, up.emit,
  up.safe_callback, fresh_when, render_nothing), Rails view helpers (link_to, form_with,
  button_to with Unpoly attributes), flash messages with [up-hungry], Turbo coexistence
  (disabling Turbo Drive in Rails 7+), CSP setup with csp_meta_tag, and global follow-all config.
---

# Unpoly Rails

The `unpoly-rails` gem integrates Unpoly's [server protocol](https://unpoly.com/up.protocol) into
Rails, exposing helper methods in controllers, views, and helpers.

**Gem:** [`unpoly-rails`](https://github.com/unpoly/unpoly-rails) — tracks Unpoly 3.x

## Installation

```ruby
# Gemfile
gem 'unpoly-rails'
```

### Frontend assets via Asset Pipeline

```js
// application.js
//= require unpoly
```
```css
/* application.css
 *= require unpoly
 */
```

### Frontend assets via npm (esbuild, Webpacker, etc.)

```js
import 'unpoly/unpoly.js'
import 'unpoly/unpoly.css'
```

## Quick reference

| Helper | Purpose |
|--------|---------|
| `up?` | Is this an Unpoly fragment request? |
| `up.target` | CSS selector being updated (success) |
| `up.target = 'body'` | Override the render target |
| `up.target?('.sel')` | Is this selector targeted? |
| `up.fail_target` | CSS selector targeted for failed responses |
| `up.any_target?('.sel')` | Is selector targeted for success or failure? |
| `up.validate?` | Is this a form validation request? |
| `up.validate_names` | Field names that triggered validation |
| `up.layer.mode` | Layer mode: `"root"`, `"modal"`, `"drawer"`, etc. |
| `up.layer.overlay?` | Is the targeted layer an overlay? |
| `up.layer.root?` | Is the targeted layer the root layer? |
| `up.layer.accept(value)` | Accept the current overlay |
| `up.layer.dismiss(value)` | Dismiss the current overlay |
| `up.layer.open(...)` | Request frontend to open a new overlay |
| `up.layer.context` | Layer context hash |
| `up.context` | Alias for `up.layer.context` |
| `up.emit(type, props)` | Emit event on `document` after update |
| `up.layer.emit(type, props)` | Emit event on the targeted layer |
| `up.cache.expire` | Expire the client-side cache |
| `up.cache.evict` | Evict entries from the client-side cache |
| `up.title = 'Title'` | Set document title from server |
| `up.safe_callback("js()")` | CSP-safe inline callback with nonce |
| `up.render_nothing` | Render empty 204 response |
| `up.no_vary { }` | Read Unpoly headers without setting Vary |

## Reference files

Load when the user's question covers that topic:

- **[server-helpers.md](references/server-helpers.md)** — Full API reference: request inspection, response control, layer API, validation, events, cache control, context, CSP callbacks, failed forms, Vary headers, conditional GETs, common Rails patterns
- **[rails-integration.md](references/rails-integration.md)** — View helper syntax (`link_to`, `form_with`, `button_to`, field helpers), Turbo coexistence (disabling Turbo Drive in Rails 7+), CSP setup with `csp_meta_tag`, global follow-all config
