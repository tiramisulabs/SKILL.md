# Dispatching (Testing Toolkit)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/dispatching

Coverage reference: testing.md

Verification status: Source-verified (core) + external package (`@slipher/testing`)

## Page Summary

Covers the `@slipher/testing` mock-bot dispatch API: driving raw `APIInteraction`/message payloads and gateway events through the *real* Seyfert pipeline (`HandleCommand`, option resolver, middlewares, REST recorder). Unlike `mockCommandContext()` (which hands `run()` a fake context), the mock bot builds a real `CommandContext` exactly as production does. Dispatchers (`bot.slash`, `bot.autocomplete`, `bot.userMenu`, `bot.clickButton`, `bot.selectMenu`, `bot.fillModal`, `bot.say`, `bot.emit`) return a lazy `Dispatch` that you `await` for the full run or step with `.until(matcher)`. The toolkit (`createMockBot`, dispatch helpers, entity-option builders, `Routes` matcher) is EXTERNAL and not in seyfert-core — verify its version/signatures in the target project. The Seyfert-core surfaces it leans on (`SeyfertRegistry` augmentation, `InteractionResponseType`, command/option/event/middleware pipeline, camelCase event names) ARE verified below against `./src`.

## Key APIs (verified)

Core seyfert APIs the toolkit consumes (all importable from `'seyfert'` root):

- `InteractionResponseType` — runtime enum, exported via `export * from './types'`. `ChannelMessageWithSource = 4`, `DeferredChannelMessageWithSource`, etc. exist. Defined `src/types/payloads/_interactions/responses.ts:71-83`.
- `SeyfertRegistry` — the empty augmentation interface (`src/client/plugins/types.ts:32` — literally `export interface SeyfertRegistry {}`). The doc's `declare module 'seyfert' { interface SeyfertRegistry { middlewares: typeof middlewares } }` is the current, correct registration shape.
- `RegisteredMiddlewares` — derived as `SeyfertRegistry extends { middlewares: infer M } ? M : {}` (`src/commands/decorators.ts:14`). `ResolvedRegisteredMiddlewares = RegisteredMiddlewares & RegisteredPluginMiddlewares` (`:19`).
- `RegisteredPlugins` — `SeyfertRegistry extends { plugins: infer T extends readonly AnySeyfertPlugin[] } ? ... ` (`src/client/plugins/types.ts:286`) — augment `plugins` in the same registry if a tested plugin contributes commands/middlewares.
- `DefaultLocale` — `SeyfertRegistry extends { langs: infer L } ? L : {}` (`src/commands/applications/shared.ts:26`).
- `UsingClient` — augmented client type carried by `ctx.client` (`src/commands/applications/shared.ts`, used `src/commands/basecontext.ts`).
- `createMiddleware(({ context, next, stop }) => ...)` — `src/commands/applications/options.ts:213`. Callback arg has NO `pass` (v5). `StopFunction = (error?: string | null) => void` (`shared.ts:20`): `stop()`/`stop(null)` = silent skip, `stop('reason')` = deny → `onMiddlewaresError`. `next()` continues.
- `middlewares('a','b')` typed helper — `src/commands/decorators.ts:184`, `<const T extends readonly MiddlewareKey[]>` — infers the registered name union; pass the result to `@Middlewares([...])`.
- Event handler names are camelCase. `createEvent({ data: { name } })` takes `ClientNameEvents`, built from raw gateway keys via `ReplaceRegex.camel(x.toLowerCase())` (`src/events/handler.ts:252`), e.g. `'guildMemberAdd'`, `'channelCreate'`.

External (NOT in seyfert-core — doc-authoritative, verify version in target project): `createMockBot`, `TEST_BOT_ID`, `bot.slash/autocomplete/userMenu/clickButton/selectMenu/fillModal/say/emit`, `bot.actor()`, `bot.defaultUser`, `registeredEvents()`, `Dispatch`, `.until()`, `.fillModal()`, `.timeoutModal()`, `waitForAction()`, result views (`result.content`, `result.deferred`, `result.replies`, `result.reply?.body`, `result.edits`, `result.followups`, `result.actions`, `result.embedView(s)`, `result.embed(s)`, `result.component(...)`, `result.components`, `result.textDisplays`), entity-option builders (`apiUser`, `userOption`, `channelOption`, `roleOption`, `mentionableOption`, `attachmentOption`).

