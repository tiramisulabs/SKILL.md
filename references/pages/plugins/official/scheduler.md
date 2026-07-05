# Scheduler (@slipher/scheduler)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/scheduler

Coverage reference: plugins.md

Verification status: Source-verified (core integration surface) + EXTERNAL package (@slipher/scheduler — verify version in target project)

## Page Summary

`@slipher/scheduler` adds cron/interval task scheduling to a Seyfert v5 bot. Tasks live on a
single **registry**; a **driver** decides where they run — `memory()` (in-process, Croner) or
`persistent()` (BullMQ/Redis, restart-surviving and replica-coordinated). The same task code runs
under either driver. Tasks are defined as decorated classes (`@Cron`/`@Interval`) or via imperative
registry calls, and are reached through `ctx.scheduler`, `client.scheduler`, or a captured
`registry` (all the same object). The package is EXTERNAL (not in core Seyfert); its
decorators/drivers/registry API below come from the MDX docs and MUST be re-verified against the
version installed in the target project. The Seyfert integration surface it relies on
(`definePlugins`, `SeyfertRegistry` augmentation, plugin `setup`/`teardown` lifecycle, and
`client`/`ctx` extension maps) IS verified against ./src.

## Key APIs (verified)

Core Seyfert APIs the page uses (all root-importable from `seyfert`):

- `definePlugins(...plugins)` / `definePlugins(plugins[])` — src/client/plugins.ts:283-285. Two
  overloads: accepts a spread OR a single array; returns the tuple unchanged. Exported from
  `seyfert` via `src/index.ts` -> `./client` -> `./client/plugins`.
- `interface SeyfertRegistry {}` — src/client/plugins/types.ts:32. Augment with
  `{ plugins: typeof plugins }` to register the plugin tuple for client/ctx type inference
  (`RegisteredPlugins` reads `SeyfertRegistry['plugins']`, types.ts:286-288). Without this
  augmentation `ctx.scheduler` / `client.scheduler` do NOT type-resolve.
- Plugin `client` map (`PluginClientMap`, types.ts:244) — how a plugin attaches `client.scheduler`
  (a factory `(client) => value`). Plugin `ctx` map (`PluginContextMap`, types.ts:250) — how it
  attaches `ctx.scheduler` (a factory `(interaction, client) => value`). These are the mechanisms
  behind `ctx.scheduler` / `client.scheduler`.
- Plugin lifecycle: `setup(client, api?)` / `teardown(client, api?)` — types.ts:510-511, driven by
  `setupClientPlugins` / `teardownClientPlugins` (src/client/plugins.ts:656,717). This is what the
  docs mean by "the Seyfert lifecycle handles setup on start, teardown on close". Both return
  `Awaitable<void>`.
- `client.start()` / `client.close()` — base lifecycle; `close()` fires the `client:close` plugin
  hook (src/client/base.ts:510, `runPluginHooks(this, 'client:close', this)`). Used by
  `persistent()` to open Redis/BullMQ on start and release them on close.
- `createPlugin` / `createPluginFactory` (src/client/plugins.ts:240,267) — the authoring helpers a
  package like this uses internally to build its plugin object.
- `createEvent({ data, run })` (src/index.ts:68) — `run(payload, client)` for gateway events
  (v5 custom/non-gateway handlers no longer get a trailing `shardId`); reach the scheduler via the
  `client` param or `entity.client.scheduler`.

EXTERNAL `@slipher/scheduler` exports (doc-authoritative — verify version in target project):

- `scheduler({ driver, tasks })` — builds the Seyfert plugin; exposes `.registry`.
- `createScheduler({ driver })` — standalone registry, no Seyfert plugin; call `registry.setup()` yourself.
- `memory()` — in-process Croner driver. Intervals tick at 1s resolution (sub-second rounds up).
- `persistent({ connection, queueName, prefix, purgeOrphansOnStartup? })` — BullMQ/Redis driver.
- Decorators `@Cron(expr, opts?)`, `@Interval(duration, opts?)`; type `ScheduledTask`.
- Registry methods: `cron(id, expr, fn)`, `interval(id, duration, fn)`, `add(id, value, fn)`
  (auto-detects cron vs interval), `get(id)`, `list()`, `snapshot()`, `pause(id)`, `resume(id)`,
  `remove(id)`, `setup()`, `on(event, cb)` (returns unsubscribe), `once(event, cb)`.
