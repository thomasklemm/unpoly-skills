# Forms

## Table of contents
- [Basic form submission](#basic-form-submission)
- [Validating after submission (server errors)](#validating-after-submission-server-errors)
- [Validating fields on change (up-validate)](#validating-fields-on-change-up-validate)
- [Reactive forms (up-watch, up-autosubmit)](#reactive-forms-up-watch-up-autosubmit)
- [Watch options](#watch-options)
- [Disabling forms during submission](#disabling-forms-during-submission)
- [Multi-step forms in overlays](#multi-step-forms-in-overlays)

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
