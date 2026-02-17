# unpoly-skills

An [Agent Skill](https://agentskills.io) for working with [Unpoly](https://unpoly.com/) — the progressive enhancement framework for server-rendered HTML. Works with Claude Code, OpenAI Codex, OpenCode, Cursor, and [35+ more agents](https://github.com/vercel-labs/skills#supported-agents).

## Install

```bash
npx skills add thomasklemm/unpoly-skills
```

This auto-detects your coding agent and installs the skill in the right place. Supports Claude Code, Codex, Cursor, OpenCode, Amp, and more.

**Install globally** (available in all projects):

```bash
npx skills add thomasklemm/unpoly-skills --global
```

**Install for a specific agent:**

```bash
npx skills add thomasklemm/unpoly-skills --agent claude-code
```

## What it does

Gives your AI agent deep knowledge of Unpoly so it can help you build, debug, and extend Unpoly-powered apps. Covers:

- **Fragment updates** — targeting, rendering options, `:main`, `:maybe`, `[up-hungry]`, `[up-keep]`
- **Forms** — submission, `[up-validate]`, reactive forms with `[up-watch]` and `[up-autosubmit]`
- **Layers** — overlays, modes (modal/drawer/popup), subinteractions, accept/dismiss patterns
- **Performance** — caching, revalidation, lazy loading, preloading, polling
- **Loading state** — CSS classes, placeholders, previews, optimistic rendering, offline handling
- **Compilers** — `up.compiler()`, destructors, data passing, macros, third-party library integration
- **Navigation** — history, scroll restoration, focus management, URL patterns
- **Lifecycle & server** — render lifecycle events, controlling rendering, full HTTP header reference

## Usage

Once installed, your agent will automatically use this skill when you ask Unpoly questions:

> "How do I open a form in a modal drawer and reload the parent page when it's accepted?"

> "Add `[up-validate]` to this registration form's email and password fields"

> "Why is my compiler not running after a fragment update?"

> "Set up caching and lazy loading for this dashboard"

## Contents

```
unpoly/
├── SKILL.md                    # Overview and quick-reference tables
└── references/
    ├── fragments.md            # Targeting, selectors, up.render() options
    ├── forms.md                # Submission, validation, reactive forms
    ├── layers.md               # Overlays, subinteractions, layer context
    ├── performance.md          # Caching, lazy loading, polling, server optimization
    ├── loading-state.md        # Previews, placeholders, optimistic rendering
    ├── compilers.md            # up.compiler(), destructors, data, macros
    ├── navigation.md           # History, scrolling, focus, URL patterns
    └── lifecycle.md            # Render lifecycle, events, server headers
```

## Unpoly version

Covers Unpoly **3.12**. See [unpoly.com/changes](https://unpoly.com/changes) for the changelog.
