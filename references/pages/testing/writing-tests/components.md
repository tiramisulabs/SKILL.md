# Testing Components (Buttons & Select Menus)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/components
Coverage reference: testing.md
Verification status: Source-verified (core, the authoritative Seyfert source) + external package (@slipher/testing)

## Page Summary

Shows how to drive Seyfert component handlers in tests with the external `@slipher/testing`
toolkit: dispatch button clicks via `bot.clickButton(customId)` and select choices via
`bot.selectMenu(customId, values)`, both flowing through the real component pipeline so
collectors AND global `ComponentCommand` handlers run exactly as in production. The toolkit API
(`createMockBot`, `clickButton`, `selectMenu`, `lastSentMessage`, `source`/`componentType`/
`resolved` options, `selectMenuInteraction`/`dispatchInteraction` for the no-collector path) is
external and doc-authoritative — verify its version in the target project. The Seyfert-core APIs
it exercises (builders, `createComponentCollector().run`, `ComponentCommand`, `ctx.write`,
`ctx.fetchResponse`, `ctx.update`) are verified below against `./src`.

## Key APIs (verified)

Core Seyfert APIs the examples build on (all root-importable from `'seyfert'`):

- `new ActionRow<Button>().addComponents(...)` — `addComponents(...component: RestOrArray<FixedComponents<T>>): this`, accepts spread or array (src/builders/ActionRow.ts).
- `new Button()` with chainable `.setCustomId(id)`, `.setStyle(style)`, `.setLabel(label)`, `.setURL`, `.setEmoji`, `.setDisabled`, `.setSKUId` (src/builders/Button.ts).
- `new StringSelectMenu()` — `.setCustomId(id)`, `.setPlaceholder(p)`, `.setValuesLength({ min, max })`, `.setOptions(...opts)` / `.addOption(...opts)` take REST params (v5: not an array) of `StringSelectOption | APISelectMenuOption` (src/builders/SelectMenu.ts:303,293). `new StringSelectOption().setLabel().setValue()`. A typed `StringSelectMenu<'a'|'b'>` rejects out-of-union raw values.
- `ButtonStyle`, `ComponentType` — enums re-exported from `'seyfert'` via `./types`.
- `Command`, `Declare`, `CommandContext`, `ComponentCommand`, `ComponentContext` — root exports (commands + components barrels).
- `message.createComponentCollector(options?)` on Message structure — delegates to `client.components.createComponentCollector(id, channelId, guildId, options)` (src/structures/Message.ts).
- Collector object returns `{ run, stop, waitFor, resetTimeouts }`. `run(customId, callback)` registers a handler; `customId` is matched via `createMatchCallback` (string, array, or RegExp); `callback` receives the `ComponentInteraction` (src/components/handler.ts:115-172). `waitFor(customId, timeout?)` resolves to the interaction or `null`.
- `ctx.write(body, fetchReply?)` and `ctx.fetchResponse()` exist on `CommandContext` and `ComponentContext` (src/components/componentcontext.ts:96,144). `write`/`editOrReply` return `void` unless `fetchReply === true`.
- `ComponentContext` (v5): `.update(body)`, `.deferUpdate()`, `.message` getter, `.customId`, `.followup`, `.modal`. **NO `editResponse`** (removed in v5 — use `editOrReply(body, true)`). `isButton()` etc. read `interaction.componentType`. Third generic `StringSelectValues` types `interaction.values`.

### `ComponentCommand` (verified, src/components/componentcommand.ts)

Global, file-based component handler (alternative to collectors — survives restarts, no message-scoped lifetime):

- `abstract componentType: keyof ContextComponentCommandInteractionMap` — `'Button' | 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'`.
- `customId?: string | RegExp` — matched against the interaction's customId.
- `filter?(ctx): boolean | Promise<boolean>` — extra gate after customId match.
- `abstract run(ctx: ComponentContext<this['componentType']>): any`.
- `middlewares: readonly (keyof ResolvedRegisteredMiddlewares)[] = []` (readonly tuple).
- Hooks: `onBeforeMiddlewares`, `onMiddlewaresError(ctx, error, metadata)`, `onRunError(ctx, error)`, `onAfterRun(ctx, error)`, and `onInternalError(client, component, error?)` — note the **`component` param before `error`** (v5).

External toolkit APIs (from `@slipher/testing`, NOT in core Seyfert — doc-authoritative, verify version in target project):

