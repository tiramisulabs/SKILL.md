# Queues (@slipher/queues)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/queues
Coverage reference: plugins.md
Verification status: Source-verified (core integration) + EXTERNAL package (queue API doc-authoritative — verify version in target project)

## Page Summary

`@slipher/queues` is an EXTERNAL package (NOT in seyfert-core) that adds typed background job queues to a Seyfert v5 bot. Producers enqueue jobs from anywhere (`ctx.queues.get(name).add(...)`); processors are decorated classes (`@Processor` + exactly one `@Process` handler, plus `@OnQueueEvent` / `@OnWorkerEvent` listeners). A driver decides where jobs run: `memory()` in-process, or `persistent()` on BullMQ/Redis — same API either way. The package plugs into Seyfert through the standard plugin surface (`definePlugins`, `SeyfertRegistry`, client/ctx extension, `client.close()` teardown), all of which are verified against ./src; the queue API itself is doc-authoritative — verify the version in the target project.

## Key APIs

### External (`@slipher/queues`, doc-authoritative — verify version in target project)
- `queues({ driver, processors, ... })` — builds the plugin; exposes `.registry`.
- Drivers: `memory({ attempts, retryDelay, concurrency, priority, delay, reportListenerError })`, `persistent({ connection, prefix, defaultJobOptions })` (needs `bullmq` installed).
- Decorators: `@Processor(name)`, `@Process()` (one per class), `@OnQueueEvent(event)`, `@OnWorkerEvent(event)`.
- Types: `QueueRegistration<Payload, Result>`, `QueueJobOf<'name'>`, interface `RegisteredQueues` (declaration-merge target).
- Producer: `registry.get(name).add(...)` / `ctx.queues.get(name).add(...)`.
  - Named-job queue: `add(jobName, payload, options?)`.
  - Simple queue (no `job` discriminant): `add(data, options?)`.
- Job object: `job.name`, `job.data`, `job.snapshot()`.
- Lifecycle events (memory): `added`, `active`, `completed`, `failed`, `retrying`, `idle`.
- Listener payloads: `{ job }`, `{ job, result }`, `{ job, error }`, `{ job, error, delay }`, `{}` (idle). Direct queue instances also support `.on()` / `.once()`.
- Errors: `InvalidDurationError` (exported for `instanceof`), warning code `SLIPHER_QUEUE_RETRY_DELAY_NO_RETRIES`, and a descriptive `TypeError` on ambiguous `add('name', objLookingLikeOptions)`.

### Core seyfert (verified in ./src — all importable from `'seyfert'`)
- `definePlugins(...plugins)` — `src/client/plugins.ts:283-289`. Accepts a spread OR a single array; returns the tuple as-is (identity helper for typing).
- `createPlugin(...)` — `src/client/plugins.ts:240` / `createPluginFactory(...)` — `:267` (the integration shape `@slipher/queues` builds on: `client` map extends the Client, `ctx` map extends command context).
- `interface SeyfertRegistry {}` — `src/client/plugins/types.ts:32` (declaration-merge `plugins: typeof plugins` to register the plugin tuple type; consumed by `RegisteredPlugins`, `types.ts:286`).
- `new Client({ plugins })` — `plugins` is a real BaseClient option; resolved into `client.plugins` (`src/client/base.ts:195,272`).
- `client.close()` — `src/client/base.ts:502` (async; runs the `client:close` plugin hook, which is how `@slipher/queues` tears queues down). NOTE: `close()` does NOT close gateway/REST/cache (`base.ts:500`) — it only runs plugin teardown.
- Barrel: plugin API re-exported via `src/client/index.ts:13` (`export * from './plugins'`) → `src/index.ts:1` (`export * from './client'`), so `definePlugins`/`SeyfertRegistry`/`createPlugin` are all root `'seyfert'` imports — no deep import needed.

## Code Examples (verified)

### Setup — plugin install (core imports confirmed against src; queue API per docs)

```ts
import { Client, definePlugins } from 'seyfert';
import {
  OnQueueEvent, OnWorkerEvent, Process, Processor,
  type QueueJobOf, type QueueRegistration,
  memory, queues,
} from '@slipher/queues';

// Discriminated union: one processor, several named jobs, each typed.
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
      case 'transcode': return transcode(job.data.fileId, job.data.format);
      case 'concatenate': return concatenate(job.data.fileIds);
    }
  }

  @OnWorkerEvent('active')
  onActive({ job }: { job: QueueJobOf<'audio'> }) { job.snapshot(); }

  @OnQueueEvent('completed')
  onCompleted({ job, result }: { job: QueueJobOf<'audio'>; result: string }) {
    job.snapshot(); void result;
  }
}

const queuesPlugin = queues({
  driver: memory({ attempts: 3, retryDelay: '5s', concurrency: 4 }),
  processors: [AudioProcessor],
});
const plugins = definePlugins(queuesPlugin);

declare module 'seyfert' {
  interface SeyfertRegistry { plugins: typeof plugins }
}

export const registry = queuesPlugin.registry;
export const client = new Client({ plugins });
```

