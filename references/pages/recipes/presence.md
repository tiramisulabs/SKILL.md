# Setting Custom Activity and Presence

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/presence
Coverage reference: setup-runtime.md
Verification status: Source-verified (branch more-qol)

## Page Summary

Presence (status + activities) can be set two ways: statically via the `presence` callback in `ClientOptions` (evaluated per-shard at connect time), or dynamically at runtime via `client.gateway.setPresence(...)` on a gateway `Client`. A mobile icon can be faked by overriding `gateway.properties` (os/browser/device). Presence is gateway-only; it does not apply to `HttpClient` (pure HTTP interaction servers have no gateway connection). In worker/sharded mode the static callback gains a second `workerId` argument.

## Key APIs (verified)

- `ClientOptions.presence?: (shardId: number) => GatewayPresenceUpdateData` — static per-shard presence callback (src/client/client.ts:306). Wired into `ShardManager` as `presence: this.options?.presence` (src/client/client.ts:170).
- `WorkerManagerOptions.presence?: (shardId: number, workerId: number) => GatewayPresenceUpdateData` — worker variant has a second `workerId` arg (src/websocket/discord/shared.ts:90).
- `ShardManager#setPresence(payload: GatewayUpdatePresence['d']): void` — broadcasts presence to ALL shards via `forEach` (src/websocket/discord/sharder.ts:248). Reachable as `client.gateway.setPresence(...)` (`gateway!: ShardManager`, src/client/client.ts:41).
- `ShardManager#setShardPresence(shardId: number, payload: GatewayUpdatePresence['d'])` — set presence for a single shard; returns the `send(...)` result (src/websocket/discord/sharder.ts:240).
- `GatewayUpdatePresence['d']` === `GatewayPresenceUpdateData` (src/types/gateway.ts:2137-2145).
- `GatewayPresenceUpdateData` (src/types/gateway.ts:2145): `{ since: number | null; activities: GatewayActivityUpdateData[]; status: PresenceUpdateStatus; afk: boolean }` — all four fields are required (none optional).
- `GatewayActivityUpdateData = Pick<GatewayActivity, 'name' | 'state' | 'type' | 'url'>` (src/types/gateway.ts:2171) — a sent activity object only accepts `name`, `state`, `type`, `url`. No `details`/`timestamps`/`emoji` on the sent payload.
- `ClientOptions.gateway?.properties?: IdentifyProperties` — `os` / `browser` / `device` overrides (src/websocket/discord/shared.ts:153; used in the mobile-icon recipe). In worker mode it is `DeepPartial<...>` (shared.ts:94).

Enums (root import from 'seyfert', defined in src/types/payloads/gateway.ts):
- `enum PresenceUpdateStatus` (src/types/payloads/gateway.ts:100): `Online = 'online'`, `DoNotDisturb = 'dnd'`, `Idle = 'idle'`, `Invisible = 'invisible'`, `Offline = 'offline'`.
- `enum ActivityType` (src/types/payloads/gateway.ts:251) — numeric: `Playing = 0`, `Streaming = 1`, `Listening = 2`, `Watching = 3`, `Custom = 4`, `Competing = 5`.
- `type GatewayPresenceUpdateData` is exported as a type (re-exported from 'seyfert').

All of the above import from the `'seyfert'` root barrel; no deep `'seyfert/lib/...'` import is required.

## Code Examples (verified)

### Static presence at client init (per shard)

```ts
import { Client, ActivityType, PresenceUpdateStatus } from 'seyfert';

const client = new Client({
    presence: (shardId) => ({
        status: PresenceUpdateStatus.Idle,
        activities: [{
            name: 'your servers',
            type: ActivityType.Watching,
        }],
        since: Date.now(),
        afk: false,
    }),
});
```

### Per-shard presence (vary text by shard)

The callback receives `shardId`, so each shard can advertise itself:

```ts
import { Client, ActivityType, PresenceUpdateStatus } from 'seyfert';

const client = new Client({
    presence: (shardId) => ({
        status: PresenceUpdateStatus.Online,
        activities: [{
            name: `shard ${shardId}`,
            type: ActivityType.Custom,
            state: `Serving shard ${shardId}`, // Custom shows `state`, not `name`
        }],
        since: null,
        afk: false,
    }),
});
```

