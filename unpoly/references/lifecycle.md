# Render Lifecycle & Events

## Table of contents
- [Lifecycle overview](#lifecycle-overview)
- [Running code after rendering](#running-code-after-rendering)
- [Key lifecycle events](#key-lifecycle-events)
- [Controlling the render process](#controlling-the-render-process)
- [Changing options before rendering](#changing-options-before-rendering)
- [Error handling](#error-handling)
- [Server integration headers (complete reference)](#server-integration-headers-complete-reference)

---

## Lifecycle overview

For every render pass, Unpoly follows this sequence:

1. **User interaction** (link click, form submit, `up.render()`)
2. **`up:link:follow`** or **`up:form:submit`** — preventable, options mutable
3. **Fetch** — request sent (or cache hit)
4. **`up:fragment:loaded`** — response received, preventable
5. **Compile** — compilers run on new elements
6. **`up:fragment:inserted`** — after insert, after compile
7. **Animation** — transition plays
8. **`up:fragment:rendered`** — after animation
9. *(Optional)* **Revalidation** — if cache expired, repeat from step 3

---

## Running code after rendering

**Await the render promise:**
```js
await up.render({ url: '/path', target: '.target' })
// Fragment is now inserted and compiled
```

**After postprocessing (animations, revalidation):**
```js
let job = up.render({ url: '/path', target: '.target' })
await job.finished
// All async postprocessing complete
```

**`{ onRendered }` callback — called after each render pass (initial + revalidation):**
```js
up.render({
  url: '/path',
  target: '.target',
  onRendered: (result) => {
    console.log('Rendered:', result.fragments)
  }
})
```

**Inspect render result:**
```js
let result = await up.render({ url: '/path', target: '.target' })
result.fragments      // Array of updated elements
result.layer          // Layer that was updated
result.response       // The up.Response object
```

---

## Key lifecycle events

### Fragment events

| Event | When | Preventable |
|-------|------|------------|
| `up:fragment:loaded` | Response received, before render | Yes |
| `up:fragment:inserted` | After insert, after compile | No |
| `up:fragment:rendered` | After animation completes | No |
| `up:fragment:destroyed` | Element removed from DOM | No |
| `up:fragment:keep` | Element is being kept (up-keep) | Yes |
| `up:fragment:poll` | Before polling reload | Yes |

### Link events

| Event | When | Preventable |
|-------|------|------------|
| `up:link:follow` | Link is about to be followed | Yes |
| `up:link:preload` | Link is about to be preloaded | Yes |

### Form events

| Event | When | Preventable |
|-------|------|------------|
| `up:form:submit` | Form is about to be submitted | Yes |
| `up:form:validate` | Form is about to validate | Yes |

### Layer events

| Event | When | Preventable |
|-------|------|------------|
| `up:layer:open` | Before opening overlay | Yes |
| `up:layer:opened` | After opening overlay | No |
| `up:layer:close` | Before closing overlay | Yes |
| `up:layer:closed` | After closing overlay | No |
| `up:layer:accept` | Layer is accepting with value | Yes |
| `up:layer:dismiss` | Layer is dismissing | Yes |

### Location events

| Event | When | Preventable |
|-------|------|------------|
| `up:location:changed` | URL changed in browser | No |
| `up:location:restore` | Back/forward navigation | Yes |

### Network events

| Event | When | Preventable |
|-------|------|------------|
| `up:network:offline` | Connection lost | No |
| `up:network:recover` | Connection restored | No |

**Listen to events:**
```js
up.on('up:fragment:inserted', '.result', function(event, element) {
  console.log('Result inserted:', element)
})

// Listen on a specific element
up.on(myElement, 'up:fragment:inserted', handler)

// One-time listener
up.once('up:fragment:inserted', handler)

// Remove listener
let off = up.on('up:fragment:inserted', handler)
off()  // Remove
```

---

## Controlling the render process

### Preventing events

```js
// Prevent a specific link from being followed
up.on('up:link:follow', function(event) {
  if (event.target.classList.contains('no-ajax')) {
    event.preventDefault()
  }
})
```

### Skipping a render pass

```js
// Skip rendering when server response hasn't changed
up.on('up:fragment:loaded', function(event) {
  if (event.response.headers['X-Content-Unchanged']) {
    event.skip()
  }
})
```

### Aborting requests

```js
// Abort all pending requests
up.network.abort()

// Abort requests targeting a fragment
up.network.abort({ target: '.content' })
```

---

## Changing options before rendering

Mutate `event.renderOptions` in lifecycle events to change render behavior:

```js
// Open all links inside a modal form in a new overlay
up.on('up:link:follow', function(event) {
  if (event.target.closest('form') && up.layer.isOverlay()) {
    event.renderOptions.layer = 'new'
  }
})

// Redirect authenticated links through login
up.on('up:link:follow', function(event) {
  if (event.target.hasAttribute('require-session') && !currentUser) {
    event.renderOptions.url = '/login?redirect=' + event.target.href
    event.renderOptions.layer = 'new'
  }
})

// Use custom animation for specific links
up.on('up:fragment:loaded', function(event) {
  if (event.response.url.includes('/admin')) {
    event.renderOptions.transition = 'cross-fade'
  }
})
```

---

## Error handling

Render promises reject on:
- HTTP error status (4xx, 5xx) — rejects with `up.Response`
- Missing target selector — rejects with `up.CannotMatch`
- Aborted request — rejects with `up.AbortError`
- Prevented event — rejects with `up.AbortError`

```js
try {
  await up.render({ url: '/path', target: '.target' })
} catch (error) {
  if (error instanceof up.CannotMatch) {
    console.log('Target not found')
  } else if (error instanceof up.AbortError) {
    console.log('Request was aborted')
  } else if (error instanceof up.Response) {
    console.log('HTTP error:', error.status)
  } else {
    throw error  // Re-throw unexpected errors
  }
}
```

**Failed responses (non-2xx):**
By default, Unpoly renders the `[up-fail-target]` on non-2xx responses.
Customize failure detection:
```js
up.network.config.fail = function(response) {
  return response.status >= 400  // default
}
```

**Errors in user code** (compilers, callbacks) don't interrupt the render pass —
they emit an `error` event on `window` and the render succeeds.

---

## Server integration headers (complete reference)

### Request headers sent by Unpoly

| Header | Example | Description |
|--------|---------|-------------|
| `X-Up-Version` | `3.12` | Marks request as Unpoly |
| `X-Up-Target` | `.content` | Fragment being updated |
| `X-Up-Fail-Target` | `.errors` | Target for failed responses |
| `X-Up-Mode` | `root` | Current layer mode |
| `X-Up-Fail-Mode` | `root` | Layer mode for failures |
| `X-Up-Context` | `{"key":"val"}` | Current layer context |
| `X-Up-Fail-Context` | `{"key":"val"}` | Fail layer context |
| `X-Up-Validate` | `email,name` | Fields being validated |

### Response headers accepted by Unpoly

| Header | Example | Description |
|--------|---------|-------------|
| `X-Up-Target` | `.main` | Override the target selector |
| `X-Up-Accept-Layer` | `{"id": 5}` | Accept overlay with JSON value |
| `X-Up-Dismiss-Layer` | `null` | Dismiss overlay with JSON value |
| `X-Up-Open-Layer` | `{}` | Force-open new overlay |
| `X-Up-Context` | `{"role":"admin"}` | Merge into current layer context |
| `X-Up-Events` | `[{"type":"user:created"}]` | Emit custom events on frontend |
| `X-Up-Location` | `/canonical/url` | Override URL pushed to history |
| `X-Up-Method` | `GET` | HTTP method for history URL |
| `X-Up-Title` | `Page Title` | Override document title |
| `X-Up-Poll` | `stop` | Stop a polling element |

### unpoly-rails gem helpers

```ruby
# Check if Unpoly request
if up?                         # Alias: up.requested_with_unpoly?

# Target info
up.target                      # Target selector string
up.target?('.content')        # Does target match?
up.layer.mode                 # 'root', 'modal', 'drawer', etc.
up.layer.root?
up.layer.overlay?
up.layer.context               # Hash of layer context
up.validate?                   # Is this a validation request?
up.validate.name               # Name of field being validated

# Render helpers
up.emit('user:created', id: @user.id)  # Emit event header
up.accept_layer(@user)                  # Accept overlay with value
up.dismiss_layer                        # Dismiss overlay
```