## Code Examples (verified)

Core-import lines are correct; toolkit imports come from `@slipher/testing`.

### 1. One-shot slash run through the real pipeline

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { GreetCommand } from '../src/commands/greet';

test('greet replies through the real pipeline', async () => {
	await using bot = await createMockBot({ commands: [GreetCommand] });
	// by-name payload; or bot.slash(GreetCommand, { options }) for option inference
	const result = await bot.slash({ name: 'greet', options: { name: 'slipher' } });

	expect(result.content).toBe('Hello, slipher!');
});
```

### 2. Fully typed test with middleware augmentation

`SeyfertRegistry` is the v5 augmentation entry (NOT `RegisteredMiddlewares`/`UsingClient` directly — those are derived). The real client boots, so the same augmentation your bot ships applies in the test.

```ts
import { InteractionResponseType } from 'seyfert';
import { TEST_BOT_ID, createMockBot } from '@slipher/testing';
import { GuardedCommand } from '../src/commands/guarded';
import { middlewares } from '../src/middlewares';

declare module 'seyfert' {
	interface SeyfertRegistry { middlewares: typeof middlewares }
}

await using bot = await createMockBot({
	botId: TEST_BOT_ID,
	commands: [GuardedCommand],
	middlewares, // readonly arrays accepted (v5)
});
const result = await bot.slash({ name: 'guarded' });

expect(result.reply?.body).toMatchObject({
	type: InteractionResponseType.ChannelMessageWithSource, // verified enum member = 4
	data: { content: 'passed' },
});
```

The `GuardedCommand` fixture's middleware uses v5 control flow (`pass()` is gone):

```ts
import { createMiddleware } from 'seyfert';

export const auth = createMiddleware<void>(({ context, next, stop }) => {
	if (!context.member) return stop();             // silent skip (was pass())
	if (isBanned(context.author.id)) return stop('You are banned'); // deny → error reply
	return next();
});

export const middlewares = { auth };
```

### 3. Step-through with `.until(...)` (assert the outgoing REST body)

`Routes` is NOT a seyfert-core runtime export (see corrections) — it is the toolkit / `discord-api-types` route matcher, or use a predicate over the recorded action body.

```ts
const dispatch = bot.slash({ name: 'ban', options: { user: userOption(apiUser({ id: '42' })) } });

const ban = await dispatch.until(Routes.ban);   // suspends when the matched REST call STARTS
expect(ban.body).toMatchObject({ delete_message_seconds: 0 });
// or a predicate matcher, no Routes import needed:
// const ban = await dispatch.until(a => a.method === 'PUT' && a.url.includes('/bans/'));

const result = await dispatch;                  // releases checkpoint, runs run() to completion
expect(result.content).toBe('Banned');
```

### 4. Autocomplete dispatch

`bot.autocomplete` names the focused option and its current value; assert the suggestion list your handler responded with.

```ts
const result = await bot.autocomplete({ name: 'search', focused: 'query', value: 'sey' });
// the choices your createXOption({ autocomplete }) handler returned via interaction.respond([...])
expect(result.choices).toEqual([{ name: 'seyfert', value: 'seyfert' }]);
```

### 5. Component + modal flow on one persistent bot

`clickButton` / `selectMenu` / `fillModal` take the component `custom_id`. A dispatch that opens a modal is driven inline with `.fillModal(...)` (or `.timeoutModal()`).

```ts
// open a modal from a button, then submit it in a single dispatch
const result = await bot
	.clickButton('profile/edit')
	.fillModal('profile-modal', { username: 'slipher', bio: 'gg' });
expect(result.content).toBe('Profile saved');

// a select menu (second arg = chosen values array)
const picked = await bot.selectMenu('pick-color', ['red']);
expect(picked.content).toContain('red');
```

### 6. Entity options (resolved block populated)

```ts
import { apiUser, userOption, channelOption } from '@slipher/testing';

