# Unit Tests (Fixtures)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/unit-tests
Coverage reference: testing.md
Verification status: Source-verified (core seyfert APIs) + external package (`@slipher/testing`, doc-authoritative)

## Page Summary

The fast testing path: exercise a single command's `run()` body in isolation using
plain fixtures from the **external** `@slipher/testing` toolkit — no client boots. You
build a stand-in `CommandContext` with `mockCommandContext()`, call `run()` directly, and
assert against a recorded `responses` array. Use fixtures for pure `run()` logic; switch to
the mock bot for option parsing, middlewares, permissions, components, or events. The
toolkit API (`mockCommandContext`, `mockUser`, `ctx.run`, `ctx.responses`,
`ctx.lastResponse`, `ctx.clearResponses`) is NOT part of core Seyfert — verify its version
in the target project. The core Seyfert APIs it dispatches against ARE verified below.

## Key APIs (verified)

Core seyfert APIs (all root-importable from `'seyfert'`):

- `Command` (abstract base for chat commands) — `src/commands/applications/chat.ts:361`
- `@Declare(options)` decorator — `src/commands/decorators.ts:195`
- `@Options(record | SubCommand[])` decorator (overloaded; record keys MUST be lowercase) — `src/commands/decorators.ts:161-163`
- `createUserOption(data)` — `src/commands/applications/options.ts:184`
- `createStringOption(...)` / `createIntegerOption(...)` / `createBooleanOption(...)` and siblings — `src/commands/applications/options.ts:146,154,178`
- `CommandContext<T>` — `src/commands/applications/chatcontext.ts`
  - `.options: ContextOptions<T>` (field, defaults to `{}`) — chatcontext.ts:68
  - `.write(body, withResponse?)` — chatcontext.ts:88 (returns `void` unless `withResponse === true`)
  - `.editOrReply(body, withResponse?)` — chatcontext.ts:145 (delegates to `write` when no interaction)
  - `.editResponse(body)` — chatcontext.ts:126 (EXISTS on CommandContext; removed only on ComponentContext in v5)
  - `.followup(body)` — chatcontext.ts:156
  - `.modal(body, options?)` — chatcontext.ts:99-101 (throws on prefix/no-interaction contexts)
  - `.deferReply(ephemeral?, withResponse?)` — chatcontext.ts:107
  - `.t` — i18n proxy (`SeyfertLocale`); resolve with `.get(locale?)` (`langs/router.ts:60`)
- Barrels: root `seyfert` re-exports `./commands` (src/index.ts:21), which re-exports
  `decorators`, `applications/chat`, `applications/chatcontext`, `applications/options`
  (src/commands/index.ts). No deep imports needed for any of the above.

External toolkit APIs (`@slipher/testing`, doc-authoritative, NOT in core Seyfert — verify version in target project):

- `mockCommandContext(CommandClass, { options })` — preferred class form; infers option
  types from the class, derives name from `@Declare`, binds the command so `ctx.run()` takes
  no argument.
- `mockCommandContext({ commandName, options })` — object form; builds a plain response sink.
- `mockUser({ id })` — user fixture factory.
- `ctx.run()` — runs the bound command's real `run()` body.
- `ctx.responses` — array every reply method pushes onto.
- `ctx.lastResponse()` — latest recorded reply.
- `ctx.clearResponses()` — reset between phases.

## Code Examples (verified)

### 1. Command under test + first test (the canonical flow)

Command file you ship — real `@Declare`/`@Options`, never rewritten for testing:

```ts
// commands/ban.ts
import { Command, Declare, Options, createUserOption, type CommandContext } from 'seyfert';

const options = {
	user: createUserOption({ description: 'User to ban', required: true }),
};

@Declare({ name: 'ban', description: 'Ban a user' })
@Options(options)
export default class BanCommand extends Command {
	async run(ctx: CommandContext<typeof options>) {
		await ctx.write({ content: `Banned ${ctx.options.user.id}` });
	}
}
```

The test (external toolkit):

```ts
// ban.test.ts
import { mockCommandContext, mockUser } from '@slipher/testing';
import { expect, test } from 'vitest';
import BanCommand from './commands/ban';

test('replies after banning', async () => {
	// options are typed from BanCommand — no manual generic, no cast
	const ctx = mockCommandContext(BanCommand, {
		options: { user: mockUser({ id: '123' }) },
	});

	// the command is already bound, so run() needs no argument
	await ctx.run();

	// every reply lands in ctx.responses, so you assert without spies
	expect(ctx.lastResponse()).toMatchObject({
		content: expect.stringContaining('Banned'),
	});
});
```

