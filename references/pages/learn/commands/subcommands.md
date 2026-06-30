# Sub-Zero (Sub Commands)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/subcommands

Coverage reference: commands.md

Verification status: Source-verified (seyfert-core branch more-qol, 2026-06)

## Page Summary

Subcommands are classes that extend `SubCommand` (abstract, must implement `run`) and are attached to a parent `Command` as options. You register them on the parent with the `@Options([...])` decorator (passing subcommand classes), or let Seyfert discover sibling files automatically with `@AutoLoad()`. Subcommands can be organized into groups declared on the parent via `@Groups`/`@GroupsT` and assigned per-subcommand with `@Group`. Discord (and Seyfert) model subcommands and subcommand-groups as command options, so a parent command with subcommands has no `run` of its own — only leaf subcommands run.

Discord allows max two nesting levels: `group > subcommand > options` OR `subcommand > options`. A `SubCommand` may carry basic options (`create*Option`) but cannot nest further subcommands.

## Key APIs (verified)

All imported from the root `'seyfert'` barrel (re-exported via `src/index.ts`).

- `abstract class SubCommand extends BaseCommand` — `src/commands/applications/chat.ts:408`. Sets `type = ApplicationCommandOptionType.Subcommand`, has optional `group?: string`, optional `declare options?: CommandOptionWithType[]` (basic options only), and an `abstract run(context: CommandContext): any`.
- `class Command extends BaseCommand` — `src/commands/applications/chat.ts:361`. Holds `options?`, `groups?`, `groupsAliases?`, `__tGroups?`. Its `toJSON()` (`chat.ts:375`) walks `options`, emitting a `SubcommandGroup` wrapper for each subcommand that carries a `group`, and pushing bare subcommands/basic options directly.
- `@Options(options)` — `src/commands/decorators.ts:161`. Two overloads: an array of subcommand constructors `(new () => SubCommand)[]` (instantiated via `new x()`), or a `LowercaseOptionsRecord` of plain options (keys MUST be lowercase at compile time). Sets `this.options`.
- `@Declare(input)` — `src/commands/decorators.ts:195`. Requires `name` + `description`; `name` is forced lowercase at the type level (`LowercaseDeclareName`). Used on both parent and subcommand.
- `@Group(groupName)` / `@Group(groupsDef, groupName)` — `src/commands/decorators.ts:151`. Sets `group` on the subcommand. The 2-arg overload accepts a `GroupDefinitions` object as the first arg purely for type-safe `keyof` name checking — the value is otherwise discarded (`resolvedGroupName` uses the second arg).
- `@Groups(groups)` — `src/commands/decorators.ts:107`. `Record<string, LocalizedGroupDefinition>`; sets `groups` and builds `groupsAliases` from each group's `aliases`.
- `@GroupsT(groups)` — `src/commands/decorators.ts:91`. Translated variant; `Record<string, TranslatedGroupDefinition>` (uses `name`/`description` localization keys). Stored on `__tGroups` plus builds `groupsAliases`.
- `defineGroups(groups)` — `src/commands/decorators.ts:85`. Identity helper returning the typed groups object (for the `@Group(def, name)` overload). Two overloads cover Localized vs Translated records.
- `@AutoLoad()` — `src/commands/decorators.ts:177`. Sets `__autoload = true`. The handler then loads every sibling file in the parent's directory and pushes their default-exported `SubCommand`s onto `options` (`src/commands/handler.ts:369-401`).
- Group types: `GroupDefinition`, `GroupDefinitions`, `LocalizedGroupDefinition`, `TranslatedGroupDefinition` — `src/commands/decorators.ts:68-83`. Both definition shapes require `defaultDescription`; optional `name`, `description`, `aliases`.
- `CommandContext<T extends OptionsRecord, M>` — `src/commands/applications/chatcontext.ts:45`. Passed to `run`. `ctx.options` is typed `ContextOptions<T>` (`chat.ts:117`) — pass `CommandContext<typeof options>` to type a subcommand's own options. Narrowing guards: `isChat()` and `inGuild(): this is GuildCommandContext<T, M>` (`chatcontext.ts:256,260`).

## Code Examples (verified)

### 1. Parent with explicit `@Options` (subcommands as named exports)

```ts
// account/parent.ts
import { Command, Declare, Options } from 'seyfert';
import { CreateCommand } from './create.command';
import { DeleteCommand } from './delete.command';

@Declare({ name: 'account', description: 'account command' })
@Options([CreateCommand, DeleteCommand])
export default class AccountCommand extends Command {}
```

```ts
// account/create.command.ts
import { type CommandContext, Declare, SubCommand } from 'seyfert';

@Declare({ name: 'create', description: 'create a new something' })
export class CreateCommand extends SubCommand {
  run(ctx: CommandContext) {
    return ctx.write({ content: 'create command executed' });
  }
}
```

### 2. AutoLoad variant — drop `@Options`, scan the parent's folder

Subcommands MUST be `export default`; files without a default export are warned and skipped (`handler.ts:379-395`).

