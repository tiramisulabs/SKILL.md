# Understanding Sharding

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/sharding
Coverage reference: setup-runtime.md
Verification status: Source-verified (seyfert-core, branch more-qol)

## Page Summary

Seyfert shards internally by default — a normal `Client` owns a `ShardManager` at `client.gateway`. You only reach for the manual worker API to run shards across OS threads (`node:worker_threads`) or processes (`node:cluster`). The `WorkerManager` (main process) spawns workers; each worker file runs a `WorkerClient`. Centralized cache across workers uses `WorkerAdapter` on each worker, while the `WorkerManager` holds the real adapter (`manager.setCache(...)`). Shard inspection (latency, per-shard ping, shard-id math) lives on `client.gateway` (a `ShardManager`) for a normal `Client`, and on `client.shards` / `client.latency` for a `WorkerClient`.

## Key APIs (verified)

- `WorkerManager` — root import from `'seyfert'` (src/index.ts:50). Constructor takes `WorkerManagerOptions`; `manager.start()` boots it. Extends `Map<number, ...>`. Has `manager.setCache(adapter)` and `manager.setRest(apiHandler)` (src/websocket/discord/workermanager.ts:98-104).
- `WorkerManagerOptions` is a discriminated union over `mode` (src/websocket/discord/shared.ts:99-114):
  - `mode: 'threads'` (default) — `path` required; spawns workers as `node:worker_threads`.
  - `mode: 'clusters'` — `path` required; spawns workers as `node:cluster` processes. NOTE: the literal is `'clusters'` (plural), not `'cluster'`.
  - `mode: 'custom'` — `adapter: CustomManagerAdapter` required; `path` optional.
  - Shared options (`WorkerManagerOptionsBase`, src/websocket/discord/shared.ts:75-97): `workers?`, `shardsPerWorker?` (default 16), `workerProxy?`, `heartbeaterInterval?` (default 15000), `presence?: (shardId, workerId) => GatewayPresenceUpdateData`, `handlePayload?`, `handleWorkerMessage?`, `getRC?`, plus everything from `ShardManagerOptions` except `handlePayload`/`presence`/`properties` — i.e. `totalShards?`, `shardStart?`, `shardEnd?`, `spawnShardDelay?` (default 5300), `resharding?`, `info`, `token`, `intents`.
  - Defaults: `mode:'threads'`, `shardsPerWorker:16` (src/websocket/discord/workermanager.ts:84-85, constants).
- `CustomManagerAdapter` (src/websocket/discord/shared.ts:70-73) — `{ postMessage(workerId, body): Awaitable<unknown>; spawn(workerData, env): Awaitable<unknown> }`. Structural; not a root export (build the literal inline).
- `WorkerClient<Ready extends boolean = boolean>` — root import from `'seyfert'`. `await client.start()` (src/client/workerclient.ts:157, async). Key members:
  - `client.workerId`, `client.workerData` (`WorkerData`), `client.latency` (average of `client.shards`), `client.shards: Map<number, Shard>`, `client.calculateShardId(guildId)` (workerclient.ts:436).
  - `client.tellWorker(workerId, (worker, vars) => ..., vars) -> Promise<R>` (workerclient.ts:456). First callback arg is the target `WorkerClient`.
  - `client.tellWorkers((worker, vars) => ..., vars) -> Promise<R[]>` — broadcasts to every worker (workerclient.ts:469).
  - `client.resumeShard(shardId, shardData)` (workerclient.ts:510).
  - `client.setServices(servicesOptions)` — wires `WorkerAdapter.postMessage` only when `options.postMessage` is set; otherwise the adapter talks to the manager via `parentPort`/`process.send` on its own (workerclient.ts:142-147).
