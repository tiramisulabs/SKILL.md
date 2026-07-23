# OpenTelemetry (`@slipher/opentelemetry`)

Coverage reference: `plugins.md`

Verification status: Source-verified against current `tiramisulabs/extra/packages/opentelemetry` + Seyfert core

## Contents

- [Contract](#contract)
- [Setup](#setup)
- [Instrumentation](#instrumentation)
- [Helpers](#helpers)
- [SDK ownership and lifecycle](#sdk-ownership-and-lifecycle)
- [Security and limitations](#security-and-limitations)
- [Source anchors](#source-anchors)

## Contract

`opentelemetry(options?)` installs automatic traces and duration histograms for Seyfert v5:

- Commands, components, and modals.
- Gateway event handlers.
- Discord REST requests.
- Cache adapter operations.

It also exposes one `TraceHandle` as `client.trace` and `ctx.trace`.

Package metadata requires `seyfert >=5.0.0` and `@opentelemetry/api ^1.9.0`.
`@opentelemetry/sdk-node` is bundled; exporters, processors, and optional metric packages are not.
Automatic instrumentation alone does not send telemetry anywhere: configure a processor/exporter
or rely on a host provider that already does.

Current `OpenTelemetryPluginOptions` extends `NodeSDK` constructor options and adds:

- `serviceName?: string` — defaults to `'seyfert'`.
- `instrument?: { interactions?, events?, rest?, cache? }` — every flag defaults `true`.
- `checkIfShouldTrace?: (source: TraceSource) => boolean`.
- `contextManager?: ContextManager`.
- `cache?: { skipResources?: string[] }` — defaults to `presence` and `voice_state`.

`serviceName` names tracers/meters and the owned SDK resource. The plugin identity remains
`@slipher/opentelemetry`.

## Setup

```ts
import { Client, definePlugins } from 'seyfert';
import { opentelemetry } from '@slipher/opentelemetry';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-node';

const plugins = definePlugins(
  opentelemetry({
    serviceName: 'my-bot',
    spanProcessors: [
      new BatchSpanProcessor(new OTLPTraceExporter()),
    ],
    instrument: {
      interactions: true,
      events: true,
      rest: true,
      cache: false,
    },
  }),
);

declare module 'seyfert' {
  interface SeyfertRegistry {
    plugins: typeof plugins;
  }
}

export const client = new Client({ plugins });
```

NodeSDK fields such as `spanProcessors`, `traceExporter`, `metricReader`, and
`instrumentations` apply only when this plugin owns and starts the SDK.

## Instrumentation

`TraceSource` is a discriminated union:

```ts
type TraceSource =
  | { kind: 'command' | 'component' | 'modal'; context: unknown }
  | { kind: 'event'; name: string; args: readonly unknown[] }
  | { kind: 'rest'; method: string; path: string }
  | { kind: 'cache'; op: string; resource: string };
```

Use `checkIfShouldTrace` to filter root spans:

```ts
opentelemetry({
  checkIfShouldTrace(source) {
    return !(source.kind === 'event' && source.name === 'RAW');
  },
});
```

If the predicate throws, instrumentation fails open and traces the operation. Instrumentation
cleanup/metrics errors are isolated from application code.

Duration histograms, in seconds:

- `seyfert.interaction.duration`.
- `seyfert.event.duration`.
- `seyfert.rest.duration`.
- `seyfert.cache.operation.duration`.

Instruments are created only for enabled surfaces.

## Helpers

Root exports:

- `record` / `startActiveSpan` — start an active child span, auto-end it, record exceptions, set
  error status, and rethrow.
- `startSpan` — manual span; the caller must end it.
- `getCurrentSpan`, `getTracer`, `getMeter`, `setAttributes`.
- `createTraceHandle`, type `TraceHandle`.

```ts
import { getCurrentSpan, record, setAttributes } from '@slipher/opentelemetry';

await record('fetch-profile', async span => {
  span.setAttribute('app.step', 'profile');
  await fetchProfile();
});

setAttributes({ 'app.feature': 'welcome' });
getCurrentSpan()?.addEvent('cache-miss');
```

`client.trace` / `ctx.trace` expose:

```ts
interface TraceHandle {
  readonly span: Span | undefined;
  setAttributes(attributes: Attributes): boolean;
  recordException(error: unknown): void;
  record: typeof record;
}
```

Outside an active span, `.span` is `undefined`, `setAttributes` returns `false`, and
`recordException` is a no-op.

## SDK ownership and lifecycle

If no real tracer provider exists, the plugin starts and owns a `NodeSDK`. On teardown it:

1. Removes event/REST/cache wrappers.
2. Shuts down only the SDK it started.
3. Removes only OpenTelemetry globals installed by that owned SDK.

If a host/preload already registered a real provider, the plugin reuses it, installs only its
instrumentation, ignores SDK-constructor fields, and does not shut the host provider down.

Teardown is terminal for one plugin instance. Setup after teardown throws. Create a fresh
`opentelemetry(...)` instance and fresh processor/exporter objects for a new client lifecycle.

## Security and limitations

The REST instrumentation omits request/response bodies, authorization/cookie headers, query
strings, and bot tokens. Discord webhook/interaction token path segments are redacted.

User, guild, channel, interaction, command, event, and custom-id data can still become span
attributes. Use `checkIfShouldTrace` when those identifiers must not be exported.

REST observer payloads do not include a request id. Concurrent requests sharing the same
method+path are correlated FIFO; out-of-order completion on the same route can associate status or
duration with the wrong span. Distinct routes are unaffected.

## Source anchors

- `packages/opentelemetry/src/index.ts` — public exports.
- `packages/opentelemetry/src/options.ts` — options/defaults/trace sources.
- `packages/opentelemetry/src/plugin.ts` — plugin lifecycle and instrumentation ownership.
- `packages/opentelemetry/src/trace-api.ts` and `handle.ts` — public helpers.
- `packages/opentelemetry/src/sdk.ts` — host-vs-owned SDK behavior.
- `packages/opentelemetry/src/instrument/**` — interaction/event/REST/cache instrumentation.