### Render the bot as "mobile"

The mobile icon is driven by `gateway.properties`, NOT by omitting `status`:

```ts
import { Client, ActivityType, PresenceUpdateStatus } from 'seyfert';

const client = new Client({
    gateway: {
        properties: {
            os: 'android',
            browser: 'Discord Android',
            device: 'android',
        },
    },
    presence: (shardId) => ({
        status: PresenceUpdateStatus.Idle,
        activities: [{ name: 'your servers', type: ActivityType.Watching }],
        since: Date.now(),
        afk: false,
    }),
});
```

### Streaming activity (the `url` field)

`ActivityType.Streaming` is the only type that renders a clickable Twitch/YouTube link, via the `url` field (one of the four `Pick`ed keys):

```ts
import { ActivityType, PresenceUpdateStatus } from 'seyfert';

ctx.client.gateway.setPresence({
    status: PresenceUpdateStatus.Online,
    activities: [{
        name: 'Live coding',
        type: ActivityType.Streaming,
        url: 'https://twitch.tv/your_channel', // must be a Twitch/YouTube URL
    }],
    since: null,
    afk: false,
});
```

### Dynamically update presence from a command (gateway `Client` only)

```ts
import { Command, Declare, type GuildCommandContext } from 'seyfert';
import { ActivityType, PresenceUpdateStatus } from 'seyfert';

@Declare({ name: 'presence', description: 'Change bot presence' })
export default class PresenceCommand extends Command {
    async run(ctx: GuildCommandContext) {
        ctx.client.gateway.setPresence({
            activities: [{ name: 'with Seyfert', type: ActivityType.Playing }],
            status: PresenceUpdateStatus.Online,
            since: Date.now(),
            afk: false,
        });

        await ctx.editOrReply({ content: 'Switched Presence' });
    }
}
```

### Custom status text (the `state` field)

`ActivityType.Custom` renders `state` (the leading text after the emoji). `name` is still required by the API even though it is not shown for Custom activities:

```ts
import { ActivityType, PresenceUpdateStatus } from 'seyfert';

ctx.client.gateway.setPresence({
    status: PresenceUpdateStatus.DoNotDisturb,
    activities: [{
        name: 'custom', // required, ignored for display
        type: ActivityType.Custom,
        state: 'Maintenance in progress',
    }],
    since: null,
    afk: false,
});
```

### Single-shard update (verified in src, not in docs)

```ts
ctx.client.gateway.setShardPresence(0, {
    activities: [{ name: 'shard 0', type: ActivityType.Custom, state: 'hi' }],
    status: PresenceUpdateStatus.Online,
    since: null,
    afk: false,
});
```

### Rotating presence on an interval (`ready` event)

A common flow: cycle through several activities after the client is ready. Use the gateway client; build the payload once and broadcast on a timer.

```ts
// src/events/botReady.ts
import { createEvent, ActivityType, PresenceUpdateStatus } from 'seyfert';

const rotation = [
    { name: 'with Seyfert', type: ActivityType.Playing },
    { name: 'the gateway', type: ActivityType.Watching },
    { name: 'your commands', type: ActivityType.Listening },
] as const;

export default createEvent({
    data: { name: 'botReady', once: true },
    run(user, client) {
        let i = 0;
        setInterval(() => {
            client.gateway.setPresence({
                status: PresenceUpdateStatus.Online,
                activities: [rotation[i % rotation.length]],
                since: null,
                afk: false,
            });
            i++;
        }, 30_000); // Discord rate-limits presence updates; keep it >= ~15s
    },
});
```

### Worker / sharded mode (per-worker callback)

In worker mode the static presence callback receives a second `workerId` argument. Runtime `setPresence` still lives on the shard manager, not on the worker client.

```ts
import { WorkerManager, ActivityType, PresenceUpdateStatus, GatewayIntentBits } from 'seyfert';

const manager = new WorkerManager({
    mode: 'threads',
    path: './bot.js',
    token: process.env.TOKEN!,
    intents: GatewayIntentBits.Guilds,
    presence: (shardId, workerId) => ({
        status: PresenceUpdateStatus.Online,
        activities: [{
            name: `worker ${workerId} / shard ${shardId}`,
            type: ActivityType.Custom,
            state: `w${workerId} s${shardId}`,
        }],
        since: null,
        afk: false,
    }),
});

await manager.start();
```

