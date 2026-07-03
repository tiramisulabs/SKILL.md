# Testing Toolkit: Assertions

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/assertions

Coverage reference: testing.md

Verification status: Source-verified (core touchpoints) + EXTERNAL package (`@slipher/testing`, doc-authoritative — verify version in target project)

## Page Summary

`@slipher/testing` ships two runner-agnostic readers that throw instead of passing green when nothing happened. The trap they solve: `expect(result.content).toContain('ok')` is satisfied by `content` being `undefined`, so a handler that silently returned still passes. The readers fail loud instead.

- `outcome(result)` reads the dispatch **lifecycle** — did it reply, defer, open a modal, get denied, or throw?
- `rendered(subject)` reads the **UI** the handler produced — messages, embeds, buttons, selects, inputs, modals, and Components V2 containers.

Both expose the same three modes: `.get.*` (throws unless exactly one match), `.query.*` (value or `undefined`), and `.all.*` (array). Neither couples to a test runner, so they behave identically under Vitest, Jest, `node:test`, or a bare script. The toolkit is an external package, not part of core Seyfert; the core APIs it dispatches against (`Routes`, `SeyfertError`, middleware `next`/`stop`, commands/components/modals) ARE verifiable in `./src`.

## External package note

`@slipher/testing` is NOT in core Seyfert. The toolkit API below (`outcome`, `rendered`, `createMockBot`, `apiUser`, `userOption`, `bot.slash`, `bot.rest.fail`, `bot.rest.intercept`, `bot.waitForAction`, `DiscordErrors`, query/error types) is doc-authoritative — verify the version and exact signatures in the target project's installed package. Everything in "Key APIs (verified — core touchpoints)" is verified against this repo.

## Key APIs (verified — core touchpoints)

- `Routes` — `seyfert` root export (`src/api/index.ts` -> `export * from './Routes'`). The doc's `Routes.ban`, `Routes.createMessage` come from here. No deep import needed.
- `SeyfertError` — `seyfert` root export; carries `code: SeyfertErrorCode` and `metadata?: Record<string, unknown>`, plus overloaded `static is(error)` / `static is(error, code)` and `toJSON()`. Constructor: `new SeyfertError(code, { metadata?, cause? })`. Confirms the doc claim that `bot.rest.fail` throws "the same code/metadata a production catch sees". `src/common/it/error.ts:21`.
- Middleware control flow — middleware receives `{ context, next, stop }`. `next(meta?)` advances (storing optional metadata); `stop()` / `stop(null)` with no arg ends the chain as a pass (`res({ pass: true })`); `stop(err)` denies with `{ error, metadata: { middleware, scope } }` where `scope` is `'global' | 'command'`. This is exactly what the toolkit's denial reader inspects. `src/commands/applications/chat.ts:247-256`.
- `StopFunction` is the middleware `stop` type (`chat.ts:41,247`); the standalone middleware `pass()` no longer exists — it was replaced by `stop()` with no argument (commit 22eb832). The toolkit's `denial({ kind: 'pass' })` classifies precisely this `res({ pass: true })` outcome.

## External API surface (doc-authoritative)

`outcome(result)` — lifecycle:
- `.get` / `.query` / `.all` — `.get.*` throws `OutcomeError` summarizing what the dispatch actually did (denied, errored, or no user-visible output).
- `response(query?)` — no arg asserts at least one reply/defer/modal/edit/followup (the case a naive `toContain` lets pass green). `ResponseQuery.kind` ∈ `'reply' | 'defer' | 'deferReply' | 'deferUpdate' | 'update' | 'modal' | 'autocomplete' | 'edit' | 'followup'` (`'defer'` matches either defer variant); also `{ ephemeral: true }`. Returns `OutcomeResponse` with `events`, `deferred`/`deferredReply`/`deferredUpdate`/`ephemeral` flags, optional `modal`, and `raw.replies`/`raw.edits`/`raw.followups`.
- `denial(query?)` — `DenialQuery` fields all optional, checked only when present: `kind` ∈ `'stop' | 'pass' | 'no-next' | 'permissions' | 'bot-permissions'`, `middleware` (name), `missing` (permission name or array — all must appear). Returns `OutcomeDenial` with `denialKind`, `middleware`, `missing`, `raw`.
- `error(matcher?)` — requires `onCommandError: 'capture'` (default `'throw'` rejects the dispatch instead, so you catch the rejection). Matcher: substring | `RegExp` | predicate over the error. Returns `OutcomeCapturedError` with the raw thrown value on `.error`.