- `WorkerClientOptions` (workerclient.ts:623): `handlePayload?`, `sendPayloadToParent?` (default false), `postMessage?`, `gateway?`, `commands?`, `handleManagerMessages?`, `logger?` (via `BaseClientOptions`). `onShardDisconnect`/`onShardReconnect` are DEPRECATED — listen to the `SHARD_DISCONNECT`/`SHARD_RECONNECT` events instead (they can double-fire).
- `WorkerAdapter` — root import from `'seyfert'`. `new WorkerAdapter(client.workerData)` (src/cache/adapters/workeradapter.ts:9). Async (`isAsync = true`); proxies every cache method to the manager process; 60s timeout rejects with `SeyfertError('CACHE_TIMEOUT')` (workeradapter.ts:42).
- `ShardManager` — root import from `'seyfert'` (src/index.ts:50); this is what a normal `Client.gateway` is. Extends `Map<number, Shard>`. Getters/methods (src/websocket/discord/sharder.ts): `latency`, `totalShards`, `calculateShardId(guildId)`, `setPresence(payload)`, `setShardPresence(shardId, payload)`, `joinVoice(...)`, `leaveVoice(guildId)`, `requestChannelInfo(guildId, fields)`, `forceIdentify(shardId)`, `disconnect(shardId, code?)`, `disconnectAll(code?)`, `resume(shardId, shardData)`.
- `Shard` (the Map values, src/websocket/discord/shard.ts): `shard.ping()` (line 124), `shard.latency`, `shard.isOpen`, `shard.resumable`, `shard.id`.
- Custom events: `SHARD_DISCONNECT` payload is `{ shardId, code, reason }`, `SHARD_RECONNECT` payload is `{ shardId }` (src/websocket/discord/shared.ts:13-21; hooks src/events/hooks/custom.ts). Also `WORKER_READY`, `WORKER_SHARDS_CONNECTED`, `GUILDS_READY`.
- `ParseClient<T>` + `SeyfertRegistry` (src/commands/applications/shared.ts; src/client/plugins/types.ts:32) — register the worker client type via module augmentation.
- Exported types from `'seyfert'`: `WorkerData`, `WorkerManagerOptions`, `ShardManagerOptions`, `ShardData` (src/index.ts:51), `WorkerInfo`, `WorkerShardInfo` (src/index.ts:52).

## Code Examples (verified)

### manager.ts (main process)
```ts
import { WorkerManager } from 'seyfert';

const manager = new WorkerManager({
	mode: 'threads', // default; 'clusters' (plural!) for separate processes, 'custom' for your own adapter
	// Node: point at the COMPILED entry. Bun/Deno: point at the TS source, e.g. './src/client.ts'
	path: './dist/client.js',
	// optional: workers, shardsPerWorker (16), totalShards, shardStart, shardEnd, presence, resharding...
});

manager.start();
```

### client.ts (each worker)
```ts
import { type ParseClient, WorkerClient } from 'seyfert';

const client = new WorkerClient();

await client.start();        // async — await it
await client.uploadCommands(); // optional: register slash commands once per process

declare module 'seyfert' {
	interface SeyfertRegistry {
		client: ParseClient<WorkerClient>;
	}
}
```

### Centralized cache (the full picture)
Each worker uses `WorkerAdapter`; the manager holds the real adapter via `setCache`. The `WorkerAdapter` reaches the manager through `parentPort`/`process.send` automatically — no `postMessage` option needed for the default transport.

```ts
// manager.ts — the manager owns the real cache backend (e.g. Redis, or default MemoryAdapter)
import { WorkerManager } from 'seyfert';
import { RedisAdapter } from '@slipher/redis-adapter'; // external — verify version in your project

const manager = new WorkerManager({ mode: 'threads', path: './dist/client.js' });
manager.setCache(new RedisAdapter({ redisOptions: { host: 'localhost', port: 6379 } }));
manager.start();
```

```ts
// client.ts — each worker proxies cache calls to the manager
import { WorkerClient, WorkerAdapter } from 'seyfert';

const client = new WorkerClient();

client.setServices({
	cache: {
		adapter: new WorkerAdapter(client.workerData),
	},
});

await client.start();
```

### Inspecting shards on a normal `Client` (`client.gateway` is a `ShardManager`)
```ts
client.gateway.latency;                       // average gateway latency (ms)
await client.gateway.get(shardId)?.ping();    // per-shard ping (Shard.ping)
client.gateway.totalShards;                   // total shard count
client.gateway.calculateShardId('guildId');   // which shard serves a guild

// Set a presence on every shard, or one shard
client.gateway.setPresence({ status: 'online', activities: [], afk: false, since: null });
client.gateway.setShardPresence(0, { status: 'idle', activities: [], afk: false, since: null });
```

