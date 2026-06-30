# Toolkit Defaults & Scope

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/defaults
Coverage reference: testing.md
Verification status: Source-verified (core seyfert anchors) + EXTERNAL package (`@slipher/testing`, doc-authoritative)

## Page Summary

How to organize a real mock-bot test suite with `@slipher/testing`, the common
pitfalls, and the package's pre-1.0 defaults (single test user, strict REST,
newest-first messages, throw-on-handler-error) and scope.

`@slipher/testing` is an EXTERNAL package — it is NOT part of seyfert-core. Every
toolkit symbol below (`createMockBot`, the `bot.*` dispatchers, `bot.world`,
`bot.rest.intercept`, the `TEST_*` ids, `onUnhandledRest`, `onCommandError`) is
**documentation-authoritative** and must be verified against the version installed
in the target project. The core seyfert surface the toolkit dispatches against —
`@Declare` commands, components/modals/events, snowflake/BigInt helpers like
`avatarURL` and `createdAt` — IS verifiable in `./src` and is confirmed below.

## Key APIs (verified)

EXTERNAL (`@slipher/testing`, verify version in target project — NOT in seyfert-core):

- `createMockBot({ commands?, loadFromConfig?, onUnhandledRest?, onCommandError?, ... })`
  → an async-disposable mock bot (`await using bot = await createMockBot(...)`).
- Dispatchers (each returns a lazy `Dispatch`; await it, or call `.until(route)` to
  step through it, or `.fillModal(...)` / `.timeoutModal()` to drive a modal open):
  - `bot.slash({ name, group?, subcommand?, options? })` — raw by-name payload.
  - `bot.slash(GreetCommand, { options })` — class form, gives option inference.
  - `bot.autocomplete({ name, focused, value })`
  - `bot.userMenu({ name, target })` / context-menu dispatch.
  - `bot.clickButton(customId)` / `bot.selectMenu(customId, values[])`
  - `bot.fillModal(customId, fields)` (also a method on a pending `Dispatch`).
  - `bot.say('!prefix ...')` — prefix/text command dispatch.
  - `bot.emit('GATEWAY_EVENT', rawPayload, { allowNoHandler? })` — raw gateway shape.
  - `bot.actor({ member, guildId, channel })` — bind an identity for later dispatches.
- `Dispatch` result fields: `result.content`, `result.replies`, `result.reply?.body`
  (`{ type, data }`), `result.deferred`, `result.edits`, `result.followups`,
  `result.actions`, typed views `result.embedView(s)`, `result.component(label)`,
  `result.components`, `result.textDisplays`, raw `result.embeds`/`result.embed`,
  and `result.error` (ONLY populated under `onCommandError: 'capture'`).
- `bot.world` (`WorldStateReader`) — seed via `world.registerGuild/Channel/Member/
  Role/VoiceState/Message/BotMember(...)`; assert via three readers:
  - `bot.world.get.*` → exactly one, throws `WorldStateError` on 0/many.
  - `bot.world.query.*` → the single match or `undefined`.
  - `bot.world.all.*` → array of every match. Plus `snapshot()` / `diff()`.
- `bot.rest.intercept(route, handler)` (or `intercept('GET', '/path/:p', fn)`) —
  simulate Discord-side behavior the world does not model; use `apiError(...)`.
- `bot.close()` — manual teardown when `await using` is unavailable.
- Constants: `TEST_BOT_ID`, `TEST_APPLICATION_ID`, `TEST_GUILD_ID`,
  `TEST_CHANNEL_ID`, `TEST_USER_ID` — real numeric-string snowflakes.
- Options: `onUnhandledRest: 'error' | 'warn' | 'silent'` (default `'error'`);
  `onCommandError: 'throw' | 'capture'` (default `'throw'`).

Core seyfert APIs the toolkit dispatches against (verified in `./src`):

