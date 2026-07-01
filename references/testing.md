# Testing Seyfert v5 Bots

Scope: how to test a Seyfert bot. The recommended toolkit is **`@slipher/testing`** — an
**EXTERNAL** package that is **NOT part of seyfert-core**. Its API (`createMockBot`, `bot.*`,
`outcome`, `rendered`, `mockCommandContext`, `mockWorld`, `TEST_*`, `apiUser`, …) is
**doc-authoritative only**: pin and verify the installed version in the target project before
relying on exact signatures (it is pre-1.0 and marks parts unstable). Everything the toolkit
*dispatches against* (commands, events, components, modals, contexts, cache, gateway, builders,
SeyfertError, middleware) is real seyfert-core and is verified below against `./src`.

## Is the toolkit even installed?

- Check `package.json` / the registry. If `@slipher/testing` is absent, fall back to focused unit
  tests with the runner's own mocks + `tsc` type checks. Do not invent it.
- It bundles **no test runner** — bring Vitest / Jest / `node:test`. Requires Seyfert v5 peer dep.
- Install (per docs): `pnpm add -D @slipher/testing`.
- **Contributing to seyfert-core itself uses plain Vitest** (`tests/*.test.mts`, config
  `tests/vitest.config.mts`); `@slipher/testing` is NOT a dependency of this repo. For core
  changes: `pnpm run build`, type-contract scripts, then `pnpm test`. Repo Vitest disables file
  parallelism/isolation (see config below) — follow that even though the toolkit docs say parallel
  files are safe.

```ts
// tests/vitest.config.mts (this repo's own pattern — share module state, run files serially)
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { fileParallelism: false, isolate: false } });
```

## Two layers

1. **Mock bot** (`createMockBot`) — boots a *real* Seyfert `Client` in-process (no token/gateway/
   network), drives your real classes through the actual pipeline (`HandleCommand` → option
   resolver → middlewares → handler), and **records every REST call instead of sending it**. Use
   for real behavior: option parsing, middlewares, permissions, components, modals, REST, events,
   world/cache, plugins.
2. **Fixtures** (`mockCommandContext`, factories) — plain mock objects, no client boot. Call a
   single `run()` body directly and assert on a recorded `responses` array. Use for pure `run()`
   logic. Rule of thumb: pure `run()` → fixtures; anything else → mock bot. Both share one import.

## Import policy

- Your command/event/component/modal code uses **root `'seyfert'` imports** — the exact same code
  runs in production; never write test-only command shapes. Verified root exports include: `Command`,
  `SubCommand`, `Declare`, `Options`, `Middlewares`, `AutoLoad`, `Group`, `create*Option`,
  `createMiddleware`, `middlewares`, `CommandContext`, `ComponentCommand`, `ComponentContext`,
  `ModalCommand`, `ModalContext`, `createEvent`, `ActionRow`, `Button`, `ButtonStyle`,
  `StringSelectMenu`, `StringSelectOption`, `ChannelSelectMenu`, `ComponentType`, `Modal`, `Label`,
  `TextInput`, `TextInputStyle`, `Embed`, `PollBuilder`, `InteractionResponseType`, `SeyfertError`,
  `CacheFrom`, `ActivityType`, `GatewayOpcodes`, `PresenceUpdateStatus`, `definePlugins`.
- Toolkit symbols + `TEST_*` ids come from **`'@slipher/testing'`**, never from `'seyfert'`.
- **`Routes` is NOT a runtime value export of `seyfert`** (re-verified against the authoritative Seyfert source:
  `src/api/Routes/index.ts` declares only `interface APIRoutes` + route-type interfaces; `cdn.ts`
  exports only `interface CDNRoute` / `type UserAvatarDefault`; `grep` finds no `export const/function
  Routes`). The runtime route proxy is per-client `client.proxy` (`src/api/Router.ts`, getter at
  `src/api/api.ts:139`). So a `Routes.ban` matcher must come from `@slipher/testing` (or
  `discord-api-types`) — **not** `import { Routes } from 'seyfert'` — or use a predicate over the
  recorded action body. (Some upstream pages wrongly call `Routes` a `seyfert` root value; ignore them.)
