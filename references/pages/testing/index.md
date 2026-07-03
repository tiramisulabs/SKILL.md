# Testing — Introduction (@slipher/testing)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/index
Coverage reference: testing.md
Verification status: External (doc-authoritative, not in core Seyfert) — core dispatch + builder/Vitest APIs verified against ./src

## Page Summary

`@slipher/testing` is a runner-agnostic testing suite for Seyfert v5 bots and Slipher plugins. It runs your bot (or a slice of it) entirely in-process — no token, no gateway, no network, never touching Discord — and bundles no test runner of its own (bring Vitest, Jest, `node:test`, etc.). The package is NOT part of core Seyfert; it is an external peer-dependency package. Verify its exact version and API in the target project before relying on signatures.

Most Discord bots are hard to unit-test because running one means opening a gateway connection — commands, components, and events have nowhere to run otherwise. This package solves that by making the bot testable in-process. The API is "actively evolving" per the docs, so always version-check.

## Two layers (per docs)

1. **Mock bot** (the core): drives a real Seyfert client pipeline in-process using your actual commands, middlewares, and plugins. It dispatches raw payloads through the same `HandleCommand` → resolver → middleware flow production uses, and captures outgoing REST calls for assertions instead of sending them. Use for testing real behavior: option parsing, middlewares, permissions, components, REST, events.
2. **Fixtures** (lightweight): plain mock objects with no assertions/spies/fake-timers bundled (so they stay runner-agnostic). Core is `mockCommandContext()` — a stand-in for a command context with the common fields plus Slipher stubs (`logger`, `queues`, `scheduler`) that record what your command does. Entity factories: `mockUser`, `mockGuild`, … with deterministic, overridable ids. Use for testing a single `run()` body in isolation.

Rule of thumb (docs): real behavior → mock bot; a single `run()` body in isolation → fixtures. Both come from the same import and coexist in one suite.

Where to go next (doc subpages): **Writing Tests** (`/docs/testing/writing-tests/setup`) for a hands-on walkthrough, and the **Toolkit** reference — Mock bot, Dispatching, World & State, Assertions, MockGateway, Fixtures, Defaults & Scope.

## Key APIs (verified — CORE only; the testing toolkit itself is external)

The testing package dispatches against these real core Seyfert APIs (the toolkit symbols `mockCommandContext`, `mockBot`, `mockUser`, `mockGuild`, `MockGateway`, etc. live in `@slipher/testing` and are NOT verifiable here):

- `HandleCommand` class — `src/commands/handle.ts:64`. The production command/interaction dispatcher. Verified async methods: `interaction(body, shardId, __reply?)` (`:274`), `chatInput(...)` (`:204`), `message(rawMessage, shardId)` (`:364`), `messageComponent(interaction)` (`:266`), `modal(interaction)` (`:259`), `autocomplete(...)` (`:67`), `contextMenu(...)` (`:112`), `entryPoint(command, interaction, context)` (`:168`); plus resolver helpers `makeResolver(...)` (`:599`), `argsParser(content, command, message)` (`:492`), `resolveCommandFromContent(...)` (`:500`). Constructed with `new HandleCommand(client)`.
  - NOTE: `HandleCommand` is NOT re-exported from the `seyfert` root barrel. `src/commands/index.ts` only re-exports `type CommandFromContent` from `./handle`. To reference the class directly, deep-import `seyfert/lib/commands/handle` (the testing package handles this internally).
- `OptionResolver` / `OptionResolverStructure` — used by `HandleCommand.makeResolver(...)` (`src/commands/optionresolver.ts`, re-exported via `src/commands/index.ts`). This is the "resolver" stage the docs reference.
- Middleware flow — commands run through registered middlewares; v5 replaced middleware `pass()` with `stop()`. The callback arg is `{ context, next, stop }`. Tests asserting on middleware control flow must use `stop()` (silent skip) / `stop('reason')` (deny), NOT `pass()`.
- `client.options.contextScopes` — `HandleCommand` wraps execution in `runContextScopes(...)` for chatInput/contextMenu/entryPoint/message. Relevant when the toolkit asserts scoped behavior.

## Code Examples (verified)

### Install (external package; exact command from docs)

```sh
pnpm add -D @slipher/testing
```