- `Declare(input)` decorator — `src/commands/decorators.ts:195`
  (`export function Declare<const T extends CommandDeclareOptions>(input: LowercaseDeclareName<T>)`).
  The toolkit resolves commands by their declared lowercase `name`.
- `User#avatarURL(options?)` — `src/structures/User.ts:41`; member/webhook variants in
  `GuildMember.ts` / `Webhook.ts`. BigInt-snowflake based.
- `DiscordBase#createdAt` (Date) / `#createdTimestamp` — `src/structures/extra/DiscordBase.ts:20,27`;
  decodes the snowflake id. This is why the toolkit's `TEST_*` ids are real numeric
  snowflakes — partial/fake ids would break these BigInt helpers.

## Code Examples (verified)

Focused spec — hand-pick the class under test, `await using` for cleanup:

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { BanCommand } from '../src/commands/moderation/ban';

test('/ban confirms', async () => {
	await using bot = await createMockBot({ commands: [BanCommand] });
	const result = await bot.slash({ name: 'ban' });

	expect(result.content).toContain('Banned');
});
```

Broad smoke spec — boot the production set from `seyfert.config` so it exercises the
same loaders `client.start()` would use:

```ts
import { createMockBot } from '@slipher/testing';
import { test } from 'vitest';

test('every command registers', async () => {
	await using bot = await createMockBot({ loadFromConfig: true });
	await bot.slash({ name: 'ping' });
});
```

Fallback when `await using` is unavailable (older runner/transpiler):

```ts
import { afterEach, beforeEach } from 'vitest';
import { createMockBot } from '@slipher/testing';
import { BanCommand } from '../src/commands/moderation/ban';

let bot: Awaited<ReturnType<typeof createMockBot>>;

beforeEach(async () => {
	bot = await createMockBot({ commands: [BanCommand] });
});
afterEach(() => bot.close());
```

Test id constants (real snowflakes that survive seyfert's BigInt helpers):

```ts
import {
	TEST_BOT_ID, // '900000000000000001'
	TEST_APPLICATION_ID, // '900000000000000002'
	TEST_GUILD_ID, // '900000000000000003'
	TEST_CHANNEL_ID, // '900000000000000004'
	TEST_USER_ID, // '900000000000000005'
} from '@slipher/testing';
```

## Recipes (new worked examples)

These exercise the core seyfert flows a bot author tests most. The seyfert-side code
(commands/components/modals/events) is `./src`-verified; the `bot.*` toolkit calls are
the external doc surface — pin them to your installed `@slipher/testing` version.

### 1. Subcommand + typed options

`@Declare` names are lowercase (v5 enforces this at compile time), and option record
keys must be lowercase too. Dispatch the leaf with `group` / `subcommand`:

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { UsersGroup } from '../src/commands/admin'; // @Declare admin -> users kick

test('/admin users kick', async () => {
	await using bot = await createMockBot({ commands: [UsersGroup] });
	const result = await bot.slash({
		name: 'admin',
		group: 'users',
		subcommand: 'kick',
		options: { reason: 'spam' }, // keys are lowercase to match the option record
	});

	expect(result.content).toContain('Kicked');
});
```

### 2. Button component flow

A `ComponentCommand` (or a collector) replies to a click. Dispatch by `customId`:

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { ConfirmButton } from '../src/components/confirm';

test('confirm button edits the message', async () => {
	await using bot = await createMockBot({ commands: [ConfirmButton] });
	const result = await bot.clickButton('confirm-ban');

	// ctx.update(...) inside the handler shows up as an edit reply
	expect(result.reply?.body).toMatchObject({ data: { content: 'Confirmed' } });
});
```

### 3. Modal flow (open then fill)

When a slash command opens a modal with `ctx.modal(...)`, drive both halves in one
call by chaining `.fillModal(...)` on the pending `Dispatch` (do NOT await first):

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { FeedbackCommand } from '../src/commands/feedback';

test('/feedback collects modal input', async () => {
	await using bot = await createMockBot({ commands: [FeedbackCommand] });

	const result = await bot
		.slash({ name: 'feedback' })
		.fillModal('feedback-modal', { rating: '5', notes: 'great' });

	expect(result.content).toContain('Thanks');
});
```

