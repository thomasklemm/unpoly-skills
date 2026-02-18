# Unpoly Rails: Common Patterns

## Table of contents
- [Drawer overlay helper](#drawer-overlay-helper)
- [Event-driven subinteractions](#event-driven-subinteractions)
- [Authorization: overlay vs root layer](#authorization-overlay-vs-root-layer)

---

## Drawer overlay helper

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
<%= link_to contact.name, contact_path(contact), **open_link_in_drawer_attributes %>

<%# Custom size and callbacks %>
<%= link_to 'View deal', deal_path(deal),
  **open_link_in_drawer_attributes(size: 'full', on_accepted: :reload, on_dismissed: :reload) %>
```

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
