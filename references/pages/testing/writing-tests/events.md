# Testing: Writing Tests — Events

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/events
Coverage reference: testing.md
Verification status: Source-verified (core) + external package (`@slipher/testing`, doc-authoritative)

## Page Summary

Shows how to test a Seyfert event handler in-process: register the real event with the test toolkit's `createMockBot`, feed it a gateway dispatch event via `bot.emit(NAME, payload)`, then assert on the effect (sent DM/message, recorded REST call, or world state). No WebSocket, no network. `emit` runs the handler through the production pipeline and resolves the awaited REST work before returning, so the assertion can run immediately. The testing harness (`@slipher/testing`) is an EXTERNAL package not part of core Seyfert; the event-authoring API it drives (`createEvent`, the `run` signature, gateway vs client event names, `client.users.write`) IS verifiable in `./src` and is confirmed below.

## Key APIs (verified)

Core (core Seyfert, root import from `seyfert`):
- `createEvent<E>({ data, run })` — `src/index.ts:68`. Signature: `createEvent<E extends ClientNameEvents | CustomEventsKeys>(data: { data: { name: E; once?: boolean }; run: (...args: ResolveEventParams<E>) => Awaitable<unknown> })`. It mutates `data.data.once ??= false` and returns the **same object** (no class wrapper). Pass the default export straight into `events: [...]`.
- Event name on `data.name` is the **camelCase** client event name (e.g. `'guildMemberAdd'`, `'messageCreate'`, `'ready'`), NOT the uppercase gateway name — `ClientNameEvents = Extract<keyof ClientEvents, string>` (`src/events/event.ts:14`). The uppercase form (`'GUILD_MEMBER_ADD'`) is only the raw gateway dispatch name used at the `emit` boundary.
- `run` params resolve via `ResolveEventParams` / `EventContext` (`src/events/handler.ts:29-36`, `src/events/event.ts:22-29`). For a gateway event the tuple is `[transformedPayload, client, shardId]`: `[K in keyof ClientEvents]: (...data: [Awaited<ClientEvents[K]>, UsingClient, number]) => unknown`.
- **Custom (non-gateway) events** (`commandsLoaded`, `componentsLoaded`, `uploadCommands` in `CustomEvents`, `src/events/event.ts:7-13`) resolve to `[...Parameters<CustomEvents[K]>, UsingClient]` — i.e. payload + client, **no trailing `shardId`** (v5 change; gateway events still get `shardId`).
- `run` return type is `Awaitable<unknown>` (v5: was `any`) — `ClientEvent.run` at `src/events/event.ts:32`.
- Transformed first-arg shapes (the value your `run` receives), per hook:
  - `guildMemberAdd` → `GuildMemberStructure` (`src/events/hooks/guild.ts:86`, `Transformers.GuildMember`). `member.id`, `member.user.username` valid.
  - `messageCreate` → `MessageStructure` (`src/events/hooks/message.ts:16`, `Transformers.Message`). `message.reply(body)` / `message.write(body)` valid (`src/structures/Message.ts:164`).
  - `ready` → `ClientUserStructure` (`src/events/hooks/dispatch.ts:10`, `Transformers.ClientUser`).
  - `guildBanAdd` → `{ ...camelCased, user: UserStructure }` (`src/events/hooks/guild.ts:41`).
- `client.users.write(userId, body)` — `src/common/shorters/users.ts:46`. Internally `createDM` then `messages.write`: `(await this.client.users.createDM(userId)).messages.write(body)`. Returns `Promise<MessageStructure>`; `body` is `MessageCreateBodyRequest`.

External (`@slipher/testing`, doc-authoritative — verify version in target project):
- `createMockBot({ events: [...] })` → returns a disposable bot (`await using`).
- `createMockBot({ loadFromConfig: true })` — boots through the same loaders `client.start()` uses, registering every event from `seyfert.config`.
- `bot.emit(name, payload, options?)` — `name` is the UPPERCASE gateway dispatch name; runs the handler through the production pipeline, resolves awaited REST before returning; fails loud if no registered handler ran.
- `bot.emit(name, payload, { allowNoHandler: true })` — opt out of the loud no-handler check when seeding world state only.
- `bot.world.query.dm({ userId })?.lastMessage?.content` — read DM world state.
- `registeredEvents()` — list wired event names for debugging a no-handler failure.

## Code Examples (verified)

### 1. Greeter event under test (core API, confirmed against src)

```ts
// src/events/guildMemberAdd.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'guildMemberAdd' }, // camelCase client name — verified ClientNameEvents
  async run(member, client) {
    await client.users.write(member.id, {
      content: `Welcome, ${member.user.username}!`,
    });
  },
});
```

### 2. Register + assert (toolkit external)

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import guildMemberAdd from '../src/events/guildMemberAdd';

test('greets new members', async () => {
  await using bot = await createMockBot({ events: [guildMemberAdd] });

  // UPPERCASE gateway name at the emit boundary, full Discord shape
  await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload);

  expect(
    bot.world.query.dm({ userId: rawMemberPayload.user.id })?.lastMessage?.content,
  ).toContain('Welcome');
});
```

### 3. Seed-only emit (no handler expected)

```ts
// World-state seeding: opt out of the loud no-handler check.
await bot.emit('CHANNEL_CREATE', rawChannelPayload, { allowNoHandler: true });
```

### 4. messageCreate handler that replies, then assert the reply

Replying in a `messageCreate` is the most common bot flow. The transformed payload is a real `MessageStructure`, so `message.reply(...)` works (`src/structures/Message.ts:164`).

```ts
// src/events/messageCreate.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'messageCreate' },
  async run(message, client) {
    if (message.author.bot) return;            // guard: ignore other bots
    if (message.content === '!ping') {
      await message.reply({ content: 'Pong!' });
    }
  },
});
```

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import messageCreate from '../src/events/messageCreate';

test('replies to !ping', async () => {
  await using bot = await createMockBot({ events: [messageCreate] });

  await bot.emit('MESSAGE_CREATE', rawPingMessagePayload); // content: '!ping'

  // Read the reply back from the channel's world state.
  const channel = bot.world.query.channel({ channelId: rawPingMessagePayload.channel_id });
  expect(channel?.lastMessage?.content).toBe('Pong!');
});
```