const result = await bot.slash({
	name: 'ban',
	options: {
		user: userOption(apiUser({ id: '42', username: 'spammer' })),
		channel: channelOption({ id: '99', type: 0 }),
		reason: 'spam',
	},
});
// ctx.options.user resolves to a real entity, not a bare id (option resolver reads `resolved`)
```

### 7. Gateway emit — RAW gateway name, camelCase handler

Pass the RAW dispatch name (UPPER_SNAKE); the toolkit routes it to your camelCase Seyfert handler (`guildMemberAdd`). `emit` throws if no handler ran unless you opt out.

```ts
await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload);                          // throws if no handler ran
await bot.emit('CHANNEL_CREATE', rawChannelPayload, { allowNoHandler: true }); // seed-only world state
// registeredEvents() lists what's actually wired if an emit unexpectedly throws.
```

### 8. Persistent actor / multi-step journey

The mock bot is long-lived: cache, collectors, cooldowns and world mutations persist across dispatches. Bind identity once with `bot.actor()`.

```ts
const alice = bot.actor({ member: aliceMember, guildId: guild.id, channel });

await alice.slash({ name: 'poll' });
await alice.clickButton('poll/yes');
await bot.emit('GUILD_MEMBER_ADD', newMember, { allowNoHandler: true });
const result = await alice.slash({ name: 'results' });

expect(result.content).toContain('1 vote');
```

### 9. Typed result views vs raw wire shapes

```ts
const result = await bot.slash({ name: 'card' });

// typed views — everyday behavior assertions:
expect(result.embedView?.title).toBe('Profile');
expect(result.component('Approve')?.customId).toBe('card/approve');
expect(result.textDisplays?.[0]).toContain('Level 5'); // components-v2