The modal handler reads fields via `ctx.getInputValue('rating', true)` on the
`ModalContext` (full accessor set in v5: `getInputValue`, `getChannels`, `getRoles`,
`getUsers`, `getRadioValues`, `getCheckboxValues`, ...).

### 4. Autocomplete

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { SearchCommand } from '../src/commands/search';

test('search autocomplete suggests tags', async () => {
	await using bot = await createMockBot({ commands: [SearchCommand] });
	const result = await bot.autocomplete({ name: 'search', focused: 'query', value: 'sey' });

	// integer/number options must respond with numeric values in v5; string ones with strings
	expect(result.choices?.map(c => c.name)).toContain('seyfert');
});
```

### 5. Gateway event + world assertion

`bot.emit` takes the RAW Discord event shape (full payload, not a partial patch).
Read the resulting state through the world reader rather than poking the cache:

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { GuildMemberAdd } from '../src/events/guildMemberAdd';

test('welcome message on join', async () => {
	await using bot = await createMockBot({ commands: [GuildMemberAdd] });
	const guild = bot.world.registerGuild({ name: 'Lab' });

	await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload, { allowNoHandler: true });

	const channel = bot.world.query.channel({ guildId: guild.id, name: 'welcome' });
	expect(channel?.lastMessage?.content).toContain('Welcome');
});
```

### 6. Permission-denied path via rest.intercept

The world models cache state, not every REST outcome. Stub a 403 to test your
command's error branch (e.g. a ban the bot can't perform):

```ts
import { createMockBot, apiError } from '@slipher/testing';
import { Routes } from 'discord-api-types/v10';
import { expect, test } from 'vitest';
import { BanCommand } from '../src/commands/moderation/ban';

test('/ban surfaces Missing Permissions', async () => {
	await using bot = await createMockBot({ commands: [BanCommand] });
	bot.rest.intercept(Routes.guildBans, () => apiError(403, 50013, 'Missing Permissions'));

	const result = await bot.slash({ name: 'ban', options: { user: TEST_USER_ID } });
	expect(result.content).toContain('cannot ban');
});
```

### 7. Multi-user / collector flow with a bound actor

Collectors only fire for the matching user. Bind an identity with `bot.actor(...)` so
the opener and the follow-up click come from the same member:

```ts
const guild = bot.world.registerGuild();
const channel = bot.world.registerChannel(guild.id);
const aliceMember = bot.world.registerMember(guild.id, { nick: 'alice' });
const alice = bot.actor({ member: aliceMember, guildId: guild.id, channel });

const dispatch = alice.slash({ name: 'poll' });
await dispatch.until(/* route the command awaits */);
const result = await alice.clickButton('vote-yes');
expect(result.content).toContain('1 vote');
```

## Current defaults (from docs, external package)

- Single default user (`TEST_USER_ID`).
- Strict `onUnhandledRest: 'error'`; opt into `'warn'` / `'silent'` per test.
- Newest-first message lists.
- An unhandled error inside a command/component/modal/event handler **rejects the
  `Dispatch` by default** (`onCommandError: 'throw'`); pass `onCommandError: 'capture'`
  to surface it on `result.error` instead (otherwise you `await` and catch the throw).

Scope: the mock bot is in-process and runner-agnostic — no WebSocket server, no fake
timers, no built-in assertions. It is long-lived: cache, collectors, plugin state,
cooldowns, and world mutations persist across dispatches in the same bot. Discord
behavior the world does not model is simulated via `bot.rest.intercept()`. Files ride
along raw as `result.reply?.files` and `action.files`; assert attachment presence
through the captured reply / recorded action.