### Inspecting shards inside a `WorkerClient` (NO `.gateway` here)
```ts
client.latency;                            // average latency of THIS worker's shards
await client.shards.get(shardId)?.ping();  // per-shard ping
client.calculateShardId('guildId');
client.workerId;                           // which worker this is
[...client.shards.keys()];                 // shard ids handled by this worker
```

### Talking to another worker (and broadcasting)
`tellWorker` serializes the function with `.toString()` and runs it in the target worker — closures DO NOT transfer; pass everything via `vars` (it is `JSON.stringify`-ed).

```ts
import { WorkerClient } from 'seyfert';
const client = new WorkerClient();

// Ask worker #1 to run something; the first arg is THAT worker's WorkerClient
const reply = await client.tellWorker(
	1,
	(worker, vars) => `Hi worker #${worker.workerId}, asked by #${vars.from}`,
	{ from: client.workerId },
);

// Broadcast to every worker; resolves to R[]
const guildCounts = await client.tellWorkers(
	worker => worker.cache.guilds?.count() ?? 0,
	{},
);
const total = (await Promise.all(guildCounts)).flat();
```

## Recipes / Common patterns

### Shard lifecycle events (replaces deprecated `onShardDisconnect`/`onShardReconnect`)
The `onShardDisconnect`/`onShardReconnect` constructor callbacks are deprecated. Use event files instead — they fire on both `Client` and `WorkerClient`.

```ts
// src/events/shardDisconnect.ts
import { createEvent } from 'seyfert';

export default createEvent({
	data: { name: 'SHARD_DISCONNECT' },
	run(data, client) {
		// data: { shardId: number; code: number; reason: string }
		client.logger.warn(`Shard #${data.shardId} disconnected (${data.code}): ${data.reason}`);
	},
});
```

```ts
// src/events/shardReconnect.ts
import { createEvent } from 'seyfert';

export default createEvent({
	data: { name: 'SHARD_RECONNECT' },
	run(data, client) {
		// data: { shardId: number }
		client.logger.info(`Shard #${data.shardId} reconnected`);
	},
});
```

### Per-shard presence at spawn (rotating status)
Pass a `presence` factory to the manager; it is called per shard/worker as shards spawn.

```ts
import { WorkerManager, ActivityType, PresenceUpdateStatus } from 'seyfert';

const manager = new WorkerManager({
	mode: 'threads',
	path: './dist/client.js',
	presence: (shardId, workerId) => ({
		status: PresenceUpdateStatus.Online,
		afk: false,
		since: null,
		activities: [{ name: `shard ${shardId} • worker ${workerId}`, type: ActivityType.Playing }],
	}),
});

manager.start();
```

### Resharding (grow shard count without downtime)
```ts
import { WorkerManager } from 'seyfert';

const manager = new WorkerManager({
	mode: 'threads',
	path: './dist/client.js',
	resharding: {
		// fetch fresh gateway info (recommended shard count) on the interval
		getInfo: () => manager.rest.proxy.gateway.bot.get(),
		interval: 8 * 60 * 60 * 1000, // every 8h
		percentage: 80,               // reshard when guilds fill 80% of capacity
	},
});

manager.start();
```

### Custom spawn mode (your own transport / process model)
```ts
import { WorkerManager } from 'seyfert';

const manager = new WorkerManager({
	mode: 'custom',
	// path optional in custom mode
	adapter: {
		// route a message from the manager to a specific worker
		postMessage(workerId, body) {
			myTransport.sendTo(workerId, body);
		},
		// spawn a worker however you like; pass workerData + env through
		spawn(workerData, env) {
			myOrchestrator.launch(workerData, env);
		},
	},
});

manager.start();
```

### Forwarding gateway packets to the manager
Workers do NOT relay every packet to the manager by default (a perf win on big bots). Opt in with `sendPayloadToParent` when the manager must observe raw dispatches.

```ts
import { WorkerClient } from 'seyfert';