// raw shapes — only when the Discord wire payload itself is the contract:
expect(result.embeds?.[0]).toMatchObject({ title: 'Profile' });
expect(result.reply?.body).toMatchObject({ type: InteractionResponseType.ChannelMessageWithSource });
```

## Common patterns / gotchas

- **Lazy dispatch.** Nothing runs until you `await` the `Dispatch` or call `.until(...)`. `.until()` must be called BEFORE the first `await`; calling it on a settled dispatch throws (nothing left to step). Awaiting always releases the checkpoint, so a half-stepped dispatch can never deadlock.
- **No rejection on command errors.** Seyfert routes thrown command errors through error hooks; a throwing/rejecting middleware **rejects the runner → `onInternalError`** (not a denial). The dispatch still resolves — assert the user-facing error reply, not a thrown exception.
- **`stop()` not `pass()`.** Middleware fixtures must use v5 control flow: `stop()` skips silently, `stop('reason')` denies and reaches `onMiddlewaresError` (whose `metadata.middleware` now names the middleware that actually called `stop()`). `pass()` no longer exists.
- **Event casing is two-layer.** `bot.emit` takes the RAW gateway name (`'GUILD_MEMBER_ADD'`); the registered handler/`registeredEvents()` name is camelCase (`'guildMemberAdd'`). A mis-cased emit silently runs no handler — `emit` throws to catch exactly that unless `{ allowNoHandler: true }`.
- **Use entity-option builders, never bare ids.** `userOption(apiUser(...))`, `channelOption`, `roleOption`, `mentionableOption`, `attachmentOption` populate the `resolved` block the option resolver reads (`src/commands/optionresolver.ts`), so `ctx.options.user` is a real entity.
- **`result.actions` is dispatch-scoped.** Work the command awaited lands in `edits`/`followups`/`actions`; fire-and-forget work (timers, un-awaited promises) only shows up via `waitForAction()`.
- **`write`/`editOrReply` return `void`** unless the response flag (`withResponse`/`fetchReply`) is `true` — relevant when a command under test expects a returned `Message` to chain off.
- **Readonly arrays accepted.** v5 `globalMiddlewares` and command option arrays are `readonly`; passing readonly tuples to `createMockBot`/client options is fine.

## Doc vs Source Corrections

- `Routes` is NOT a runtime value exported from seyfert. `'seyfert'` only exports route *types* (`APIRoutes`, `GuildRoutes`, ...) via `src/api/index.ts` -> `./Routes`; the runtime route proxy is built per-client by the `Router` class (`src/api/Router.ts`, typed `APIRoutes`) and reached as `client.proxy`, not as a standalone `Routes` symbol. The `Routes.ban` matcher in the doc therefore comes from the toolkit or `discord-api-types`, NOT from `import { Routes } from 'seyfert'`. -> Verify its import in the target project, or use a predicate matcher.
- Event name casing: the doc emits raw gateway names (`'GUILD_MEMBER_ADD'`), correct for the toolkit's `emit`, but the Seyfert handler/`registeredEvents()` name is camelCase (`'guildMemberAdd'`) per `ClientNameEvents` (`src/events/handler.ts:252`). The doc's own warning about mis-casing already reflects this; just be aware the two layers use different casings.
- Middleware control flow changed in v5: `pass()` was replaced by `stop()` (commit 22eb832). A `GuardedCommand` middleware that "blocks" should call `stop()`/`stop('reason')`/`next()` — `pass()` no longer exists. Not in the doc's code, but relevant when authoring the guarded fixture (Example 2).
- Augmentation surface: the doc prose mentions "`RegisteredMiddlewares`, `UsingClient`, and `DefaultLocale` apply" — true, but in v5 you AUGMENT `SeyfertRegistry` (its `client`/`middlewares`/`langs`/`plugins` keys); those three types are derived from it. The doc's actual code already augments `SeyfertRegistry` correctly. `ParseMiddlewares` is gone — `typeof middlewares` is enough.
- `globalMiddlewares` and command option arrays are now `readonly` (commits 22eb832, base.ts) — passing readonly arrays to `createMockBot`/client options is accepted.

## Source Anchors

- `src/index.ts` (root barrel; `createEvent`, `export * from './types'`, `./api`, `./commands`)
- `src/types/payloads/_interactions/responses.ts:71-83` (`InteractionResponseType` enum; `ChannelMessageWithSource = 4`)
- `src/client/plugins/types.ts:32` (`SeyfertRegistry`), `:286` (`RegisteredPlugins`)
- `src/commands/decorators.ts:14-20` (`RegisteredMiddlewares`, `ResolvedRegisteredMiddlewares`), `:184` (`middlewares(...)` helper)
- `src/commands/applications/shared.ts:20` (`StopFunction`), `:26` (`DefaultLocale`), `:57-58` (`{ next, stop }` middleware payload — no `pass`)
- `src/commands/applications/options.ts:213` (`createMiddleware`)
- `src/api/index.ts`, `src/api/Router.ts` (route types vs runtime proxy — no `Routes` value export)
- `src/events/handler.ts:252` (camelCase `ClientNameEvents`)
- `src/commands/optionresolver.ts` (resolved-entity reading)

## Agent Guidance

- Treat this page as EXTERNAL: the dispatch surface lives in `@slipher/testing`. Confirm the package is installed and pin/verify its version before relying on exact signatures; the only hard guarantees from seyfert-core are the augmentation interface (`SeyfertRegistry`), the `InteractionResponseType` enum, `createMiddleware`/`stop` semantics, and the command/event pipeline behavior.
- When writing the `Routes.ban` matcher, do not auto-suggest `import { Routes } from 'seyfert'` — it will not resolve. Point users to the toolkit's matcher import (or `discord-api-types` `Routes`), or a predicate over the recorded action body.
- Always author middleware fixtures with v5 `stop()` (not `pass()`); show `createMiddleware(({ context, next, stop }) => ...)`.
- Use the `*Option(apiUser(...))` builders for entity options, not bare ids.
- Prefer typed result views (`result.content`, `result.embedView`, `result.component(...)`) for behavior assertions; reserve raw shapes (`result.reply?.body`, `result.embeds`) for wire-contract tests.
- Repeat the execution-model gotchas: lazy dispatch, `.until()` before `await`, awaiting releases checkpoints, dispatches don't reject on command errors, `emit` throws on no-handler unless `{ allowNoHandler: true }`.
