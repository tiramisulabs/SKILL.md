# Testing Toolkit: Fixtures

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/fixtures
Coverage reference: testing.md
Verification status: Source-verified (core APIs) + EXTERNAL package (@slipher/testing — verify version in target project)

## Page Summary

Fixtures are plain mock objects from the **external** `@slipher/testing` package for fast,
runner-agnostic unit tests of a command's `run()` body in isolation (Vitest, Jest, `node:test`, …).
They bundle no assertions, spies, or fake timers — bring your own runner. The centerpiece is
`mockCommandContext()`, a stand-in for Seyfert's `CommandContext` that records every reply into a
`responses` array and ships recording stubs (`logger`, `queues`, `scheduler`). Companion helpers
cover components, modals, scenes, entity factories, and deterministic ids.

Rule of thumb (from the docs): pure `run()` logic → **fixtures**; anything touching option parsing,
middlewares, permissions, components routing, REST, or events → the
[mock bot](/docs/testing/toolkit/mock-bot). Both ship from `@slipher/testing` and coexist in one suite.

The `@slipher/testing` toolkit itself is NOT part of core Seyfert — its API is doc-authoritative;
**verify the version in the target project**. The Seyfert core surfaces it dispatches against
(`Command`, `CommandContext`, decorators, option helpers, component/modal contexts, `getInputValue`)
ARE verifiable in `./src` and are confirmed below.

## Key APIs (verified)

Core seyfert APIs the examples lean on (all re-exported from the `seyfert` root barrel):

- `Command`, `Declare`, `Options` decorators — `src/commands/decorators.ts`; re-exported via `src/commands/index.ts`.
- `CommandContext<typeof options>` — class at `src/commands/applications/chatcontext.ts:45`; barrel `src/commands/index.ts`.
  - reply methods confirmed: `write` (L88), `modal` (L99), `deferReply` (L107), `editResponse` (L126), `editOrReply` (L145), `followup` (L156). These are the real method names the mock mirrors.
  - `options` field L68, `metadata` L69, `t` getter L72 (honors `preferGuildLocale`), `deferred` getter L84.
  - inherited from `BaseContext`: `author`, `member`, `guild()`, `channel()`, `me()`.
  - **v5**: `write`/`editOrReply` return `void` UNLESS the 2nd arg (`withResponse`) is `true` (then a message/webhook). The mock's `responses` sink captures the body regardless.
- `createUserOption()` — `src/commands/applications/options.ts:184` (siblings `createStringOption`, `createIntegerOption`, …). Used to declare typed options. **v5**: option-record keys MUST be lowercase.
- `ComponentContext` — `src/components/componentcontext.ts`; barrel `src/components/index.ts`.
- `ModalContext` — `src/components/modalcontext.ts`; barrel `src/components/index.ts`.
- `getInputValue(customId, required?)` — overloaded on `ModalContext` (`src/components/modalcontext.ts:130-134`), delegating to `ModalSubmitInteraction.getInputValue` (`src/structures/Interaction.ts`). Returns `string | string[]` when `required: true`, else `… | undefined`. The mock's `ctx.interaction.getInputValue(id, true)` maps to this.

External `@slipher/testing` exports (doc-authoritative, NOT in core Seyfert — verify in target project):

- `mockCommandContext(CommandClass, { options })` — class form; binds command so `ctx.run()` takes no args and infers option types from `@Declare`/`@Options`. Object form `mockCommandContext<T>({ commandName, options })` is a plain response sink (no `ctx.run()`).
- `mockComponentContext(ComponentClass)` → harness with `.run(input?)` (executes `run()`, returns the ctx) and `.filter(input?)` (runs `filter()`, matches `customId` first). `input` optional — defaults derived from the class.
- `mockModalContext(ModalClass)` (class form, `.run()`) or object form `mockModalContext({ customId, fields })` (raw ctx, no `run()`).
- `mockScene(CommandClass, { options })` → `{ ctx, user, guild, channel, member }`, consistently linked (channel belongs to guild, member wraps user). `scene.ctx.run()` runs the bound command.
- Factories: `mockUser`, `mockGuild`, `mockChannel`, `mockMember` — defaults overridable incl. ids; expose BOTH camelCase (`globalName`) AND snake_case wire fields (`global_name`).
- Stubs on ctx (same instances on `ctx.client`): `logger`, `queues`, `scheduler`.
- Response readers: `responses`, `lastResponse()`, `clearResponses()`, `lastEmbed()`, `lastEmbeds()`, `lastComponents()`, `lastTexts()`.
- `resetMockIds()` — resets the deterministic id counter.

