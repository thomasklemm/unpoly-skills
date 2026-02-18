# Unpoly Rails: Common Patterns

## Table of contents
- [Event-driven subinteractions](#event-driven-subinteractions)
- [Authorization: overlay vs root layer](#authorization-overlay-vs-root-layer)
- [Drawer overlay links](#drawer-overlay-links)

---

## Event-driven subinteractions

Use `[up-accept-event]` when an overlay should close upon a custom event emitted from the
server — useful when the standard redirect flow stays in the overlay, but the parent layer
needs to react to the newly created record's ID.

See [layers.md](../../unpoly/references/layers.md) for the HTML attribute reference and
pure client-side examples.

### Accept overlay on record save

The opener link declares which event closes the overlay and what to do on acceptance.
The server emits the event after a successful save:

```html
<!-- Opens a drawer to create a tag; reloads the tag list when server emits 'tag:created' -->
<a href="/tags/new"
   up-layer="new"
   up-mode="drawer"
   up-accept-event="tag:created"
   up-on-accepted="up.reload('.tag-list')">
  New tag
</a>
```

```ruby
# tags_controller.rb
def create
  @tag = Tag.new(tag_params)
  if @tag.save
    up.layer.emit('tag:created', id: @tag.id)  # accepted; value.id available in up-on-accepted
    redirect_to @tag                            # overlay continues to the tag show page
  else
    render :new, status: :unprocessable_entity
  end
end
```

### Create related record inline, inject FK into parent form

A common CRM pattern: a deal form needs a `contact_id` for a contact that doesn't exist yet.
Open a drawer to create the contact, emit an event with its ID, and re-validate the parent
form with the new value so the server-rendered form reflects the association.

```html
<!-- In the deal form: "New Contact" link next to a contact_id select -->
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

```ruby
# contacts_controller.rb
def create
  @contact = Contact.new(contact_params)
  if up.validate?
    @contact.valid?
    render :new, status: :unprocessable_entity
  elsif @contact.save
    up.layer.emit('contact:created', id: @contact.id)
    redirect_to @contact  # overlay shows the new contact's page; event fires acceptance
  else
    render :new, status: :unprocessable_entity
  end
end
```

After the overlay closes, `up.validate` re-submits the deal form with the new `contact_id`,
so the server-rendered form reflects the newly associated contact.

### Dismiss overlay on destructive action

Use `[up-dismiss-event]` to close an overlay when a destructive action completes on the server:

```html
<a href="/companies/1"
   up-layer="new"
   up-dismiss-event="company:destroyed"
   up-on-dismissed="up.reload('.company-list')">
  View company
</a>
```

```ruby
# companies_controller.rb
def destroy
  @company.destroy
  up.layer.emit('company:destroyed')
  redirect_to companies_path
end
```

---

## Authorization: overlay vs root layer

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
      # redirect_back_or_to validates the referrer is on the same host before using it.
      # Never use redirect_to(request.referrer || root_path) — the Referer header is
      # attacker-controllable and can be used to redirect users to external sites.
      redirect_back_or_to root_path, allow_other_host: false
    end
  end
end
```

---

## Drawer overlay links

### With an Unpoly macro (preferred)

`up.macro` is the idiomatic Unpoly approach: register a macro that expands a custom
`[data-drawer]` attribute into the full set of `up-*` attributes before any compiler runs.
This keeps drawer configuration in one JS place rather than a Ruby helper.

```js
// app/javascript/macros/drawer_link.js
up.macro('[data-drawer]', (link) => {
  link.setAttribute('up-layer', 'new')
  link.setAttribute('up-mode', 'drawer')
  link.setAttribute('up-position', link.dataset.drawerPosition || 'right')
  link.setAttribute('up-size', link.dataset.drawerSize || 'large')
})
```

Use in any ERB template — no helper, no splat:

```erb
<%# Default drawer (large, right) %>
<%= link_to contact.name, contact_path(contact), data: { drawer: true } %>

<%# Full-width drawer %>
<%= link_to 'View deal', deal_path(deal), data: { drawer: true, drawer_size: 'full' } %>

<%# Still composable with up-* attributes directly %>
<%= link_to 'New contact', new_contact_path,
  data: { drawer: true },
  'up-accept-event': 'contact:created',
  'up-on-accepted': 'up.reload()' %>
```

> Rails converts `drawer_size:` to `data-drawer-size` on the element, which JavaScript reads
> as `link.dataset.drawerSize` (camelCase). This is standard Rails/browser behaviour.

### With a Ruby view helper (alternative)

If you prefer to keep everything in Ruby or aren't using a JS bundler, a helper that returns
an attribute hash works well too:

```ruby
# app/helpers/overlay_helper.rb
module OverlayHelper
  def drawer_link_attributes(size: 'large', position: 'right', on_accepted: :reload)
    {
      'up-layer': 'new',
      'up-mode': 'drawer',
      'up-position': position,
      'up-size': size,
      'up-on-accepted': on_accepted == :reload ? 'up.reload()' : on_accepted
    }
  end
end
```

```erb
<%= link_to contact.name, contact_path(contact), **drawer_link_attributes %>
<%= link_to 'View deal', deal_path(deal), **drawer_link_attributes(size: 'full') %>
```
