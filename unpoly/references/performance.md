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
up.cache.expire('/users')                  // Mark specific URL stale (URL pattern)
up.cache.evict()                           // Remove all entries
up.cache.evict({ url: '/users' })          // Remove specific entry
up.cache.get({ url: '/users' })            // Look up cached request (may still be in-flight)
```

**Network utilities:**
```js
up.network.isBusy()                        // true if any request is in flight
up.request('/api/data', { method: 'GET' }) // Make a raw request (returns up.Request)
up.network.abort()                         // Abort all pending requests
up.network.abort({ target: '.content' })   // Abort requests targeting a fragment
```

**Network config:**
```js
up.network.config.concurrency = 6         // max concurrent requests (extras queued)
up.network.config.timeout = 90_000        // default request timeout (ms)
up.network.config.lateDelay = 400         // ms before showing progress bar
up.network.config.cacheExpireAge = 15_000 // ms until entry triggers revalidation
up.network.config.cacheEvictAge = 90 * 60 * 1000  // ms until entry is removed
up.network.config.cacheSize = 70          // max cached responses
up.network.config.wrapMethod = true       // wrap PATCH/PUT/DELETE as POST + _method param
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

**`[up-defer]` timing values:**

| Value | Behavior |
|-------|---------|
| `insert` (default) | Load immediately when inserted into DOM |
| `reveal` | Load when element scrolls into viewport |
| `manual` | Only load when explicitly triggered via `up.deferred.load()` |

```html
<!-- Load on viewport entry -->
<div up-defer="/recommendations" up-defer="reveal">
  <p class="placeholder">Loading recommendations…</p>
</div>

<!-- Manual load (e.g. triggered by a button click) -->
<div id="heavy-chart" up-defer="/chart" up-defer="manual">
  <p class="placeholder">Chart not loaded yet</p>
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
up.deferred.load(deferElement)  // also: up.link.deferred.load(element)
```

**`up:deferred:load` event** — fires before deferred content is loaded; preventable:
```js
up.on('up:deferred:load', '[up-defer]', function(event) {
  if (!userHasPermission) event.preventDefault()
})
```

**`[up-intersect-margin]`** — expand/contract intersection zone for `reveal` mode:
```html
<div up-defer="/content" up-defer="reveal" up-intersect-margin="200px">
```

**Caching:** Deferred fragments respect the cache. If the URL is already cached,
the fragment renders instantly.

**Multiple fragments from one URL:**
```html
<div up-defer="/dashboard #sales-chart"></div>
<div up-defer="/dashboard #revenue-chart"></div>
<!-- Both use the same cached response -->
```

**Pattern: infinite scrolling** — trigger on reveal + append with `:after`:
```html
<div id="pages">
  <div class="page"><!-- items page 1 --></div>
</div>

<a id="next-page" href="/items?page=2"
   up-defer="reveal"
   up-target="#next-page, #pages:after">
  Load more
</a>
```
The response returns the next page items plus an updated `#next-page` link for page 3.
On the last page, render `#next-page` as a non-link element to stop loading.

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

**`[up-poll]` attributes:**

| Attribute | Default | Description |
|-----------|---------|-------------|
| `[up-interval]` | `30000` | Reload interval in ms |
| `[up-if-layer]` | `'front'` | `'front'` pauses under overlays; `'any'` continues |
| `[up-href]` | source URL | Override the URL to poll |
| `[up-keep-data]` | — | Preserve fragment data object across reloads |

**Control polling from JS:**
```js
up.radio.startPolling(element, { interval: 10000 })
up.radio.startPolling(element, { interval: 10000, ifLayer: 'any' })
up.radio.stopPolling(element)
```

**Smart polling:** Unpoly pauses polling when:
- The browser tab is hidden
- The user is offline
- The layer with the polling element is not frontmost (when `up-if-layer="front"`)

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
| `X-Up-Origin-Mode` | `modal` | Mode of layer where interaction originated |
| `X-Up-Context` | `{...}` | Current layer context (JSON) |
| `X-Up-Fail-Context` | `{...}` | Context of failure layer (JSON) |
| `X-Up-Validate` | `email` | Field being validated |

### Server can control cache via response headers

| Header | Example | Meaning |
|--------|---------|---------|
| `X-Up-Expire-Cache` | `"/users/*"` | Expire matching cache entries |
| `X-Up-Evict-Cache` | `"/users/*"` | Remove matching cache entries entirely |
