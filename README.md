# unpoly-skills

[Agent Skills](https://agentskills.io) for working with [Unpoly](https://unpoly.com/) — the progressive enhancement framework for server-rendered HTML. Works with Claude Code, OpenAI Codex, OpenCode, Cursor, and [35+ more agents](https://github.com/vercel-labs/skills#supported-agents).

---

## `unpoly` — Frontend skill

Covers Unpoly fragment updates, overlays, forms, compilers, caching, animations, and more.

### Install

```bash
npx skills add thomasklemm/unpoly-skills/unpoly
```

**Install globally** (available in all projects):

```bash
npx skills add thomasklemm/unpoly-skills/unpoly --global
```

### What it covers

- **Fragment updates** — targeting, rendering options, `:main`, `:maybe`, `[up-hungry]`, `[up-keep]`
- **Forms** — submission, `[up-validate]`, reactive forms with `[up-watch]` and `[up-autosubmit]`
- **Layers** — overlays, modes (modal/drawer/popup), subinteractions, accept/dismiss patterns
- **Performance** — caching, revalidation, lazy loading, preloading, polling
- **Loading state** — CSS classes, placeholders, previews, optimistic rendering, offline handling
- **Compilers** — `up.compiler()`, destructors, data passing, macros, third-party library integration
- **Navigation** — history, scroll restoration, focus management, URL patterns
- **Animations** — built-in transitions, custom animations, duration/easing
- **Error handling** — failed responses, `[up-fail-target]`, conditional requests, network errors
- **Lifecycle & server** — render lifecycle events, controlling rendering, full HTTP header reference

### Example prompts

> "How do I open a form in a modal drawer and reload the parent page when it's accepted?"

> "Add `[up-validate]` to this registration form's email and password fields"

> "Why is my compiler not running after a fragment update?"

> "Set up caching and lazy loading for this dashboard"

### Contents

```
unpoly/
├── SKILL.md                    # Overview and quick-reference tables
└── references/
    ├── fragments.md            # Targeting, selectors, up.render() options
    ├── forms.md                # Submission, validation, reactive forms, [up-switch]
    ├── layers.md               # Overlays, subinteractions, layer context
    ├── performance.md          # Caching, lazy loading, polling, server optimization
    ├── loading-state.md        # Previews, placeholders, optimistic rendering
    ├── compilers.md            # up.compiler(), destructors, data, macros
    ├── navigation.md           # History, scrolling, focus, URL patterns
    ├── animations.md           # Transitions, custom animations
    ├── error-handling.md       # Failed responses, ETags, network errors
    └── lifecycle.md            # Render lifecycle, events, server headers
```

---

## `unpoly-rails` — Ruby on Rails server-side integration

Covers the `unpoly-rails` gem: server-side helpers for inspecting Unpoly requests, controlling
rendering, managing overlays, emitting events, and more.

### Install

```bash
npx skills add thomasklemm/unpoly-skills/unpoly-rails
```

**Install globally:**

```bash
npx skills add thomasklemm/unpoly-skills/unpoly-rails --global
```

### What it covers

- **Request inspection** — `up?`, `up.target`, `up.target?`, `up.fail_target`, `up.any_target?`
- **Response control** — override target, set title, render nothing
- **Layer API** — `up.layer.mode`, `up.layer.overlay?`, `up.layer.accept`, `up.layer.dismiss`, `up.layer.open`
- **Events** — `up.emit`, `up.layer.emit`
- **Form validation** — `up.validate?`, `up.validate_names`
- **Cache control** — `up.cache.expire`, `up.cache.evict`
- **Context** — `up.context` read/write/delete
- **CSP callbacks** — `up.safe_callback`
- **Conditional GET** — `fresh_when`, `stale?` with Unpoly polling

### Example prompts

> "How do I accept a layer overlay from my Rails controller?"

> "How do I handle `[up-validate]` in my create action without saving?"

> "How do I expire the Unpoly cache after a background job updates data?"

> "How do I emit a frontend event from a Rails action?"

### Contents

```
unpoly-rails/
├── SKILL.md                    # Overview, installation, quick-reference table
└── references/
    └── server-helpers.md       # Full API reference for all server-side helpers
```

---

## Unpoly version

Both skills cover Unpoly **3.12**. See [unpoly.com/changes](https://unpoly.com/changes) for the changelog.