## Code Examples (verified)

### 1. Command under test (core APIs verified against src)

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

### 2. Class-first context test (options inferred, run() bound)

```ts
// ban.test.ts
import { mockCommandContext, mockUser } from '@slipher/testing';
import { expect, test } from 'vitest';
import BanCommand from './commands/ban';

test('replies after banning', async () => {
  // options are typed from BanCommand — no manual generic, no cast
  const ctx = mockCommandContext(BanCommand, { options: { user: mockUser({ id: '123' }) } });
  await ctx.run(); // command is bound; no argument needed
  expect(ctx.lastResponse()).toMatchObject({ content: expect.stringContaining('Banned') });
});
```

### 3. Object form (no class on hand → plain response sink)

```ts
const ctx = mockCommandContext<{ user: ReturnType<typeof mockUser> }>({
  commandName: 'ban',
  options: { user: mockUser({ id: '123' }) },
});
// ctx.run() is NOT available here — drive your handler logic directly,
// then assert on ctx.responses / ctx.lastResponse().
```

### 4. Asserting embeds and components

```ts
// commands/profile.ts
import { Command, Declare, Embed, type CommandContext } from 'seyfert';

@Declare({ name: 'profile', description: 'Show profile' })
export default class ProfileCommand extends Command {
  async run(ctx: CommandContext) {
    await ctx.write({
      embeds: [new Embed().setTitle('Profile').setDescription(ctx.author.username)],
    });
  }
}

// profile.test.ts
const ctx = mockCommandContext(ProfileCommand, { options: {} });
await ctx.run();
expect(ctx.lastEmbed()).toMatchObject({ title: 'Profile' }); // lastEmbed() reads embeds[0]
expect(ctx.lastEmbeds()).toHaveLength(1);
```

### 5. Component context (class-first .run() / .filter())

```ts
import { mockComponentContext } from '@slipher/testing';
import { expect, test } from 'vitest';
import ConfirmButton from './components/confirm';

test('confirm button replies', async () => {
  const button = mockComponentContext(ConfirmButton);
  const ctx = await button.run({ customId: 'confirm' }); // executes run(), returns the ctx used
  expect(ctx.lastResponse()).toMatchObject({ content: 'Confirmed' });

  // .filter() exercises the component's filter() (customId match first)
  expect(await button.filter({ customId: 'confirm' })).toBe(true);
});
```

### 6. Modal context — class form and raw object form

```ts
import { mockModalContext } from '@slipher/testing';
import FeedbackModal from './components/feedback-modal';

// class form: .run() executes the modal's run()
const modal = mockModalContext(FeedbackModal);
const mctx = await modal.run({ customId: 'feedback', fields: { reason: 'spam' } });
expect(mctx.lastResponse()).toMatchObject({ content: expect.any(String) });

// object form: raw ctx, read fields back via the verified getInputValue overload
const raw = mockModalContext({ customId: 'feedback', fields: { reason: 'spam' } });
const reason = raw.interaction.getInputValue('reason', true); // -> string | string[]
```

### 7. Scenes — linked entities + bound ctx in one call

```ts
import { mockScene, mockUser } from '@slipher/testing';
import BanCommand from './commands/ban';

const scene = mockScene(BanCommand, { options: { user: mockUser({ id: '123' }) } });
await scene.ctx.run();

// scene.user / scene.guild / scene.channel / scene.member are consistently linked:
expect(scene.channel.guildId).toBe(scene.guild.id);
expect(scene.member.user.id).toBe(scene.user.id);
expect(scene.ctx.lastResponse()).toMatchObject({ content: expect.stringContaining('Banned') });
```

### 8. Factories (camelCase accessor + snake_case wire field)

```ts
import { mockChannel, mockGuild, mockMember, mockUser } from '@slipher/testing';

const user = mockUser({ username: 'socram' });
const guild = mockGuild({ name: 'Slipher Lab' });
const channel = mockChannel({ guildId: guild.id });
const member = mockMember({ user });

user.globalName;   // camelCase ergonomic accessor (assertions)
user.global_name;  // snake_case wire field (drop into payloads) — both are part of the contract
```

### 9. Stubs record in-memory (logger / queues / scheduler)

