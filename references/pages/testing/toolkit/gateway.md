# MockGateway (@slipher/testing)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/gateway

Coverage reference: testing.md

Verification status: Source-verified (core) + external package (doc-authoritative)

## Page Summary

`createMockBot()` (from the external `@slipher/testing` toolkit) installs an in-process `MockGateway` where Seyfert expects its `ShardManager`. It lets tests feed raw gateway dispatch payloads into the real bot (`bot.emit(...)`) so your actual event handlers run through the production pipeline, and it records presence updates, raw sends, and controllable shards so disconnect/reconnect infra hooks can be exercised without ever opening a WebSocket. The toolkit API is doc-authoritative and NOT part of seyfert-core; the core gateway surface it stands in for (`client.gateway` / `ShardManager`: `setPresence`, `send`, `latency`, shard lifecycle callbacks) IS verifiable in `./src`. `MockGateway` is explicitly NOT a transport/protocol emulator — it only models the surface bots read in tests, and that shape is unstable during 0.x.

## Key APIs (verified)

### External — @slipher/testing (NOT in seyfert-core; verify version in target project)
- `createMockBot({ events?, shards?, shardLatency? })` -> disposable mock bot (use with `await using`).
- `bot.emit(name, payload, opts?)` — name is the UPPERCASE Discord dispatch name (`'GUILD_MEMBER_ADD'`); resolves the REST work the handler awaited before returning. Fails loud if no registered handler ran unless `{ allowNoHandler: true }`.
- `bot.registeredEvents()` — list wired handlers (debug a green-but-silent assertion).
- `bot.world` / `bot.world.query.*` — mock world/cache inspection (e.g. `bot.world.query.dm({ userId })`).
- `bot.gateway` (alias `bot.client.gateway`) MockGateway recording surface:
  - `presences` — ordered `setPresence` payloads; `.at(-1)` is current.
  - `sent` — ordered raw sends, each `{ shardId, payload }`.
  - `values()` — controllable mock shards, each `{ id, latency }` (mock shape; `latency` = average; see corrections).
  - `simulateDisconnect(shardId, code?, reason?)`, `simulateReconnect(shardId)` — invoke the wrapped client's disconnect/reconnect callbacks.
  All of the above are MockGateway-specific and unspecified/subject to change during 0.x.

### Core seyfert APIs the mock dispatches against (verified in ./src)
- `client.gateway: ShardManager` — `src/client/client.ts:41` (`gateway!: ShardManager`).
- `ShardManager extends Map<number, Shard>` — `src/websocket/discord/sharder.ts:30`. So real `.values()` yields `Shard` instances, not `{ id, latency }`.
- `ShardManager.setPresence(payload: GatewayUpdatePresence['d'])` — `sharder.ts:248` (fans out to every shard via `setShardPresence`).
- `ShardManager.send<T extends GatewaySendPayload>(shardId, payload): Promise<boolean>` (async) — `sharder.ts:299`.
- `get ShardManager.latency` — `sharder.ts:81` (average across shards).
- Shard-lifecycle callbacks on `ShardManagerOptions`: `onShardDisconnect?(data: ShardDisconnectData)`, `onShardReconnect?(data: ShardReconnectData)`, `handlePayload(shardId, packet)`, `handleSendPayload?(shardId, payload)` — `src/websocket/discord/shared.ts:41-47`. `client.setServices({ gateway })` WRAPS each of these and also wires the `SHARD_DISCONNECT` / `SHARD_RECONNECT` events — `client.ts:63-106`.
- `createEvent({ data: { name, once? }, run })` — `src/index.ts:68`; `run` is typed `Awaitable<void>`. Event file `name` is the camelCase client name (`'guildMemberAdd'`); you DISPATCH it in tests by emitting the UPPERCASE gateway name (`'GUILD_MEMBER_ADD'`).
- Enums imported from `'seyfert'` are real root exports (NOT only discord-api-types passthroughs):
  - `PresenceUpdateStatus` — `src/types/payloads/gateway.ts:100`.
  - `ActivityType` — `src/types/payloads/gateway.ts:251`.
  - `GatewayOpcodes` — `src/types/utils/index.ts:164`.
  Re-exported via `src/types/index.ts` -> root `src/index.ts:48` (`export * from './types'`).

## Code Examples (verified)

### 1. Emitting a dispatch event into real handlers

External toolkit; root enums/types from `seyfert`. Emit the UPPERCASE Discord dispatch name with a FULL event payload:

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import guildMemberAdd from '../src/events/guildMemberAdd';