- Task options: `{ id, runImmediately? }`. `ScheduledTask` exposes `id`, `runCount`.
- Events: `scheduled`, `started`, `completed`, `failed`, `paused`, `resumed`, `removed`.
  `completed` payload `{ task, result }`; `failed` payload `{ task, error }`.
- Durations: ms number or strings `'30s'`/`'5m'`/`'1h'`/`'1d'`, compound `'1h30m'`.

## Code Examples (verified)

Plugin setup with in-process driver (core imports verified from `seyfert`):

```ts
import { Client, definePlugins } from 'seyfert';
import { Interval, memory, scheduler } from '@slipher/scheduler';

class MaintenanceTasks {
	@Interval('5m', { id: 'heartbeat' })
	heartbeat() {
		// run work
	}
}

const schedulerPlugin = scheduler({ driver: memory(), tasks: [MaintenanceTasks] });
const plugins = definePlugins(schedulerPlugin);

declare module 'seyfert' {
	interface SeyfertRegistry {
		plugins: typeof plugins;
	}
}

export const client = new Client({ plugins });
```

Decorated tasks; `runImmediately` runs once at setup then on schedule:

```ts
import { Cron, Interval, type ScheduledTask } from '@slipher/scheduler';

class Tasks {
	@Cron('0 9 * * *', { id: 'morning-report' })
	report(task: ScheduledTask) {
		return task.id;
	}

	@Interval('1h', { id: 'heartbeat', runImmediately: true })
	heartbeat() {}
}
```

Scheduling imperatively from a command (`ctx.scheduler` is the same registry):

```ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'refresh-cache', description: 'Schedule a cache refresh' })
export default class RefreshCacheCommand extends Command {
	async run(ctx: CommandContext) {
		ctx.scheduler.interval('refresh-cache', '30s', async task => {
			void task.id;
		});
		// write(body) returns void in v5 unless you pass the response flag
		await ctx.write({ content: 'Cache refresh scheduled.' });
	}
}
```

Capturing the registry for non-client code (avoids a circular import on `client`):

```ts
// index.ts
const schedulerPlugin = scheduler({ driver: memory(), tasks: [MaintenanceTasks] });
const plugins = definePlugins(schedulerPlugin);
export const registry = schedulerPlugin.registry;
export const client = new Client({ plugins });

// services/reports.ts — no client, no ctx needed
import { registry } from '../index';
export const pauseReports = () => registry.pause('morning-report');
```

Persistent driver + graceful shutdown:

```ts
import { Client, definePlugins } from 'seyfert';
import { persistent, scheduler } from '@slipher/scheduler';

const schedulerPlugin = scheduler({
	driver: persistent({
		connection: { host: '127.0.0.1', port: 6379 },
		queueName: 'scheduler',
		prefix: 'slipher',
		purgeOrphansOnStartup: true, // delete Redis schedules with no matching task on start
	}),
	tasks: [MaintenanceTasks],
});
const plugins = definePlugins(schedulerPlugin);
const client = new Client({ plugins });
await client.start(); // opens Redis/BullMQ during plugin setup

process.on('SIGTERM', () => void client.close().then(() => process.exit(0)));
```

Standalone (no Seyfert plugin):

```ts
import { createScheduler, memory } from '@slipher/scheduler';

const registry = createScheduler({ driver: memory() });
registry.cron('daily-cleanup', '0 0 * * *', async () => {});
registry.add('poller', '10s', async task => void task.runCount); // add() auto-detects kind
await registry.setup(); // no client lifecycle, so call setup yourself
```

## Recipes (new — verified core surface)

Scheduling reactively from an event handler (reach the registry via the `client` param):

```ts
// events/guildCreate.ts
import { createEvent } from 'seyfert';

export default createEvent({
	data: { name: 'guildCreate' },
	// v5: run(payload, client) — non-gateway handlers no longer receive shardId
	run(guild, client) {
		// run a one-off onboarding cron per guild; stable id keyed by guild id
		client.scheduler.cron(`welcome-digest:${guild.id}`, '0 12 * * MON', async () => {
			await client.messages.write(guild.systemChannelId!, {
				content: 'Weekly digest is ready.',
			});
		});
	},
});
```

Admin command that lists / pauses / resumes / removes tasks at runtime:

