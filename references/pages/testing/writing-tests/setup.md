# Testing — Writing Tests: Setup

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/setup

Coverage reference: testing.md

Verification status: Source-verified (core seyfert APIs) + external package (`@slipher/testing`, doc-authoritative)

## Page Summary

Foundation of the "Writing Tests" guide. You install the external `@slipher/testing` dev dependency, stand up an in-process mock bot with `createMockBot(...)`, dispatch a real slash command with `bot.slash(...)`, and assert on the captured REST reply (`result.content`). No token, no gateway, no network: the mock turns your payload into a raw `APIInteraction`, runs it through Seyfert's `HandleCommand`, parses options with the real resolver, runs your middlewares, and records outgoing REST calls instead of sending them.

Split of responsibility:
- The **toolkit surface** (`createMockBot`, `bot.slash`, `bot.clickButton`, `bot.selectMenu`, `bot.emit`, `result.*`) is EXTERNAL and doc-authoritative — NOT in seyfert-core. Verify the installed version's API in the target project.
- The **seyfert classes it dispatches against** (`Command`, `Declare`, `Options`, `createStringOption`, `CommandContext`, `ctx.write`, `ctx.options`, `ActionRow`, `Button`, `createEvent`, `createComponentCollector`, `client.users.write`) are ALL verified in `./src` and run identically in production.

## Key APIs (verified)

Core seyfert APIs (root imports from `'seyfert'` via `src/index.ts -> export * from './commands'`):

- `Command` — `class Command extends BaseCommand` (`src/commands/applications/chat.ts:361`). User commands subclass it and implement `run`.
- `Declare(options)` — decorator (`src/commands/decorators.ts:195`). `{ name, description, ... }`; names validated lowercase via `LowercaseDeclareName` (v5: lowercase enforced at compile time).
- `Options(record | (new () => SubCommand)[])` — overloaded decorator (`src/commands/decorators.ts:161-163`). Options-record keys MUST be lowercase (v5).
- `createStringOption(data)` — option factory (`src/commands/applications/options.ts:146`). Returns `{ ...data, type: ApplicationCommandOptionType.String } as const`. Siblings: `createIntegerOption`, `createNumberOption`, `createBooleanOption`, `createChannelOption`, `createUserOption`, `createRoleOption`, `createMentionableOption`, `createAttachmentOption`. NOTE (more-qol): choices typed as `readonly SeyfertChoice<string>[]`; readonly option/choice arrays now accepted (commit 22eb832). Autocomplete choice `value` is typed to the option type (v5).
- `CommandContext<T extends OptionsRecord = {}, M = never>` — interface + class (`src/commands/applications/chatcontext.ts:36,45`). The `run(ctx: CommandContext<typeof options>)` parameter type.
- `ctx.write(body, withResponse?)` — `async write<WR extends boolean = false>(...)` (`chatcontext.ts:88`). Returns `void` UNLESS `withResponse` is `true` (v5).
- `ctx.options` — `ContextOptions<T>` (`chatcontext.ts:68`). Parsed values, e.g. `ctx.options.name`.
- `ctx.fetchResponse()` — `chatcontext.ts:163`. Resolves the sent message (used to attach a collector).
- `ActionRow<T>` (`src/builders/ActionRow.ts:16`), `Button` (`src/builders/Button.ts:9`), `ButtonStyle` — component builders for component tests.
- `createEvent({ data: { name }, run })` — `src/index.ts:68`. v5: `run` is `Awaitable<void>`; custom (non-gateway) handlers no longer receive a trailing `shardId`.
- `Message.createComponentCollector(options?)` (`src/structures/Message.ts:76`) and `client.users.write(userId, body)` (`src/common/shorters/users.ts:46`, DMs the user).

External toolkit API (doc-authoritative — verify version in target project; NOT in seyfert-core):

- `createMockBot(options)` from `@slipher/testing` — boots a real Seyfert client in-process; every REST call is recorded. Returns an `await using` disposable. Full option set (per toolkit reference): `commands`, `components`, `events`, `middlewares`, `globalMiddlewares`, `world`, `simulateGateway` (default `true`), `onUnhandledRest`, `plugins`, `clientOptions`, `loadFromConfig`, `commandsDir`/`componentsDir`/`eventsDir`/`langsDir`, `shards`/`shardLatency`, `botId`/`applicationId`, `prefixes`/`mentionAsPrefix`.
- `bot.slash(...)` — dispatch a slash interaction. Two forms: `bot.slash({ name, options })` (raw by-name) or `bot.slash(GreetCommand, { options })` (option inference from the class).
- `bot.clickButton(customId, { source? })`, `bot.selectMenu(customId, values, { source?, componentType? })` — dispatch components; default target is `bot.lastSentMessage()`.
- `bot.emit(name, payload, { allowNoHandler? })` — feed an UPPERCASE gateway event into real handlers.
- `result.*` — parsed view of captured output: `result.content`, `result.deferred`, `result.edits`, `result.followups`, `result.replies`, `result.embedView?.title`, `result.components`, `result.component('id')?.customId`, `result.textDisplays`; raw wire shape at `result.reply?.body` / `result.embeds`.

