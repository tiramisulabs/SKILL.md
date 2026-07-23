# Queues (`@slipher/queues`)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/queues

Coverage reference: `plugins.md`

Verification status: Source-verified against current `tiramisulabs/extra/packages/queues` + Seyfert core

## Contents

- [Contract](#contract)
- [Typed setup](#typed-setup)
- [Producing and controlling jobs](#producing-and-controlling-jobs)
- [Events](#events)
- [Drivers](#drivers)
- [Dependency resolution](#dependency-resolution)
- [Lifecycle and gotchas](#lifecycle-and-gotchas)
- [Source anchors](#source-anchors)

## Contract

`queues(options)` installs one `QueuesRegistry` as `client.queues`, `ctx.queues`, and
`queuesPlugin.registry`.

Current plugin options:

- `driver: QueueDriver` — `memory()` or `persistent()`.
- `queueDefaults?: QueueOptions`.
- `processors?: readonly QueueConstructor[]`.
- `resolve?: (target) => instance` — dependency injection for processor classes.

Standalone `createQueues(options)` returns a registry without a Seyfert plugin; call
`registry.setup()` and `registry.close()` yourself.

Decorators:

- `@Processor(name, queueOptions?)`.
- Exactly one `@Process()` method per processor class; it receives the full `QueueJob`.
- `@OnWorkerEvent(event)` — process-local worker events.
- `@OnQueueEvent(event)` / `@QueueEvent(event)` — queue-global events.

## Typed setup

```ts
import { Client, definePlugins } from 'seyfert';
import {
  memory,
  OnQueueEvent,
  OnWorkerEvent,
  Process,
  Processor,
  queues,
  type QueueJobOf,
  type QueueRegistration,
} from '@slipher/queues';

type AudioJob =
  | { job: 'transcode'; fileId: string; format: 'mp3' | 'ogg' }
  | { job: 'concatenate'; fileIds: string[] };

declare module '@slipher/queues' {
  interface RegisteredQueues {
    audio: QueueRegistration<AudioJob, string>;
  }
}

@Processor('audio')
class AudioProcessor {
  @Process()
  async handle(job: QueueJobOf<'audio'>) {
    switch (job.name) {
      case 'transcode':
        return transcode(job.data.fileId, job.data.format);
      case 'concatenate':
        return concatenate(job.data.fileIds);
    }
  }

  @OnWorkerEvent('active')
  onActive({ job }: { job: QueueJobOf<'audio'> }) {
    void job.snapshot();
  }

  @OnQueueEvent('completed')
  onCompleted({ jobId, job, result }: {
    jobId: string;
    job: QueueJobOf<'audio'> | undefined;
    result: string;
  }) {
    auditCompletion(jobId, result, job?.snapshot());
  }
}

const queuesPlugin = queues({
  driver: memory({ attempts: 3, retryDelay: '5s', concurrency: 4 }),
  processors: [AudioProcessor],
});
const plugins = definePlugins(queuesPlugin);

declare module 'seyfert' {
  interface SeyfertRegistry {
    plugins: typeof plugins;
  }
}

export const registry = queuesPlugin.registry;
export const client = new Client({ plugins });
```

## Producing and controlling jobs

Named jobs:

```ts
await ctx.queues.get('audio').add(
  'transcode',
  { fileId: 'file-2', format: 'ogg' },
  { attempts: 5, retryDelay: '10s', priority: 1 },
);
```

Simple queue:

```ts
declare module '@slipher/queues' {
  interface RegisteredQueues {
    welcome: QueueRegistration<{ userId: string }, void>;
  }
}

await ctx.queues.get('welcome').add({ userId: ctx.author.id }, { delay: '5s' });
```

Each queue supports `add`, `process`, `start`, `pause`, `clear`, `close`, `counts`, `getJob`,
`on`, `once`, and `off`. Registry `get(name, options?)` returns the existing queue or creates it.
Reusing a queue name with different options throws.

`priority` and `delay` are `JobOptions`, not `memory()`/`QueueOptions`.

## Events

Queue/driver-neutral event payloads are intentionally different:

- Queue-global: `added`, `completed`, `failed`. Payloads include `jobId`; `job` may be
  `undefined` when an event originated from BullMQ `QueueEvents`.
- Worker-local: `active`, `completed`, `failed`, `retrying`, `idle`. Worker payloads carry a
  concrete `job`.
- Direct queue `.on()` also exposes the combined queue event map used by that driver.

Use `@OnWorkerEvent` when the handler must inspect the concrete job that ran locally. Use
`@OnQueueEvent` for cluster-visible outcomes and code defensively around `job === undefined`.
Listener failures are isolated and routed to `reportListenerError`.

## Drivers

`memory(options?: QueueOptions)` supports:

- `concurrency`, `attempts`, `retryDelay`, `autostart`, `retention`.
- Test/control hooks `now`, `idGenerator`, `reportListenerError`.
- Function-form `retryDelay(job, error)`.

`persistent(options?: PersistentQueueOptions)` supports:

- `connection`, `prefix`, `defaultJobOptions`.
- `queueOptions`, `queueEventsOptions`, `workerOptions`.
- Optional injected `bullmq` module for testing/custom loading.

Install `bullmq@^5` only when using `persistent()`. Function-form `retryDelay` is not supported
by the persistent driver; use a duration or BullMQ backoff object.

The persistent driver refuses to produce before setup. With the plugin, await `client.start()`
before adding jobs. Queue/Worker/QueueEvents resources close during plugin teardown.

## Dependency resolution

Processors are instantiated eagerly while `queues(options)` constructs its registry, using
`new target()` by default. This happens before plugin setup and commonly before the Seyfert client
exists, so constructors do not receive the client automatically. Use `resolve` only with
dependencies that already exist at composition time:

```ts
const queuesPlugin = queues({
  driver: memory(),
  processors: [AudioProcessor],
  resolve: (target) => container.resolve(target),
});
```

Alternatively keep processors constructor-free and capture a service/registry from the
composition module. Avoid importing the exported client from modules that the composition module
itself imports.

## Lifecycle and gotchas

- `ctx.queues === client.queues === queuesPlugin.registry`.
- `QueuesRegistry.setup()` delegates to the driver; `close()` closes the driver or every queue
  and aggregates close failures.
- On SIGTERM/SIGINT, await `client.close()` to tear persistent workers down, but also disconnect
  gateway shards and close any non-plugin resources you own; Seyfert core `close()` only tears
  plugins down.
- `retryDelay` only matters when `attempts > 1`; otherwise the package emits
  `SLIPHER_QUEUE_RETRY_DELAY_NO_RETRIES`.
- Invalid duration strings throw exported `InvalidDurationError`.
- Ambiguous `add('name', optionsShapedObject)` throws. Force a named job with
  `add(name, data, {})`, or use non-string data for the simple form.
- Put routing data such as `channelId`, `userId`, and locale inside the job payload.

## Source anchors

- `packages/queues/src/index.ts` — registry, plugin, decorators, processor instantiation.
- `packages/queues/src/core.ts` — public queue/driver/job/event contracts.
- `packages/queues/src/memory.ts` — in-process behavior and retries.
- `packages/queues/src/persistent.ts` — BullMQ lifecycle and restrictions.
- Seyfert `src/client/plugins.ts` and `src/client/plugins/types.ts` — plugin lifecycle/typing.