### Producing jobs

```ts
// named-job queue
await ctx.queues.get('audio').add('concatenate', { fileIds: ['a', 'b'] });
await ctx.queues.get('audio').add(
  'transcode', { fileId: 'file-2', format: 'ogg' }, { attempts: 5, retryDelay: '10s' },
);

// simple queue (no `job` discriminant): add(data, options)
declare module '@slipher/queues' {
  interface RegisteredQueues { welcome: QueueRegistration<{ userId: string }> }
}
await ctx.queues.get('welcome').add({ userId: ctx.author.id });
```

### Enqueue from a slash command (full command file)

`ctx.queues` is the plugin-extended context — typed because `SeyfertRegistry.plugins` was augmented. `write`/`editOrReply` return `void` in v5 unless you pass the response flag `true`.

```ts
import { Command, Declare, Options, createStringOption } from 'seyfert';

const options = {
  // v5: option keys MUST be lowercase (compile-time check)
  fileid: createStringOption({ description: 'File to transcode', required: true }),
} as const;

@Declare({ name: 'transcode', description: 'Queue an audio transcode' })
@Options(options)
export default class TranscodeCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    await ctx.queues.get('audio').add('transcode', {
      fileId: ctx.options.fileid,
      format: 'mp3',
    });
    // deferReply earlier if the enqueue or downstream UX is slow
    await ctx.write({ content: 'Queued. I will ping you when it finishes.' });
  }
}
```

### Enqueue from an event (use `entity.client.queues`)

Inside an event handler there is no `ctx`; reach the registry through the client carried on the entity. Custom event `run` is `Awaitable<unknown>` in v5; gateway handlers no longer get a trailing `shardId` for custom events.

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'guildMemberAdd' },
  run(member) {
    // member.client === the running Client, extended with .queues by the plugin
    return member.client.queues.get('welcome').add({ userId: member.id });
  },
});
```

### Notify the user when a job completes (carry routing data in the payload)

The processor only gets `job.data`, so stash whatever you need to reply (channel id, user id) inside the payload. Use `client.queues` from the processor's listeners to reach REST — but capture `registry`/`client` carefully to avoid circular imports (see gotchas).

```ts
type NotifyJob = {
  job: 'render';
  prompt: string;
  channelId: string;
  userId: string;
};

declare module '@slipher/queues' {
  interface RegisteredQueues { render: QueueRegistration<NotifyJob, string> }
}

@Processor('render')
class RenderProcessor {
  constructor(private readonly client: UsingClient) {}

  @Process()
  async handle(job: QueueJobOf<'render'>) {
    return render(job.data.prompt); // returns the result string (an image url, etc.)
  }

  @OnQueueEvent('completed')
  async onDone({ job, result }: { job: QueueJobOf<'render'>; result: string }) {
    await this.client.messages.write(job.data.channelId, {
      content: `<@${job.data.userId}> done: ${result}`,
    });
  }

  @OnQueueEvent('failed')
  async onFail({ job, error }: { job: QueueJobOf<'render'>; error: Error }) {
    await this.client.messages.write(job.data.channelId, {
      content: `<@${job.data.userId}> render failed: ${error.message}`,
    });
  }
}
```

> Note: whether a `@Processor` constructor receives the client is a `@slipher/queues` convention — confirm against the installed package. If it does not inject, capture `registry`/`client` from your composition module instead.

### ctx-less producer (capture `registry`, not `client`, to dodge circular imports)

```ts
// services/media.ts
import { registry } from '../index';
export function scheduleTranscode(fileId: string) {
  return registry.get('audio').add('transcode', { fileId, format: 'mp3' });
}
```

### Isolated listener errors

A throwing listener never changes job state, retry counts, or whether later listeners run — it is routed to `reportListenerError`.

```ts
queues({
  driver: memory({
    reportListenerError(event, error) {
      logger.error('queue listener failed', { event, error });
    },
  }),
  processors: [AudioProcessor],
});
```

### Persistent (BullMQ/Redis) driver + clean shutdown

```ts
import { persistent, queues } from '@slipher/queues';

const queuesPlugin = queues({
  driver: persistent({
    connection: { host: '127.0.0.1', port: 6379 },
    prefix: 'slipher',
    defaultJobOptions: { removeOnComplete: true, attempts: 3 },
  }),
  processors: [AudioProcessor],
});

