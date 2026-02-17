# unpoly-skills

Give your AI coding agent deep knowledge of [Unpoly](https://unpoly.com/) — the progressive
enhancement framework that makes server-rendered apps feel like SPAs without the complexity.

Two skills, install either or both:

| Skill | What it's for |
|-------|--------------|
| ⚡ [`unpoly`](#unpoly--frontend) | Fragment updates, overlays, forms, compilers, caching, animations |
| 💎 [`unpoly-rails`](#unpoly-rails--ruby-on-rails) | Server-side gem helpers, Rails view syntax, Turbo coexistence |

Works with Claude Code, Cursor, OpenAI Codex, OpenCode, Amp, and [35+ more agents](https://github.com/vercel-labs/skills#supported-agents).

```bash
# Install — the wizard will ask which skills you want
npx skills add thomasklemm/unpoly-skills
```

---

## ⚡ `unpoly` — Frontend

Covers the full [Unpoly API](https://unpoly.com/api) so your agent can add fragment updates, overlays,
reactive forms, lazy loading, animations, and more to any server-rendered app.

### Install

```bash
# Add to the current project
npx skills add thomasklemm/unpoly-skills --skill unpoly

# Or install globally for all projects
npx skills add thomasklemm/unpoly-skills --skill unpoly --global
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

## 💎 `unpoly-rails` — Ruby on Rails

Covers the [`unpoly-rails`](https://github.com/unpoly/unpoly-rails) gem and everything
specific to using Unpoly in a Rails app: server-side request helpers, Rails view syntax,
flash messages, Turbo coexistence, and more.

### Install

```bash
# Add to the current project
npx skills add thomasklemm/unpoly-skills --skill unpoly-rails

# Or install globally for all projects
npx skills add thomasklemm/unpoly-skills --skill unpoly-rails --global
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
- **Rails view helpers** — `link_to`, `form_with`, `button_to` with Unpoly attributes; `html:` and `form:` option gotchas
- **Flash messages** — `[up-hungry]` pattern for flash that updates on every fragment response
- **Turbo coexistence** — disabling Turbo Drive in Rails 7+ apps

### Example prompts

> "How do I accept a layer overlay from my Rails controller?"

> "How do I handle `[up-validate]` in my create action without saving?"

> "How do I expire the Unpoly cache after a background job updates data?"

> "How do I use `button_to` with `up-confirm` for a delete action?"

### Contents

```
unpoly-rails/
├── SKILL.md                    # Overview, installation, quick-reference table
└── references/
    ├── server-helpers.md       # Full API reference for all server-side helpers
    └── rails-integration.md   # View helpers, flash messages, Turbo, CSP, global config
```

---

## Unpoly version

Both skills cover Unpoly **3.12**. See the [API docs](https://unpoly.com/api), [changelog](https://unpoly.com/changes), and [unpoly-rails gem](https://github.com/unpoly/unpoly-rails).

---

## Acknowledgements

A huge thank you to [Henning Koch](https://github.com/triskweline) at [makandra](https://makandra.com) for creating and maintaining Unpoly. It's a joy to work with — a thoughtful, well-documented framework that makes server-rendered apps genuinely great without the SPA complexity tax.

- [unpoly](https://github.com/unpoly/unpoly)
- [unpoly-rails](https://github.com/unpoly/unpoly-rails)

The skills in this repo are based on the official Unpoly documentation and source. This repo is independently maintained and not affiliated with makandra or the Unpoly project.

---

## License

MIT — see [LICENSE](LICENSE).