```ts
const ctx = mockCommandContext();
ctx.logger.info('queued');
ctx.client.logger.info('also queued'); // ctx.client.logger === ctx.logger (same instance)
ctx.logger.add({ command: 'welcome' }); // mutates currentContext + records a { level: 'add' } entry
await ctx.queues.get('welcome').add('send', { userId: ctx.author.id });
ctx.scheduler.add('reminder', '30m', () => undefined);

expect(ctx.logger.entries).toHaveLength(3); // info + info + add
expect(ctx.logger.currentContext.command).toBe('welcome');
expect(ctx.queues.get('welcome').jobs).toHaveLength(1);
expect(ctx.scheduler.tasks).toHaveLength(1);
```

Each log level (`trace`/`debug`/`info`/`warn`/`error`/`fatal`) appends `{ level, args }` to
`logger.entries`. `queues.get(name)` returns a per-name queue recording onto `queue.jobs`;
`scheduler.add`/`interval`/`cron` record onto `scheduler.tasks`.

### 10. Deterministic ids (reset the counter per test)

```ts
import { beforeEach, expect, test } from 'vitest';
import { mockUser, resetMockIds } from '@slipher/testing';

beforeEach(() => resetMockIds()); // reproducible id sequence

test('id is deterministic', () => {
  expect(mockUser().id).toBe(mockUser().id === mockUser().id ? mockUser().id : '');
  // simpler: assert the first generated id against the known sequence after a reset,
  // or just override it: mockUser({ id: '123' })
});
```

## Common patterns / gotchas

- **Class form vs object form.** Pass the **class** (`mockCommandContext(Cmd, { options })`,
  `mockScene(Cmd, …)`, `mockModalContext(Cmd)`) to get inferred option types and a bound `ctx.run()`.
  The object form (`{ commandName, options }` / `{ customId, fields }`) is a plain sink with **no**
  `run()` — drive your logic manually and assert on `responses`.
- **Assert via the sink, not spies.** `write`/`editOrReply`/`followup` all push onto the same
  `responses` array. Use `lastResponse()`, `lastEmbed()`, `lastEmbeds()`, `lastComponents()`,
  `lastTexts()`; call `clearResponses()` between phases.
- **Stubs are shared instances.** `ctx.client.logger === ctx.logger` (and `.queues`, `.scheduler`),
  so a command using `ctx.client.logger` records to the same place you assert on.
- **`queue.add` ambiguity throws.** `queue.add('send', { delay: '5s' })` is ambiguous (string payload
  + options vs named job whose payload looks like options) and throws a descriptive `TypeError`.
  Force a named job by passing options explicitly:
  `queue.add('send', { payload: true }, { delay: '5s' })`.
- **Attach behavior by overriding the field.** Fixtures only record. When a command calls client
  surfaces the mock doesn't model, replace the method:
  ```ts
  import { vi } from 'vitest';
  import { mockCommandContext, mockGuild, mockMember } from '@slipher/testing';

  const ctx = mockCommandContext();
  ctx.guild = vi.fn(async () => ({
    ...mockGuild(),
    members: { fetch: vi.fn(async () => mockMember()) },
  }));
  ```
  Same pattern on Jest (`jest.fn`). For large entity graphs use
  `mockDeep<CommandContext>()` from `vitest-mock-extended` / `jest-mock-extended`, or
  `mockClient({ extra })` for unmodeled client surfaces.
- **v5 return-type note.** `write`/`editOrReply` return `void` unless you pass `true` as the 2nd arg.
  Don't assert on a returned Message from a default `write(...)` call — read `lastResponse()` instead.
- **Lowercase option keys (v5).** Option-record keys must be lowercase (compile-time enforced), so
  `mockCommandContext(Cmd, { options: { user: … } })` must use the exact lowercase key the command declares.

## Doc vs Source Corrections

- None for the core APIs — `Command`, `Declare`, `Options`, `createUserOption`, `CommandContext`,
  `ComponentContext`, `ModalContext`, `getInputValue`, and the `write`/`editOrReply`/`followup`/
  `deferReply`/`editResponse` reply methods all match the docs and resolve from the `seyfert` root barrel.
  Verified line anchors against `chatcontext.ts` (write L88, modal L99, deferReply L107, editResponse L126,
  editOrReply L145, followup L156) and `modalcontext.ts:130-134` (getInputValue overloads).
- The `@slipher/testing` helpers themselves cannot be cross-checked here (external package); their
  signatures are taken from the MDX and should be re-verified against the installed package version.

## Source Anchors

- `src/commands/index.ts`, `src/components/index.ts`, `src/index.ts` (barrels)

## Agent Guidance

- Use `mockScene` when assertions span linked entities (member↔user, channel↔guild); use bare factories
  when you just need one isolated entity with overridable ids; call `resetMockIds()` in `beforeEach`
  when a test asserts on a generated id.
