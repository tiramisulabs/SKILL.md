# Seyfert Skill

An [Agent Skill](https://code.claude.com/docs/en/skills) for building, updating, debugging, reviewing, and testing Discord bots with the **[Seyfert](https://github.com/tiramisulabs/seyfert) v5** TypeScript framework.

It gives an AI coding agent (Claude Code, Codex, and other skill-aware runtimes) source-verified guidance on Seyfert: project setup, commands, options, middlewares, components, modals, events, plugins, i18n, cache, builders, sharding, and the official ecosystem packages.

## Why

The public docs are a good map, but they drift from the published package. Every API, signature, export, and code example in this skill was verified against the real Seyfert source, and the code examples are type-checked with `tsc` against it. When docs and source disagree, **source wins** — `references/source-truth.md` lists the exact corrections.

## Install

### Claude Code — plugin (one command)

This repo self-hosts as a plugin marketplace. Add it, then install:

```text
/plugin marketplace add tiramisulabs/SKILL.md
/plugin install seyfert@seyfert
```

Run `/reload-plugins` (or restart) to activate. Invoke as **`/seyfert:seyfert`**, or just let it auto-activate when you work in a project that imports `seyfert`. Update later with `/plugin marketplace update seyfert`. CLI equivalent:

```bash
claude plugin marketplace add tiramisulabs/SKILL.md
claude plugin install seyfert@seyfert
```

### Claude Code — manual (loose skill)

Clone into your skills directory as a folder named `seyfert`. The repo is named `SKILL.md`, so clone into an **explicit folder** — a bare clone would nest `SKILL.md/SKILL.md`:

```bash
git clone https://github.com/tiramisulabs/SKILL.md ~/.claude/skills/seyfert   # personal (all projects)
git clone https://github.com/tiramisulabs/SKILL.md .claude/skills/seyfert      # project-scoped (commit to share)
```

Detected live (restart only if `~/.claude/skills/` did not exist before). Since the repo also ships a plugin manifest, a manual clone loads as a skills-directory plugin too, so it is also invoked as **`/seyfert:seyfert`**. Verify with the `/` menu or by asking "what skills are available?".

### Codex

Clone into the [Agent Skills](https://agentskills.io) directory (`SKILL.md` is read directly; the `.claude-plugin/` files are ignored):

```bash
git clone https://github.com/tiramisulabs/SKILL.md ~/.agents/skills/seyfert   # user scope
git clone https://github.com/tiramisulabs/SKILL.md .agents/skills/seyfert      # repo scope
```

Invoke with `/skills` or `$`.

## Structure

```
.claude-plugin/                       # plugin + marketplace manifests (enables one-command install)
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