test('greets new members', async () => {
  await using bot = await createMockBot({ events: [guildMemberAdd] });

  await bot.emit('GUILD_MEMBER_ADD', rawMemberPayload);

  expect(
    bot.world.query.dm({ userId: rawMemberPayload.user.id })?.lastMessage?.content,
  ).toContain('Welcome');
});
```

### 2. The event file under test (core `createEvent`)

This is the real handler `events: [...]` registers. Note the camelCase `name`, dispatched as `'GUILD_MEMBER_ADD'`:

```ts
// src/events/guildMemberAdd.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'guildMemberAdd' },
  async run(member, client) {
    // run is Awaitable<void>; non-gateway custom events get no trailing shardId in v5
    await client.users.write(member.id, { content: `Welcome, ${member.user?.username}!` });
  },
});
```

### 3. Seed-only emit (no handler expected)

Opt out of the loud no-handler check explicitly when you only want to populate world state:

```ts
await bot.emit('CHANNEL_CREATE', rawChannelPayload, { allowNoHandler: true });
```

### 4. Presence, sends, shards, and lifecycle hooks

Enums resolve from `seyfert`; recording surfaces are mock-only:

```ts
import { ActivityType, GatewayOpcodes, PresenceUpdateStatus } from 'seyfert';
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';

test('records presence, sends, and shards', async () => {
  await using bot = await createMockBot({ shards: 3, shardLatency: 12 });

  bot.client.gateway.setPresence({
    activities: [{ name: 'testing', type: ActivityType.Playing }],
    afk: false,
    since: null,
    status: PresenceUpdateStatus.Online,
  });
  await bot.client.gateway.send(0, { op: GatewayOpcodes.Heartbeat, d: null });
  await bot.gateway.simulateDisconnect(0);

  expect(bot.gateway.presences.at(-1)).toMatchObject({ status: PresenceUpdateStatus.Online });
  expect([...bot.gateway.values()]).toHaveLength(3);
  expect(bot.gateway.sent.at(-1)).toMatchObject({ shardId: 0 });
});

await bot.gateway.simulateDisconnect(0, 1006, 'connection reset');
await bot.gateway.simulateReconnect(0);
```

## Recipes / Common patterns

### Recipe A — testing presence logic that derives a status

Code that computes a presence and pushes it via `gateway.setPresence` is testable by asserting the recorded `presences`:

```ts
import { ActivityType, PresenceUpdateStatus } from 'seyfert';
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { applyMaintenancePresence } from '../src/lib/presence';

test('maintenance mode flips presence to DND with a notice', async () => {
  await using bot = await createMockBot({ shards: 1 });

  applyMaintenancePresence(bot.client); // calls client.gateway.setPresence(...)

  expect(bot.gateway.presences.at(-1)).toMatchObject({
    status: PresenceUpdateStatus.DoNotDisturb,
    activities: [{ type: ActivityType.Custom }],
  });
});
```

### Recipe B — exercising shard-lifecycle infra (SHARD_DISCONNECT / SHARD_RECONNECT)

`setServices({ gateway })` wires `onShardDisconnect`/`onShardReconnect` to fire the `SHARD_DISCONNECT` / `SHARD_RECONNECT` client events (`client.ts:63-106`). Register those events and use the simulate hooks to drive them, no socket required:

```ts
import { createEvent } from 'seyfert';
import { createMockBot } from '@slipher/testing';
import { expect, test, vi } from 'vitest';

const alerted = vi.fn();
const shardDown = createEvent({
  data: { name: 'SHARD_DISCONNECT' },
  run(data) {
    alerted(data.shardId);
  },
});

test('alerts on shard disconnect, clears on reconnect', async () => {
  await using bot = await createMockBot({ events: [shardDown], shards: 2 });

  await bot.gateway.simulateDisconnect(1, 1006, 'connection reset');
  await bot.gateway.simulateReconnect(1);

  expect(alerted).toHaveBeenCalledWith(1);
});
```

### Recipe C — asserting outbound gateway traffic (e.g. voice/op-4 join)

Code that calls `client.gateway.send(shardId, payload)` (voice connect, request guild members, etc.) records into `sent`:

```ts
import { GatewayOpcodes } from 'seyfert';
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { joinVoice } from '../src/lib/voice';

