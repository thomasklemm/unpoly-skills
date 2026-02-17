# Unpoly in Rails Views

## Table of contents
- [View helper syntax](#view-helper-syntax)
- [Turbo coexistence](#turbo-coexistence)
- [CSP setup](#csp-setup)
- [Global follow-all config](#global-follow-all-config)

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

Also add preloading and instant-follow globally for snappier navigation:

```js
up.link.config.preloadSelectors.push('a[href]')
up.link.config.instantSelectors.push('a[href]')
```
