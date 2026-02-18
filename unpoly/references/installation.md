# Installing Unpoly

## Table of contents
- [CDN](#cdn)
- [npm with a bundler](#npm-with-a-bundler)
- [Ruby on Rails](#ruby-on-rails)

---

## CDN

The quickest way to get started — no build step required:

```html
<script src="https://cdn.jsdelivr.net/npm/unpoly@3.12.1/unpoly.min.js"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/unpoly@3.12.1/unpoly.min.css">
```

Load Unpoly before your own stylesheets and scripts. Unpoly boots automatically on
`DOMContentLoaded` and exposes its API as `window.up`.

**Manual boot** — to configure Unpoly before it boots, add `[up-boot="manual"]` to `<html>`:
```html
<html up-boot="manual">
```
Then configure and call `up.boot()` when ready:
```js
up.fragment.config.mainTargets.push('.my-main')
up.boot()
```

If you're migrating from an older version, also load the migration shim after Unpoly:

```html
<script src="https://cdn.jsdelivr.net/npm/unpoly@3.12.1/unpoly-migrate.min.js"></script>
```

See [migration.md](migration.md) for details on the upgrade workflow.

---

## npm with a bundler

Install the package:

```sh
npm install unpoly
```

Import in your application entry point (esbuild, webpack, Vite, jsbundling-rails):

```js
import 'unpoly/unpoly.js'
import 'unpoly/unpoly.css'
```

For migration from an older version, add the shim after the main import:

```js
import 'unpoly/unpoly.js'
import 'unpoly/unpoly-migrate.js'  // load after unpoly, before your app code
import 'unpoly/unpoly.css'
```

---

## Ruby on Rails

### Add the gem

`unpoly-rails` provides the server-side integration (request helpers, response helpers,
layer context, etc.) for any Rails app. Add it to your `Gemfile`:

```ruby
gem 'unpoly-rails'
```

Then `bundle install` and restart your server.

With the gem installed, `up` is available in controllers and views — see
[server-helpers.md](../../unpoly-rails/references/server-helpers.md) for the full API.

### Load the frontend

Choose the approach that matches your Rails asset setup:

#### Importmap (importmap-rails)

Pin Unpoly in `config/importmap.rb`:

```ruby
pin "unpoly", to: "https://cdn.jsdelivr.net/npm/unpoly@3.12.1/unpoly.min.js"
pin "unpoly-migrate", to: "https://cdn.jsdelivr.net/npm/unpoly@3.12.1/unpoly-migrate.min.js"
```

Import in `app/javascript/application.js`:

```js
import "unpoly"
// import "unpoly-migrate"  // uncomment when upgrading from an older version
```

Add the stylesheet to your layout:

```erb
<%= stylesheet_link_tag "https://cdn.jsdelivr.net/npm/unpoly@3.12.1/unpoly.min.css" %>
```

#### npm + bundler (jsbundling-rails / esbuild / Vite)

```sh
npm install unpoly
```

In `app/javascript/application.js`:

```js
import 'unpoly/unpoly.js'
import 'unpoly/unpoly.css'
```

#### Asset Pipeline (Sprockets)

The `unpoly-rails` gem adds Unpoly's frontend files to the asset pipeline automatically.

In `app/assets/javascripts/application.js`:

```js
//= require unpoly
//= require unpoly-migrate  // include when upgrading from an older version
```

In `app/assets/stylesheets/application.css`:

```css
/*
 *= require unpoly
 */
```
