# unpoly-rails: Gotchas & Lessons Learned

Real-world pitfalls encountered when building Rails apps with Unpoly.

## Table of contents
- [up-accept-location must not match the opening URL](#up-accept-location-must-not-match-the-opening-url)
- [up-hungry flash wiped by up-defer responses](#up-hungry-flash-wiped-by-up-defer-responses)
- [up-validate system test submit race](#up-validate-system-test-submit-race)

---

## up-accept-location must not match the opening URL

`[up-accept-location]` closes the overlay as soon as the browser URL matches the pattern.
If the pattern also matches the URL used to *open* the overlay, the overlay closes immediately
upon opening — before the user has done anything.

**Example of the bug:**

```erb
<%# BUG: /contacts/* matches /contacts/new the moment the overlay opens %>
<%= link_to 'New contact', new_contact_path,
  'up-layer': 'new',
  'up-accept-location': '/contacts/*' %>
```

**Fix option 1 — use the server-side accept pattern:**

Remove `[up-accept-location]` and call `up.layer.accept` from the controller after a
successful save. The server decides when to close, not a URL pattern:

```ruby
def create
  @contact = Contact.new(contact_params)
  if @contact.save
    if up.layer.overlay?
      up.layer.accept(contact_path(@contact))
      head :no_content
    else
      redirect_to contact_path(@contact)
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

Client-side URL patterns like `/contacts/*`, `/companies/*` or `/companies/$id`
(or `/companies/:id`) cannot distinguish between `/companies/new` and `/companies/123`:
they will match both the opening URL and the post-save URL and can still cause the
immediate-close bug described above. For this reason the server-side accept pattern
shown above is the reliable way to close the overlay after a successful save.

> Always verify that a URL pattern doesn't match the overlay's opening URL.

---

## up-hungry flash wiped by up-defer responses

> **Using `[up-flashes]`?** Unpoly 3.x's built-in `[up-flashes]` attribute automatically
> applies `up-keep` alongside `up-hungry`. When a subsequent `[up-defer]` response arrives
> with an empty `[up-flashes]`, `up-keep` causes Unpoly to preserve the existing flash
> container — including the displayed toast — rather than replace it. Apps using
> `[up-flashes]` are not affected by this gotcha. The pattern below applies to apps that
> manage their flash container with a plain `[up-hungry]` element.

When a page triggers multiple Unpoly requests in sequence (e.g. a modal creates a record,
which causes both a detail fragment update and a lazy `[up-defer]` panel to reload), the
flash is consumed by the first response. Subsequent `[up-defer]` responses have no flash,
and their empty `#flash[up-hungry]` element wipes the toast that was just shown.

**Root cause:** `[up-hungry]` replaces the flash element in the DOM whenever it appears
in *any* response. An `[up-defer]` response renders with an empty flash wrapper, which
overwrites the toast set by the earlier response.

**Fix:** Omit `#flash` entirely from Unpoly responses that have nothing to render, so the
existing DOM toast is left untouched. Always render the empty wrapper on full page loads
so `[up-hungry]` stays wired for the next real flash message:

```erb
<%# app/views/shared/_flash.html.erb %>
<% if flash.any? %>
  <div id="flash" up-hungry>
    <% flash.each do |type, message| %>
      <div class="alert alert-<%= type %>"><%= message %></div>
    <% end %>
  </div>
<% elsif !up? %>
  <div id="flash" up-hungry></div>  <%# full page load: keep wrapper wired in DOM %>
<% end %>
<%# Unpoly requests with no flash: omit entirely — leaves any existing toast untouched %>
```

---

## up-validate system test submit race

In Capybara system tests, clicking the submit button blurs the last focused field. If that
field has `[up-validate]`, this fires a validation request *concurrently* with the form
submission. Two races can occur:

1. **Typed value lost:** a validate re-render completes while the user (or test) is typing
   into a field, replacing the DOM and discarding the partial input.
2. **Flash wiped:** the validate response arrives after a successful save has set flash,
   re-rendering the form with an empty `#flash[up-hungry]` and wiping the success toast.

**Fix:** Before clicking submit, blur the active element and wait for the network to idle.
This drains any pending validation before the form submission begins:

```ruby
# In ApplicationSystemTestCase
def settle_form
  page.execute_script("document.activeElement.blur()")
  sleep 0.05  # give Unpoly a tick to queue the request
  Timeout.timeout(5) do
    loop do
      break unless page.evaluate_script("typeof up !== 'undefined' && up.network.isBusy()")
      sleep 0.05
    end
  end
end

# Usage in tests — call before every submit in forms that use up-validate
settle_form
click_button "Save"
```

For filling fields in sequence in a form with `[up-validate]`, also wait for idle after
each field focus to let the previous field's validation complete before typing:

```ruby
def fill_validated_field(label, value)
  find_field(label).click    # focus → blurs previous field → fires up-validate
  settle_form                 # wait for validate round-trip
  find_field(label).set(value)
end
```
