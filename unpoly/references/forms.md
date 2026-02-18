# Forms

## Table of contents
- [Basic form submission](#basic-form-submission)
- [Validating after submission (server errors)](#validating-after-submission-server-errors)
- [Validating fields on change (up-validate)](#validating-fields-on-change-up-validate)
- [Reactive forms (up-watch, up-autosubmit)](#reactive-forms-up-watch-up-autosubmit)
- [Watch options](#watch-options)
- [Switching form state (up-switch)](#switching-form-state-up-switch)
- [Disabling forms during submission](#disabling-forms-during-submission)
- [Multi-step forms in overlays](#multi-step-forms-in-overlays)
- [Nested forms and up.hello()](#nested-forms-and-uphello)

---

## Basic form submission

Add `[up-submit]` to submit without a full page load:

```html
<form action="/users" up-submit>
  <input name="email">
  <button type="submit">Save</button>
</form>
```

The response replaces the form's target (by default the form itself or the main element).

**Set a custom target:**
```html
<form action="/search" up-submit up-target=".results">
  <input name="q">
  <button>Search</button>
</form>
```

**Failed responses** (non-2xx) update `[up-fail-target]` instead (defaults to the form element):
```html
<form action="/users" up-submit up-target=".main" up-fail-target="form">
```

---

## Validating after submission (server errors)

When a form has validation errors, the server should:
1. Respond with **HTTP 422** (Unprocessable Entity)
2. Re-render the form with error markup

```ruby
# Rails example
def create
  @user = User.new(user_params)
  if @user.save
    redirect_to @user
  else
    render :new, status: :unprocessable_entity  # ← important
  end
end
```

Unpoly detects the non-2xx status and renders the form fragment (not the main target).
No extra JS needed — error messages appear inside the re-rendered form.

---

## Validating fields on change (up-validate)

`[up-validate]` validates a field against the server as soon as the user blurs it:

```html
<form action="/users" up-submit>
  <fieldset>
    <label for="email" up-validate>E-mail</label>
    <input type="text" id="email" name="email">
  </fieldset>
  <fieldset>
    <label for="password" up-validate>Password</label>
    <input type="password" id="password" name="password">
  </fieldset>
  <button type="submit">Register</button>
</form>
```

What happens:
1. User blurs "email" input
2. Unpoly submits the form to the same action with `X-Up-Validate: email`
3. Server re-renders the form with validation feedback for that field
4. Unpoly updates only the changed `<fieldset>` (the **form group**)

**Server-side detection (Rails):**
```ruby
def create
  @user = User.new(user_params)
  if request.headers['X-Up-Validate']
    @user.valid?  # run validations without saving
    render :new, status: :unprocessable_entity
  elsif @user.save
    redirect_to @user
  else
    render :new, status: :unprocessable_entity
  end
end
```

**Validate on custom target:**
```html
<input name="email" up-validate=".email-group">
```

**Validate multiple fields together:**
```html
<input name="password" up-validate=".password-group">
<input name="password_confirmation" up-validate=".password-group">
```

**Mark explicit form groups with `[up-form-group]`:**

Unpoly looks for a "form group" element around the validated field — an ancestor element that
acts as the unit of replacement. By default Unpoly recognizes `<fieldset>`, `<label>`, and the
`<form>` itself (configured via `up.form.config.groupSelectors`). Use `[up-form-group]` to mark
a container explicitly as the form group:

```html
<!-- Table rows as form groups for nested forms -->
<tr up-form-group>
  <td><input name="item[name]" up-validate></td>
  <td><input name="item[price]" up-validate></td>
  <td><button type="button" data-action="nested-form#remove">Remove</button></td>
</tr>

<!-- Div wrapper as form group -->
<div up-form-group class="address-fields">
  <input name="address[street]" up-validate>
  <input name="address[city]" up-validate>
</div>
```

This is especially important in complex layouts (tables, custom card structures) where Unpoly
cannot infer the intended grouping boundary automatically.

**Scope `[up-validate]` to a specific form to avoid collisions:**

When multiple `<form>` tags may appear in a rendered page (e.g., a user-facing form plus a
logout form in the nav), set `[up-validate]` to the specific form's selector rather than leaving
it empty. Otherwise Unpoly may replace the wrong form element in the response:

```html
<!-- ❌ Risky if other forms exist in the response -->
<form class="registration-form" up-submit up-validate>

<!-- ✅ Scoped to this form's CSS class -->
<form class="registration-form" up-submit up-validate="form.registration-form">
```

---

## Reactive forms (up-watch, up-autosubmit)

### Dependent fields (up-watch)

Update part of the form when a field changes:

```html
<form action="/invoices/new">
  <!-- When country changes, reload the VAT rate field -->
  <select name="country" up-watch up-target=".vat-group">
    <option>Germany</option>
    <option>France</option>
  </select>

  <div class="vat-group">
    <select name="vat_rate">
      <option>19%</option>
    </select>
  </div>
</form>
```

The server receives the full form params and responds with an updated fragment.

**Multiple dependent fragments:**
```html
<select name="country" up-watch up-target=".vat-group, .shipping-options">
```

**Watch a field and call a callback:**
```js
up.watch('input[name=country]', { delay: 300 }, function(value, name) {
  console.log(`${name} changed to ${value}`)
})
```

### Auto-submit (up-autosubmit)

Submit the whole form whenever any field changes:

```html
<form action="/products" up-autosubmit up-target=".product-list">
  <select name="category">...</select>
  <input name="price_max" type="range">
</form>
```

Attach to specific field to submit on that field's change only:
```html
<form action="/products">
  <input name="search" up-autosubmit up-target=".results">
</form>
```

---

## Watch options

| Attribute | Description |
|-----------|-------------|
| `[up-watch-delay]` | Debounce delay in ms before submitting (default 0) |
| `[up-watch-event]` | DOM event to watch (default `'input'` for text, `'change'` for others) |
| `[up-watch-disable]` | Disable fields while reloading (`true`, `false`, or CSS selector) |
| `[up-watch-feedback]` | Show loading state while reloading |

```html
<!-- Only submit 300ms after the user stops typing -->
<input name="search" up-autosubmit up-watch-delay="300" up-target=".results">
```

From JS:
```js
up.watch('input[name=search]', { delay: 300, event: 'input' }, handler)
```

---

## Switching form state (up-switch)

`[up-switch]` controls show/hide/enable/disable of other elements based on a field's value.
This is for **client-side** light effects — use `[up-watch]` for server-rendered changes.

**Show/hide elements:**
```html
<select name="level" up-switch=".level-dependent">
  <option value="beginner">Beginner</option>
  <option value="expert">Expert</option>
</select>

<div class="level-dependent" up-show-for="beginner">Shown for beginners only</div>
<div class="level-dependent" up-hide-for="beginner">Hidden for beginners</div>
<div class="level-dependent" up-show-for="intermediate expert">Multiple values</div>
```

**Enable/disable fields:**
```html
<select name="role" up-switch=".role-dependent">
  <option value="trainee">Trainee</option>
  <option value="manager">Manager</option>
</select>

<input class="role-dependent" name="department" up-enable-for="manager">
<input class="role-dependent" name="mentor" up-disable-for="manager">
```

**Special values:**
- `:blank` / `:present` — react to empty or non-empty values
- `:checked` / `:unchecked` — for checkboxes

**Custom switching effects** via `up:form:switch` event:
```js
up.on('up:form:switch', '[highlight-for]', (event) => {
  let value = event.target.getAttribute('highlight-for')
  event.target.style.outline = (event.field.value === value) ? '2px solid orange' : ''
})
```

---

## Disabling forms during submission

`[up-disable]` disables matched elements while the form request is in flight:

```html
<!-- Disable all fields + submit button -->
<form action="/checkout" up-submit up-disable>

<!-- Disable only the submit button -->
<form action="/checkout" up-submit up-disable="button[type=submit]">
```

From JS:
```js
up.submit(form, { disable: true })
```

Configure default disable behavior:
```js
up.form.config.submitButtons = 'button[type=submit], input[type=submit]'
```

---

## Multi-step forms in overlays

Open a sub-form in an overlay, then return a value to the parent form:

```html
<!-- Main form -->
<form action="/orders" up-submit>
  <input type="hidden" name="address_id" value="">
  <a href="/addresses/new"
     up-layer="new modal"
     up-accept-location="/addresses/:id"
     up-on-accepted="up.element.get('[name=address_id]').value = value.addressId">
    Add address
  </a>
  <button type="submit">Place order</button>
</form>
```

See [layers.md](layers.md) for subinteraction patterns.

**Nested forms (accepts_nested_attributes_for) + `up.hello()`:**

When adding nested form rows dynamically via JS (e.g., a Stimulus controller that clones a
template and appends it to the DOM), call `up.hello()` on the new element so Unpoly
compilers run on the inserted fields — enabling `[up-validate]` and other Unpoly attributes:

```js
// Stimulus controller adding a nested record row
add() {
  let content = this.templateTarget.innerHTML.replace(/NEW_RECORD/g, new Date().getTime())
  this.containerTarget.insertAdjacentHTML('beforeend', content)
  let newRow = this.containerTarget.lastElementChild
  up.hello(newRow)  // ← must call so [up-validate] etc. work in the new row
}
```

Without `up.hello()`, Unpoly compilers won't run on dynamically inserted content, and
attributes like `[up-validate]`, `[up-watch]`, and `[up-autosubmit]` will be silently ignored.