- `HandleCommand` (the dispatcher) is also **not** on the root barrel — `src/commands/index.ts`
  re-exports only `type CommandFromContent` from `./handle`. Deep-import `seyfert/lib/commands/handle`
  if hand-wiring a harness (the toolkit does this internally; `HandleCommand` class at `handle.ts:64`,
  prefix path at `:368`).

## Mock bot — setup & options

```ts
import { createMockBot } from '@slipher/testing';
await using bot = await createMockBot({ commands: [GreetCommand] }); // auto-disposes at scope end
```

`await using` (TS 5.2+/Node 20+) disposes automatically — runs `bot.close()` and plugin
`teardown`. The double `await` is intentional: outer `await using` registers disposal, inner
`await` is the async boot. No explicit-resource-management? `afterEach(() => bot.close())` (or
`await bot[Symbol.asyncDispose]?.()`). Options seen in docs: `commands`, `components`, `events`,
`modals`, `middlewares` (same record as `client.setServices`), `globalMiddlewares`, `world`,
`plugins`, `clientOptions`, `loadFromConfig`, `commandsDir`/`componentsDir`/`eventsDir`/`langsDir`,
`shards`, `shardLatency`, `botId`, `applicationId`, `prefixes`, `mentionAsPrefix`, `langs`,
`defaultLang`, `simulateGateway` (default `true`), `onUnhandledRest: 'error'|'warn'|'silent'`
(default `'error'`), `onCommandError: 'throw'|'capture'` (default `'throw'`). `loadFromConfig: true`
runs Seyfert's real loaders reading `seyfert.config` (which usually points at **compiled output**) —
build first and run from a cwd where the config resolves; `*Dir` options override per-kind.

## Commands

```ts
// real command, root imports — the exact code you ship
import { Command, Declare, Options, createStringOption, type CommandContext } from 'seyfert';
const options = { name: createStringOption({ description: 'Who', required: true }) }; // lowercase keys (v5)
@Declare({ name: 'greet', description: 'Greet' })
@Options(options)
export class GreetCommand extends Command {
  async run(ctx: CommandContext<typeof options>) { await ctx.write({ content: `Hi ${ctx.options.name}` }); }
}
```

```ts
import { expect, test } from 'vitest';
import { createMockBot } from '@slipher/testing';

test('greet replies through the real pipeline', async () => {
  await using bot = await createMockBot({ commands: [GreetCommand] });
  const result = await bot.slash({ name: 'greet', options: { name: 'x' } });   // by-name
  expect(result.content).toBe('Hi x');
});
// class form: bot.slash(GreetCommand, { options: { name: 'x' } }) — infers option types (wrong key won't compile)
```