Asserting directly on the array (equivalent to `lastResponse()`):

```ts
expect(ctx.responses.at(-1)).toMatchObject({
	content: expect.stringContaining('Banned'),
});
```

### 2. Branching logic — assert each path independently

The reason fixtures exist: cover every `if` in `run()` with a one-call test. Note option
record keys are lowercase (v5 enforces this at compile time) and `choices` use `as const`.

```ts
// commands/roll.ts
import { Command, Declare, Options, createIntegerOption, type CommandContext } from 'seyfert';

const options = {
	sides: createIntegerOption({
		description: 'Die size',
		required: true,
		choices: [
			{ name: 'D6', value: 6 },
			{ name: 'D20', value: 20 },
		] as const,
	}),
};

@Declare({ name: 'roll', description: 'Roll a die' })
@Options(options)
export default class RollCommand extends Command {
	async run(ctx: CommandContext<typeof options>) {
		if (ctx.options.sides <= 0) {
			return ctx.write({ content: 'Invalid die', flags: 64 });
		}
		const result = 1 + (ctx.options.sides % 7); // deterministic stub for the test
		await ctx.write({ content: `You rolled ${result}` });
	}
}
```

```ts
// roll.test.ts
import { mockCommandContext } from '@slipher/testing';
import { expect, test } from 'vitest';
import RollCommand from './commands/roll';

test('valid roll replies with a result', async () => {
	const ctx = mockCommandContext(RollCommand, { options: { sides: 20 } });
	await ctx.run();
	expect(ctx.lastResponse()).toMatchObject({ content: expect.stringContaining('rolled') });
});

test('rejects a non-positive die', async () => {
	const ctx = mockCommandContext(RollCommand, { options: { sides: 0 } });
	await ctx.run();
	expect(ctx.lastResponse()).toMatchObject({ content: 'Invalid die', flags: 64 });
});
```

### 3. Multiple responses — `clearResponses()` between phases

`write`, `editOrReply`, and `followup` all push onto the SAME `ctx.responses` array, in
call order. Assert the sequence, then reset for the next phase.

```ts
// commands/setup.ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'setup', description: 'Multi-step setup' })
export default class SetupCommand extends Command {
	async run(ctx: CommandContext) {
		await ctx.write({ content: 'Step 1: creating roles…' });
		await ctx.followup({ content: 'Step 2: done' });
	}
}
```

```ts
// setup.test.ts
import { mockCommandContext } from '@slipher/testing';
import { expect, test } from 'vitest';
import SetupCommand from './commands/setup';

test('emits both steps in order', async () => {
	const ctx = mockCommandContext(SetupCommand, { options: {} });
	await ctx.run();

	expect(ctx.responses).toHaveLength(2);
	expect(ctx.responses[0]).toMatchObject({ content: expect.stringContaining('Step 1') });
	expect(ctx.responses[1]).toMatchObject({ content: expect.stringContaining('Step 2') });

	// reuse the same fixture for a second phase
	ctx.clearResponses();
	await ctx.run();
	expect(ctx.responses).toHaveLength(2);
});
```

### 4. Mocking a service the command calls (stub injection)

`run()` logic usually leans on a service. Stub it so the test stays a single function call
and never touches Discord/DB. Inject via your own service locator, or spy with Vitest.

```ts
// commands/balance.ts
import { Command, Declare, Options, createUserOption, type CommandContext } from 'seyfert';
import { economy } from '../services/economy'; // your module

const options = { user: createUserOption({ description: 'Target', required: true }) };

@Declare({ name: 'balance', description: 'Show a balance' })
@Options(options)
export default class BalanceCommand extends Command {
	async run(ctx: CommandContext<typeof options>) {
		const coins = await economy.getBalance(ctx.options.user.id);
		await ctx.editOrReply({ content: `Balance: ${coins}` });
	}
}
```

```ts
// balance.test.ts
import { mockCommandContext, mockUser } from '@slipher/testing';
import { expect, test, vi } from 'vitest';
import BalanceCommand from './commands/balance';
import { economy } from '../services/economy';

test('shows the fetched balance', async () => {
	vi.spyOn(economy, 'getBalance').mockResolvedValue(42);

	const ctx = mockCommandContext(BalanceCommand, {
		options: { user: mockUser({ id: '999' }) },
	});
	await ctx.run();

	expect(economy.getBalance).toHaveBeenCalledWith('999');
	expect(ctx.lastResponse()).toMatchObject({ content: 'Balance: 42' });
});
```