## Common patterns / gotchas

- ALL FOUR fields are required by the type. You cannot omit `status`, `since`, `activities`, or `afk` inside the payload object — TypeScript will error (src/types/gateway.ts:2145). Use `since: null` when not afk/idle, `since: Date.now()` when signalling idle.
- Activity payloads are stripped to `name | state | type | url` (the `Pick`). Setting `details`, `timestamps`, `emoji`, etc. is a type error / no-op — those are receive-only fields on `GatewayActivity`, not sendable.
- `name` is always required, even for `ActivityType.Custom` where only `state` is displayed.
- `ActivityType.Streaming` (`url`) only shows the "Live" badge for valid Twitch/YouTube URLs.
- `client.gateway` is the `ShardManager` and exists ONLY on the gateway `Client`. On `HttpClient` there is no gateway, so presence cannot be set at all. Guard with `ctx.client.gateway` access only in gateway bots.
- `setPresence` broadcasts to every shard (`forEach`); `setShardPresence(id, ...)` targets one. Neither is awaited in practice — `setPresence` returns `void`, `setShardPresence` returns the raw `send(...)` result (fire-and-forget over the WS).
- Presence updates are gateway-rate-limited (a few per minute per shard). Don't spam them in a tight loop; rotate on a >=15s interval.
- The mobile icon trick depends on `gateway.properties` (os/browser/device = android). It is non-official behavior and may break; the docs' "leave status undefined" note is imprecise — `status` is a required field, so the trick is the properties override, not the missing status.

## Doc vs Source Corrections

- Docs import `GatewayPresenceUpdateData` in the mobile example but never use it. Harmless; corrected examples drop the unused import. (type exists: src/types/gateway.ts:2145)
- Docs say "Avoiding `status`" makes the mobile icon appear, yet every doc example still passes `status`. In source `GatewayPresenceUpdateData.status` is REQUIRED (src/types/gateway.ts:2161), so you cannot omit it in the callback object without a type error. The mobile trick is `gateway.properties`, not omitting status. Treat the "leave status undefined" note as imprecise.
- The doc's dynamic example augments `SeyfertRegistry { client: ParseClient<Client<true>> }` (v5 augmentation — replaces the old `UsingClient`). That is correct for v5; the standalone command example here assumes that augmentation already lives in your project.
- Otherwise the doc examples are accurate against source: `ClientOptions.presence` callback shape, `client.gateway.setPresence(...)`, and enum names all match.

## Source Anchors

- src/client/client.ts:41, 170, 306 (ClientOptions.presence, gateway: ShardManager)
- src/websocket/discord/sharder.ts:240, 248 (ShardManager setShardPresence / setPresence)
- src/websocket/discord/shared.ts:55, 90, 94, 153 (presence + properties option types, shard + worker)
- src/types/gateway.ts:2137, 2145, 2161, 2171 (GatewayUpdatePresence, GatewayPresenceUpdateData, GatewayActivityUpdateData)
- src/types/payloads/gateway.ts:100, 251 (PresenceUpdateStatus, ActivityType enums)

## Agent Guidance

- Use the `presence` callback in `ClientOptions` for the initial/static status; it receives `shardId` (and `workerId` in worker mode) so you can vary presence per shard/worker.
- For runtime changes use `client.gateway.setPresence(payload)` (all shards) or `client.gateway.setShardPresence(shardId, payload)` (one shard). `client.gateway` is the `ShardManager` and only exists on the gateway `Client` — NOT on `HttpClient`.
- The payload requires all four fields: `since` (number | null), `activities`, `status`, `afk`. Activity objects only honor `name`, `state`, `type`, `url` — anything else is stripped by the `Pick`.
- `ActivityType.Custom` uses the `state` field for displayed text (the `name` is still required by the API). `ActivityType.Streaming` uses `url`.
- Rotate presence on a timer inside a `once` `botReady`/ready event; keep the interval >=15s to respect gateway rate limits.
- Mobile-icon trick relies on faking `gateway.properties`; it is non-official and may break.