`rendered(subject)` — UI (accepts a result, bot, message array, or builder):
- `.get` throws `RenderedOutputError` (lists what was rendered + near misses).
- `message(query?)` — `content`, `id`, `channelId`, `ephemeral`, `transport`.
- `embed(query?)` — `title`, `description`, `contains`, `author`, `footer`, `color`, `field`.
- `component('button', query?)`, `select(query?)`, `input(query?)` — query object or string shorthand for `customId`.
- `modal(query?)` — `customId` (string shorthand) or `title`.
- `container(query?)` — Components V2 by `id`, `accentColor`, `content`, `has` (e.g. `has: { kind: 'select', query: 'reason' }`).
- generic `component(kind, query?)` — escape hatch for any kind: `content`, `section`, `media`, `separator`, `label`, `fileUpload`.
- Returned `RenderedMessage`/modal/container/section are themselves scoped — drill into the controls a specific subject owns.
- String matchers are EXACT (`{ label: 'Edit' }` matches only literal `Edit`); use a `RegExp` for partial/case-insensitive (`{ label: /edit/i }`).

## Code Examples (verified against doc; imports are external package)

Lifecycle:
```ts
import { outcome } from '@slipher/testing';

outcome(result).get.response();                                  // throws if no reply/defer/edit/followup
outcome(result).get.response({ kind: 'reply' });
outcome(result).get.response({ kind: 'deferReply' });
outcome(result).get.response({ kind: 'modal' });
outcome(result).get.response({ ephemeral: true });

outcome(result).get.denial({ kind: 'permissions', missing: 'BanMembers' });
outcome(result).get.denial({ kind: 'stop', middleware: 'blocker' });

const captured = outcome(result).get.error(/timeout/);           // needs onCommandError: 'capture'
outcome(result).get.error('already replied');
outcome(result).get.error(err => err instanceof RangeError);
```

For the simplest assertions you don't even need a reader — read the result fields directly (use `outcome` when you want the *kind* of response, not just its text):
```ts
expect(result.content).toBe('Banned spammer');
```

UI + scoping:
```ts
import { rendered } from '@slipher/testing';

rendered(result).get.message({ content: 'Banned spammer' });
rendered(result).get.embed({ title: 'Profile' });
rendered(result).get.component('button', { customId: 'confirm' });

const settings = rendered(result).get.message({ content: 'Settings' });
settings.get.component('button', 'edit');                        // Edit button on the Settings message only

const modal = rendered(result).get.modal('reject-request');
modal.get.select('reason');
modal.get.input({ customId: 'notes', required: true });
modal.get.component('fileUpload', 'evidence');

const panel = rendered(result).get.container({ content: /Settings/, has: { kind: 'select', query: 'reason' } });
panel.get.content({ text: 'Settings' });
panel.get.section({ content: /Danger/ }).get.component('button', 'delete');
```

## Recipes (worked, copy-paste-ready)

### Recipe 1 — Assert a command's REST effect (reply + the ban it issued)
The mock records every REST call. Assert both the user-facing reply AND the underlying call.
```ts
import { Routes, apiUser, createMockBot, outcome, userOption } from '@slipher/testing';
import { expect, test } from 'vitest';
import { BanCommand } from '../../src/commands/ban';

test('/ban bans the target and confirms', async () => {
  await using bot = await createMockBot({ commands: [BanCommand] });
  const target = apiUser({ id: '42', username: 'spammer' });

  const result = await bot.slash({
    name: 'ban',
    options: { user: userOption(target), reason: 'raid' },
  });

  outcome(result).get.response({ kind: 'reply' });
  expect(result.content).toBe('Banned spammer');
  const ban = await bot.waitForAction(Routes.ban);   // wait for the recorded REST call
  expect(ban.reason).toBe('raid');
});
```
Note: option keys are lowercase (`user`, `reason`) — v5 enforces this at compile time. `await using` triggers `bot.close()` (and plugin teardown) at scope end.

### Recipe 2 — Assert a structured denial (shape over copy)
A permission guard rejects before `run`. Assert the denial's *kind/missing*, not the reply text, so the test survives message rewording.
```ts
import { outcome, rendered } from '@slipher/testing';

const denied = await bot.slash({ name: 'ban', memberPermissions: [] });

outcome(denied).get.denial({ kind: 'permissions', missing: 'BanMembers' });
rendered(denied).get.message({ content: /missing/i });   // RegExp = partial/case-insensitive
```
For a middleware `stop('reason')` denial, match `{ kind: 'stop', middleware: 'name' }`; for `stop()` with no arg (a silent skip, the old `pass()`), match `{ kind: 'pass' }`.

### Recipe 3 — Capture and assert a thrown error
`get.error` only works when the dispatch was configured `onCommandError: 'capture'`. Under the default `'throw'`, the dispatch rejects — await-reject and `try/catch` instead.
```ts
import { createMockBot, outcome } from '@slipher/testing';
import { expect, test } from 'vitest';

test('/profile surfaces a timeout', async () => {
  await using bot = await createMockBot({ commands: [ProfileCommand], onCommandError: 'capture' });
  const result = await bot.slash({ name: 'profile' });
  const captured = outcome(result).get.error(/timeout/);
  expect(captured.error).toBeInstanceOf(Error);
});

// Default 'throw' mode — catch the rejection yourself:
await expect(bot.slash({ name: 'profile' })).rejects.toThrow(/timeout/);
```

