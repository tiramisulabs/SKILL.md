# Mock Bot (@slipher/testing)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/mock-bot

Coverage reference: testing.md

Verification status: Source-verified (core) + external package (@slipher/testing — verify version in target project)

## Page Summary

`createMockBot()` boots a *real* Seyfert `Client` in-process with no token, gateway, or network, then drives your real command/component/event/modal classes through the actual Seyfert pipeline (raw `APIInteraction` -> `HandleCommand` -> option resolver -> middlewares -> handler), recording every REST call instead of sending it. Unlike `mockCommandContext()` (which hands `run()` a fake context), the mock bot builds a genuine `CommandContext` exactly as production does. The `createMockBot` API itself lives in the EXTERNAL `@slipher/testing` package — not in core Seyfert — so treat its option names/signatures as doc-authoritative and verify the installed version in the target project. The Seyfert-core surfaces it dispatches against (client, services, plugins, loaders, contexts, REST routes) ARE verified below against `./src`.

## Key APIs (verified)

External (NOT in core Seyfert — verify version in target project):
- `createMockBot(options)` from `@slipher/testing` — async, returns a disposable mock bot (`await using`).
- Dispatchers: `bot.slash(CommandClass, { options })` (option inference from the class) or `bot.slash({ name, options })` (raw by-name payload); `bot.say(...)` for prefix/message commands.
- Result shape: `result.content`, `result.deferred`, `result.edits`, `result.followups`, `result.actions`, `result.replies`, `result.reply?.body` (`{ type, data }`), `result.embedView(s)`, `result.embeds`/`result.embed`, `result.components`, `result.component(id)`, `result.textDisplays`, `result.error`.
- Dispatch control: lazy `Dispatch`; `dispatch.until(Routes.x)`, `.fillModal(...)`, `.timeoutModal()`, `waitForAction()`.
- Options: `commands`, `components`, `events`, `middlewares` (same record as `client.setServices`), `globalMiddlewares`, `world` (cache seed), `plugins`, `clientOptions` (forwarded to `Client`, minus plugin loading), `onCommandError: 'throw' | 'capture'` (default `'throw'`), `onUnhandledRest` (throw by default; warn/silent for allowed fallback reads), `simulateGateway` (default `true`), `shards`, `shardLatency`, `botId`, `applicationId`, `prefixes`, `mentionAsPrefix`, `loadFromConfig`, `commandsDir`, `componentsDir`, `eventsDir`, `langsDir`.

Core seyfert APIs this toolkit dispatches against (verified in ./src):
- `HandleCommand` — `src/commands/handle.ts:64` (the real entrypoint the mock bot drives; prefix dispatch path at `handle.ts:368`).
- `CommandContext` — `src/commands/applications/chatcontext.ts:45` (real context the mock builds).
- `client.setServices({ rest, cache, langs, middlewares, handleCommand })` — `src/client/base.ts:309` (the `middlewares` record the doc references; note `handleCommand` takes the `HandleCommand` *constructor* in v5, not an instance).
- `globalMiddlewares` option — `src/client/base.ts:1255` (`readonly (keyof ResolvedRegisteredMiddlewares)[]`).
- `plugins` / `clientOptions` — `ClientOptions<TPlugins>` at `src/client/client.ts:303`, `plugins?: TPlugins` at `client.ts:305`.
- Loaders used by `loadFromConfig` / `*Dir` options: `loadCommands(dir?)` `base.ts:1127`, `loadComponents(dir?)` `base.ts:1135`, `loadLangs(dir?)` `base.ts:1160`, `loadEvents(dir?)` `client.ts:108`. Defaults resolve from `getRC()` config `locations` (`base.ts:1188`).
- `Routes` (used as `Routes.x` in `dispatch.until`) is NOT a runtime `seyfert` export — `export * from './Routes'` in `src/api/index.ts` re-exports only route *type* interfaces (`./Routes` is a directory of them). Import the matcher from `@slipher/testing` or `discord-api-types/v10` (as example 4 does), or pass a predicate over the recorded action.

## Code Examples (verified)

Imports of `createMockBot` come from `@slipher/testing`; core classes come from `seyfert`; the `Routes` matcher comes from `discord-api-types/v10` (or `@slipher/testing`), not `seyfert`.

### 1. Slash command through the real pipeline

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { GreetCommand } from '../src/commands/greet';

test('greet replies through the real pipeline', async () => {
	await using bot = await createMockBot({ commands: [GreetCommand] });
	// by-name payload; or bot.slash(GreetCommand, { options }) for class inference
	const result = await bot.slash({ name: 'greet', options: { name: 'slipher' } });
	expect(result.content).toBe('Hello, slipher!');
});
```

### 2. Programmatic registration of your real classes

```ts
import { createMockBot } from '@slipher/testing';
import { BanCommand } from '../src/commands/moderation/ban';
import { ConfirmButton } from '../src/components/confirm';
import guildMemberAdd from '../src/events/guildMemberAdd';