// client.close() runs plugin teardown (verified base.ts:502). Wire it to signals.
process.on('SIGTERM', () => {
  void client.close().then(() => process.exit(0));
});
```

## Common patterns / gotchas

- `ctx.queues === client.queues === queuesPlugin.registry` — one object, three entry points (command/component/modal: `ctx.queues`; event: `entity.client.queues`; service: captured `registry`).
- ONE `@Process()` per `@Processor` class. There is NO framework per-name dispatch — `switch (job.name)` yourself for named-job queues. Typos surface through the typed producer at compile time, not as runtime "process not found".
- Ambiguous `add`: `add('name', { delay: '5s' })` throws a descriptive `TypeError` (string payload + options vs. named job whose payload looks like options). Force it: `add('name', { delay: '5s' }, {})`, or use non-string data with `add(data, options)`.
- `retryDelay` only applies when `attempts > 1`; otherwise you get a `SLIPHER_QUEUE_RETRY_DELAY_NO_RETRIES` warning. Bad duration strings throw `InvalidDurationError` (exported for `instanceof`).
- Processors get only `job.data` — put routing/context (channelId, userId, locale) INTO the payload if a listener needs to reply.
- Circular imports: processors/services loaded by `index.ts` must NOT import the exported `client`; capture `registry` at composition time instead.
- `memory()` for single-process bots; `persistent()` when jobs must survive restarts or span workers — queue/processor code is byte-identical, only the driver changes. `persistent` needs `bullmq` installed.
- Choose `@OnWorkerEvent` for process-local reactions (the worker that ran the job) and `@OnQueueEvent` for queue-wide/global reactions. With `memory()` both observe the same single-process queue.

## Doc vs Source Corrections

- None for core APIs. `definePlugins` (`:283`), `createPlugin` (`:240`), `createPluginFactory` (`:267`), `SeyfertRegistry` (`types.ts:32`), `new Client({ plugins })` (`base.ts:195/272`), and `client.close()` (`base.ts:502`) all match ./src exactly and resolve from the `'seyfert'` root barrel.
- The MDX comment says jobs are "switched on the `job` field" but the handler/registration switches on `job.name` — the `job` key is the discriminant in the `RegisteredQueues` registration; `name` is the runtime field on the job object. Package convention, not a core conflict; verify against the installed `@slipher/queues` types.
- Queue decorators/drivers/registry methods are NOT in seyfert-core; they cannot be source-verified here. Treat the queue API as doc-authoritative and confirm against the installed package version.
- v5 reminders that touch these examples (verified in checklist): option keys must be lowercase; `write`/`editOrReply` return `void` unless the response flag is `true`; middleware uses `stop()` (no `pass()`); custom event `run` is `Awaitable<unknown>` with no trailing `shardId`.

## Source Anchors

- `src/client/plugins.ts` — `createPlugin` :240, `createPluginFactory` :267, `definePlugins` :283-289, `createContextScope` :291
- `src/client/plugins/types.ts` — `SeyfertRegistry` :32, `PluginClientMap` :244, `RegisteredPlugins` :286 (consumes `SeyfertRegistry.plugins`)
- `src/client/plugins/api.ts`, `registry.ts`, `shared.ts`, `order.ts`, `errors.ts` (plugin runtime: registry, shared services, ordering, error wrapping)
- `src/client/base.ts` — plugins option :195, resolved :272, `close()` :502 (runs `client:close` hook; gateway/REST/cache untouched, :500)
- `src/client/index.ts:13`, `src/index.ts:1` — barrel re-exports → root `'seyfert'` imports

## Agent Guidance

- Use when a bot needs deferred/background work (media transcode, welcome flows, scheduled side effects, rendering). Reach for `memory()` for single-process bots; switch to `persistent()` only when jobs must survive restarts or span workers — the queue/processor code is identical.
- EXTERNAL package: before editing production code, confirm it is installed and check its version/types — the queue API here is from docs, not seyfert-core. `pnpm add @slipher/queues` (+ `bullmq` for persistent).
- One `@Process()` per `@Processor`; dispatch named jobs via `switch (job.name)`. There is no per-name framework dispatch.
- `ctx.queues === client.queues === queuesPlugin.registry` (same object). Capture `registry` (not `client`) in services loaded by `index.ts` to avoid circular imports.
- Gotchas: `add('name', { delay: '5s' })` is ambiguous → `TypeError`; pass an explicit `{}` third arg or non-string data. `retryDelay` only applies when `attempts > 1` (else `SLIPHER_QUEUE_RETRY_DELAY_NO_RETRIES`). Invalid duration strings → `InvalidDurationError`. Listener errors are isolated via `reportListenerError` and never alter job state.
- Always wire `client.close()` to process signals so queues tear down cleanly (close() invokes plugin teardown — verified base.ts:502; note it does NOT close gateway/REST/cache).
