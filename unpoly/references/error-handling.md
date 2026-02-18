# Error Handling & Failed Responses

## Table of contents
- [What counts as a failed response](#what-counts-as-a-failed-response)
- [Rendering failed responses differently](#rendering-failed-responses-differently)
- [Ignoring error codes](#ignoring-error-codes)
- [Customizing failure detection](#customizing-failure-detection)
- [Handling unexpected content](#handling-unexpected-content)
- [Skipping rendering](#skipping-rendering)
- [Network errors and offline](#network-errors-and-offline)
- [Aborted requests](#aborted-requests)
- [Flash messages (up-flashes)](#flash-messages-up-flashes)

---

## What counts as a failed response

Any HTTP status other than 2xx or 304 is **failed**. Unpoly renders failed responses
differently from successful ones — typically targeting the form or an error element
instead of the main content area.

---

## Rendering failed responses differently

Prefix any render option with `fail` to apply it only on failure:

```html
<form method="post" action="/action"
  up-submit
  up-target=".content"
  up-fail-target="form"
  up-scroll="auto"
  up-fail-scroll=".errors">
</form>
```

```js
up.render({
  url: '/action', method: 'post',
  target: '.content',          // success: update .content
  failTarget: 'form',          // failure: update the form
  scroll: 'auto',
  failScroll: '.errors',       // failure: scroll to errors
  onRendered: () => { },
  onFailRendered: () => { },
})
```

**Default fail target for forms** is the `<form>` element itself — so validation errors
render inside the form automatically with no extra config.

**Signal failure from the server** with HTTP 422:
```ruby
# Rails
render :new, status: :unprocessable_entity
```

---

## Ignoring error codes

Treat all responses as successful regardless of status:

```html
<a href="/api/data" up-follow up-fail="false">Load</a>
```

```js
up.render({ url: '/api/data', fail: false })
```

---

## Customizing failure detection

```js
// Fail if response has X-Unauthorized header
let defaultFail = up.network.config.fail
up.network.config.fail = (response) =>
  defaultFail(response) || !!response.header('X-Unauthorized')

// Or dynamically in an event listener
up.on('up:fragment:loaded', function(event) {
  if (event.response.header('X-Unauthorized')) {
    event.renderOptions.fail = true
  }
})
```

---

## Handling unexpected content

When a response doesn't contain the target selector (e.g. a login redirect or maintenance page),
Unpoly throws `up.CannotMatch` by default. Use `[up-fallback]` to gracefully show the response
in the main content area instead:

```html
<a href="/dashboard" up-follow up-fallback="true">Dashboard</a>
```

```js
up.render({ url: '/dashboard', fallback: true })
```

Fallback to main is **enabled by default** when navigating.

---

## Skipping rendering

### From the server

```http
# No content, skip rendering
HTTP/1.1 204 No Content

# Unchanged content (from conditional request), skip rendering
HTTP/1.1 304 Not Modified

# Explicitly tell Unpoly to update nothing
X-Up-Target: :none
```

### From JS (after response arrives)

```js
up.on('up:fragment:loaded', function(event) {
  // Inspect response before rendering
  if (event.response.header('X-No-Change')) {
    event.skip()   // abort this render pass, keep current DOM
  }

  // Or prevent entirely
  if (event.response.header('X-User-Created')) {
    event.preventDefault()
    alert('User created!')
  }
})
```

### Global skip rules

```js
up.fragment.config.skipResponse = function(response) {
  // Skip if body is empty or content matches current
  return !response.text || response.text === currentContent
}
```

### Conditional requests (304 optimization)

Unpoly stores ETags and modification times on rendered fragments and sends them on
reload/revalidation/polling requests:

```http
# Unpoly sends:
GET /messages HTTP/1.1
If-None-Match: "x234dff"
If-Modified-Since: Wed, 21 Oct 2024 07:28:00 GMT
```

The server can respond with `304 Not Modified` to skip re-rendering:

```ruby
# Rails
def show
  @post = Post.find(params[:id])
  if stale?(etag: @post, last_modified: @post.updated_at)
    render :show
  end
  # stale? responds 304 automatically when ETags match
end
```

Unpoly stores the received ETag in `[up-etag]` on the fragment automatically.
Set it manually to override:

```html
<div class="messages" up-etag='"abc123"'>...</div>
```

---

## Network errors and offline

Fatal errors (timeout, loss of connectivity) emit `up:request:offline` — no
`up:fragment:loaded` is fired since there's never a response.

```js
// Low-level: fired for each failed request
up.on('up:request:offline', function(event) {
  showOfflineBanner()
  // event.request — the failed up.Request
})

// Fragment-level: fired on the fragment being rendered; has retry()
up.on('up:fragment:offline', function(event) {
  if (confirm('Connection lost. Retry?')) {
    event.retry()  // re-runs the render with same options
  }
})

// When slow requests complete (not specifically offline)
up.on('up:network:recover', function(event) {
  hideLateBanner()
})
```

---

## Aborted requests

```js
// Abort all pending requests
up.network.abort()

// Abort requests targeting a specific fragment
up.network.abort({ target: '.content' })

// Abort by URL pattern or predicate
up.network.abort('/users/*')
up.network.abort((request) => request.url.includes('/slow'))

// Detect abort in promise chain
try {
  await up.render({ url: '/slow', target: '.content' })
} catch (e) {
  if (e instanceof up.AbortError) {
    console.log('Request was aborted')
  }
}
```

Two abort events exist at different levels:
- **`up:request:aborted`** — fired on the underlying HTTP request
- **`up:fragment:aborted`** — fired on a fragment element when its requests are aborted; also fired even when no requests are pending, so async code (timers, etc.) can use it as a general cleanup signal

```js
// Use up:fragment:aborted to cancel a timer when the fragment is aborted
up.compiler('.live-clock', function(element) {
  let timer = setInterval(updateClock, 1000)
  up.fragment.onAborted(element, () => clearInterval(timer))
})
```

---

## Flash messages (up-flashes)

Unpoly has built-in support for server-side flash/notification messages that
automatically insert into a designated container on every response.

**1. Place the container in your layout:**
```html
<nav>...</nav>
<div up-flashes></div>  <!-- outside main -->
<main>...</main>
```

**2. Include flash content in any server response:**
```html
<div up-flashes>
  <div class="alert alert-success">User was updated!</div>
</div>

<main>
  <!-- rest of the response -->
</main>
```

Unpoly finds the `[up-flashes]` element in the response and renders it into the
layout's `[up-flashes]` container — regardless of what fragment was targeted.

Internally, `[up-flashes]` applies `up-hungry`, `up-if-layer="subtree"`, `up-keep`, and `role="alert"` automatically.

**Flashes from overlays** are shown on the parent layer when the overlay closes.

**Custom transitions:**
```html
<div up-flashes up-transition="cross-fade" up-duration="300"></div>
```

**Auto-clear after delay:**
```html
<div up-flashes>
  <div class="flash" up-dismiss="destroy" up-duration="3000">Done!</div>
</div>
```

**Suppress stale cached flashes** — flashes from cached responses are automatically
suppressed if the cache entry is expired (they come from a previous request).