await using bot = await createMockBot({
	commands: [BanCommand],
	components: [ConfirmButton],
	events: [guildMemberAdd],
});
```

### 3. Boot the production set through Seyfert's real loaders

Build first — `seyfert.config` `locations` usually point at compiled output.

```ts
await using bot = await createMockBot({ loadFromConfig: true });
await bot.slash({ name: 'ping' });
```

Explicit directories override the config:

```ts
import { join } from 'node:path';

await using bot = await createMockBot({
	loadFromConfig: true,
	commandsDir: join(process.cwd(), 'dist/commands'),
});
```

### 4. Step through a dispatch and capture handler errors

Default `onCommandError: 'throw'` rejects the `Dispatch` on a handler throw (mirroring Seyfert routing throws to error hooks). Pass `'capture'` to assert on `result.error`.

```ts
import { Routes } from 'discord-api-types/v10';

await using bot = await createMockBot({ commands: [BanCommand], onCommandError: 'capture' });
const dispatch = bot.slash({ name: 'ban', options: { user: '123' } });
await dispatch.until(Routes.ban); // matched call started; response still undefined while suspended
const result = await dispatch;     // releases checkpoint; cannot deadlock
expect(result.error).toBeUndefined();
```

### 5. Assert embeds and components without casting

The mock parses Discord payloads back into typed views, so you avoid wire-shape casts.

```ts
import { expect, test } from 'vitest';
import { ProfileCommand } from '../src/commands/profile';

test('profile renders an embed and an approve button', async () => {
	await using bot = await createMockBot({ commands: [ProfileCommand] });
	const result = await bot.slash({ name: 'profile', options: { user: '42' } });

	expect(result.embedView?.title).toBe('Profile: alice');
	expect(result.component('Approve')?.customId).toBe('approve-42');
	// raw flattened payloads stay available for wire-shape contracts:
	expect(result.embed?.fields).toHaveLength(2);
});
```

### 6. Button component flow (real ComponentContext)

A button handler is a real `ComponentCommand`; in v5 its context uses `update(...)` and `ctx.message` (NO `editResponse` on component contexts). Register the component and dispatch it the way the toolkit exposes (verify the exact dispatcher name in your installed version — the result/assertion surface is the stable part).

```ts
import { expect, test } from 'vitest';
import { ConfirmButton } from '../src/components/confirm';

test('confirm button edits the original message', async () => {
	await using bot = await createMockBot({ components: [ConfirmButton] });

	// dispatch a component interaction by its customId through the real pipeline
	const result = await bot.component({ customId: 'confirm', userId: '99' });

	// component handler called ctx.update(...) -> recorded as an edit
	expect(result.edits.at(-1)?.content).toBe('Confirmed.');
});
```

### 7. Modal flow — open, fill, submit in one dispatch

A command that calls `ctx.modal(...)` opens a modal; drive it without deadlocking using `.fillModal(...)`. The submit runs your real `ModalCommand`, whose v5 context exposes `getInputValue` / `getChannels` / `getRoles` / `getUsers` / `getRadioValues` / `getCheckboxValues` and `update` / `deferUpdate`.

```ts
import { expect, test } from 'vitest';
import { SetupCommand } from '../src/commands/setup';

test('setup modal stores the submitted name', async () => {
	await using bot = await createMockBot({ commands: [SetupCommand] });

	const result = await bot
		.slash({ name: 'setup' })
		.fillModal({ username: 'alice' }); // keys = modal text-input customIds

	expect(result.content).toBe('Saved: alice');
});

// Or assert the timeout branch (ctx.modal({ waitFor }) never resolves):
test('setup handles a modal timeout', async () => {
	await using bot = await createMockBot({ commands: [SetupCommand] });
	const result = await bot.slash({ name: 'setup' }).timeoutModal();
	expect(result.content).toBe('Setup cancelled — no response.');
});
```

### 8. Seed cache + simulate gateway for stateful commands

`world` seeds entities into the client cache so a command reading `ctx.client.cache` / `ctx.guild()` sees realistic data; `simulateGateway` (default `true`) emits matching member update/remove events for stateful writes.

```ts
import { expect, test } from 'vitest';
import { KickCommand } from '../src/commands/moderation/kick';

test('kick removes the member and replies', async () => {
	await using bot = await createMockBot({
		commands: [KickCommand],
		world: {
			guilds: [{ id: 'g1', name: 'Test Guild' }],
			members: [{ id: 'u2', guildId: 'g1', username: 'troublemaker' }],
		},
		simulateGateway: true, // emits guildMemberRemove for the kick write
	});

	const result = await bot.slash({
		name: 'kick',
		guildId: 'g1',
		options: { user: 'u2', reason: 'spam' },
	});

	expect(result.content).toContain('Kicked troublemaker');
});
```

### 9. Prefix / message command via say()

Enable prefix dispatch with `prefixes` (+ optional `mentionAsPrefix`) and drive it with `bot.say(...)`. This exercises the prefix path of `HandleCommand` (`handle.ts:368`).

```ts
import { expect, test } from 'vitest';
import { PingCommand } from '../src/commands/ping';

