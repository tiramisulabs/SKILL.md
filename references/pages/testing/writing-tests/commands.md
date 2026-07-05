# Testing Commands

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/commands
Coverage reference: testing.md
Verification status: Source-verified (core seyfert APIs) + EXTERNAL package (`@slipher/testing`, doc-authoritative)

## Page Summary

A walkthrough for testing Seyfert commands end-to-end with the external `@slipher/testing`
toolkit: dispatch a slash command through the real pipeline (`HandleCommand`, option parsing,
middlewares), assert the reply and REST effects, run dispatches as specific users, populate
entity options, exercise autocomplete, prefix commands, i18n, and permission denials. The
toolkit (`createMockBot`, `bot.slash`, `outcome`, `apiUser`, etc.) is NOT part of core Seyfert —
treat its API as doc-authoritative and verify the version in the target project. The
command-side surface it drives (Command, decorators, option creators, CommandContext, the bans
shorter, middleware `stop()`, permission-fail / error hooks) IS verified against `./src` below.

## Key APIs (verified)

Core seyfert APIs that the test toolkit dispatches against — all importable from `'seyfert'`:

- `Command` (abstract base) — `src/commands/applications/chat.ts`.
- `@Declare(input)` — decorator, name is lowercased at the type level (`LowercaseDeclareName`); `src/commands/decorators.ts:195`.
- `@Options(options)` — accepts a lowercase options record OR a `(new () => SubCommand)[]` array; `src/commands/decorators.ts:161-163`. Option arrays may be `readonly` (v5).
- `@Middlewares(cbs)` — accepts a `readonly MiddlewareKey[]`; `src/commands/decorators.ts:188`. `command.middlewares` is a readonly tuple (no `.push`).
- `@AutoLoad()` — `src/commands/decorators.ts:177`.
- Option creators in `src/commands/applications/options.ts`:
  `createStringOption` (146), `createIntegerOption` (154), `createNumberOption` (162),
  `createChannelOption` (170), `createBooleanOption` (178), `createUserOption` (184),
  `createRoleOption` (188), `createMentionableOption` (192), `createAttachmentOption` (199).
  Option record keys MUST be lowercase (compile-time); autocomplete choice `value` is typed to the option type.
- `CommandContext<typeof options>` — `src/commands/applications/chatcontext.ts`. Provides:
  - `ctx.options` — parsed/resolved options.
  - `ctx.write(body, withResponse?)` — `chatcontext.ts:88` (interaction.write, or message reply for prefix).
  - `ctx.editOrReply(body, withResponse?)` — `chatcontext.ts:145`; `ctx.editResponse(body)` — `chatcontext.ts:126` (EXISTS on CommandContext).
  - `ctx.deferReply(ephemeral?, withResponse?)` — `chatcontext.ts:107`; `ctx.modal(...)` — `chatcontext.ts:99-104` (throws on prefix: no interaction); `ctx.followup(...)` — `chatcontext.ts:156`.
  - `ctx.t` — i18n accessor returning the typed `SeyfertLocale` proxy (`chatcontext.ts:72`).
  - `ctx.client` (UsingClient), `ctx.guildId`, `ctx.author`, `ctx.channelId`.
  - `ctx.inGuild()` narrows to `GuildCommandContext` (channel()/fetchMember() narrowed).
  - `write`/`editOrReply` return `void` UNLESS the response flag is `true` (then a Message/WebhookMessage).
- `ctx.client.bans.create(guildId, memberId, options?)` — `BanShorter.create`, `src/common/shorters/bans.ts:56`. Signature `(guildId: string, memberId: string, options?: BanOptions)`; v5 `BanOptions` is `{ deleteMessageSeconds?, reason? }` (camelCase), serialized to raw `delete_message_seconds` for REST by `resolveBanOptions`.
- Middleware: `createMiddleware(({ context, next, stop }) => ...)` — `src/commands/applications/shared.ts:55`. NO `pass` (removed). `stop()`/`stop(null)` = silent skip; `stop('reason')` = deny → `onMiddlewaresError`.
- Command hooks (`src/commands/applications/chat.ts:349-355`): `onBeforeMiddlewares(ctx)`, `onRunError(ctx, error)`, `onOptionsError(ctx, metadata)`, `onMiddlewaresError(ctx, error, metadata)`, plus `onPermissionsFail(ctx, permissions)` / `onBotPermissionsFail(ctx, permissions)` (dispatched in `src/commands/handle.ts`, defaults in `src/commands/handler.ts`).

External toolkit API (doc-authoritative, verify version in target project): `createMockBot`,
`bot.slash`, `bot.say`, `bot.autocomplete`, `bot.actor`, `bot.waitForAction`, `bot.clickButton`,
`bot.defaultUser`, `apiUser`, `userOption`, `outcome`, `Routes`, `registerBotMember`.

## Code Examples (verified)

The command under test (core APIs verified; imports from `'seyfert'`):