## Common patterns / gotchas

- **One bot per test by default.** Boot is in-process and cheap; a fresh bot per test
  gives free state isolation. Reserve `loadFromConfig: true` for "every command
  registers" smoke specs.
- **Module-level state leaks.** `createMockBot` deep-clones the world, but module-level
  state in your command files persists within a worker — reset it in `beforeEach` to
  avoid order-dependent flakes (the classic "passes alone, fails in CI").
- **Naive green assertions.** `expect(result.content).toContain('ok')` also passes when
  `content` is `undefined`. Use the throwing readers (`response()`, the assertion
  helpers) or assert against a typed view so a silently-returning handler fails loud.
- **Don't await before driving a modal.** Chain `.fillModal(...)` / `.timeoutModal()`
  on the pending `Dispatch`; awaiting first releases the checkpoint. Awaiting always
  releases any active checkpoint, so a dispatch cannot deadlock.
- **`emit` wants full event shapes**, not partial patches, or the cache looks stale.
- **Same user for collectors.** Send a real message/click as the same user via
  `bot.actor(...)`; a different identity between opener and follow-up makes modal/
  collector flows hang (`waitFor` uses REAL timers).
- **`result.error` needs `'capture'`.** Under the default `'throw'`, await the dispatch
  and catch the rejection instead.

## Doc vs Source Corrections

- The MDX writes the interceptor as `bot.rest.intercept(...)` (a method on the bot),
  not a free `rest.intercept(...)`. The earlier note's shorthand `rest.intercept` is
  corrected here to `bot.rest.intercept`.
- World access is three readers — `bot.world.get.*` (throws on 0/many),
  `bot.world.query.*` (one or `undefined`), `bot.world.all.*` (array) — plus
  `snapshot()`/`diff()`. The earlier note described `bot.world` only generically.
- `@slipher/testing` is not in seyfert-core, so its API cannot be diffed against
  `./src`; treat the MDX as authoritative and pin behavior to the installed version.
- Default command-error behavior is unambiguous: `onCommandError: 'throw'` rejects the
  `Dispatch`; `'capture'` puts the error on `result.error`. No real conflict.
- Core anchors confirm the toolkit's design assumptions: `@Declare` is a real
  decorator (`src/commands/decorators.ts:195`) and the `TEST_*` ids must be valid
  snowflakes because `avatarURL`/`createdAt` decode them via BigInt.

## Source Anchors

- `src/commands/decorators.ts:195` — `Declare` command declaration decorator
  (`LowercaseDeclareName<T>` enforces the lowercase command name).
- `src/structures/User.ts:41` — `avatarURL(options?)` and snowflake-derived helpers.
- `src/structures/extra/DiscordBase.ts:20,27` — `createdTimestamp` / `createdAt`
  snowflake decode (`createdAt` returns a `Date`).
- `src/structures/GuildMember.ts`, `src/structures/Webhook.ts` — additional `avatarURL` variants.

## Agent Guidance

- Use this page when writing or debugging mock-bot tests. Because `@slipher/testing`
  is external and pre-1.0, ALWAYS verify the exact signature/option names against the
  version in the target project's `package.json` before relying on them.
- Prefer one fresh bot per test (`await using`); the boot is in-process and cheap.
  Use `loadFromConfig: true` only for broad smoke specs.
- Pick the dispatcher that matches the interaction: `slash` / `autocomplete` /
  `userMenu` / `clickButton` / `selectMenu` / `fillModal` / `say` / `emit`. For modal
  flows, chain `.fillModal(...)` on the pending dispatch instead of awaiting first.
- Assert state the reply doesn't surface (bans, reactions, pins, voters, ...) through
  the read-only `bot.world` readers (`get`/`query`/`all`) rather than poking internals.
- Root seyfert imports come from `'seyfert'`; the testing helpers and `TEST_*`
  constants come from `'@slipher/testing'`, never from `'seyfert'`.
