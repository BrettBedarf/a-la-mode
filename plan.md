# Pi à la Mode — Implementation Plan

## Goal

Segment [pi](vendor/pi/README.md) into on-demand modes similar to [opencode](./vendor/opencode/README.md) "primary agents" — each agent is a named configuration of model, thinking level, instructions, and optional `disabled_tools`, switchable mid-session without restart, using `disabled_tools` for tool blacklisting and `!entry` syntax for subtractive overrides on skill lists.

## Architecture

### 1. Mode Config Files

Support Markdown + YAML frontmatter config files.

```text
~/.pi/agent/modes/
├── build.md
├── reviewer.md
├── plan.md
└── docs-writer.jsonc
```

Markdown example:

```md
---
name: build
description: Write and edit code
disabled_tools:
  - bash
  - write
model: anthropic/claude-sonnet-4-5
thinkingLevel: medium
skills:
  - ~/dev/pi/skills/rust
  - ~/dev/pi/skills/testing
  - "!~/dev/pi/skills/legacy-rust"
extensions:
  - ~/dev/pi/exts/git-checkpoint.ts
---

...appended to system prompt...
```

List semantics:

- `disabled_tools` is a blacklist. Entries name tools to remove from the active tool set.
- `skills` is a whitelist. Normal entries add/include items.
- In `skills`, an entry prefixed with `!` removes that item from the resolved config during layered merging.
- In YAML frontmatter, negated skill entries must be quoted because bare `!value` is YAML tag syntax.
- `extensions` is additive-only in config; no `!` removal syntax.

### 2. Mode Manager Extension

- **Discovers** configs on `session_start` from `~/.pi/agent/modes/*.md`
- **Merges** user + project agents/modes by name, with project-local agents/modes overriding scalar fields, applying blacklist semantics for `disabled_tools`, using negation-aware merging for `skills`, and treating `extensions` as additive-only
- **Registers** `/mode <name>` command + optional configurable shortcut
- **Switches** via `pi.setActiveTools()`, `pi.setModel()`, `pi.setThinkingLevel()`
- **Injects** instructions via `before_agent_start` event (chained system prompt)
- **Persists** active mode via `pi.appendEntry("active-agent", { name })`
- **Restores** active mode on `session_start` from session entries (like [`preset.ts`](vendor/pi/packages/coding-agent/examples/extensions/preset.ts) does)
- **Contributes** skill paths via `resources_discover` event when mode has `skills` array

### 3. On-Demand Loading

| Resource     | Mechanism                                      | Trigger                      |
| ------------ | ---------------------------------------------- | ---------------------------- |
| Tools        | `pi.setActiveTools([...allToolsExceptDisabled])` | Mode switch                |
| Instructions | `before_agent_start` → `{ systemPrompt: ... }` | Every turn while mode active |
| Model        | `pi.setModel(model)`                           | Mode switch                  |
| Thinking     | `pi.setThinkingLevel(level)`                   | Mode switch                  |
| Skills       | `resources_discover` → `{ skillPaths: [...] }` | Mode switch + `/reload`      |
| Extensions   | Load from our managed npm install dir via a minimal package manager/runtime | Mode switch + startup |

### 4. Session Persistence

Store active mode in session JSONL via `pi.appendEntry()`. On resume, scan branch for last `"active-mode"` entry and reapply config (no model re-switch, only tools + instructions). See [`plan-mode/index.ts`](vendor/pi/packages/coding-agent/examples/extensions/plan-mode/index.ts) for state restoration pattern.

### 5. CLI / SDK Integration

```bash
pi --mode build          # start with build agent
pi --mode planner -p     # one-shot with planner agent
```

Register `--mode` flag via `pi.registerFlag()`.

## Implementation Phases

### Phase 0: Skeleton

- `index.ts` with `ExtensionAPI` factory, `session_start` handler, config discovery
- Shared loaders for markdown frontmatter and JSON/JSONC agent configs
- Normalize config fields, parse `disabled_tools`, parse quoted `!entry` removals for `skills`, and keep `extensions` additive-only