test('joinVoice sends an op-4 VoiceStateUpdate on the guild shard', async () => {
  await using bot = await createMockBot({ shards: 4 });

  await joinVoice(bot.client, { guildId: '1', channelId: '99' });

  expect(bot.gateway.sent.at(-1)).toMatchObject({
    payload: { op: GatewayOpcodes.VoiceStateUpdate },
  });
});
```

### Gotchas

- Emit UPPERCASE Discord dispatch names (`'MESSAGE_CREATE'`), and FULL Discord event payloads — not partial patches. The mock applies canonical reducers (member/channel/message/reaction/voice/thread) to world state, so partial payloads leave the cache stale.
- `emit` throws if NO handler ran. Pass `{ allowNoHandler: true }` only for pure world-seeding emits. Use `registeredEvents()` to debug a passing-but-silent assertion (mis-cased name or missing `events: [...]`).
- Event file `name` is camelCase (`'guildMemberAdd'`); the dispatch you emit is UPPERCASE (`'GUILD_MEMBER_ADD'`). Don't conflate them.
- `await emit(...)` already resolves the REST work the handler awaited — no manual flush/tick needed before asserting.
- Always use `await using bot = await createMockBot(...)` so the bot is disposed at end of test scope.
- When asserting against a REAL (non-mock) client: `gateway.values()` yields `Shard` instances (not `{ id, latency }`), and presence/send are NOT recorded — recording is mock-only.

## Doc vs Source Corrections

- No factual errors in the page. Examples match upstream MDX (`seyfert-web@seyfert-v5`) and core source exactly.
- Enum origin (confirmed, not a bug): docs import `ActivityType` / `GatewayOpcodes` / `PresenceUpdateStatus` from `'seyfert'`. These are seyfert's OWN root exports (`src/types/payloads/gateway.ts`, `src/types/utils/index.ts`, re-exported at `src/index.ts:48`), not just a discord-api-types passthrough. Documented for agent confidence.
- `values()` shape: docs describe each mock shard as `{ id, latency }`. That is a MockGateway-only shape. The REAL `ShardManager` extends `Map<number, Shard>` (`sharder.ts:30`), so its `.values()` yields full `Shard` instances. Mock `latency` averaging is a toolkit feature, distinct from the core `ShardManager.latency` getter (`sharder.ts:81`).
- `presences` / `sent` / `simulateDisconnect` / `simulateReconnect` / `bot.emit` / `bot.world` / `registeredEvents()` are NOT part of core `ShardManager` — they exist only on the external mock. No core equivalent.
- `gateway.send` core signature is `send(shardId, payload)` returning `Promise<boolean>` (`sharder.ts:299`); matches doc usage (the mock records into `sent`).

## Source Anchors

- `src/client/client.ts` (lines 41, 63-106) — `gateway: ShardManager`; `setServices` wraps `handlePayload` / `onShardDisconnect` / `onShardReconnect` / `handleSendPayload` and fires `SHARD_DISCONNECT` / `SHARD_RECONNECT` events.
- `src/websocket/discord/sharder.ts` (lines 30, 81, 248, 299) — `ShardManager extends Map<number, Shard>`, `latency`, `setPresence`, `send`.
- `src/websocket/discord/shared.ts` (lines 41-47) — `ShardManagerOptions` callbacks (`handlePayload`, `handleSendPayload`, `onShardDisconnect`, `onShardReconnect`).
- `src/websocket/discord/shard.ts` (line 170) — `Shard.send`.
- `src/index.ts` (lines 48, 68) — `export * from './types'` (enum re-export); `createEvent(...)` signature (`run: Awaitable<void>`).
- `src/types/payloads/gateway.ts` (lines 100, 251), `src/types/utils/index.ts` (line 164) — `PresenceUpdateStatus`, `ActivityType`, `GatewayOpcodes` definitions.

## Agent Guidance

- Use this page when writing unit/integration tests that drive gateway-facing logic (event handlers, presence logic, outbound `gateway.send`, shard-lifecycle/infra code) without a live connection. It pairs with `@slipher/testing`'s `createMockBot`.
- External dependency: `@slipher/testing` is NOT shipped by seyfert-core. Confirm it is installed and check its installed version in the target project before relying on exact method names — the toolkit explicitly marks the mock gateway shape as unstable during 0.x.
- For testing INTERACTION-driven logic (commands/components/modals) instead of gateway dispatch, see the sibling testing/toolkit pages — this one is specifically the gateway/world surface.
- Enums (`ActivityType`, `GatewayOpcodes`, `PresenceUpdateStatus`) import cleanly from `'seyfert'`; no deep import needed.
- Drive shard infra via Recipe B: simulate hooks -> wrapped callbacks -> `SHARD_DISCONNECT` / `SHARD_RECONNECT` events. Register those events on the mock bot to assert your reconnect/alerting logic.