const client = new WorkerClient({ sendPayloadToParent: true });
await client.start();
```

## Doc vs Source Corrections

- Docs prose/example say `mode: 'cluster'` (singular) for processes -> src is `mode: 'clusters'` (plural) (src/websocket/discord/shared.ts:106). Valid modes: `'threads'` (default), `'clusters'`, `'custom'`.
- Docs use `Worker`/`WorkerManager` interchangeably -> the only exported class is `WorkerManager` (src/index.ts:50); there is no `Worker` class for this.
- Docs put the `client.gateway.*` "Accessing shards at runtime" snippet on the WorkerClient page -> `gateway` is a `ShardManager` that only exists on a normal `Client` (src/client/client.ts:41). A `WorkerClient` has NO `.gateway`; use `client.shards` (Map) + `client.latency` + `client.calculateShardId` (workerclient.ts:91,115,436).
- Docs `client.start()` (manager and client) is not awaited -> `WorkerClient.start()` is async; prefer `await client.start()`.
- Docs centralized-cache example sets only the worker `WorkerAdapter` -> for an actual shared backend you must ALSO call `manager.setCache(...)` on the manager (defaults to `MemoryAdapter`); the `WorkerAdapter` reaches it via `parentPort`/`process.send` automatically.
- The `WorkerClientOptions.postMessage` option is NOT required for centralized cache — it only overrides the adapter's transport (workerclient.ts:144).

## Source Anchors

- src/index.ts:50-52 (exports of `ShardManager`, `WorkerManager`, and types `ShardData`/`ShardManagerOptions`/`WorkerData`/`WorkerManagerOptions`/`WorkerInfo`/`WorkerShardInfo`)
- src/websocket/discord/shared.ts (`WorkerManagerOptions` union :99-114, `WorkerManagerOptionsBase` :75-97, `CustomManagerAdapter` :70-73, `WorkerData` :181-196, `ShardData` :116-131, `ShardManagerOptions` :23+, `ShardDisconnectData`/`ShardReconnectData` :13-21, `ShardSocketCloseCodes` :172-179)
- src/websocket/discord/workermanager.ts (`WorkerManager`, `setCache`/`setRest` :98-104, constructor defaults :81-96, `start`)
- src/client/workerclient.ts (`WorkerClient`, `setServices` :142, `latency` :115, `calculateShardId` :436, `tellWorker` :456, `tellWorkers` :469, `resumeShard` :510, `WorkerClientOptions` :623, deprecated `onShardDisconnect`/`onShardReconnect` :627-633)
- src/cache/adapters/workeradapter.ts (`WorkerAdapter`, `CACHE_TIMEOUT` :42)
- src/websocket/discord/sharder.ts + shard.ts (`ShardManager` methods, `Shard.ping`/`latency`/`isOpen`/`resumable`)
- src/events/hooks/custom.ts (`SHARD_DISCONNECT`, `SHARD_RECONNECT`, `WORKER_READY`, `WORKER_SHARDS_CONNECTED`)
- src/commands/applications/shared.ts + src/client/plugins/types.ts:32 (`ParseClient`, `SeyfertRegistry`)

## Agent Guidance

- Default to letting `Client` shard internally. Introduce `WorkerManager` + `WorkerClient` only when you need true parallelism across threads/processes.
- The `path` points to the COMPILED worker entry (`./dist/client.js`) for Node; for Bun/Deno point at the TS source. Required for `threads`/`clusters`, optional for `custom`.
- The mode literal is `'clusters'` (plural) — `'cluster'` fails the type check.
- For shared cache across workers: put `WorkerAdapter` on every worker AND set the real backend on the manager with `manager.setCache(...)`. Default manager cache is `MemoryAdapter`. Do NOT also pass `postMessage` unless you are replacing the transport.
- `tellWorker`/`tellWorkers` `eval` a stringified function in the target worker — closures over outer scope DO NOT transfer; pass data via `vars` (JSON-serialized). Both reject with `SeyfertError('WORKER_TIMEOUT')` after 60s.
- Set `sendPayloadToParent: true` only if the manager must receive every gateway packet; it defaults to `false` for startup/perf on large bots.
- Don't reach for `client.gateway` on a `WorkerClient` — it's undefined; use `client.shards` / `client.latency`.
- Prefer the `SHARD_DISCONNECT`/`SHARD_RECONNECT` events over the deprecated `onShardDisconnect`/`onShardReconnect` callbacks (those can double-fire).
- Register the worker client type with `interface SeyfertRegistry { client: ParseClient<WorkerClient> }` — NOT the old `UsingClient` augmentation.