```ts
// account/parent.ts
import { AutoLoad, Command, Declare } from 'seyfert';

@Declare({ name: 'account', description: 'account command' })
@AutoLoad()
export default class AccountCommand extends Command {}
```

```ts
// account/create.command.ts  (default export required for AutoLoad)
import { type CommandContext, Declare, SubCommand } from 'seyfert';

@Declare({ name: 'create', description: 'create a new something' })
export default class CreateCommand extends SubCommand {
  run(ctx: CommandContext) {
    return ctx.write({ content: 'create command executed' });
  }
}
```

### 3. A subcommand with its OWN basic options (typed `ctx.options`)

A `SubCommand` carries basic options the same way a flat command does — via `@Options({...})` with `create*Option` helpers (`src/commands/applications/options.ts`). Type the context with `CommandContext<typeof options>` to get inferred `ctx.options`.

```ts
// economy/pay.command.ts
import {
  type CommandContext,
  createNumberOption,
  createUserOption,
  Declare,
  Options,
  SubCommand,
} from 'seyfert';

const options = {
  // keys MUST be lowercase (compile-time enforced in v5)
  to: createUserOption({ description: 'Who to pay', required: true }),
  amount: createNumberOption({ description: 'How much', required: true }),
};

@Declare({ name: 'pay', description: 'Send coins to another user' })
@Options(options)
export default class PayCommand extends SubCommand {
  run(ctx: CommandContext<typeof options>) {
    const target = ctx.options.to; // typed User
    const amount = ctx.options.amount; // typed number
    return ctx.write({
      content: `Sent ${amount} coins to ${target.username}.`,
    });
  }
}
```

### 4. Groups — declare on the parent, assign on each subcommand

```ts
// settings/parent.ts
import { Command, Declare, Groups, Options } from 'seyfert';
import GetSetting from './get';
import SetSetting from './set';

@Declare({ name: 'settings', description: 'Manage settings' })
@Options([GetSetting, SetSetting])
@Groups({
  config: { defaultDescription: 'Read or change configuration' },
})
export default class SettingsCommand extends Command {}
```

```ts
// settings/set.ts
import { type CommandContext, Declare, Group, SubCommand } from 'seyfert';

@Declare({ name: 'set', description: 'Change a config value' })
@Group('config')
export default class SetSetting extends SubCommand {
  run(ctx: CommandContext) {
    return ctx.write({ content: 'Setting updated.' });
  }
}
```

This produces the Discord shape `/settings config set` (group > subcommand).

### 5. Type-safe group names with `defineGroups` + `@Group(def, name)`