```ts
// src/commands/ban.ts
import { Command, Declare, Options, createUserOption, createStringOption, type CommandContext } from 'seyfert';

const options = {
  user: createUserOption({ description: 'Member to ban', required: true }),
  reason: createStringOption({ description: 'Why' }),
};

@Declare({ name: 'ban', description: 'Ban a member' })
@Options(options)
export class BanCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    await ctx.client.bans.create(ctx.guildId!, ctx.options.user.id, { reason: ctx.options.reason });
    await ctx.write({ content: `Banned ${ctx.options.user.username}.` });
  }
}
```

Dispatch a slash command and assert the reply (toolkit API — verify version):

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

- `result.content` = parsed reply text; `result.messages` = message-create calls;
  `result.reply?.body` = raw `{ type, data }` wire payload; typed views like
  `result.embedView?.title` and `result.component('Approve')?.customId`.
- `bot.slash(GreetCommand, { options })` infers options from the class; `bot.slash({ name, ... })` is by-name.

Assert REST effects (each action carries `{ seq, method, route, body, query, response }`):

```ts
import { Routes } from '@slipher/testing';
const ban = await bot.waitForAction(Routes.ban);
// NOTE: action.body is the RAW Discord wire body (snake_case), even though
// BanOptions is camelCase in your command code — resolveBanOptions converts it.
expect(ban.body).toMatchObject({ delete_message_seconds: 0 });
```

Run as a specific user / bind an actor for multi-step journeys:

```ts
import { apiUser } from '@slipher/testing';
await bot.slash({ name: 'profile', user: apiUser({ id: '99', username: 'mara' }) });

const alice = bot.actor({ member: aliceMember, guildId: guild.id, channel });
await alice.slash({ name: 'poll' });
await alice.clickButton('poll/yes');
expect((await alice.slash({ name: 'results' })).content).toContain('1 vote');
```

Entity options + autocomplete:

```ts
import { apiUser, userOption } from '@slipher/testing';
await bot.slash({ name: 'ban', options: { user: userOption(apiUser({ id: '42' })), reason: 'spam' } });

const result = await bot.autocomplete({ name: 'search', focused: 'query', value: 'sey' });
expect(result.choices).toEqual([{ name: 'result:sey', value: 'sey' }]);
```

Prefix commands + i18n (flag-style args via default parser; messages populate `result.messages`):

```ts
await using bot = await createMockBot({ commands: [EchoCommand], prefixes: ['!'] });
expect((await bot.say('!echo -text hello')).content).toBe('echo: hello');

await using lbot = await createMockBot({
  commands: [HelloCommand],
  langs: { 'en-US': { greeting: 'Hello!' }, 'es-ES': { greeting: '¡Hola!' } },
  defaultLang: 'en-US',
});
expect((await lbot.slash({ name: 'hello' })).content).toBe('Hello!');
expect((await lbot.slash({ name: 'hello', locale: 'es-ES' })).content).toBe('¡Hola!');
```

Permission denials (drives `onPermissionsFail`; restricting bot `permissions` drives `onBotPermissionsFail`):

```ts
import { createMockBot, outcome } from '@slipher/testing';
const result = await bot.slash({ name: 'ban', memberPermissions: [] });
outcome(result).get.denial({ kind: 'permissions', missing: 'BanMembers' });
// world-seeded faithful path: compute perms from roles/overwrites
await bot.slash({ name: 'ban', guildId: guild.id, channel: denied, user: member.user });
```

`get.denial(...)` throws unless denied for exactly that reason (doubles as assertion). For
unhandled errors, build with `createMockBot({ onCommandError: 'capture' })` and read via
`outcome(result).get.error(...)`. After `registerBotMember(...)`, moderation REST routes
enforce computed permissions/hierarchy and return Discord `403 50013`.

## Recipes / Common patterns (verified core, external toolkit)

### Test an embed + button reply, then assert the typed view

```ts
// src/commands/confirm.ts
import { Command, Declare, ActionRow, Button, Embed, type CommandContext } from 'seyfert';
import { ButtonStyle } from 'seyfert';

@Declare({ name: 'confirm', description: 'Ask to confirm' })
export class ConfirmCommand extends Command {
  async run(ctx: CommandContext) {
    const row = new ActionRow<Button>().addComponents(
      new Button().setCustomId('confirm/yes').setStyle(ButtonStyle.Success).setLabel('Approve'),
    );
    await ctx.write({ embeds: [new Embed().setTitle('Deploy?')], components: [row] });
  }
}
```

```ts
test('confirm shows the embed and Approve button', async () => {
  await using bot = await createMockBot({ commands: [ConfirmCommand] });
  const result = await bot.slash({ name: 'confirm' });
  expect(result.embedView?.title).toBe('Deploy?');
  expect(result.component('Approve')?.customId).toBe('confirm/yes');
});
```

### Test a command guarded by a middleware that denies with `stop('reason')`

```ts
// src/middlewares/onlyOwner.ts
import { createMiddleware } from 'seyfert';
export const onlyOwner = createMiddleware<void>(({ context, next, stop }) => {
  if (context.author.id !== process.env.OWNER_ID) return stop('Owner only.'); // deny → onMiddlewaresError
  return next();
});
```

