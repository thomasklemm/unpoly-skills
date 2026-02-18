# unpoly-rails Server Helpers

## Table of contents
- [Detecting an Unpoly request](#detecting-an-unpoly-request)
- [Changing the render target](#changing-the-render-target)
- [Rendering nothing](#rendering-nothing)
- [Setting the document title](#setting-the-document-title)
- [Layer API](#layer-api)
- [Opening a new overlay](#opening-a-new-overlay)
- [Emitting frontend events](#emitting-frontend-events)
- [Form validation](#form-validation)
- [Context](#context)
- [Cache control](#cache-control)
- [Conditional GET (polling/reload)](#conditional-get-pollingreload)
- [CSP-safe callbacks](#csp-safe-callbacks)
- [Common Rails patterns](#common-rails-patterns)
- [Vary headers](#vary-headers)
- [Failed form submissions](#failed-form-submissions)

---

## Detecting an Unpoly request

```ruby
up?  # => true or false
```

Retrieve and test the CSS selector being updated:

```ruby
up.target           # => '.content'
up.target?('.sidebar')     # Is this selector being targeted?
up.fail_target             # Selector targeted for failed responses (4xx/5xx)
up.fail_target?('.errors') # Is this selector targeted on failure?
up.any_target?('.flash')   # Targeted for success OR failure?
```

> **Gotcha:** `up.target?` always returns `true` for non-Unpoly requests (regular page loads,
> tests). This is by design — a full-page request is assumed to include any fragment.
> Always guard with `up?` first when the check should only apply to fragment updates:
>
> ```ruby
> # Wrong — always true for regular requests, breaks tests
> if up.target?('#sidebar')
>
> # Correct
> if up? && up.target?('#sidebar')
> ```

---

## Changing the render target

Override which fragment the frontend will update:

```ruby
unless signed_in?
  up.target = 'body'
  render 'sign_in'
end
```

The new target applies to both successful and failed responses.

---

## Rendering nothing

When the action's side effects are complete (e.g., closing an overlay) and there's nothing to render:

```ruby
head :no_content
```

> **Deprecated:** `up.render_nothing` is deprecated as of unpoly-rails 3.12. Use `head(:no_content)` instead.
> Calling `up.render_nothing` now emits an `ActiveSupport::Deprecation` warning.

Typical pattern for overlay acceptance:

```ruby
def create
  @note = Note.new(note_params)
  if @note.save
    if up.layer.overlay?
      up.layer.accept(@note.id)
      head :no_content
    else
      redirect_to @note
    end
  else
    render 'form', status: :unprocessable_entity
  end
end
```

---

## Setting the document title

Force Unpoly to update the browser title (useful when skipping `<head>` in fragment responses):

```ruby
up.title = 'Users – My App'
```

---

## Layer API

Access properties of the layer being targeted by the current request.

> Note: A request may target different layers for successful (2xx) vs. failed (4xx/5xx) responses.
> Use `up.fail_layer` to inspect the failure target layer.

### Detecting the targeted layer

```ruby
up.layer.mode       # => "root", "modal", "drawer", "popup", "cover"
up.layer.root?      # => true if root layer
up.layer.overlay?   # => true if any overlay
```

### Accepting / dismissing overlays

```ruby
up.layer.accept        # Accept overlay (close with success signal)
up.layer.accept(@user) # Accept with a value (passed to up-on-accepted callback)

up.layer.dismiss         # Dismiss overlay (close with cancel signal)
up.layer.dismiss(:cancel) # Dismiss with a value
```

After calling these, always render or redirect:

```ruby
up.layer.accept(@note.id)
head :no_content
```

### Fail layer helpers

```ruby
up.fail_layer.mode      # Mode of layer targeted for failed responses
up.fail_layer.root?
up.fail_layer.overlay?
up.fail_layer.context
```

### Origin layer

```ruby
up.origin_layer.mode    # Mode of the layer that caused the request
up.origin_layer.root?
up.origin_layer.overlay?
```

---

## Opening a new overlay

Request the frontend to render the response in a new overlay:

```ruby
up.layer.open
```

By default targets `:main`. Pass an explicit target or options:

```ruby
up.layer.open(target: '#overlay-content')
up.layer.open(mode: 'drawer', position: 'right')
```

---

## Emitting frontend events

Emit an event on the `document` after the fragment is updated:

```ruby
class UsersController < ApplicationController
  def show
    @user = User.find(params[:id])
    up.emit('user:selected', id: @user.id)
  end
end
```

Emit on the targeted layer instead:

```ruby
up.layer.emit('user:selected', id: @user.id)
```

---

## Form validation

Detect a validation request (from `[up-validate]`):

```ruby
up.validate?  # => true when Unpoly sends X-Up-Validate header
```

On validation, validate but do not save; re-render the form:

```ruby
def create
  @user = User.new(user_params)
  if up.validate?
    @user.valid?        # run validations, don't save
    render 'form'       # re-render with errors
  elsif @user.save
    redirect_to @user
  else
    render 'form', status: :unprocessable_entity
  end
end
```

Inspect which fields triggered validation:

```ruby
up.validate_names          # => ['email', 'password']
up.validate_name?('email') # => true
```

---

## Context

The context is a JSON object shared between the frontend and server, persisted across Unpoly
navigations (but cleared on full page loads). Each layer typically has its own context.

Read and write context values:

```ruby
up.context[:lives] = 3
up.context[:lives]         # => 3
up.context['lives']        # => 3 (string or symbol keys both work)
```

Delete a key:

```ruby
up.context.delete(:foo)
```

Replace the entire context:

```ruby
up.context.replace(JSON.parse(File.read('context.json')))
```

`up.context` is an alias for `up.layer.context`.

---

## Cache control

### Expiring cache entries

Mark cached responses as stale (Unpoly will revalidate them in the background):

```ruby
up.cache.expire           # Expire entire cache
up.cache.expire('/notes/*')  # Expire matching URL pattern
```

### Evicting cache entries

Remove entries from the cache entirely (no revalidation):

```ruby
up.cache.evict            # Evict entire cache
up.cache.evict('/notes/*')   # Evict matching URL pattern
```

---

## Conditional GET (polling/reload)

When Unpoly reloads or polls a fragment, it sends `If-Modified-Since` and `If-None-Match`
headers. Use Rails' conditional GET helpers to skip rendering when content hasn't changed:

```ruby
class MessagesController < ApplicationController
  def index
    @messages = current_user.messages.order(created_at: :desc)
    # Renders 304 Not Modified if ETag/Last-Modified match
    fresh_when(@messages)
  end
end
```

Or use `stale?` for more control:

```ruby
def index
  @messages = current_user.messages
  if stale?(@messages)
    render :index
  end
end
```

This saves server CPU and reduces responses to ~1 KB when content hasn't changed.

---

## CSP-safe callbacks

When your CSP disallows `eval()`, inline Unpoly callbacks like `[up-on-loaded]` will fail.
Use `up.safe_callback` to prefix the callback with the response's CSP nonce:

```ruby
link_to 'Click me', '/path',
  'up-follow': true,
  'up-on-loaded': up.safe_callback("alert('loaded')")
```

Also include the nonce meta tag in your layout `<head>`:

```erb
<%= csp_meta_tag %>
```

> Only works for `[up-on-...]` attributes. Not for native attributes like `[onclick]`.

---

## Vary headers

Accessing Unpoly request headers through helper methods automatically sets a `Vary` response
header, ensuring caches store separate responses per header value.

```ruby
# Automatically sets: Vary: X-Up-Mode
if up.layer.mode == 'modal'
  up.layer.accept
else
  redirect_to :show
end
```

To read an Unpoly header without affecting `Vary` (e.g., for logging):

```ruby
up.no_vary do
  Rails.logger.info("Unpoly mode: #{up.layer.mode}")
end
```

Accessing `response.headers[]` directly also never sets a `Vary` header.

---

## Failed form submissions

Unpoly detects failed form submissions by HTTP status code. Always render failed forms with
a non-200 status — `422 Unprocessable Entity` is conventional:

```ruby
def create
  @user = User.new(user_params)
  if @user.save
    redirect_to @user
  else
    render 'new', status: :unprocessable_entity
  end
end
```

Without a non-200 status, Unpoly treats the response as successful and won't show errors
in the form fragment.

---

## Automatic behaviors

`unpoly-rails` sets up the following automatically (no configuration required):

- **Redirect preservation** — `redirect_to` preserves Unpoly request/response headers across redirects
- **`X-Up-Location` echo** — Each response includes the request URL, so Unpoly detects redirects
- **`_up_method` cookie** — Lets Unpoly detect the HTTP method of the initial page load

---

## Common Rails patterns

### DRY validate + save across create and update

Extract a private method that handles `up.validate?`, save, and render in one place,
shared between `create` and `update`:

```ruby
class CompaniesController < ApplicationController
  def create
    build_company
    save_company(form: 'new')
  end

  def update
    load_company
    build_company
    save_company(form: 'edit')
  end

  private

  def save_company(form:)
    if up.validate?
      @company.valid?              # run validations, don't save
      render form                  # re-render form with errors
    elsif @company.save
      redirect_to @company, notice: 'Company saved'
    else
      render form, status: :unprocessable_entity
    end
  end
end
```

### Overlay auto-close via `up-accept-location` with Rails path helpers

Use `path_helper('$id')` as a URL pattern for `[up-accept-location]`. Unpoly treats `$id`
as a wildcard, so the overlay closes automatically when the browser lands on any matching URL
after a normal `redirect_to` — no `up.layer.accept` needed in the controller.

```erb
<%= link_to 'New company', new_company_path,
  'up-layer': 'new',
  'up-accept-location': company_path('$id'),
  'up-on-accepted': "up.reload('#companies')" %>
```

`company_path('$id')` generates `/companies/$id`. After `redirect_to @company`, Unpoly sees
the URL `/companies/42` matches the pattern and closes the overlay.

### Server-side event triggers overlay dismissal

Emit an event from the controller after a destructive action. Any open overlay watching
for that event with `[up-dismiss-event]` automatically closes.

Controller:
```ruby
def destroy
  @company.destroy
  up.layer.emit('company:destroyed')
  redirect_to companies_path
end
```

View (on the opener page):
```erb
<%= link_to company.name, company,
  'up-layer': 'new',
  'up-dismiss-event': 'company:destroyed',
  'up-on-dismissed': "up.reload('#companies')" %>
```

When the company is destroyed in any layer, the overlay watching for `company:destroyed`
closes and the list reloads — without the controller needing to know which overlays are open.

### Passing context to overlays from views

Use `.to_json` to pass a context hash to a new overlay:

```erb
<%= link_to 'Add project', new_project_path,
  'up-layer': 'new',
  'up-accept-location': project_path('$id'),
  'up-context': { from_company: true }.to_json %>
```

The overlay's layer context is then accessible on the server:

```ruby
up.context[:from_company]  # => true
```

Use this to adjust responses based on where an overlay was opened from.

### Emit event + accept overlay on save (event-driven subinteraction)

When a form is submitted inside an overlay and the parent layer needs the new record's ID
(e.g., to populate a foreign key select), emit an event with the ID and let `[up-accept-event]`
on the opener link close the overlay:

Controller:
```ruby
def create
  @patient = Patient.new(patient_params)
  if up.validate?
    @patient.valid?
    render :new, status: :unprocessable_entity
  elsif @patient.save
    # Emit event so any overlay listening with [up-accept-event] closes with this payload
    up.layer.emit('patient:created', id: @patient.id)
    redirect_to @patient  # normal redirect continues in the overlay
  else
    render :new, status: :unprocessable_entity
  end
end
```

Opener (parent form):
```erb
<%= link_to icon_tag(:new), new_patient_path,
  'up-layer': 'new',
  'up-mode': 'drawer',
  'up-accept-event': 'patient:created',
  'up-on-accepted': "up.validate('form.appointment-form', {
    params: { 'appointment[patient_id]': value.id }
  })" %>
```

`value` in `[up-on-accepted]` is the hash passed to `up.layer.emit` — here `{ id: 42 }`.
`up.validate` re-submits the form with the new `patient_id` so the server-rendered form
reflects the newly associated record.

This pattern works for any "create related record inline" flow — patients, tags, addresses, etc.

### Authorization concern: redirect vs overlay close

When authorization fails, check whether the request targets an overlay. If so, return
`head :no_content` rather than redirecting — a redirect inside an overlay would navigate
the overlay to the redirect target (e.g. sign-in page) instead of closing it cleanly.

`head :no_content` returns HTTP 204: Unpoly discards the empty response and leaves the
overlay unchanged. Note that any `flash` message set before a 204 response won't render
on that request — it carries over to the next full-page render.

```ruby
module DoesPunditAuthorization
  extend ActiveSupport::Concern
  include Pundit::Authorization

  included do
    rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized
  end

  private

  def user_not_authorized
    if up.layer.overlay?
      # HTTP 204: Unpoly discards the response, overlay stays open unchanged.
      # Don't set flash here — it won't render on a 204 response.
      head :no_content
    else
      flash[:alert] = 'You are not authorized to perform this action.'
      redirect_to(request.referrer || root_path)
    end
  end
end
```

### Guard ApplicationController callbacks with `up?`

Only run server-side side effects that are specific to Unpoly fragment requests:

```ruby
class ApplicationController < ActionController::Base
  after_action :set_cache_headers

  private

  def set_cache_headers
    if up?
      response.headers['Cache-Control'] = 'no-store'
    end
  end
end
```