- `createMockBot({ commands: [...] })` → disposable bot (`await using`).
- `bot.slash({ name })`, `bot.clickButton(customId, opts?)`, `bot.selectMenu(customId, values, opts?)`, `bot.lastSentMessage()`.
- Dispatch options: `{ source }` (message object or id; defaults to `lastSentMessage()`), `{ componentType }` (`'user' | 'role' | 'mentionable' | 'channel'` for entity selects), `{ resolved }` (raw payload override).
- `selectMenuInteraction()` + `dispatchInteraction()` for a `ComponentCommand` select path with no collector.
- Every dispatch defaults to a single test user, so per-user state correlates across calls.

## Code Examples (verified)

### 1. Command + collector (the canonical flow)

```ts
// src/commands/poll.ts
import { ActionRow, Button, ButtonStyle, Command, type CommandContext, Declare } from 'seyfert';

@Declare({ name: 'poll', description: 'Open a poll' })
export class PollCommand extends Command {
  async run(ctx: CommandContext) {
    const row = new ActionRow<Button>().addComponents(
      new Button().setCustomId('poll/yes').setStyle(ButtonStyle.Success).setLabel('Yes'),
    );
    await ctx.write({ content: 'Vote now', components: [row] });
    const message = await ctx.fetchResponse();
    message.createComponentCollector().run('poll/yes', async interaction => {
      await interaction.write({ content: 'Voted!' });
    });
  }
}
```

```ts
// tests/poll.test.ts
import { createMockBot } from '@slipher/testing'; // external; verify version in target project
import { expect, test } from 'vitest';
import { PollCommand } from '../src/commands/poll';

test('clicking yes replies', async () => {
  await using bot = await createMockBot({ commands: [PollCommand] });

  await bot.slash({ name: 'poll' });
  const result = await bot.clickButton('poll/yes'); // defaults to lastSentMessage()

  expect(result.content).toBe('Voted!');
});
```

### 2. Target a specific message

```ts
const sent = bot.lastSentMessage();
await bot.clickButton('approve', { source: sent?.id });
```

### 3. String select menu (builder + test)

```ts
// src/commands/settings.ts
import { ActionRow, Command, type CommandContext, Declare, StringSelectMenu, StringSelectOption } from 'seyfert';

@Declare({ name: 'settings', description: 'Bot settings' })
export class SettingsCommand extends Command {
  async run(ctx: CommandContext) {
    const menu = new StringSelectMenu()
      .setCustomId('settings/theme')
      .setPlaceholder('Pick a theme')
      .setOptions( // v5: rest params, not an array
        new StringSelectOption().setLabel('Dark').setValue('dark'),
        new StringSelectOption().setLabel('Light').setValue('light'),
      );
    await ctx.write({ content: 'Choose', components: [new ActionRow<StringSelectMenu>().addComponents(menu)] });
    const message = await ctx.fetchResponse();
    message.createComponentCollector().run('settings/theme', async interaction => {
      // interaction.values is string[]
      await interaction.update({ content: `Theme set to ${interaction.values[0]}` });
    });
  }
}
```

```ts
test('selecting dark updates', async () => {
  await using bot = await createMockBot({ commands: [SettingsCommand] });
  await bot.slash({ name: 'settings' });
  const result = await bot.selectMenu('settings/theme', ['dark']);
  expect(result.content).toBe('Theme set to dark');
});
```

### 4. Entity select (needs `componentType`; ids auto-resolve)

```ts
const sent = bot.lastSentMessage();
await bot.selectMenu('settings/mod', [role.id], { source: sent, componentType: 'role' });
// handler reads a fully-resolved role via interaction.roles — no manual payload building
```

### 5. Global `ComponentCommand` (no collector) + test

A `ComponentCommand` lives in a file the handler auto-loads; it matches by `customId` and is not
bound to a single message — ideal for buttons whose handlers must survive restarts.

```ts
// src/components/confirm.ts
import { type ComponentContext, ComponentCommand } from 'seyfert';

export default class ConfirmButton extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'confirm'; // or a RegExp, e.g. /^ticket\/\d+$/

  filter(ctx: ComponentContext<'Button'>) {
    return ctx.interaction.user.id === ctx.message.interactionMetadata?.user.id; // author-only
  }

  async run(ctx: ComponentContext<'Button'>) {
    await ctx.update({ content: 'Confirmed!', components: [] });
  }
}
```