### Recipe 4 — Plugins run for real
Plugins go through the `plugins` option; their real `setup()` runs inside `createMockBot()` and `teardown()` runs on `bot.close()` (or `await using`).
```ts
const bot = await createMockBot({
  commands: [PingCommand],
  plugins: [myPlugin()],
});
await bot.close();   // runs plugin teardown
```

### Recipe 5 — Bots with a custom Client (Lavalink, database, …)
Attach fakes for services the package doesn't model; commands read them via `ctx.client` by duck typing, so the fake only needs the methods the path under test touches.
```ts
const bot = await createMockBot({ commands: [PlayCommand] });
Object.assign(bot.client, {
  manager: { getPlayer: () => ({ get: () => true, set: () => {} }), useable: true },
  database: { getPrefix: async () => '!' },
});
// Audio/Lavalink playback is out of scope — stub the manager, don't emulate it.
```

### Recipe 6 — Simulate Discord REST failures
`bot.rest.fail` throws a faithful `SeyfertError` (same `code`/`metadata` a production `catch` sees), not a bespoke mock. Exercise your error handling against the real shape.
```ts
import { DiscordErrors, Routes, SeyfertError } from '@slipher/testing';

bot.rest.fail(Routes.ban, DiscordErrors.MissingPermissions);          // 403 / 50013
bot.rest.fail(Routes.createMessage, { status: 429, retryAfter: 5 });  // raw shape
bot.rest.fail(Routes.ban, DiscordErrors.MissingAccess, { times: 1 }); // fail once, then normal

// In the command's catch, SeyfertError.is narrows by code (verified: src/common/it/error.ts):
try { /* ... */ } catch (err) {
  if (SeyfertError.is(err)) { /* err.code / err.metadata available */ }
}

// For sequential or request-conditional failures, use a closure:
bot.rest.intercept(Routes.ban, (call) => { /* inspect call, throw or pass through */ });
```

## Common patterns / gotchas

- `.get.*` throws on ZERO **or** MULTIPLE matches. Use `.query.*` when absence is legitimate, `.all.*` to collect every match.
- String matchers are EXACT. Reach for `RegExp` for partial/case-insensitive. When two subjects share a `customId`, scope from the parent `RenderedMessage`/modal/container instead of querying the whole tree.
- Prefer asserting denial SHAPE (`kind`/`missing`) over user-facing copy — security tests then survive message rewording.
- `outcome(result).get.error()` REQUIRES `onCommandError: 'capture'`; under the default `'throw'` the dispatch rejects — use `await expect(...).rejects` / `try/catch`. This option lives in the toolkit, not core Seyfert config.
- `'defer'` as a `kind` matches either `deferReply` or `deferUpdate`; use the specific variant when you care which.
- Option keys in `bot.slash({ options })` must be lowercase (v5 compile-time rule) and built with the toolkit's typed helpers (`userOption`, etc.).
- Everything is external — verify signatures against the installed package version. The core touchpoints (`Routes`, `SeyfertError`, middleware `next`/`stop`) are stable in core Seyfert.

## Doc vs Source Corrections

- No corrections to the toolkit API itself (external, not in core Seyfert); examples match the upstream MDX (`seyfert-v5`).
- Adjacency confirmed: the denial `kind: 'pass'` reflects middleware that ended the chain via `stop()` with no argument (`res({ pass: true })`), since the standalone middleware `pass()` was removed in favor of `stop()` (commit 22eb832; `src/commands/applications/chat.ts:247-256`). The `'pass'` denial kind remains a valid lifecycle classification.
- Core export shape confirmed: `Routes` and `SeyfertError` are both `seyfert` root exports — no deep import needed. `SeyfertError.is` is overloaded (`is(err)` / `is(err, code)`); constructor is `new SeyfertError(code, { metadata?, cause? })`.

## Source Anchors

- `src/api/index.ts` -> `export * from './Routes'` (Routes root export)
- `src/common/it/error.ts:21` (SeyfertError: `code`, `metadata`, overloaded static `is`, `toJSON`, constructor `(code, { metadata, cause })`)
- `src/commands/applications/chat.ts:247-256` (`StopFunction`; `stop()`→`res({ pass: true })`, `stop(err)`→`deny` with `metadata: { middleware, scope }`)
- `src/index.ts` (root barrel)

## Agent Guidance

- Use `outcome` for lifecycle, `rendered` for UI. The whole point of these readers is failing loud where naive `toContain`/`toBe` on `undefined` would pass green — reach for them on any "did it actually respond?" assertion.
- Default to asserting structure: `outcome().get.denial({ kind, missing })` and `outcome().get.response({ kind })` over reply-text matching.
- `.get` = exactly one (else throw); `.query` = optional; `.all` = collect. Scope from a `RenderedMessage`/modal/container when subjects share a `customId`.
- For thrown-error assertions, set `onCommandError: 'capture'` and use `get.error()`; otherwise the dispatch rejects and you `try/catch`.
- This package is external — confirm the installed version's signatures before relying on them; the core dispatch targets (`Routes`, `SeyfertError`, middleware `next`/`stop`) are stable in core Seyfert.
