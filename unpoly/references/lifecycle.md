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
3. **`up:request:load`** — before request is sent; preventable, options mutable
4. **Fetch** — request sent (or cache hit)
5. **`up:request:loaded`** — response received (including HTTP errors)
6. **`up:fragment:loaded`** — response received, before DOM changes; preventable
7. **Compile** — compilers run on new elements
8. **`up:fragment:inserted`** — after insert, after compile
9. **Animation** — transition plays
10. **`up:fragment:rendered`** — after animation
11. *(Optional)* **Revalidation** — if cache expired, repeat from step 3

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

**`up.Response` properties** (available in `event.response`, render results, `up:fragment:loaded`):
```js
response.url          // URL from which the response was loaded
response.status       // HTTP status code (e.g. 200, 422)
response.ok           // true if 2xx
response.text         // response body as string
response.json         // body parsed as JSON (cached)
response.contentType  // content-type header
response.etag         // ETag header value
response.lastModified // Last-Modified header as Date
response.expired      // true if cached response has expired
response.age          // milliseconds since the response was received
response.header(name) // read any response header (case-insensitive)
response.isHTML()     // true if content-type is text/html
```

---

## Key lifecycle events

### Fragment events

| Event | When | Preventable |
|-------|------|------------|
| `up:fragment:loaded` | Response received, before render | Yes |
| `up:fragment:inserted` | After insert, after compile | No |
| `up:fragment:rendered` | After animation completes | No |
| `up:fragment:destroyed` | Element removed from DOM (emitted on parent) | No |
| `up:fragment:keep` | Element is about to be kept (`[up-keep]`) | Yes |
| `up:fragment:kept` | Element was kept (after keep) | No |
| `up:fragment:hungry` | Hungry element about to be included in render | Yes |
| `up:fragment:poll` | Before polling reload | Yes |
| `up:fragment:aborted` | Fragment requests were aborted | No |
| `up:fragment:offline` | Network failure during render | No |

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
| `up:layer:opened` | After opening, before animation | No |
| `up:layer:accept` | Layer is about to accept | Yes |
| `up:layer:accepted` | Layer was accepted | No |
| `up:layer:dismiss` | Layer is about to dismiss | Yes |
| `up:layer:dismissed` | Layer was dismissed | No |
| `up:layer:location:changed` | URL changed within a layer | No |

### Location events

| Event | When | Preventable |
|-------|------|------------|
| `up:location:changed` | URL changed in browser | No |
| `up:location:restore` | Back/forward navigation | Yes |

### Network / request events

| Event | When | Preventable |
|-------|------|------------|
| `up:request:load` | Before request is sent; listeners may mutate request options | Yes |
| `up:request:loaded` | Response received (including HTTP errors) | No |
| `up:request:offline` | Fatal network failure for a specific request (timeout, disconnect) | No |
| `up:request:aborted` | HTTP request was aborted | No |
| `up:network:late` | Requests taking longer than `lateDelay` ms | No |
| `up:network:recover` | All late requests have completed | No |

**Listen to events:**
```js
up.on('up:fragment:inserted', '.result', function(event, element) {
  console.log('Result inserted:', element)
})

// Listen on a specific element
up.on(myElement, 'up:fragment:inserted', handler)

// Remove listener
let off = up.on('up:fragment:inserted', handler)
off()  // Remove

// Emit a custom event
up.emit(element, 'user:selected', { id: 5 })

// Current input device ('key', 'pointer', 'unknown')
up.event.inputDevice
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
| `X-Up-Origin-Mode` | `modal` | Mode of the layer that triggered the request |
| `X-Up-Context` | `{"key":"val"}` | Current layer context |
| `X-Up-Fail-Context` | `{"key":"val"}` | Fail layer context |
| `X-Up-Validate` | `email name` | Space-separated field names being validated |
| `If-None-Match` | `"abc123"` | ETag for conditional requests |
| `If-Modified-Since` | `Wed, 21 Oct 2024…` | Last-modified for conditional requests |

### Response headers accepted by Unpoly

| Header | Example | Description |
|--------|---------|-------------|
| `X-Up-Target` | `.main` | Override the target selector (`':none'` skips rendering) |
| `X-Up-Accept-Layer` | `{"id": 5}` | Accept overlay with JSON value (ignored on root layer) |
| `X-Up-Dismiss-Layer` | `null` | Dismiss overlay with JSON value (ignored on root layer) |
| `X-Up-Open-Layer` | `{}` | Force-open new overlay |
| `X-Up-Context` | `{"role":"admin"}` | Merge into current layer context (only changed keys) |
| `X-Up-Events` | `[{"type":"user:created"}]` | Emit custom events on frontend |
| `X-Up-Location` | `/canonical/url` | Override URL pushed to history |
| `X-Up-Method` | `GET` | HTTP method for history URL |
| `X-Up-Title` | `"Page Title"` | Override document title (JSON-encoded string) |
| `X-Up-Poll` | `stop` | Stop a polling element |
| `X-Up-Expire-Cache` | `"/users/*"` | Expire matching client-side cache entries |
| `X-Up-Evict-Cache` | `"/users/*"` | Remove matching cache entries entirely |

### unpoly-rails gem helpers

```ruby
# Check if Unpoly request
up?                                     # true/false

# Target info
up.target                               # Target selector string
up.target?('.content')                 # Does target match?
up.layer.mode                          # 'root', 'modal', 'drawer', etc.
up.layer.root?
up.layer.overlay?
up.layer.context                        # Hash of layer context
up.validate?                            # Is this a validation request?
up.validate_names                       # Array of field names being validated

# Render helpers
up.emit('user:created', id: @user.id)  # Emit event header
up.layer.accept(@user)                  # Accept overlay with value
up.layer.dismiss                        # Dismiss overlay
```
