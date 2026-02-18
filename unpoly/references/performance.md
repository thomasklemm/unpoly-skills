# Performance

## Table of contents
- [Caching](#caching)
- [Revalidation](#revalidation)
- [Preloading](#preloading)
- [Lazy loading (up-defer)](#lazy-loading-up-defer)
- [Polling (up-poll)](#polling-up-poll)
- [Optimizing server responses](#optimizing-server-responses)

---

## Caching

Unpoly caches GET responses so revisited pages load instantly.

**Enable cache:**
```js
// Auto-cache all GET requests (default when navigating)
up.network.config.autoCache = (request) => request.method === 'GET'
```

**Per-link control:**
```html
<a href="/stock" up-cache="false">Live prices</a>
<a href="/report" up-cache="true">Cached report</a>
```

**Per-request:**
```js
up.render({ url: '/data', cache: false })
```

**Disable globally:**
```js
up.fragment.config.navigateOptions.cache = false
```

**Expire after form submission:**
Unpoly automatically expires the entire cache after any non-GET request (form submission).
The expired entries are still used for instant rendering, then revalidated.

**Manually manage cache:**
```js
up.cache.expire()                          // Mark all stale (will revalidate)
up.cache.expire('/users')                  // Mark specific URL stale
up.cache.evict()                           // Remove all entries
up.cache.evict({ url: '/users' })          // Remove specific entry
```

---

## Revalidation

Cache entries expire after **15 seconds** by default. When expired content is rendered,
Unpoly automatically fetches fresh content from the server and re-renders.
The user sees cached content instantly, then fresh content replaces it.

**Configure expiry:**
```js
up.network.config.cacheExpireAge = 20_000  // 20 seconds
```

**Skip re-rendering if unchanged:**
Support [conditional requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Conditional_requests)
to let the server respond with `304 Not Modified` when content hasn't changed:

```ruby
# Rails — using ETags
def show
  @post = Post.find(params[:id])
  if stale?(etag: @post)
    render :show
  end
  # stale? automatically renders 304 if ETag matches
end
```

Unpoly sends `If-None-Match` / `If-Modified-Since` headers on revalidation requests.

**Skip revalidation render from JS:**
```js
up.on('up:fragment:loaded', function(event) {
  // Don't replace a playing video with revalidated content
  let video = document.querySelector('video')
  if (event.revalidating && video && !video.paused) {
    event.skip()
  }
})
```

**Detect revalidation in a compiler:**
```js
up.compiler('[track-page-view]', function(element, data, meta) {
  if (!meta.revalidating) {
    trackPageView(meta.layer.location)
  }
})
```

---

## Preloading

Load a page in the background before the user clicks:

```html
<!-- Preload on mouse hover -->
<a href="/dashboard" up-preload>Dashboard</a>

<!-- Preload on intersection (when link enters viewport) -->
<a href="/dashboard" up-preload="insert">Dashboard</a>
```

**Global preload on hover:**
```js
up.link.config.preloadSelectors.push('a[href]')
```

**`[up-instant]`** — follow on mousedown instead of click (~100ms faster feel):
```html
<a href="/dashboard" up-follow up-instant>Dashboard</a>
```

---

## Lazy loading (up-defer)

Load a fragment only when needed, not on initial page render:

**Load immediately when page loads (deferred from initial render):**
```html
<div id="stats" up-defer="/expensive-stats">
  <!-- Shown while loading -->
  <p class="placeholder">Loading…</p>
</div>
```

> **Gotcha:** The placeholder **must have a derivable target selector** — typically an `[id]`.
> Unpoly targets itself by default, sending `X-Up-Target: #stats`. The server must respond
> with an element using the **same id** as a wrapper:
>
> ```html
> <!-- Server response must wrap content in the matching id -->
> <div id="stats">
>   …actual content…
> </div>
> ```
>
> If the ids don't match, Unpoly cannot find the fragment to replace.

**Load when element scrolls into view:**
```html
<div up-defer="/recommendations" up-defer-reveal>
  <p class="placeholder">Loading recommendations…</p>
</div>
```

**Custom target (partial URL):**
The server returns a full page; only the matching fragment is used:
```html
<div id="sales-chart" up-defer="/dashboard #sales-chart">
```

**Trigger manually from JS:**
```js
let deferElement = document.querySelector('[up-defer]')
up.deferred.load(deferElement)
```

**Caching:** Deferred fragments respect the cache. If the URL is already cached,
the fragment renders instantly.

**Multiple fragments from one URL:**
```html
<div up-defer="/dashboard #sales-chart"></div>
<div up-defer="/dashboard #revenue-chart"></div>
<!-- Both use the same cached response -->
```

---

## Polling (up-poll)

Reload a fragment on a timer:

```html
<!-- Reload .inbox every 30 seconds -->
<div class="inbox" up-poll>...</div>

<!-- Custom interval -->
<div class="notifications" up-poll up-interval="5000">...</div>

<!-- Poll a different URL than the current page -->
<div class="status" up-poll up-source="/api/status">...</div>
```

**Control polling from JS:**
```js
up.radio.startPolling(element, { interval: 10000 })
up.radio.stopPolling(element)
```

**Smart polling:** Unpoly pauses polling when:
- The browser tab is hidden
- The user is offline
- The layer with the polling element is not frontmost

**Server can stop polling** by responding with `X-Up-Poll: stop` header.

---

## Optimizing server responses

Unpoly sends request metadata so the server can optimize responses.

### Detect Unpoly requests

```ruby
# unpoly-rails gem
if up?
  # This is an Unpoly request
end
```

```python
# Check header directly
if request.headers.get('X-Up-Version'):
    # Unpoly request
```

### Omit content not targeted

The server can skip rendering content that isn't in the target selector:

```ruby
# Rails with unpoly-rails
def index
  if up.target?('.users-list')
    @users = User.all
  end
  # Skip loading expensive sidebar data if not targeted
end
```

### Different content per layer mode

```ruby
# Rails
def new
  if up.layer.overlay?
    render :new_modal   # Lean modal without layout
  else
    render :new         # Full page with layout
  end
end
```

Check layer mode:
```ruby
up.layer.mode          # 'root', 'modal', 'drawer', 'popup', etc.
up.layer.root?
up.layer.overlay?
```

### Vary responses (HTTP Vary header)

When content differs by Unpoly request headers, set the `Vary` header so
intermediary caches know:

```http
Vary: X-Up-Mode
```

### Key Unpoly request headers

| Header | Value | Meaning |
|--------|-------|---------|
| `X-Up-Version` | `3.12` | Identifies Unpoly request |
| `X-Up-Target` | `.content` | Fragment selector being updated |
| `X-Up-Fail-Target` | `.errors` | Target for failed responses |
| `X-Up-Mode` | `root` | Current layer mode |
| `X-Up-Fail-Mode` | `root` | Layer mode for failed responses |
| `X-Up-Context` | `{...}` | Current layer context (JSON) |
| `X-Up-Validate` | `email` | Field being validated |