```ts
import {
	Command,
	Declare,
	Options,
	createStringOption,
	type CommandContext,
} from 'seyfert';

const options = {
	// option record keys MUST be lowercase in v5 (compile-time enforced)
	action: createStringOption({
		description: 'What to do',
		required: true,
		choices: [
			{ name: 'list', value: 'list' },
			{ name: 'pause', value: 'pause' },
			{ name: 'resume', value: 'resume' },
			{ name: 'remove', value: 'remove' },
		] as const, // choices is readonly in v5 — use `as const`
	}),
	id: createStringOption({ description: 'Task id', required: false }),
};

@Declare({ name: 'tasks', description: 'Manage scheduled tasks' })
@Options(options)
export default class TasksCommand extends Command {
	async run(ctx: CommandContext<typeof options>) {
		const { action, id } = ctx.options;
		const registry = ctx.scheduler;

		if (action === 'list') {
			const ids = registry.list().map(t => t.id);
			return ctx.write({ content: ids.length ? ids.join(', ') : 'No tasks.' });
		}
		if (!id) return ctx.write({ content: 'An id is required for that action.' });

		if (action === 'pause') registry.pause(id);
		else if (action === 'resume') registry.resume(id);
		else if (action === 'remove') registry.remove(id);

		await ctx.write({ content: `Done: ${action} ${id}.` });
	}
}
```

Observability — feed scheduler events into metrics/logging:

```ts
// index.ts (after building schedulerPlugin)
const registry = schedulerPlugin.registry;

// on() returns an unsubscribe fn — keep it if you ever need to detach
const offFail = registry.on('failed', ({ task, error }) => {
	client.logger.error(`Task ${task.id} failed`, error);
});

registry.on('completed', ({ task, result }) => {
	client.logger.debug(`Task ${task.id} ok (run #${task.runCount})`, result);
});

// later, when shutting a subsystem down: offFail();
```

Mixing decorated classes with imperative tasks (both share one registry):

```ts
import { Client, definePlugins } from 'seyfert';
import { Cron, memory, scheduler } from '@slipher/scheduler';

class CoreTasks {
	@Cron('*/15 * * * *', { id: 'sweep' })
	sweep() {/* every 15 min */}
}

const schedulerPlugin = scheduler({ driver: memory(), tasks: [CoreTasks] });
const plugins = definePlugins(schedulerPlugin);
const client = new Client({ plugins });

// add more tasks at composition time alongside the decorated ones
schedulerPlugin.registry.interval('ping-upstream', '1m', async () => {/* ... */});

await client.start();
```

## Common patterns / gotchas

- `ctx.scheduler`, `client.scheduler`, and `schedulerPlugin.registry` are the SAME object — pick
  whichever is reachable: `ctx.scheduler` in commands/components/modals, `entity.client.scheduler`
  in events, the captured `registry` in plain modules.
- Types only resolve after the `declare module 'seyfert' { interface SeyfertRegistry { plugins: typeof plugins } }`
  augmentation. v5 augments `SeyfertRegistry` (NOT `UsingClient`/`RegisteredMiddlewares`); reuse the
  same block to also register `client`, `middlewares`, `langs`.
- Always give tasks a stable explicit `id`. With `memory()` the method name is an acceptable
  default; with `persistent()` ids ARE the Redis scheduler ids — renaming a method without a fixed
  `id` orphans the old Redis schedule and creates a new one.
- Removing a task from code does NOT stop a `persistent()` schedule — call `registry.remove(id)` or
  start with `persistent({ purgeOrphansOnStartup: true })`. `pause(id)`/`resume(id)` drop and
  recreate the BullMQ job scheduler.
- `memory()` intervals are 1s-resolution (sub-second values like `'500ms'` round up). Cron uses
  Croner's runtime timezone — set `TZ` (e.g. `TZ=UTC`) explicitly if timezone matters.
- For `persistent()`, wire `client.close()` to process signals (SIGTERM/SIGINT) so Redis/BullMQ
  release on shutdown — `close()` fires the `client:close` plugin hook (verified, base.ts:510).
  Install `bullmq@^5.23.0` only when you actually use `persistent()`.
- `on(...)` returns an unsubscribe function; listener errors are isolated and reported via the
  configured logger. With `persistent()`, `started`/`completed`/`failed` come from BullMQ
  `QueueEvents`, so every replica sees the cluster-level outcome.

## Doc vs Source Corrections

None for the core integration surface. The MDX's use of `definePlugins`, the
`declare module 'seyfert' { interface SeyfertRegistry { plugins } }` augmentation, and the
`setup`/`teardown` lifecycle all match ./src exactly. The `@slipher/scheduler`
package itself is not in core Seyfert, so its decorator/driver/registry signatures could not be
diffed against source — treat them as doc-authoritative and verify against the installed package
version. Surrounding command/event examples follow the standard v5 conventions checklist — see plugins.md.