Requires Seyfert v5 as a peer dependency. This index page itself contains no TypeScript usage examples — concrete `mockBot` / `mockCommandContext` snippets live on the toolkit subpages. Treat any code there as external/doc-authoritative and verify exports against the installed version.

### EXTERNAL (conceptual — verify against installed `@slipher/testing`)

The two layers, sketched the way the docs describe them. Symbols are external and may change; the shapes shown match the doc's stated intent, not a verified signature.

```ts
// fixtures layer — isolate a single run() body
import { describe, expect, test } from 'vitest';
import { mockCommandContext } from '@slipher/testing'; // external symbol — version-verify
import PingCommand from '../src/commands/ping';

test('ping replies with pong', async () => {
  const ctx = mockCommandContext(); // records writes/edits; stubs logger/queues/scheduler
  await new PingCommand().run(ctx as any);
  // assert on what the command did to the recorded context (shape is package-defined)
  expect(ctx /* e.g. ctx.replies / ctx.write calls */).toBeDefined();
});
```

```ts
// mock-bot layer — drive the real pipeline, assert on captured REST
import { mockBot } from '@slipher/testing'; // external symbol — version-verify
import { client } from '../src/index'; // your real configured Client

const bot = await mockBot(client); // boots HandleCommand pipeline in-process, no gateway/token
await bot.dispatchSlash?.('ping');  // dispatches a raw interaction through HandleCommand
// assert against captured outgoing REST instead of a real network send
```

### CORE (verified against ./src) — unit-testing without the toolkit

For testing core Seyfert itself, or for unit tests that need no `@slipher/testing`, you assert directly on verified core APIs. v5 builders validate on `toJSON()` — perfect for fast, dependency-free unit tests. This mirrors `tests/builder-validation.test.mts`.

```ts
import { describe, expect, test } from 'vitest';
import { PollBuilder, Modal, SeyfertError } from 'seyfert';

describe('builder validation (v5)', () => {
  test('PollBuilder.toJSON() throws when the question is missing', () => {
    const poll = new PollBuilder().setAnswers({ text: 'Yes' }, { text: 'No' });
    expect(() => poll.toJSON()).toThrow(SeyfertError); // code: MISSING_POLL_QUESTION
  });

  test('a complete poll serializes fine', () => {
    const poll = new PollBuilder()
      .setQuestion({ text: 'Deploy?' })
      .setAnswers({ text: 'Yes' }, { text: 'No' });
    expect(poll.toJSON()).toMatchObject({ question: { text: 'Deploy?' } });
  });

  test('Modal.toJSON() throws without a title', () => {
    expect(() => new Modal().setCustomId('m').toJSON()).toThrow(SeyfertError);
  });
});
```

Narrow a caught error by code with the v5 `SeyfertError.is(...)` static guard:

```ts
import { SeyfertError } from 'seyfert';

try {
  somePrefixCtx.modal(myModal); // throws CANNOT_USE_MODAL outside an interaction
} catch (err) {
  if (SeyfertError.is(err, 'CANNOT_USE_MODAL')) {
    // typed-narrowed branch
  }
}
```

### CORE (verified) — Vitest config used by this repo

core Seyfert's own suite (`tests/*.test.mts`) runs on plain Vitest with serial execution. Replicate when in-process global state (a shared `Client`) must not be clobbered by parallel files:

```ts
// tests/vitest.config.mts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    fileParallelism: false, // run files serially
    isolate: false,         // share module state across the run
  },
});
```

## Recipes / Common patterns & gotchas

