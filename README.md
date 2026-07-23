# Seyfert Skill

An [Agent Skill](https://agentskills.io) for building, updating, debugging, reviewing, and testing Discord bots with the **[Seyfert](https://github.com/tiramisulabs/seyfert) v5** TypeScript framework.

It gives an AI coding agent (Claude Code, Codex, and other skill-aware runtimes) source-verified guidance on Seyfert: project setup, commands, options, middlewares, components, modals, events, plugins, i18n, cache, builders, sharding, and the official ecosystem packages.

## Why

The public docs are a good map, but they drift from the published package. Every API, signature, export, and code example in this skill was verified against the real Seyfert source, and the code examples are type-checked with `tsc` against it. When docs and source disagree, **source wins** — `references/source-truth.md` lists the exact corrections.

## Install

### Any supported agent — automatic (recommended)

Use the open [`skills`](https://github.com/vercel-labs/skills) installer. It detects the coding agents installed on your machine and installs the skill in the right location:

```bash
npx skills add tiramisulabs/SKILL.md
```

The installer supports Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot, OpenCode, and many other Agent Skills-compatible runtimes. Add `-g` for a user-wide installation, or target agents explicitly:

```bash
npx skills add tiramisulabs/SKILL.md -g -a codex -a cursor -a gemini-cli
```

### Claude Code plugin

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

### Manual installation

If you do not want to use the installer, clone the repository into your agent's skills directory as a folder named `seyfert`. The repo is named `SKILL.md`, so use an **explicit destination folder** — a bare clone would nest `SKILL.md/SKILL.md`.

For Claude Code:

```bash
git clone https://github.com/tiramisulabs/SKILL.md ~/.claude/skills/seyfert   # personal (all projects)
git clone https://github.com/tiramisulabs/SKILL.md .claude/skills/seyfert      # project-scoped (commit to share)
```

Detected live (restart only if `~/.claude/skills/` did not exist before). Since the repo also ships a plugin manifest, a manual clone loads as a skills-directory plugin too, so it is also invoked as **`/seyfert:seyfert`**. Verify with the `/` menu or by asking "what skills are available?".

For Codex:

```bash
git clone https://github.com/tiramisulabs/SKILL.md ~/.codex/skills/seyfert   # user scope
git clone https://github.com/tiramisulabs/SKILL.md .agents/skills/seyfert    # repo scope
```

Other runtimes use their own skills directory or the universal `.agents/skills` project directory. Provider-specific metadata is optional: every compatible runtime reads the same root `SKILL.md`, while Claude Code alone uses `.claude-plugin/` for its plugin marketplace.

## Structure

```
.claude-plugin/                       # optional Claude Code plugin + marketplace adapter
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
  pages/                              # per-page deep notes (each carries its source URL, coverage ref, verification label)
```

Open only the reference a task needs; `SKILL.md` routes you there.

## Contributing

When Seyfert's source changes, update the affected `references/**` notes (and `source-truth.md`) and re-verify against the installed package or the `seyfert` source. Keep code examples type-correct.

## License

[MIT](./LICENSE) — © tiramisulabs. Seyfert itself is MIT-licensed.