```ts
// tests/confirm.test.ts — no collector: build the interaction and dispatch it directly
import { createMockBot, selectMenuInteraction, dispatchInteraction } from '@slipher/testing'; // external
import { expect, test } from 'vitest';

test('confirm button updates the message', async () => {
  await using bot = await createMockBot({ commands: [/* slash that posts the button */] });
  await bot.slash({ name: 'open' });
  const result = await bot.clickButton('confirm'); // dispatch hits the global ComponentCommand
  expect(result.content).toBe('Confirmed!');
});
```

### 6. `waitFor` instead of `run` (single awaited interaction)

```ts
// inside a command after writing a message with a 'confirm' button
const collector = message.createComponentCollector({ timeout: 15_000 });
const interaction = await collector.waitFor('confirm', 15_000); // null on timeout
if (!interaction) return ctx.editOrReply({ content: 'Timed out' });
await interaction.update({ content: 'Got it' });
```

## Common patterns / gotchas

- **Send before you dispatch.** `clickButton`/`selectMenu` default `source` to `lastSentMessage()`.
  If nothing was sent (or the wrong message is last), the dispatch lands nowhere. Send a real
  message first; pass `source` explicitly when a flow emits several messages.
- **Same user.** Every dispatch uses one test user. If a collector has a `filter` keyed on user id,
  it sees that same user — so author-only guards pass naturally in tests.
- **`update` vs `write`.** Use `interaction.update(...)` (or `ctx.update`) to EDIT the source
  message in place; `write(...)` sends a NEW reply. Tests assert on whichever your handler called —
  `bot.clickButton(...)` returns the parsed reply/update body.
- **No `editResponse` on ComponentContext (v5).** It was removed. Use `editOrReply(body, true)` to
  get the `Message` back, or `update(...)` to edit the component message.
- **`setOptions` takes rest params (v5).** `new StringSelectMenu().setOptions(a, b)`, not
  `setOptions([a, b])`. Same for `RadioGroup.setOptions(...)`.
- **Collectors run ALL matching handlers (v5).** The old first-match `break` is gone — if two
  collectors on the same message match a customId, both fire. Tighten `customId`/`filter` if you
  relied on first-match-wins.
- **Entity selects need `componentType`.** Without it the mock can't resolve `[id]` into a role/
  user/channel. Pass `{ componentType: 'role' }` (etc.); ids resolve against seeded world entities.
- **No-collector handlers** (`ComponentCommand`) are reached by the same `clickButton`/`selectMenu`
  dispatch as collectors — they go through the real component handler. For a raw payload assertion,
  build it with `selectMenuInteraction()` and feed `dispatchInteraction()`.

## Doc vs Source Corrections

- No corrections for core APIs — every Seyfert symbol the page uses (`ActionRow`, `Button`,
  `ButtonStyle`, `Command`, `Declare`, `CommandContext`, `createComponentCollector().run`,
  `ctx.write`, `ctx.fetchResponse`, `interaction.write`/`update`) is verified accurate against
  `more-qol` src.
- Note (not a doc error): the MDX imports `ButtonStyle` in a second statement; it can be merged
  into the single `'seyfert'` import (shown in example 1). Both work.
- Added (not in MDX, verified against src): `ComponentCommand` global-handler example, the
  `StringSelectMenu` builder with v5 rest-param `setOptions`, the `waitFor` collector variant, and
  the `update` vs `write` distinction.
- `@slipher/testing` toolkit surface (`createMockBot`, `clickButton`, `selectMenu`,
  `lastSentMessage`, `selectMenuInteraction`, `dispatchInteraction`, `source`/`componentType`/
  `resolved`) cannot be verified here — not in core Seyfert. Treat the MDX as authoritative and
  pin/verify the installed version in the consuming project.

## Source Anchors

- src/index.ts (barrel exports: builders, commands, components, types)
- tests/plugin-authoring-contract.ts (confirms `@slipher/testing` is NOT a core Seyfert dep)

## Agent Guidance

- Use this page when writing Vitest/component tests for a Seyfert bot. The component pipeline
  (collectors, `ComponentCommand` handlers) is real core Seyfert; only the dispatch harness is
  external.
- Two handler models: (1) message-scoped **collector** via `createComponentCollector().run(id, cb)`
  (`id` = string | array | RegExp; also `waitFor`, `stop`, `resetTimeouts`, options `timeout`,
  `idle`, `filter`, `onPass`, `onStop`, `onError`); (2) global **`ComponentCommand`** file matched
  by `customId` + `filter`. Both are dispatched the same way by the toolkit.
