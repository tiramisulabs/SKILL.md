# Seyfert Skill

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for building, updating, debugging, reviewing, and testing Discord bots with the **[Seyfert](https://github.com/tiramisulabs/seyfert) v5** TypeScript framework.

It gives an AI coding agent (Claude Code, Codex, and other skill-aware runtimes) source-verified guidance on Seyfert: project setup, commands, options, middlewares, components, modals, events, plugins, i18n, cache, builders, sharding, and the official ecosystem packages.

## Why

The public docs are a good map, but they drift from the published package. Every API, signature, export, and code example in this skill was verified against the real Seyfert source, and the code examples are type-checked with `tsc` against it. When docs and source disagree, **source wins** — `references/source-truth.md` lists the exact corrections.

## Install

Clone (or copy) this repository into your skills directory as `seyfert`:

```bash
# Claude Code — user-level (loads in every project)
git clone https://github.com/tiramisulabs/SKILL.md ~/.claude/skills/seyfert

# Claude Code — project-level
git clone https://github.com/tiramisulabs/SKILL.md .claude/skills/seyfert

# Codex
git clone https://github.com/tiramisulabs/SKILL.md ~/.codex/skills/seyfert
```

The agent auto-loads it when you work on a project that imports `seyfert` (or invoke it explicitly, e.g. `/seyfert` in Claude Code).

## Structure

```
SKILL.md                              # entry point: routing, source-of-truth rules, critical v5 facts, quick start
references/
  source-truth.md                     # curated, source-cited doc-vs-source corrections + local landmarks
  setup-runtime.md                    # clients, start(), seyfert.config, intents, sharding, presence, module augmentation
  commands.md                         # commands, options, middlewares, subcommands, context menus, prefix, errors
  components.md                       # components, modals, collectors, polls, Components v2
  builders.md                         # Embed, Button, ActionRow, select menus, Modal, v2, attachments, Formatter
  plugins.md                          # plugin authoring + official @slipher/* plugins
  i18n-cache-recipes.md               # i18n/langs, cache, structures, and recipes
  testing.md                          # @slipher/testing toolkit + unit tests
  link-verification-matrix.md         # URL-by-URL coverage/verification audit
  pages/                              # per-page deep notes for every docs URL
```

Open only the reference a task needs; `SKILL.md` routes you there.

## Contributing

When Seyfert's source changes, update the affected `references/**` notes (and `source-truth.md`) and re-verify against the installed package or the `seyfert` source. Keep code examples type-correct.

## License

[MIT](./LICENSE) — © tiramisulabs. Seyfert itself is MIT-licensed.