```ts
// the denial surfaces through onMiddlewaresError; the toolkit reports it as an error outcome
test('non-owner is blocked', async () => {
  await using bot = await createMockBot({ commands: [AdminCommand], onCommandError: 'capture' });
  const result = await bot.slash({ name: 'admin', user: apiUser({ id: 'not-owner' }) });
  // assert the user-facing effect: the command did not produce its normal reply
  expect(result.content).toBeUndefined();
});
```

Note: `stop()` with no argument (v5) skips silently — no `onMiddlewaresError`, command body never runs.
The old `pass()` is gone. `onMiddlewaresError(ctx, error, metadata)` receives `metadata.middleware`
naming the middleware that actually called `stop()`.

### Test an autocomplete option callback in isolation

```ts
// src/commands/search.ts  (option with an autocomplete handler)
import { createStringOption } from 'seyfert';
const options = {
  query: createStringOption({
    description: 'Search term',
    required: true,
    autocomplete(interaction) {
      const value = interaction.getInput(); // current focused value
      return interaction.respond([{ name: `result:${value}`, value }]);
    },
  }),
};
```

```ts
test('search autocomplete echoes the focused value', async () => {
  await using bot = await createMockBot({ commands: [SearchCommand] });
  const r = await bot.autocomplete({ name: 'search', focused: 'query', value: 'sey' });
  expect(r.choices).toEqual([{ name: 'result:sey', value: 'sey' }]);
});
```

### Test i18n output via `ctx.t` without hardcoding strings

```ts
// HelloCommand: await ctx.write({ content: ctx.t.greeting.get() });
test('greeting resolves per locale', async () => {
  await using bot = await createMockBot({
    commands: [HelloCommand],
    langs: { 'en-US': { greeting: 'Hello!' }, 'es-ES': { greeting: '¡Hola!' } },
    defaultLang: 'en-US',
  });
  expect((await bot.slash({ name: 'hello', locale: 'es-ES' })).content).toBe('¡Hola!');
});
```

`ctx.t` is the typed `SeyfertLocale` proxy; resolve a leaf with `.get(locale?)`. The dispatcher's
`locale` option feeds the same real resolution path, so you test the wiring, not a stub.

### Gotchas

- Prefix commands do NOT use interaction responses — assert on `result.messages` (or `result.content` parsed from them), not `result.reply`.
- `ctx.modal(...)` throws `Cannot use modal without an interaction.` on prefix dispatches — only reachable via `bot.slash`/component dispatches.
- Use `userOption(apiUser(...))` (not a bare id) for entity options so `resolved` populates and `ctx.options.user` is a real entity.
- REST action `body` is raw snake_case Discord wire shape; your command code uses camelCase `BanOptions` — assert on the wire shape (`delete_message_seconds`) when reading `action.body`.
- Set `onCommandError: 'capture'` explicitly when you intend to assert thrown errors via `outcome(result).get.error(...)`; otherwise errors propagate.
- v5 middleware uses `stop()` (silent) / `stop('reason')` (deny) — there is no `pass()`. `command.middlewares` and `@Options([...])` arrays are readonly.
- `write`/`editOrReply` return `void` unless you pass the response flag `true` — don't assert on a returned Message without it.

## Doc vs Source Corrections

- None for the core APIs. Decorator name/option-record signatures, all option creators,
  `ctx.write`, `ctx.editOrReply`, `ctx.t`, `ctx.client.bans.create(guildId, memberId, options?)`,
  and the `onPermissionsFail` / `onBotPermissionsFail` / `onMiddlewaresError` hooks all match the doc examples.
- Toolkit example uses `delete_message_seconds: 0` in the REST body — correct: that is the raw
  Discord wire body (`resolveBanOptions` converts camelCase `BanOptions` to snake_case before REST).
- The toolkit surface (`createMockBot`, `bot.*`, `outcome`, `apiUser`, `Routes`,
  `registerBotMember`) could not be verified — it lives in the external `@slipher/testing`
  package, not in core Seyfert. Verify its exact signatures and version in the target project.

## Source Anchors

- `src/client/base.ts` (bans shorter wiring)
- `src/index.ts` / `src/commands/index.ts` (root + command barrel exports)

## Agent Guidance

- Use this page when writing automated tests for Seyfert commands. The toolkit is EXTERNAL:
  before relying on any `bot.*` / `outcome` / `apiUser` shape, check the version installed in
  the user's project (`@slipher/testing`) — its API may have drifted from these docs.
- The command code you test is plain seyfert: decorators, option creators, CommandContext, hooks.
  Those are stable and verified here; if a test fails, suspect the toolkit binding or a recent
  core change, not these signatures.
- Reach for `bot.actor(...)` for multi-step flows (poll → click button → results) so later
  dispatches see earlier in-process state; use `outcome(result).get.denial(...)` / `.get.error(...)`
  to assert intent instead of digging through the raw action list.
