# Unpoly in Rails Views

## Table of contents
- [View helper syntax](#view-helper-syntax)
- [Flash messages](#flash-messages)
- [Turbo coexistence](#turbo-coexistence)
- [CSP setup](#csp-setup)
- [Global follow-all config](#global-follow-all-config)
- [Drawer helper for consistent overlay links](#drawer-helper-for-consistent-overlay-links)
- [Migration: unpoly-migrate.js](#migration-unpoly-migratejs)

---

## View helper syntax

### `link_to`

Pass Unpoly attributes directly in the options hash:

```erb
<%= link_to 'Users', users_path, 'up-follow': true, 'up-target': '#content' %>
<%= link_to 'New user', new_user_path, 'up-layer': 'new modal' %>
<%= link_to 'Dashboard', root_path, 'up-follow': true, 'up-preload': true, 'up-instant': true %>
```

### `form_with` and `form_for`

Unpoly attributes must go inside the `html:` option — not directly on `form_with`:

```erb
<%# Correct — attributes inside html: %>
<%= form_with model: @user, html: { 'up-submit': true, 'up-validate': true, 'up-disable': true } do |f| %>
  <%= f.text_field :email %>
  <%= f.submit 'Save' %>
<% end %>

<%# Wrong — attributes at the top level are ignored by Rails for HTML output %>
<%= form_with model: @user, 'up-submit': true do |f| %>
```

### `form_tag`

```erb
<%# Search form that updates a fragment without full navigation %>
<%= form_tag users_path, method: :get, 'up-target': '#users', 'up-autosubmit': true do %>
  <%= text_field_tag :query, params[:query], placeholder: 'Search…' %>
<% end %>
```

### `button_to`

`button_to` generates a `<form>`. Unpoly attributes must go in the `form:` option:

```erb
<%# Correct — form-level Unpoly attributes in form: %>
<%= button_to 'Delete', user,
  method: :delete,
  form: { 'up-follow': true, 'up-confirm': 'Delete this user?' } %>

<%# Wrong — these become attributes on the <button>, not the <form> %>
<%= button_to 'Delete', user, method: :delete, 'up-follow': true %>
```

### Destroy actions: prefer `button_to` over `link_to method: :delete`

In Rails 7 (without `rails-ujs`), `link_to 'Delete', resource, method: :delete` generates
a `<form>` wrapper. Unpoly won't follow it automatically unless the form has `up-follow`.

`button_to` with `form:` is cleaner and explicit:

```erb
<%= button_to 'Delete', @company,
  method: :delete,
  form: {
    'up-follow': true,
    'up-confirm': 'Really delete?',
    'up-preview': 'btn-spinner'
  },
  class: 'btn btn-danger' %>
```

### Field helpers with `[up-validate]`

Add `[up-validate]` at the form level to validate all fields on blur, or per-field:

```erb
<%# Form-level: validates any field on blur %>
<%= form_with model: @user, html: { 'up-validate': true } do |f| %>
  <%= f.text_field :email %>
  <%= f.text_field :username %>
<% end %>

<%# Per-field: only validates email on blur %>
<%= form_with model: @user do |f| %>
  <%= f.text_field :email, 'up-validate': true %>
  <%= f.text_field :username %>
<% end %>
```

### Boolean attributes

Rails renders `'up-follow': true` as `up-follow="up-follow"`, which Unpoly accepts as truthy.
You can also use an empty string: `'up-follow': ''`.

Both are correct:
```erb
<%= link_to 'Users', users_path, 'up-follow': true %>
<%= link_to 'Users', users_path, 'up-follow': '' %>
```

### `data:` shorthand does NOT work for `up-*`

Rails' `data:` hash renders as `data-*` attributes. Never use it for Unpoly:

```erb
<%# Wrong — renders as data-up-follow="true", which Unpoly ignores %>
<%= link_to 'Users', users_path, data: { up: { follow: true } } %>

<%# Correct — string keys render as up-follow="up-follow" %>
<%= link_to 'Users', users_path, 'up-follow': true %>
```

---

## Flash messages

With fragment updates, the full page isn't reloaded, so flash messages rendered in the
layout won't appear unless their fragment is also updated. The standard Rails pattern is
to put flash in its own partial with `[up-hungry]`, which tells Unpoly to update it
whenever it appears in any response — regardless of what fragment was targeted.

**Layout:**
```erb
<%# app/views/layouts/application.html.erb %>
<%= render 'shared/flash' %>
```

**Partial (`app/views/shared/_flash.html.erb`):**
```erb
<div id="flash" up-hungry>
  <% flash.each do |type, message| %>
    <div class="alert alert-<%= type %>"><%= message %></div>
  <% end %>
</div>
```

Because the partial has `[up-hungry]`, Unpoly updates `#flash` on every response that
includes it — including fragment responses that target something else entirely.

To include the flash partial in fragment responses without rendering a full layout, render
it alongside your main content. Rails does this automatically when using `respond_to` or
the layout is rendered as part of the fragment. Alternatively, include the flash in your
`:main` target or any frequently-updated fragment.

**Controller flash works normally — no changes needed:**
```ruby
def create
  if @post.save
    redirect_to @post, notice: 'Post created'  # flash[:notice] works as usual
  else
    render :new, status: :unprocessable_entity
  end
end
```

---

## Turbo coexistence

**Rails 7+ ships with Turbo (via `turbo-rails`) by default.** Running Turbo and Unpoly
together causes conflicts: both intercept link clicks and form submissions, leading to
double-handling, broken history, and unpredictable rendering.

**The solution is to disable Turbo when using Unpoly.**

### Option 1: Remove `turbo-rails` from the Gemfile (recommended for new apps)

```ruby
# Gemfile — remove or comment out:
# gem 'turbo-rails'
```

Also remove from `application.js`:
```js
// Remove:
import "@hotwired/turbo-rails"
```

### Option 2: Disable Turbo drive globally (keep gem for Action Cable/Strada)

If you need `turbo-rails` for Turbo Streams or Strada but not Turbo Drive:

```js
// application.js
import { Turbo } from "@hotwired/turbo-rails"
Turbo.session.drive = false
```

Or in HTML:
```html
<body data-turbo="false">
```

### Stimulus is fine

Stimulus controllers are independent of Turbo Drive and work alongside Unpoly without
any issues.

---

## CSP setup

If your app uses a Content Security Policy with `script-src` nonces, add the nonce meta
tag to your layout so Unpoly can rewrite fragment nonces and run `[up-on-...]` callbacks:

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%= csp_meta_tag %>
  ...
</head>
```

Then use `up.safe_callback` in views for inline `[up-on-...]` callbacks:

```erb
<%= link_to 'New', new_user_path,
  'up-layer': 'new',
  'up-on-accepted': up.safe_callback("up.reload('#users')") %>
```

See [server-helpers.md](server-helpers.md) for the `up.safe_callback` server-side helper.

---

## Global follow-all config

Rather than adding `[up-follow]` and `[up-submit]` to every link and form, configure
Unpoly to follow all links and submit all forms by default in `application.js`:

```js
// Make all links navigate as fragment updates
up.link.config.followSelectors.push('a[href]')

// Make all forms submit as fragment updates
up.form.config.submitSelectors.push('form')
```

Opt specific links or forms out with `[up-follow="false"]`:

```erb
<%= link_to 'Download PDF', report_path(format: :pdf), 'up-follow': false %>
<%= form_with url: external_url, html: { 'up-follow': false } do |f| %>
```

To follow only internal links (not external URLs with `://`), scope the selector:

```js
// Only follow internal links — leaves mailto:, external URLs, etc. unaffected
up.link.config.followSelectors.push('a[href]:not([href*="://"])')
```

Also add preloading and instant-follow globally for snappier navigation:

```js
up.link.config.preloadSelectors.push('a[href]')
up.link.config.instantSelectors.push('a[href]')
```

---

## Drawer helper for consistent overlay links

Extract a view helper to consistently open links in a drawer overlay with standard options:

```ruby
# app/helpers/overlay_helper.rb
module OverlayHelper
  def open_link_in_drawer_attributes(
    layer: 'new', mode: 'drawer', size: 'large',
    on_accepted: :reload, on_dismissed: nil,
    preload: false
  )
    attrs = {
      'up-layer': layer,
      'up-mode': mode,
      'up-size': size
    }

    attrs['up-position'] = 'right' if mode.to_s == 'drawer'

    if on_accepted.present?
      attrs['up-on-accepted'] = on_accepted == :reload ? 'up.reload()' : on_accepted
    end

    if on_dismissed.present?
      attrs['up-on-dismissed'] = on_dismissed == :reload ? 'up.reload()' : on_dismissed
    end

    if preload
      attrs['up-preload'] = ''
      attrs['up-instant'] = ''
    end

    attrs
  end
end
```

Use in views:
```erb
<%# Opens in a right-hand drawer; reloads parent on accept %>
<%= link_to record.name, record_path(record), **open_link_in_drawer_attributes %>

<%# Custom size and callbacks %>
<%= link_to 'View prescription', prescription_path(prescription),
  **open_link_in_drawer_attributes(size: 'full', on_accepted: :reload, on_dismissed: :reload) %>
```

---

## Migration: `unpoly-migrate.js`

When upgrading Unpoly across major version ranges, include `unpoly-migrate.js` alongside
`unpoly.js` to polyfill deprecated APIs and keep your existing HTML/JS working:

```js
import 'unpoly/unpoly.js'
import 'unpoly/unpoly-migrate.js'  // polyfills renamed/removed APIs
import 'unpoly/unpoly.css'
```

**Notable 3.11 breaking changes to be aware of:**

- Scripts (`<script>` tags) in updated fragments no longer execute by default. If your app
  relies on inline scripts in fragment responses, restore the old behavior:
  ```js
  up.fragment.config.runScripts = true
  ```
- `[up-scroll='reset']` was renamed to `[up-scroll='top']`. `unpoly-migrate.js` polyfills
  this automatically.
- `up.link.config.followSelectors` no longer treats `[up-href]`-only elements as followable
  by default — they now require `[up-follow]` too. `unpoly-migrate.js` polyfills this.