## Code Examples (verified)

Command under test — all imports verified as root `'seyfert'` exports:

```ts
// src/commands/greet.ts
import { Command, Declare, Options, createStringOption, type CommandContext } from 'seyfert';

const options = {
  name: createStringOption({ description: 'Who to greet', required: true }),
};

@Declare({ name: 'greet', description: 'Greet someone' })
@Options(options)
export class GreetCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    await ctx.write({ content: `Hello, ${ctx.options.name}!` });
  }
}
```

Mock bot from explicit classes (external API):

```ts
import { createMockBot } from '@slipher/testing';
import { GreetCommand } from '../src/commands/greet';

await using bot = await createMockBot({ commands: [GreetCommand] });
// components / events accept real classes the same way
```

Mock bot from config (external API). `loadFromConfig: true` boots through the same loaders `client.start()` uses, reading `seyfert.config`. Configs usually point at compiled output, so build first and run from a cwd where the config resolves:

```ts
await using bot = await createMockBot({ loadFromConfig: true });
await bot.slash({ name: 'ping' });
```

First test (Vitest + external toolkit):

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { GreetCommand } from '../src/commands/greet';

test('greet replies through the real pipeline', async () => {
  await using bot = await createMockBot({ commands: [GreetCommand] });

  const result = await bot.slash({ name: 'greet', options: { name: 'slipher' } });

  expect(result.content).toBe('Hello, slipher!');
});
```

`await using` disposes the bot automatically at the end of scope — no teardown needed.

## Recipes / Worked examples

### 1. Typed dispatch with option inference

`bot.slash({ name })` is concise but stringly-typed. Passing the command class gives option inference, so a wrong option name fails to type-check:

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { GreetCommand } from '../src/commands/greet';

test('greet — typed form', async () => {
  await using bot = await createMockBot({ commands: [GreetCommand] });
  const result = await bot.slash(GreetCommand, { options: { name: 'ada' } });
  expect(result.content).toBe('Hello, ada!');
});
```

### 2. Testing a button + collector flow

The command writes a message, fetches it, and registers a component collector. `bot.clickButton(customId)` dispatches the click against the last sent message — the collector picks it up and your handler replies for real. Builders (`ActionRow`, `Button`, `ButtonStyle`) and `ctx.fetchResponse()` / `createComponentCollector()` are all core seyfert:

```ts
// src/commands/poll.ts
import { ActionRow, Button, ButtonStyle, Command, Declare, type CommandContext } from 'seyfert';

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
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { PollCommand } from '../src/commands/poll';

test('clicking yes replies', async () => {
  await using bot = await createMockBot({ commands: [PollCommand] });

  await bot.slash({ name: 'poll' });
  const result = await bot.clickButton('poll/yes');

  expect(result.content).toBe('Voted!');
});
```

### 3. Testing a gateway event handler

Event handlers are real `createEvent(...)` classes registered via `events: [...]`. `bot.emit(NAME, payload)` feeds an UPPERCASE gateway event into the real pipeline and resolves the REST work it awaited, so you can assert immediately. `createEvent` and `client.users.write` are core seyfert:

```ts
// src/events/guildMemberAdd.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'guildMemberAdd' },
  async run(member, client) {
    await client.users.write(member.id, { content: `Welcome, ${member.user.username}!` });
  },
});
```

```ts
// tests/welcome.test.ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import guildMemberAdd from '../src/events/guildMemberAdd';

test('greets new members', async () => {
  await using bot = await createMockBot({ events: [guildMemberAdd] });

  await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload); // full Discord event shape

  expect(
    bot.world.query.dm({ userId: rawMemberPayload.user.id })?.lastMessage?.content,
  ).toContain('Welcome');
});
```

Note (v5): `createEvent.run` is `Awaitable<void>` and custom (non-gateway) handlers no longer get a trailing `shardId` — gateway-event handlers like `guildMemberAdd` are unaffected (`(payload, client)`).

### 4. loadFromConfig with explicit dirs

When the production set lives in `seyfert.config`, `loadFromConfig: true` boots through the real loaders. Explicit `*Dir` options override the config (handy when pointing at compiled `dist`):

```ts
import { join } from 'node:path';
import { createMockBot } from '@slipher/testing';

await using bot = await createMockBot({
  loadFromConfig: true,
  commandsDir: join(process.cwd(), 'dist/commands'),
});
```

### 5. Manual dispose (no `await using` support)

