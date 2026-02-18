# Animations & Transitions

## Table of contents
- [Animations vs transitions](#animations-vs-transitions)
- [Built-in animations](#built-in-animations)
- [Built-in transitions](#built-in-transitions)
- [Using animations and transitions](#using-animations-and-transitions)
- [Tuning duration and easing](#tuning-duration-and-easing)
- [Custom animations and transitions](#custom-animations-and-transitions)
- [Disabling animation](#disabling-animation)

---

## Animations vs transitions

- **Animation** — applied to a single element being inserted or removed (e.g. fade in a new element)
- **Transition** — applied when swapping an old element for a new one (both old and new animate simultaneously)

Use `{ animation }` / `[up-animation]` when opening/closing overlays.
Use `{ transition }` / `[up-transition]` when swapping fragments.

---

## Built-in animations

| Name | Effect |
|------|--------|
| `fade-in` | Fade from transparent to opaque |
| `fade-out` | Fade from opaque to transparent |
| `move-to-top` | Slide out to top |
| `move-from-top` | Slide in from top |
| `move-to-bottom` | Slide out to bottom |
| `move-from-bottom` | Slide in from bottom |
| `move-to-left` | Slide out to left |
| `move-from-left` | Slide in from left |
| `move-to-right` | Slide out to right |
| `move-from-right` | Slide in from right |
| `none` | No animation |

---

## Built-in transitions

| Name | Effect |
|------|--------|
| `cross-fade` | Old fades out while new fades in |
| `move-left` | Old slides left while new slides in from right |
| `move-right` | Old slides right while new slides in from left |
| `move-up` | Old slides up while new slides in from below |
| `move-down` | Old slides down while new slides in from above |
| `none` | No transition (default) |

---

## Using animations and transitions

**Fragment updates:**
```html
<a href="/next" up-target=".content" up-transition="cross-fade">Next page</a>
```

```js
up.render({ url: '/next', target: '.content', transition: 'cross-fade' })
up.render({ url: '/next', target: '.content', transition: 'move-left' })
```

**Overlays** (use `animation`, not `transition`):
```html
<a href="/menu" up-layer="new drawer" up-animation="move-from-left">Open menu</a>
```

```js
up.layer.open({ url: '/menu', mode: 'drawer', animation: 'move-from-right' })
```

**Configure default overlay animations:**
```js
up.layer.config.drawer.openAnimation = 'move-from-right'
up.layer.config.drawer.closeAnimation = 'move-to-right'
up.layer.config.modal.openAnimation = 'fade-in'
up.layer.config.modal.closeAnimation = 'fade-out'
```

---

## Tuning duration and easing

```html
<a href="/next" up-target=".content"
   up-transition="cross-fade"
   up-duration="300"
   up-easing="ease-in-out">
  Next
</a>
```

```js
up.render({
  url: '/next',
  target: '.content',
  transition: 'cross-fade',
  duration: 300,       // milliseconds (default: 175)
  easing: 'ease-out'  // CSS timing function (default: 'ease')
})
```

**Change global defaults:**
```js
up.motion.config.duration = 200
up.motion.config.easing = 'ease-in-out'
```

---

## Composing transitions inline

Transitions can be composed from two animation names using a `/` separator:

```html
<!-- Old fades out, new slides in from bottom -->
<a href="/next" up-target=".content" up-transition="fade-out/move-from-bottom">Next</a>
```

```js
up.render({ target: '.content', url: '/next', transition: 'fade-out/move-from-top' })
```

---

## Custom animations and transitions

**Register a custom animation:**
```js
up.animation('pulse', function(element, options) {
  element.animate(
    [{ transform: 'scale(1)' }, { transform: 'scale(1.1)' }, { transform: 'scale(1)' }],
    { duration: options.duration, easing: options.easing }
  )
})
```

**Register a custom transition:**
```js
up.transition('my-slide', function(oldElement, newElement, options) {
  return Promise.all([
    up.animate(oldElement, 'move-to-left', options),
    up.animate(newElement, 'move-from-right', options)
  ])
})
```

**Use it:**
```html
<a href="/next" up-target=".content" up-transition="my-slide">Next</a>
```

---

## Motion utility functions

```js
// Animate an element (string name, CSS props object, or custom function)
up.animate(element, 'fade-in', { duration: 300 })
up.animate(element, { opacity: 0 }, { duration: 200 })

// Transition between old and new element
up.morph(oldElement, newElement, 'cross-fade', { duration: 300 })

// Jump all active animations to their final frame
up.motion.finish()           // all animations
up.motion.finish(element)    // animations on specific element

// Check if animations are enabled
up.motion.isEnabled()        // false when prefers-reduced-motion or config.enabled = false
```

---

## Disabling animation

**Globally (e.g. in tests):**
```js
up.motion.config.enabled = false
```

**Per render:**
```js
up.render({ url: '/next', target: '.content', transition: 'none' })
```

Unpoly automatically respects the user's OS-level "reduce motion" preference
(`prefers-reduced-motion`). When enabled, animations are skipped by default.
To force animations regardless:
```js
up.motion.config.enabled = true
```