test('prefix ping responds', async () => {
	await using bot = await createMockBot({
		commands: [PingCommand],
		prefixes: ['!'],
		mentionAsPrefix: true,
	});

	const result = await bot.say('!ping');
	expect(result.content).toBe('Pong!');
});
```

### 10. Middleware order and denial

Pass the same `middlewares` record you give `client.setServices`, plus `globalMiddlewares` to run them on every command. A v5 middleware denies with `stop('reason')` (routes to `onMiddlewaresError`) or skips silently with `stop()` — there is no `pass()` anymore.

```ts
import { createMiddleware } from 'seyfert';
import { expect, test } from 'vitest';
import { SecretCommand } from '../src/commands/secret';

const requireOwner = createMiddleware<void>(({ context, next, stop }) => {
	if (context.author.id !== 'owner-id') return stop('Owner only'); // deny
	return next();
});

test('non-owner is denied before the handler runs', async () => {
	await using bot = await createMockBot({
		commands: [SecretCommand],
		middlewares: { requireOwner },
		globalMiddlewares: ['requireOwner'],
	});

	const result = await bot.slash({ name: 'secret', userId: 'someone-else' });
	// the handler never ran; the denial reply is what the user sees
	expect(result.content).toContain('Owner only');
});
```

### 11. Test a plugin end-to-end

Plugins load through the mock client's real lifecycle (`setup` / `teardown` run), so you can assert a plugin's client extension, middleware, or command additions.

```ts
import { definePlugins } from 'seyfert';
import { economyPlugin } from '../src/plugins/economy';

await using bot = await createMockBot({
	plugins: definePlugins(economyPlugin),
	commands: [/* commands that read client.economy */],
});
// economyPlugin.setup ran on boot; teardown runs on `await using` disposal
```

## Common patterns / gotchas

- `await using bot = await createMockBot(...)` — the double `await` is intentional: the outer `await using` registers disposal (teardown / plugin cleanup), the inner `await` is the async boot. Without `await using`, plugin `teardown` and resource cleanup will not run.
- Awaiting a `Dispatch` guarantees everything `run()` *awaited* (replies, edits, followups, dispatch actions). It does NOT wait for fire-and-forget work the command did not await — use `waitForAction()` for timers / detached promises / scheduler side effects.
- `dispatch.until(Routes.x)` suspends mid-flight at the first matching REST call; `result.response`/return stays `undefined` while suspended. Awaiting the same dispatch afterwards releases the checkpoint — it cannot deadlock.
- For modal-opening commands, drive the modal in the SAME dispatch chain (`.fillModal(...)` / `.timeoutModal()`); awaiting a bare dispatch that is waiting on a modal would otherwise hang on `ctx.modal({ waitFor })`.
- v5 context deltas that matter inside the handlers you test: component contexts have NO `editResponse` (use `ctx.update(...)`), `ctx.message` is a getter, `isButton()` reads `interaction.componentType`; modal contexts keep `editResponse` and gained the full accessor set; `write`/`editOrReply` return `void` unless the response flag is `true`.
- `onUnhandledRest` throws by default — any REST route your command hits that the mock did not classify will fail the test, surfacing accidental network calls. Relax to warn/silent only for intentional fallback reads.
- `loadFromConfig` reads `seyfert.config` `locations` (compiled output) — build before broad integration specs, or override per-kind with `commandsDir` / `componentsDir` / `eventsDir` / `langsDir`.
- Default `onCommandError: 'throw'` rejects the Dispatch on a handler throw; `'capture'` surfaces it on `result.error`. This mirrors real Seyfert routing handler throws to error hooks rather than the user reply.

## Doc vs Source Corrections

- None for core surfaces — `setServices` `middlewares`, `globalMiddlewares`, `plugins`/`clientOptions`, the loaders, `CommandContext`, and `HandleCommand` all match the doc's usage against `./src`. `Routes` is NOT a runtime `seyfert` value (only route *type* interfaces via `export * from './Routes'`) — import the `Routes.x` matcher from `@slipher/testing`/`discord-api-types` (example 4 uses `discord-api-types/v10`).
- v5 reminder (not a doc error, but relevant when writing the handlers under test): middleware uses `stop()`/`stop('reason')` (no `pass()`); `setServices({ handleCommand })` takes the `HandleCommand` *constructor*, not an instance; component contexts dropped `editResponse`.
- `createMockBot` and all `bot.*`/`Dispatch`/`result.*` members are EXTERNAL (`@slipher/testing`) and are NOT present anywhere in `./src` (grep for `createMockBot`/`MockGateway`/`onUnhandledRest`/`simulateGateway` returns no core hits). Do not assume these exist in core Seyfert; pin/verify the toolkit version installed in the consuming project. In particular the exact component/modal dispatcher names (`bot.component(...)`, `.fillModal(...)`) shown above are doc/version-dependent — confirm them in your installed `@slipher/testing`.

## Source Anchors

- `src/api/index.ts` (`Routes` export) + `src/index.ts:16` (`export * from './api'`)

## Agent Guidance

- Use the mock bot for true end-to-end behavior tests (option parsing, middleware order, error hooks, REST side effects, component/modal flows, plugin lifecycle) where `mockCommandContext()` is too shallow.