If your runner/TS target lacks explicit resource management, dispose manually — check the toolkit's API for the dispose method (it returns a disposable):

```ts
import { afterEach, beforeEach, expect, test } from 'vitest';
import { createMockBot } from '@slipher/testing';
import { GreetCommand } from '../src/commands/greet';

let bot: Awaited<ReturnType<typeof createMockBot>>;

beforeEach(async () => { bot = await createMockBot({ commands: [GreetCommand] }); });
afterEach(async () => { await bot[Symbol.asyncDispose]?.(); }); // verify the actual dispose API

test('greet', async () => {
  expect((await bot.slash({ name: 'greet', options: { name: 'x' } })).content).toBe('Hello, x!');
});
```

## Common patterns / gotchas

- **`await using` lifetime**: the bot disposes at end of scope. Inside a `test(...)` body that is exactly the test — no `afterEach` teardown needed. Fall back to manual dispose only where explicit resource management is unavailable (Recipe 5).
- **`loadFromConfig` reads compiled output**: bot configs usually point at `dist`, so build before config-loaded specs and run the test from a cwd where `seyfert.config` resolves. Use `*Dir` overrides to be explicit.
- **Event names are UPPERCASE for `emit`**: `bot.emit('GUILD_MEMBER_ADD', ...)`, never `'guildMemberAdd'`. `emit` fails loud when no registered handler ran (catches mis-casing / forgotten `events: [...]`). Pass `{ allowNoHandler: true }` only when emitting purely to seed world state. Note this is the raw gateway name; this is distinct from `createEvent({ data: { name: 'guildMemberAdd' } })`, which uses the camelCase client name.
- **Emit full Discord shapes, not partial patches** — the mock applies canonical events to world state; partial payloads leave the cache stale.
- **`result` is parsed, not raw**: prefer `result.content`, `result.deferred`, `result.edits`, `result.components`, `result.embedView?.title`. Reach for `result.reply?.body` / `result.embeds` only when the Discord wire shape is the contract.
- **Single test user by default** — dispatches correlate per-user automatically, so a `clickButton`/`selectMenu` after a `slash` lands on the same user. A click that never fires usually means a stale `source`; send a message first, then click `lastSentMessage()`.
- **Command errors don't reject**: Seyfert routes errors through its error hooks; assert on the user-facing error reply (or pass `onCommandError: 'capture'` to read `result.error`). (Toolkit behavior — verify in target version.)
- **Same code as production**: decorators, option factories, `CommandContext`, `ctx.write`, `ctx.options`, builders, collectors, events are standard Seyfert. Don't introduce test-only command shapes.

## Doc vs Source Corrections

- None for the core seyfert APIs — every import, decorator, builder, and method in the doc examples resolves exactly as shown in `./src` and matches the v5 checklist (lowercase option keys, `ctx.write` returns `void` unless `withResponse: true`, `createEvent.run` is `Awaitable<void>`).
- The `@slipher/testing` surface (`createMockBot` + all options, `bot.slash`/`clickButton`/`selectMenu`/`emit`, `result.*`, `bot.world`) is NOT in seyfert-core and cannot be source-verified; treat the MDX as authoritative and confirm the installed version's API in the target project.
- Freshness note (not a doc error): on `more-qol`, `createStringOption` choices and `Options` arrays now accept `readonly`; doc examples type-check unchanged.

## Source Anchors

- `src/index.ts` (root barrel — `export * from './commands'`; `createEvent` at `:68`)
- `src/commands/applications/chat.ts` (`Command`, `SubCommand`)
- `src/commands/decorators.ts` (`Declare`, `Options`)
- `src/commands/applications/options.ts` (`createStringOption` and siblings)
- `src/commands/applications/chatcontext.ts` (`CommandContext`, `write`, `options`, `fetchResponse`)
- `src/builders/ActionRow.ts`, `src/builders/Button.ts` (component builders)
- `src/structures/Message.ts:76` (`createComponentCollector`), `src/common/shorters/users.ts:46` (`users.write`)

## Agent Guidance

- Use this page when a user wants to unit/integration test Seyfert commands/components/events without a live gateway. The mechanism is the external `@slipher/testing` package — always have the user install it as a dev dependency (`pnpm add -D @slipher/testing`) and verify its installed version, since its API is not pinned by seyfert-core.
- The command/component/event-authoring half is fully standard Seyfert; the same code runs in production. Don't introduce test-only shapes.
- Do not invent `createMockBot` options or `result`/`bot` methods beyond those listed (sourced from the toolkit reference). When unsure, point the user at The Toolkit > Mock bot / Dispatching reference pages.
- For deeper flows, route users to sibling pages: `writing-tests/commands`, `writing-tests/components` (buttons/selects/collectors), `writing-tests/events`, `writing-tests/modals`, and `toolkit/*` (mock-bot, dispatching, world, gateway, assertions).