### Phase 1: Discovery + Switching

- Load `*.md` from ~/.pi/agent/modes/`
- Merge with precedence `project > user`, with blacklist semantics for `disabled_tools`, negation-aware merging for `skills`, and additive-only merging for `extensions`
- `/mode` command with `ctx.ui.select()` picker
- Apply effective active tools, `model`, and `thinkingLevel` on switch
- Status bar via `ctx.ui.setStatus()`

### Phase 2: Instructions + Persistence

- `before_agent_start` injection of markdown body or `config.instructions`
- `pi.appendEntry("active-mode", ...)` on switch
- Restore from session entries on `session_start`

### Phase 3: Skills + Extensions

- `resources_discover` to add per-mode skill paths
- Keep a dedicated npm install directory for mode-managed extensions
- Build a minimal package manager/runtime that installs, resolves, and loads extensions from that directory
- Document composition patterns for reuse: shared skills, prompt fragments, tool presets

### Phase 4: Polish

- Optional configurable shortcut for `/mode`
- `--mode` CLI flag
- Error handling for missing models/tools
- Conflict diagnostics for duplicate agent names and invalid configs

## Design Decisions

1. Use composition, not parent inheritance, in v1. Reuse via shared skills, prompt fragments, tool presets, or copied configs; represent tool removal via `disabled_tools`, allow quoted `!entry` removals for `skills`, and keep `extensions` additive-only without introducing full inheritance.
2. Support Markdown frontmatter.
3. Project-local modes override user modes with same name after project trust is granted.
4. Use `build` vernacular to match opencode-style primary agent naming.
5. Do not hardcode mode-switch keyboard shortcut in plan; treat it as optional configurable UX.

## References

- [pi extensions docs: `ExtensionAPI Methods`](vendor/pi/packages/coding-agent/docs/extensions.md#extensionapi-methods) — `setActiveTools`, `setModel`, `setThinkingLevel`, `registerCommand`, `registerShortcut`, `registerFlag`, `appendEntry`
- [pi extensions docs: `before_agent_start`](vendor/pi/packages/coding-agent/docs/extensions.md#before_agent_start) — system prompt injection per turn
- [pi extensions docs: `resources_discover`](vendor/pi/packages/coding-agent/docs/extensions.md#resources_discover) — dynamic skill/prompt/theme paths
- [pi keybindings docs](vendor/pi/packages/coding-agent/docs/keybindings.md) — shortcut format and conflict model
- [`preset.ts`](vendor/pi/packages/coding-agent/examples/extensions/preset.ts) — prior art for model/tools/thinking switching
- [`plan-mode/index.ts`](vendor/pi/packages/coding-agent/examples/extensions/plan-mode/index.ts) — mode toggle with state persistence
- [`tools.ts`](vendor/pi/packages/coding-agent/examples/extensions/tools.ts) — interactive tool toggle UI pattern
- [`dynamic-tools.ts`](vendor/pi/packages/coding-agent/examples/extensions/dynamic-tools.ts) — runtime tool registration
- [`subagent/agents.ts`](vendor/pi/packages/coding-agent/examples/extensions/subagent/agents.ts) — agent config discovery from markdown
- [`subagent/README.md`](vendor/pi/packages/coding-agent/examples/extensions/subagent/README.md) — project-local agent precedence and trust model
- [pi skills docs](vendor/pi/packages/coding-agent/docs/skills.md) — skills are already on-demand; leverage for domain knowledge
- [pi packages docs](vendor/pi/packages/coding-agent/docs/packages.md) — package structure if distributing agent configs as pi-package

## File Layout

```text
├── plan.md              # this file
├── extension/
│   ├── index.ts         # agent manager extension
│   └── types.ts         # AgentConfig, AgentScope types
├── agents/
│   ├── build.md         # preferred frontmatter form
│   ├── reviewer.md
│   └── planner.md
└── README.md            # usage docs
```
