# Scheduler (`@slipher/scheduler`)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/scheduler

Coverage reference: `plugins.md`

Verification status: Source-verified against current `tiramisulabs/extra/packages/scheduler` + Seyfert core

## Contents

- [Contract](#contract)
- [Plugin setup](#plugin-setup)
- [Registry API](#registry-api)
- [Persistent schedules](#persistent-schedules)
- [Events](#events)
- [Lifecycle and gotchas](#lifecycle-and-gotchas)
- [Source anchors](#source-anchors)

## Contract

`scheduler(options)` installs one `SchedulerRegistry` as `client.scheduler`, `ctx.scheduler`, and
`schedulerPlugin.registry`. `createScheduler(options)` creates the same registry without a
Seyfert plugin.

Required options:

- `driver: SchedulerDriver` — `memory()` or `persistent()`.

Optional options:

- `tasks?: SchedulerTaskSource[]` — decorated classes or instances.
- `resolveTask?: (source) => instance` — dependency injection.
- `logger?: SchedulerLogger`.

Decorators:

- `@Cron(expression, { id?, runImmediately?, data? })`.
- `@Interval(duration, { id?, runImmediately?, data? })`.

Persistent decorated tasks require an explicit non-empty `id`. Method-name defaults are accepted
only by the memory driver.

## Plugin setup

```ts
import { Client, definePlugins } from 'seyfert';
import { Interval, memory, scheduler } from '@slipher/scheduler';

class MaintenanceTasks {
  @Interval('5m', { id: 'heartbeat', runImmediately: true })
  heartbeat() {
    // work
  }
}

const schedulerPlugin = scheduler({
  driver: memory(),
  tasks: [MaintenanceTasks],
});
const plugins = definePlugins(schedulerPlugin);

declare module 'seyfert' {
  interface SeyfertRegistry {
    plugins: typeof plugins;
  }
}

export const registry = schedulerPlugin.registry;
export const client = new Client({ plugins });
```

With the plugin, `setup` prepares driver resources and the `plugins:ready` hook activates tasks
only after every plugin completed setup. Teardown closes the driver.

Standalone:

```ts
import { createScheduler, memory } from '@slipher/scheduler';

const registry = createScheduler({ driver: memory() });
registry.cron('daily-cleanup', '0 0 * * *', async () => {});
await registry.setup();
// ...
await registry.close();
```

## Registry API

Define:

- `cron(id, expression, runner, options?)`.
- `interval(id, duration, runner, options?)`.
- `add(id, durationOrCron, runner, options?)` — duration first, then cron fallback.
- `register(tasks, resolveTask?)`.

Read:

- `get(id)`, `list()`, `snapshot()`.

Mutate — all asynchronous:

- `pause(id)`, `resume(id)`, `remove(id)`.
- Deprecated alias `start(id)` delegates to `resume(id)`.
- `removeOrphan(id)` removes a driver schedule that is no longer registered in this registry.

Lifecycle:

- `prepare(client?)`, `activate(client?)`, `setup(client?)`, `close()`.

Events:

- `on(event, listener)`, `once(event, listener)`, `off(event, listener)`.

Always await mutations:

```ts
if (action === 'pause') await ctx.scheduler.pause(id);
else if (action === 'resume') await ctx.scheduler.resume(id);
else if (action === 'remove') await ctx.scheduler.remove(id);

await ctx.write({ content: `Done: ${action} ${id}.` });
```

## Persistent schedules

```ts
import { persistent, scheduler } from '@slipher/scheduler';

const schedulerPlugin = scheduler({
  driver: persistent({
    connection: { host: '127.0.0.1', port: 6379 },
    queueName: 'scheduler',
    prefix: 'slipher',
    purgeOrphansOnStartup: false,
    immediateRunDeduplicationMs: 60_000,
  }),
  tasks: [MaintenanceTasks],
});
```

Current persistent options:

- `connection`, `queueName`, `prefix`.
- `purgeOrphansOnStartup`.
- `immediateRunDeduplicationMs` — positive integer; defaults to 60 seconds.
- `logger`.
- Optional injected `bullmq` module.

Install `bullmq@^5.23.0` only when using persistent scheduling.

Removing a decorated task from source leaves its Redis scheduler behind. Choose one:

```ts
await registry.removeOrphan('removed-task-id');
```

or enable `purgeOrphansOnStartup`. Do not call `remove(id)` for an orphan: `remove` requires a
currently registered task and throws otherwise.

`pause` removes the BullMQ job scheduler; `resume` re-upserts it. `runImmediately` is deduplicated
across replicas inside the configured window.

## Events

Events and payloads:

- `scheduled`, `started`, `paused`, `resumed`, `removed` → `{ task }`.
- `completed` → `{ task, result }`.
- `failed` → `{ task, error }`.
- `error` → `{ source: 'queue' | 'queue-events' | 'worker', error }`.

```ts
const offFailed = registry.on('failed', ({ task, error }) => {
  client.logger.error(`Task ${task.id} failed`, error);
});

registry.on('error', ({ source, error }) => {
  client.logger.error(`Scheduler ${source} error`, error);
});

// later
offFailed();
```

Listener failures do not alter task state or block later listeners. They are reported through the
configured logger or a process warning.

## Lifecycle and gotchas

- `ctx.scheduler === client.scheduler === schedulerPlugin.registry`.
- `client.start()` prepares persistent Queue/Worker/QueueEvents, then activates work at
  `plugins:ready`. Preparation failures reject startup before tasks activate.
- `client.close()` runs plugin teardown and closes scheduler resources. On SIGTERM/SIGINT, also
  disconnect gateway shards and close any non-plugin resources you own; Seyfert core `close()`
  does not close gateway, REST, or cache.
- `memory()` uses Croner in-process. Intervals have one-second scheduling resolution; set an
  explicit process timezone such as `TZ=UTC` when cron timezone matters.
- Persistent task ids are Redis scheduler ids. Renaming a method without a stable id creates an
  orphan.
- `ScheduledTask` exposes id/kind/expression/interval/status/runCount/timestamps/error/data and
  `snapshot()`.
- A torn-down registry/driver instance should not be restarted; create a fresh plugin instance
  for a fresh client lifecycle.

## Source anchors

- `packages/scheduler/src/index.ts` — public exports.
- `packages/scheduler/src/manager.ts` — registry and plugin lifecycle.
- `packages/scheduler/src/types.ts` — options, events, driver/task contracts.
- `packages/scheduler/src/drivers/{memory,persistent}.ts` — driver behavior.
- `packages/scheduler/src/events.ts` — listener isolation.
- Seyfert `src/client/base.ts` — `plugins:ready` and teardown order.