Result views: `result.content`, `result.reply?.body` (`{ type, data }` wire payload), `result.messages`,
`result.actions`, `result.edits`, `result.followups`, `result.deferred`, `result.embeds`/`embed`,
`result.embedView(s)`, `result.components`, `result.component('id')`, `result.textDisplays`,
`result.error` (only under `onCommandError: 'capture'`). Run as a user:
`bot.slash({ name, user: apiUser({ id, username }) })`. Persistent multi-step actor:
`const a = bot.actor({ member, guildId, channel })` (cache/collectors/cooldowns persist across its
dispatches). Subcommands: `bot.slash({ name: 'admin', group: 'users', subcommand: 'kick', options })`.
Entity options need builders: `userOption(apiUser({ id }))` / `channelOption` / `roleOption` /
`mentionableOption` / `attachmentOption` (a bare id won't populate `resolved`, so `ctx.options.user`
won't be a real entity). Autocomplete: `bot.autocomplete({ name, focused, value })` → `result.choices`
(numeric options must respond numeric, string options string — v5 types enforce this). Context menu:
`bot.userMenu({ name, target })`. Prefix commands: `createMockBot({ prefixes: ['!'], mentionAsPrefix })`
then `bot.say('!echo hi')` — assert on `result.messages`, NOT `result.reply` (no interaction response).
i18n: `langs` + `defaultLang` options; `bot.slash({ name, locale: 'es-ES' })`.

v5 context facts (carry into the handlers you test): `ctx.write(body, withResponse?)` /
`ctx.editOrReply(body, withResponse?)` return **`void` unless the flag is `true`** — assert on
`result`/`responses`, not a returned `Message`. `CommandContext` HAS `editResponse(body)`,
`deferReply`, `followup`, `modal`, `fetchResponse`. `ctx.inGuild()` → `GuildCommandContext`
(narrows `channel()`/`fetchMember()`). `ctx.t` is the typed `SeyfertLocale` proxy — resolve a leaf
with `ctx.t.greeting.get(locale?)`.

```ts
// middleware-guarded command (v5: stop(), no pass())
import { createMiddleware } from 'seyfert';
export const onlyOwner = createMiddleware<void>(({ context, next, stop }) => {
  if (!context.member) return stop();                  // silent skip (was pass())
  if (context.author.id !== process.env.OWNER_ID) return stop('Owner only.'); // deny → onMiddlewaresError
  return next();
});
```

```ts
import { outcome, apiUser } from '@slipher/testing';
const denied = await bot.slash({ name: 'admin', user: apiUser({ id: 'not-owner' }), onCommandError: 'capture' });
outcome(denied).get.denial({ kind: 'stop', middleware: 'onlyOwner' });   // assert shape, not copy
```

Permission hooks are real: `memberPermissions: []` drives `onPermissionsFail`; bot `permissions: []`
drives `onBotPermissionsFail` (`src/commands/handle.ts`, defaults `handler.ts`). With
`registerBotMember(...)` seeded, moderation REST routes enforce computed perms/hierarchy and return
Discord `403 50013`. `outcome(result).get.denial({ kind: 'permissions', missing: 'BanMembers' })`.

## Components

Two handler models, both dispatched the same way:

1. **Collector** (message-scoped): `message.createComponentCollector(opts?).run(customId, cb)`
   (`customId` = string/array/RegExp; matched via `createMatchCallback`). Also `.waitFor(customId,
   timeout?)` → interaction or `null`, `.stop()`, `.resetTimeouts()`. Options: `timeout`, `idle`,
   `filter`, `onPass`, `onStop`, `onError`. **v5: ALL matching collectors fire (the first-match
   `break` is gone)** — tighten `customId`/`filter` if you relied on first-match-wins.
2. **`ComponentCommand`** (global, file-based, survives restarts): abstract `componentType`
   (`'Button' | 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'`),
   `customId?: string | RegExp`, `filter?(ctx)`, `run(ctx)`, `readonly middlewares`. Hook
   `onInternalError(client, component, error?)` — note the **`component` before `error`** (v5).

```ts
// command writes a button, then a collector handles the click
import { ActionRow, Button, ButtonStyle, Command, Declare, type CommandContext } from 'seyfert';
@Declare({ name: 'poll', description: 'Open a poll' })
export class PollCommand extends Command {
  async run(ctx: CommandContext) {
    const row = new ActionRow<Button>().addComponents(
      new Button().setCustomId('poll/yes').setStyle(ButtonStyle.Success).setLabel('Yes'),
    );
    await ctx.write({ content: 'Vote now', components: [row] });
    const message = await ctx.fetchResponse();
    message.createComponentCollector().run('poll/yes', async i => { await i.write({ content: 'Voted!' }); });
  }
}
```

```ts
await using bot = await createMockBot({ commands: [PollCommand] });
await bot.slash({ name: 'poll' });
const result = await bot.clickButton('poll/yes');          // defaults to bot.lastSentMessage()
expect(result.content).toBe('Voted!');
```

String select: `bot.selectMenu('settings/theme', ['dark'])`. Build menus with v5 rest-param
`setOptions(a, b)` (NOT an array). Use `interaction.update(body)` to edit the source message in
place vs `write(body)` for a new reply. Entity selects need
`bot.selectMenu('id', [role.id], { source, componentType: 'role' })` (ids auto-resolve against
seeded world). Target another message via `{ source: msg | id }`. **v5: `ComponentContext` has NO
`editResponse`** — use `editOrReply(body, true)` or `update(...)`; it gained `ctx.message` getter and
`isButton()` reads `interaction.componentType`.

## Modals

A modal is two-step. Build inputs with v5 `Label` + `setComponent(...)`, NOT legacy
`ActionRow<TextInput>`. `Label` accepts a wide union (`TextInput | select menus | FileUpload |
Checkbox | CheckboxGroup | RadioGroup`). `Modal.toJSON()`/`Label.toJSON()` throw if
customId/title/component are missing (surfaces inside `ctx.modal` as a test failure — a feature).

**Awaited (inline)** — opener returns the submit:
```ts
import { Command, Declare, Label, Modal, TextInput, TextInputStyle, type CommandContext } from 'seyfert';
@Declare({ name: 'feedback', description: 'Send feedback' })
export class FeedbackCommand extends Command {
  async run(ctx: CommandContext) {
    const modal = new Modal().setCustomId('feedback-modal').setTitle('Feedback').addComponents(
      new Label().setLabel('Rating').setComponent(new TextInput().setCustomId('rating').setStyle(TextInputStyle.Short)),
    );
    const submit = await ctx.modal(modal, { waitFor: 60_000 });  // ModalSubmitInteraction | null
    if (!submit) return ctx.write({ content: 'timed out' });      // handle the null/timeout branch
    const rating = submit.getInputValue('rating', true);          // required → string
    await submit.write({ content: `thanks (${rating})` });
  }
}
```
```ts
const r = await bot.clickButton('open', { user }).fillModal('feedback-modal', { rating: '5' });
expect(r.content).toContain('thanks');
const t = await bot.clickButton('open', { user }).timeoutModal();  // forces the null/timeout branch
expect(t.content).toBe('timed out');
```

**Decoupled** — opener `ctx.modal(modal)` (one-arg → `Promise<undefined>`, just opens) plus a
registered `ModalCommand` (matched by `customId`, register via `createMockBot({ modals: [...] })`):
```ts
import { ModalCommand, type ModalContext } from 'seyfert';
export default class NicknameModal extends ModalCommand {
  customId = 'nickname-modal';                 // string OR RegExp
  async run(ctx: ModalContext) { await ctx.write({ content: `Nick set to ${ctx.getInputValue('nick', true)}` }); }
}
// bot = createMockBot({ commands: [NicknameCommand], modals: [NicknameModal] });
// await bot.clickButton('open', { user }).fillModal('nickname-modal', { nick: 'Glorp' });
```

Gotchas: keep the **same `user`** across opener + `fillModal`, or the submit won't correlate and it
hangs (`waitFor` uses REAL timers). `fillModal` keys = the **inner component** customIds (text →
string, select/channel/role/user → array, checkbox → bool/array); read with the matching accessor.
Awaiting an opener without `.fillModal`/`.timeoutModal` fails loud by design. `waitFor` only arms
when `> 0`. `ctx.modal(...)` on a prefix/no-interaction context throws `CANNOT_USE_MODAL`
("Cannot use modal without an interaction.") — modals are interaction-only. `ModalContext` (unlike
component context) STILL has `editResponse`, plus full accessors `getChannels/getRoles/getUsers/
getMentionables/getRadioValues/getCheckbox/getCheckboxValues/getInputValue/getFiles`, `update`, `deferUpdate`.

## Events

`createEvent` data name is **camelCase** client name (`ClientNameEvents`), e.g. `'guildMemberAdd'`.
`bot.emit` takes the **UPPERCASE** raw gateway dispatch name. Mixing them is the classic silent trap.
v5: `run` is `Awaitable<unknown>`; gateway handlers are `(payload, client, shardId)`; **custom
(non-gateway) handlers drop the trailing `shardId`** → `(payload, client)`. Throwing in a `once`
handler resets it so it can fire again.

```ts
import { createEvent } from 'seyfert';
export default createEvent({ data: { name: 'guildMemberAdd' }, async run(member, client) {
  await client.users.write(member.id, { content: `Welcome ${member.user.username}` });  // recorded, not sent
} });
```

```ts
await using bot = await createMockBot({ events: [guildMemberAdd] });
await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload);                    // throws if no handler ran
await bot.emit('CHANNEL_CREATE', rawPayload, { allowNoHandler: true });  // seed-only world state
expect(bot.world.query.dm({ userId: rawMemberPayload.user.id })?.lastMessage?.content).toContain('Welcome');
```

Emit **full** Discord payloads, not partial patches (the mock applies canonical reducers to world
state). `emit` awaits the REST the handler awaited before returning — read the effect immediately,
no tick/flush. `registeredEvents()` lists wired handlers for debugging green-but-silent assertions.

## World & state

```ts
import { mockWorld, createMockBot } from '@slipher/testing';
const world = mockWorld();
const guild = world.registerGuild({ name: 'Lab' });   // auto-creates @everyone (role id == guild id)
world.registerChannel(guild.id);
const member = world.registerMember(guild.id);
await using bot = await createMockBot({ commands: [C], world });  // world is deep-cloned (safe to share a builder)
```

Registrars: `registerGuild/Channel/Thread/Role/Member/BotMember/VoiceState/Message` (+ emoji,
sticker, invite, webhook, automod, scheduledEvent, stageInstance, …); registering under an unseeded
id throws a descriptive `TypeError`. `setData(k,v)` → read back via `bot.worldData`. Entities are
written under **`CacheFrom.Test`** (`=== 3`; `CacheFrom` is `Gateway=1, Rest, Test`), so real cache
reads resolve without interceptors: `ctx.guild('cache')`, `member.voice('cache')`,
`client.cache.roles.values(guildId)`, `client.channels.fetchMessages(id)` (mock returns
newest-first). Readers: `bot.world.get.*` (throws on 0/many — use for uniqueness), `bot.world.query.*`
(`undefined`-able), `bot.world.all.*` (array); plus `snapshot()`/`diff()`. Defaults are deliberately
permissive — seed `registerBotMember` and roles/overwrites when the permission/hierarchy contract
matters; `memberPermissions: 'all'` for an admin invoker.

```ts
// full permission/hierarchy test: ban denied by a channel overwrite (mock runs the real perm algorithm)
const guild = world.registerGuild({ id: 'g1', ownerId: 'owner', everyonePermissions: ['SendMessages'] });
const mod = world.registerRole(guild.id, { permissions: ['BanMembers'], position: 5 });
const target = world.registerMember(guild.id, { roles: [mod.id] });
const denied = world.registerChannel(guild.id, { overwrites: [{ id: mod.id, type: 'role', deny: ['BanMembers'] }] });
world.registerBotMember(guild.id, { roles: [mod.id] });
await using bot = await createMockBot({ commands: [BanCommand], world });
await bot.slash({ name: 'ban', guildId: guild.id, channel: denied, user: target.user });
// world state proves WHAT happened; bot.actions proves the calls happened
expect(bot.world.query.ban({ guildId: guild.id, userId: target.user.id })).toBeUndefined();
```

## REST, assertions

- Actions carry `{ seq, method, route, body, query, response }`. `await bot.waitForAction(Routes.x)`
  (for fire-and-forget the command did NOT await); `bot.actions` (all, in order). REST `body` is the
  **raw snake_case** Discord wire shape even though your code uses camelCase (e.g. `BanOptions
  { deleteMessageSeconds, reason }` serializes to `delete_message_seconds`).
- Stubs: `bot.rest.intercept(route, fn)` (also `intercept('GET', '/path/:p', fn)`);
  `bot.rest.fail(route, DiscordErrors.MissingPermissions)` throws a faithful `SeyfertError` (real
  root export; `code`, `metadata`, `static is`, `toJSON`; `bot.rest.fail(route, err, { times: 1 })`
  fails once). Unmatched REST throws unless `onUnhandledRest: 'warn'|'silent'`.
- `outcome(result)` reads the lifecycle: `.get.response({ kind: 'reply'|'defer'|'modal'|'update'|'edit'|… , ephemeral })`,
  `.get.denial({ kind: 'permissions'|'stop'|'pass'|'no-next'|'bot-permissions', middleware, missing })`,
  `.get.error(/re/)` (needs `onCommandError: 'capture'`; default `'throw'` rejects the dispatch, so
  `await expect(bot.slash(...)).rejects.toThrow(/re/)` there). `kind: 'pass'` = a middleware that
  ended via `stop()` with no arg (the old `pass()`).
- `rendered(result)` reads UI: `.get.message/embed/component('button',{customId})/select/input/modal/container`;
  scopes drill in (`rendered(r).get.message({content}).get.component('button','edit')`).
- All three readers expose `.get` (throws unless exactly one — fails loud where a naive
  `toContain` on `undefined` passes green), `.query` (or `undefined`), `.all` (array). **String
  matchers are exact** — use RegExp for partial/case-insensitive. Prefer denial *shape* over copy.

## Dispatch execution model

A dispatch is a **lazy** `Dispatch` (nothing runs until `await` or `.until()`). `.until(matcher)`
suspends mid-flight when the matched REST call STARTS (`response` stays `undefined`) and must be
called BEFORE the first `await`; awaiting always releases the checkpoint (cannot deadlock).
Dispatches do **not reject** on command errors — Seyfert routes handler throws through error hooks; a throwing/rejecting middleware rejects the
runner → `onInternalError` (not a denial). Assert the user-facing reply, or set `onCommandError: 'capture'` and read `result.error`.

```ts
import { Routes } from '@slipher/testing';      // NOT from 'seyfert'
const d = bot.slash({ name: 'ban', options: { user: userOption(apiUser({ id: '42' })) } });
const ban = await d.until(Routes.ban);           // or: a => a.method === 'PUT' && a.url.includes('/bans/')
expect(ban.body).toMatchObject({ delete_message_seconds: 0 });
const result = await d;                           // releases the checkpoint, runs to completion
```

## MockGateway

`createMockBot({ shards, shardLatency })` installs a `MockGateway` for `client.gateway`
(`ShardManager`). Real methods you can call: `gateway.setPresence(...)`, `gateway.send(shardId,
payload)` (`Promise<boolean>`), `gateway.latency`. Mock-only recording: `gateway.presences`,
`gateway.sent` (`{ shardId, payload }`), `gateway.values()` (`{ id, latency }`),
`simulateDisconnect/Reconnect` — unstable in 0.x. `simulate*` drive the wrapped client callbacks,
which `setServices({ gateway })` wires to the `SHARD_DISCONNECT`/`SHARD_RECONNECT` events, so you can
test reconnect/alerting logic. Enums import from `'seyfert'`: `ActivityType`, `GatewayOpcodes`,
`PresenceUpdateStatus`. NB the real `ShardManager.values()` yields `Shard` instances (it extends
`Map<number, Shard>`), not the mock's `{ id, latency }` — don't assume the mock shape on a real client.

## Fixtures (unit)

```ts
import { mockCommandContext, mockUser } from '@slipher/testing';
const ctx = mockCommandContext(BanCommand, { options: { user: mockUser({ id: '123' }) } });
await ctx.run();   // command bound (class form) → no arg; SKIPS option parsing/middlewares/permissions
expect(ctx.lastResponse()).toMatchObject({ content: expect.stringContaining('Banned') });
```

Class form infers option types from `@Declare`/`@Options` and binds `ctx.run()`. Object form
`mockCommandContext({ commandName, options })` is a plain sink (no bound run). Options you pass are
injected as-is into `ctx.options` — they are NOT resolved from raw interaction data; if a command
relies on a resolved option's side effects, use the mock bot. Readers: `responses`, `lastResponse()`,
`clearResponses()`, `lastEmbed(s)()`, `lastComponents()`, `lastTexts()` (`write`/`editOrReply`/
`followup` all push onto the same `responses` array in call order). Also `mockComponentContext`
(`.run(input?)` / `.filter(input?)`), `mockModalContext` (class `.run()` or object form +
`getInputValue`), `mockScene` (linked `{ ctx, user, guild, channel, member }`), factories
`mockUser/Guild/Channel/Member` (expose both camelCase `globalName` and snake_case `global_name`),
`resetMockIds()` (deterministic ids; `TEST_*` ids are real snowflakes so BigInt helpers like
`avatarURL`/`createdAt` work). Stubs `logger`/`queues`/`scheduler` record in-memory
(`ctx.client.logger === ctx.logger`). For unmodeled surfaces, override the field directly
(`ctx.guild = vi.fn(async () => …)`).

## Plugins

Plugins load through the mock client's real lifecycle — `setup()` runs on boot, `teardown()` on
disposal — so you can assert client extensions, middleware, and command additions end-to-end.

```ts
import { definePlugins } from 'seyfert';
import { economyPlugin } from '../src/plugins/economy';
await using bot = await createMockBot({
  plugins: definePlugins(economyPlugin),
  commands: [/* commands that read client.economy */],
});
// economyPlugin.setup ran on boot; teardown runs on `await using` disposal
```

Augment `SeyfertRegistry` (NOT `UsingClient`/`RegisteredMiddlewares` — those are derived; v5):
```ts
import { middlewares } from '../src/middlewares';
declare module 'seyfert' { interface SeyfertRegistry { middlewares: typeof middlewares } }
```

## Gotchas (source-verified)

- Middleware control flow uses **`stop()`**, not the removed `pass()` (commit 22eb832; arg is
  `{ context, next, stop }`; `stop()`/`stop(null)` skip silently → denial kind `'pass'`,
  `stop(err)` denies → `onMiddlewaresError` whose `metadata.middleware` names the caller). Any
  carried-over `pass()` example is stale.
- Command **option arrays and `globalMiddlewares` are `readonly`-accepting**; `command.middlewares`
  is a readonly tuple (no `.push`). Passing readonly arrays type-checks fine.
- `ctx.t` returns the typed `SeyfertLocale` alias (commit 7411731); resolve with `.get(locale?)`.
- `write`/`editOrReply` return `void` unless the response flag is `true` — assert on `result`/`responses`.
- Collectors fire ALL matching handlers (no `break`); collector run param type is
  `CollectorRunParameters` (old `CollectorRunPameters` typo removed); `client.collectors.run(...)`
  takes the camelCase client name (`'messageCreate'`), not the raw gateway name.
- `@Declare`/`@Options` need `experimentalDecorators` in the test transform.
- `loadFromConfig` reads compiled output — build first; reset module-level state in command files in
  `beforeEach` to avoid order-dependent CI flakes ("passes alone, fails in CI").

## Builder validation (dependency-free unit tests)

v5 builders validate on `toJSON()` — fast tests with no toolkit. Mirrors `tests/builder-validation.test.mts`.

```ts
import { describe, expect, test } from 'vitest';
import { PollBuilder, Modal, SeyfertError } from 'seyfert';

describe('builder validation (v5)', () => {
  test('PollBuilder.toJSON() throws without a question', () => {
    expect(() => new PollBuilder().setAnswers({ text: 'Yes' }, { text: 'No' }).toJSON()).toThrow(SeyfertError);
  });
  test('Modal.toJSON() throws without a title', () => {
    expect(() => new Modal().setCustomId('m').toJSON()).toThrow(SeyfertError);
  });
});
// Narrow a caught error by code: if (SeyfertError.is(err, 'CANNOT_USE_MODAL')) { … }
```

## Review checklist

- [ ] Is `@slipher/testing` actually installed for this project's package manager, and its version's
      API confirmed? (Not part of seyfert-core; do not assume signatures.)
- [ ] Command/event/component/modal code imports from **root `'seyfert'`**; toolkit + `TEST_*` from
      **`'@slipher/testing'`** — never `Routes`/`HandleCommand` from `'seyfert'` (not runtime/root).
- [ ] `createEvent` name camelCase; `bot.emit` name UPPERCASE; emit full payloads. Custom events
      `(payload, client)` (no `shardId`); gateway events `(payload, client, shardId)`.
- [ ] Modal submits chained to the opener with the same `user`; awaited modal uses `{ waitFor > 0 }`
      and handles the `null` timeout branch; inputs built with `Label` + `setComponent`, not `ActionRow`.
- [ ] Entity options use `*Option(apiUser(...))` builders, not bare ids.
- [ ] Permission tests seed `registerBotMember` + roles/overwrites or use `memberPermissions`/`permissions`.
- [ ] REST asserted/intercepted under strict `onUnhandledRest`; `onCommandError: 'capture'` set when
      asserting thrown errors via `outcome().get.error` (default `'throw'` rejects the dispatch).
- [ ] Fixtures only for pure `run()` logic (they skip parsing/middlewares/permissions); mock bot for
      pipeline behavior (option parsing, middleware order, components, modals, plugins).
- [ ] Assertions target user-visible output / meaningful side effects (world state, recorded
      actions), not internal shapes; RegExp for partial matches (string matchers are exact).
- [ ] Middleware uses `stop()` not `pass()`; `ComponentContext` has no `editResponse` (use
      `update`/`editOrReply(body, true)`); `setOptions(...)` is rest-param.