## Common patterns / gotchas

- `ctx.run()` SKIPS option parsing, middlewares, and permissions on purpose. Options you
  pass to `mockCommandContext` are injected as-is into `ctx.options` — they are NOT resolved
  from raw interaction data. If a command relies on a resolved option's side effects (cache
  hydration, attachment download), use the full mock bot instead.
- Prefer the class form `mockCommandContext(BanCommand, { options })` over the object form:
  it infers option types and binds the command, removing manual generics and `as unknown as`
  casts.
- Always pass `options` even when the command has none (`options: {}`) so `ctx.options` is a
  real object, mirroring core's default of `{}`.
- v5: option record keys are lowercase or it won't compile; `choices`/`channel_types` are
  readonly, so annotate them `as const`. (changelog: "Lowercase option keys".)
- v5 return shapes: `write`/`editOrReply` return `void` unless the response flag is `true`.
  Don't assert on a returned `Message` in a fixture test — assert on `ctx.responses`.
- `editResponse` EXISTS on `CommandContext` (chatcontext.ts:126). It was removed only from
  `ComponentContext` in v5 — that distinction matters when you later test components.
- `ctx.modal(...)` throws `Cannot use modal without an interaction.` in a prefix/no-interaction
  context. Test modal-driven flows with the mock bot, not fixtures.
- i18n: if `run()` reads `ctx.t`, supply a locale mock via the toolkit or assert on the
  resolved key; `ctx.t.get(locale?)` resolves the `SeyfertLocale` proxy (langs/router.ts:60).
- Rule of thumb: pure `run()` logic → fixtures. Option parsing, middlewares, permissions,
  components, or events → the mock bot (`/docs/testing/writing-tests/setup`).

## Doc vs Source Corrections

- None for core APIs — `Command`, `@Declare`, `@Options`, `createUserOption`,
  `createStringOption`/`createIntegerOption`, `CommandContext`, and
  `ctx.write`/`editOrReply`/`editResponse`/`followup`/`modal` all match the docs and are
  root-importable from `'seyfert'`. Signatures confirmed against chatcontext.ts.
- Note (not a correction): the `mockCommandContext`/`mockUser`/`ctx.responses`/`ctx.run`
  surface is entirely from `@slipher/testing` and cannot be validated against core Seyfert.
  Confirm names/signatures against the installed toolkit version before relying on them.
- v5 reminder enforced in examples above: lowercase option keys, `as const` on `choices`,
  and `void` returns from `write`/`editOrReply` (assert on `ctx.responses`, not the return).

## Source Anchors

- src/index.ts:21 (root re-export of `./commands`)
- src/commands/index.ts (commands barrel)
- src/commands/decorators.ts:161-163, 195 (`Options`, `Declare`)
- src/commands/applications/chat.ts:361 (`Command`)
- src/commands/applications/options.ts:146, 154, 178, 184 (`createStringOption`, `createIntegerOption`, `createBooleanOption`, `createUserOption`)
- src/commands/applications/chatcontext.ts:68, 88, 99-101, 107, 126, 145, 156 (`options`, `write`, `modal`, `deferReply`, `editResponse`, `editOrReply`, `followup`)
- src/langs/router.ts:60 (`ctx.t` / `SeyfertLocale.get`)

## Agent Guidance

- Use this fixture path when you only care about the logic inside one `run()` — branching,
  the reply body, stub calls. It is dramatically faster than the mock bot because nothing
  boots; each test is one function call.
- Rule of thumb: pure `run()` logic -> fixtures. Anything touching option parsing,
  middlewares, permissions, components, or events -> the mock bot
  (`/docs/testing/writing-tests/setup`).
- Mock services/IO (DB, REST, economy) with `vi.spyOn` so the test stays a pure function
  call; assert both the call (`toHaveBeenCalledWith`) and the reply (`ctx.responses`).
- For multi-reply commands, assert `ctx.responses` length and order; use `clearResponses()`
  to reuse one fixture across phases.
- Prefer the class form `mockCommandContext(BanCommand, { options })` over the object form:
  it infers option types and binds the command, removing manual generics and casts.
- Gotcha: `@slipher/testing` is a third-party package, NOT shipped by seyfert. It is not in
  this repo. Install it separately and pin/verify its version in the target project; its API
  may drift independently of core Seyfert.
- The command file used in tests must be the REAL command (real `@Declare`/`@Options`); do
  not rewrite it for testing. The default-export class is imported directly.