`defineGroups(...)` lets you declare the group map once and reuse it; the `@Group(def, name)` overload then type-checks `name` against `keyof def`, so a typo fails at compile time instead of throwing at build (the parent's `toJSON` does `this.groups![i.group].defaultDescription`).

```ts
// moderation/groups.ts
import { defineGroups } from 'seyfert';

export const modGroups = defineGroups({
  warns: { defaultDescription: 'Manage warnings', aliases: ['w'] },
  bans: { defaultDescription: 'Manage bans' },
});
```

```ts
// moderation/parent.ts
import { Command, Declare, Groups, Options } from 'seyfert';
import { modGroups } from './groups';
import AddWarn from './add-warn';

@Declare({ name: 'mod', description: 'Moderation tools' })
@Options([AddWarn])
@Groups(modGroups)
export default class ModCommand extends Command {}
```

```ts
// moderation/add-warn.ts
import { type CommandContext, Declare, Group, SubCommand } from 'seyfert';
import { modGroups } from './groups';

@Declare({ name: 'add', description: 'Warn a member' })
@Group(modGroups, 'warns') // 'warns' is checked against keyof modGroups
export default class AddWarn extends SubCommand {
  run(ctx: CommandContext) {
    return ctx.write({ content: 'Warning added.' });
  }
}
```

### 6. Guild-only subcommand with `inGuild()` narrowing

Inside `run`, narrow with `ctx.inGuild()` to get a `GuildCommandContext`: `guildId`/`member` become non-null and `channel()`/`guild()`/`me()`/`fetchMember()` return guild-typed values (`chatcontext.ts:260,265`).

```ts
// role/give.command.ts
import { type CommandContext, createRoleOption, Declare, Options, SubCommand } from 'seyfert';

const options = {
  role: createRoleOption({ description: 'Role to grant', required: true }),
};

@Declare({ name: 'give', description: 'Give yourself a role' })
@Options(options)
export default class GiveRole extends SubCommand {
  async run(ctx: CommandContext<typeof options>) {
    if (!ctx.inGuild()) {
      return ctx.write({ content: 'This command only works in a server.' });
    }
    // ctx is now GuildCommandContext: guildId/member are non-null
    const member = await ctx.fetchMember(); // GuildMemberStructure (not | undefined)
    await member.roles.add(ctx.options.role.id);
    return ctx.write({ content: `Gave you ${ctx.options.role.name}.` });
  }
}
```

### 7. Localized groups for i18n with `@GroupsT`

Use `@GroupsT` (+ `TranslatedGroupDefinition`) instead of `@Groups` when you want the group name/description pulled from your lang files. `name`/`description` are `FlatObjectKeys<DefaultLocale>` (dotted keys into your locale).

```ts
import { Command, Declare, GroupsT, Options } from 'seyfert';
import Today from './today';

@Declare({ name: 'weather', description: 'Weather utilities' })
@Options([Today])
@GroupsT({
  forecast: {
    defaultDescription: 'Forecast commands',
    name: 'groups.forecast.name',
    description: 'groups.forecast.description',
  },
})
export default class WeatherCommand extends Command {}
```

## Common patterns / gotchas

- Parent commands that hold subcommands must NOT define `run`. `SubCommand` is abstract with an abstract `run`, so each leaf must implement it; the parent has nothing to run.
- `@Group('x')` without a matching `@Groups`/`@GroupsT` entry for `'x'` throws at build time — `Command.toJSON()` reads `this.groups![i.group].defaultDescription` (`chat.ts:388`). Use the `defineGroups` + `@Group(def, name)` overload to catch the typo in the editor instead.
- Do NOT combine `@AutoLoad()` and manual `@Options([...])` on the same parent — pick one loading strategy.
- `@AutoLoad()` only picks up `export default` SubCommands from sibling files; a named-only export is logged (`no default export found ... ignoring it as a SubCommand`) and skipped. `@Options([Class])` works with either named or default exports because you import the class yourself.
- Option record keys must be lowercase in v5 (compile-time error otherwise) — applies to a subcommand's own `@Options({...})` too. `choices`/`channel_types` are readonly; use `as const`.
- `@Declare`'s `name` is forced lowercase at the type level; Discord rejects non-lowercase subcommand/group names anyway.
- Two nesting levels max: a `SubCommand` cannot contain another `SubCommand`. Want `/a b c`? `a` = Command, `b` = group (`@Groups` + `@Group`), `c` = subcommand.
- `ctx.write`/`ctx.editOrReply` return `void` unless you pass the response flag `true` (v5: then you get a message/webhook back) — `chatcontext.ts:88`.
- Per-subcommand lifecycle hooks (`onRunError`, `onMiddlewaresError`, `onOptionsError`, `onBeforeOptions`, etc.) live on `BaseCommand` and can be defined directly on a `SubCommand` class, scoped to that leaf.

## Doc vs Source Corrections

- Docs callout writes `@Autoload()` (lowercase L) — src exports `AutoLoad` (`src/commands/decorators.ts:177`). The correct decorator is `@AutoLoad()`.
- Docs example imports `CommandContext` as a value in the groups example; it is type-only — prefer `import { type CommandContext }` (avoids verbatim-module-syntax errors). Not a runtime bug.
- Docs only show `@Group` / `@Groups`. Src also provides `@GroupsT` + `TranslatedGroupDefinition`, the `defineGroups` helper, and the type-safe `@Group(groupsDef, name)` overload — none mentioned in docs.
- Docs imply `@Options([...])` and AutoLoad are interchangeable. Src distinction: `@Options([Class])` accepts named or default exports; `@AutoLoad()` only picks up `export default` SubCommands and warns/skips files without one (`src/commands/handler.ts:379-395`).
- Otherwise aligned: `SubCommand`, `@Options([SubCommand])`, `@Group`, `@Groups`, `defaultDescription`, and `ctx.write` all match source.

## Source Anchors

- `src/commands/applications/chat.ts` (`Command`, `SubCommand`, `BaseCommand`; `Command.toJSON` group/subcommand emission at :375-405; `ContextOptions` at :117)
- `src/commands/decorators.ts` (`@Declare`, `@Options`, `@Group`, `@Groups`, `@GroupsT`, `defineGroups`, `@AutoLoad`; group definition types :68-83)
- `src/commands/handler.ts` (`__autoload` scanning :369-401, default-export warnings :379-395)
- `src/commands/applications/chatcontext.ts` (`CommandContext`, `ctx.options`/`ctx.write`, `isChat`/`inGuild` guards :256-262, `GuildCommandContext` :265)
- `src/commands/applications/options.ts` (`create*Option` helpers for subcommand options)

## Agent Guidance

- Choose `@Options([...])` for explicit control (named exports OK) or `@AutoLoad()` for folder-convention loading (every sibling subcommand must be `export default`).
- Reach for `defineGroups` + `@Group(def, name)` whenever you use groups — it turns a runtime build crash (typo'd group name) into a compile error.
- Type a subcommand's own options by declaring the `options` object, decorating with `@Options(options)`, and writing `run(ctx: CommandContext<typeof options>)`.
- Narrow with `ctx.inGuild()` inside `run` before touching guild-only data (`member`, guild channels) to get a `GuildCommandContext` with non-null fields.
- For i18n group labels use `@GroupsT` + translation keys; for static labels use `@Groups` with `defaultDescription`.
- All decorators/classes import from `'seyfert'`; no deep imports needed for the APIs on this page.