- **Pick the right layer.** Single `run()` body → fixtures (`mockCommandContext`). Whole-pipeline behavior (option parsing, middleware order, permissions, components) → mock bot. Don't reimplement dispatch by hand if the toolkit covers it.
- **Middleware control flow is `stop()`, not `pass()` (v5).** Any test or example carried over from v4 docs that calls `pass()` is stale. `stop()` / `stop(null)` = silent skip; `stop('reason')` = deny → routes to `onMiddlewaresError`. A middleware that throws/rejects is an internal error → routes to `onInternalError` (not a denial).
- **`onMiddlewaresError(ctx, error, metadata)`**: in v5, `metadata.middleware` names the middleware that actually called `stop()` / rejected — assert on that exact name, not the index-current one.
- **Builders fail at `toJSON()`, not at the API.** Assert with `expect(() => builder.toJSON()).toThrow(SeyfertError)`. Validators: `PollBuilder` (needs a question + answers), `Modal` (needs title), `MediaGalleryItem`, `StringSelectOption`, `RadioGroupOption`. `resolveColor('#zzz')` throws on malformed hex.
- **`HandleCommand` is not a root export.** Deep-import `seyfert/lib/commands/handle` if wiring a custom in-process harness. The barrel only exposes `type CommandFromContent` from that module.
- **No real token/gateway required.** The mock-bot layer captures outgoing REST for assertions instead of sending it — that's how you assert side effects without a network. Frame "what did the bot try to send?" as "what REST calls were captured?".
- **Context return types changed (v5).** `write(...)` / `editOrReply(...)` return `void` unless the response flag (`withResponse` / `fetchReply`) is `true`. If a test relied on a returned `Message` from the default call, pass the flag explicitly.
- **Collectors use camelCase client event names (v5).** `client.collectors.run('messageCreate', filter)` — not `'MESSAGE_CREATE'`. Also every matching collector now runs (the first-match `break` is gone); tighten filters in tests that assumed first-match-wins. Type is `CollectorRunParameters` (old `CollectorRunPameters` typo removed).
- **Custom (non-gateway) event handlers no longer receive a trailing `shardId`** — `createEvent({ data:{name}, run(payload, client) {} })`. Update event-test signatures accordingly.
- **Contributing to core Seyfert uses plain Vitest, not `@slipher/testing`.** This repo has no `@slipher/testing` dependency; tests live in `tests/*.test.mts` (config `tests/vitest.config.mts`). Don't suggest the toolkit for core contributions.

## Doc vs Source Corrections

- The docs describe the dispatch path as `HandleCommand` → resolver → middleware. Confirmed against `src/commands/handle.ts` (the real production dispatcher), with the caveat that `HandleCommand` is reachable from core only via deep import `seyfert/lib/commands/handle`, not the `seyfert` root barrel (`src/commands/index.ts` re-exports only `type CommandFromContent`).
- Draft note's source URL omitted `/index`; header uses the canonical `/docs/testing/index`.
- Verification status clarified to "External (doc-authoritative) — core dispatch + builder/Vitest verified".
- Middleware control flow updated to v5 `stop()` (the `pass()` API was removed). Any toolkit example using `pass()` is stale.
- The page is conceptual/overview and introduces no API surface that contradicts source.

## Source Anchors

- `src/commands/handle.ts` (HandleCommand class `:64`; methods `interaction:274`, `chatInput:204`, `message:364`, `modal:259`, `messageComponent:266`, `autocomplete:67`, `contextMenu:112`, `entryPoint:168`, `makeResolver:599`, `argsParser:492`, `resolveCommandFromContent:500`; runContextScopes usage)
- `src/commands/index.ts` (barrel — only `type CommandFromContent` re-exported from handle; `optionresolver` re-exported)
- `src/commands/optionresolver.ts` (OptionResolver / OptionResolverStructure)
- `src/index.ts:21` (root barrel; `export * from './commands'`)
- `tests/` (repo's own Vitest suite — separate from `@slipher/testing`)
- `tests/builder-validation.test.mts` (model for builder `toJSON()` validation assertions)
- `tests/vitest.config.mts` (`fileParallelism:false`, `isolate:false`)

## Agent Guidance

- This is an OVERVIEW page for an EXTERNAL package. Do not invent `@slipher/testing` exports — when a user asks for concrete test code, consult the installed package's types or the toolkit subpages, and always add "verify version in target project". The EXTERNAL examples above are conceptual sketches matching doc intent, not verified signatures.
- For testing core Seyfert itself (this repo), there is NO `@slipher/testing` dependency; use plain Vitest. The CORE examples above are verified and copy-paste-ready.
- Gotcha (v5): middleware control flow now uses `stop()` instead of `pass()`. Carried-over `pass()` is stale.
- Gotcha: `HandleCommand` is not on the `seyfert` root export — deep-import `seyfert/lib/commands/handle` for a custom harness.
- The mock-bot layer asserts on captured outgoing REST rather than real sends — the framing for "assertions without a gateway/token".