### 5. A `once` event (ready) — runs a single time

`once: true` registers a one-shot handler. v5 note: if a `once` handler throws it is **reset** so it can fire again (changelog → Events). `ready` hands you a `ClientUserStructure`.

```ts
// src/events/ready.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'ready', once: true },
  run(user, client, shardId) {
    client.logger.info(`Logged in as ${user.username} on shard #${shardId}`);
  },
});
```

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test, vi } from 'vitest';
import ready from '../src/events/ready';

test('ready logs once', async () => {
  await using bot = await createMockBot({ events: [ready] });
  const spy = vi.fn();
  // (Pattern depends on the toolkit's logger hook; verify in your project.)

  await bot.emit('READY', rawReadyPayload);
  // assert your side effect (logger call / world flag) here
});
```

### 6. Custom (non-gateway) event — NO trailing `shardId` (v5)

Custom events declared in `CustomEvents` (`commandsLoaded`, `componentsLoaded`, `uploadCommands`) receive `[payload, client]` — the trailing `shardId` that gateway events carry is gone in v5.

```ts
// src/events/commandsLoaded.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'commandsLoaded' },
  run(metadata, client) {
    // metadata: PluginLoadedMetadata<'commands', ...> — note: no shardId param
    client.logger.info('Commands loaded');
  },
});
```

### 7. Using the real config (`loadFromConfig`)

Boots through the same loaders `client.start()` uses, so every event in `seyfert.config` is wired — closest to production.

```ts
test('greets via production config', async () => {
  await using bot = await createMockBot({ loadFromConfig: true });
  await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload);
  expect(
    bot.world.query.dm({ userId: rawMemberPayload.user.id })?.lastMessage?.content,
  ).toContain('Welcome');
});
```

## Common patterns / gotchas

- **Casing is the #1 trap.** `createEvent({ data: { name: ... } })` wants the **camelCase** `ClientNameEvents` key (`'guildMemberAdd'`). `bot.emit(...)` wants the **uppercase** gateway name (`'GUILD_MEMBER_ADD'`). Mixing them up silently fires nothing; `emit` then fails loud (no-handler). Call `registeredEvents()` to see what is wired.
- **Emit full Discord shapes, not partials.** The mock applies a canonical set of events (member, channel, message, reaction, voice, thread) to world state; a partial payload can leave the cache stale and assertions misleading.
- **`emit` awaits REST before returning** — no need to `await` a tick or poll; read the effect right after the `emit` call.
- **`await using bot`** relies on `Symbol.asyncDispose` (TS 5.2+ / Node 20+). One bot per test keeps world state isolated.
- **`shardId` presence differs by event kind.** Gateway events: `run(payload, client, shardId)`. Custom events (`CustomEvents`): `run(payload, client)` only — adding a third param is a type error in v5.
- **`createEvent` returns the plain input object** with `once` defaulted to `false`. There is no class instance — the default export is exactly what `events: [...]` expects.
- **Throwing in a `once` handler resets it** (v5) so it can fire on the next dispatch — don't assume a one-shot is permanently consumed after a failed run.
- **`run` returns `Awaitable<unknown>`** (v5: was `any`, then `void`, now widened to `unknown`). Returning a value — promise or not — is allowed and typed as `unknown`.
- **DMs are recorded, not sent.** `client.users.write` does a real `createDM` + `messages.write` round-trip in production; the toolkit records it so you read it back via `bot.world.query.dm({ userId })`.

## Doc vs Source Corrections

- No v4/stale APIs in the upstream MDX — it already uses camelCase `createEvent` names and uppercase `emit` names. The dual-casing is correct and matches src: `createEvent` requires the camelCase `ClientNameEvents` key (`src/events/event.ts:14`); `emit` takes the raw gateway dispatch name.
- Minor doc wording: the MDX calls event handlers "real classes" ("Like commands, event handlers are real classes"). Per src, `createEvent` returns the **plain input object** (`src/index.ts:72-73`), not a class instance — functionally identical for `events: [...]`, but there is no class wrapper.
- `client.users.write` confirmed to internally `createDM` first, then `messages.write` — the page's "DMs each new member" description is accurate (`src/common/shorters/users.ts:46-47`).
- v5 deltas relevant to this page (changelog → Events, confirmed in src): `run` is `Awaitable<unknown>`; custom (non-gateway) handlers dropped the trailing `shardId`; throwing in a `once` event resets it; `VOICE_CHANNEL_STATUS_UPDATE` now hands a resolved `VoiceChannel | undefined` instead of a raw cache promise.
- All `@slipher/testing` surface (`createMockBot`, `emit`, `world.query.dm`, `world.query.channel`, `allowNoHandler`, `registeredEvents`, `loadFromConfig`) is NOT in core Seyfert — cannot be source-verified here; treat as doc-authoritative and verify the installed `@slipher/testing` version in the target project.