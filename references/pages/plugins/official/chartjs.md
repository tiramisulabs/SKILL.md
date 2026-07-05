# Chart.js (@slipher/chartjs)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/chartjs
Coverage reference: plugins.md
Verification status: Source-verified (core Seyfert surface) + EXTERNAL package (`@slipher/chartjs` — verify version in target project)

## Page Summary

`@slipher/chartjs` is an external official plugin that renders [Chart.js](https://www.chartjs.org/) charts to PNG image buffers using [`@napi-rs/canvas`](https://github.com/Brooooooklyn/canvas) — no headless browser required. You construct a fixed-size `NapiChartjsCanvas`, render a standard Chart.js config to a `Buffer`, then attach it to a message via Seyfert's `AttachmentBuilder` and the `files` array.

Important: it is NOT a `createPlugin`/`definePlugins` runtime plugin (it never touches the `Client` plugin lifecycle). It is a standalone render utility you call inside a command. Its API is documented here from the upstream MDX — it is not part of core Seyfert and cannot be source-verified; confirm the installed version. Every Seyfert-side API it touches (`AttachmentBuilder`, `EmbedBuilder`, `ctx.write`/`ctx.editOrReply` with `files`) IS verified against `./src`.

## Key APIs (verified)

Core Seyfert APIs the examples use (confirmed in `./src`, all importable from the `seyfert` root barrel):

- `AttachmentBuilder` — class in `src/builders/Attachment.ts`, re-exported via `src/builders/index.ts` -> `src/index.ts`.
  - `new AttachmentBuilder(data?)` — default `filename` is a random `*.jpg` if none set (`randomBytes(8).toString('base64url')`).
  - `.setName(name: string): this` — sets `data.filename`.
  - `.setFile<T extends AttachmentDataType>(type: T, data: AttachmentResolvableMap[T]): this` — `type` is `'url' | 'path' | 'buffer'`.
  - `.setDescription(desc: string): this` (alt text), `.setSpoiler(spoiler: boolean): this`, getter `spoiler`.
  - `AttachmentResolvableMap` (`src/builders/Attachment.ts:9`): `url: string`, `path: string`, `buffer: ReadableStream | Buffer | ArrayBuffer | Uint8Array | Uint8ClampedArray`. So `setFile('buffer', buffer)` is valid for a Chart.js PNG `Buffer`. (`resolveAttachmentData` accepts `Buffer`, `ArrayBuffer`, `Uint8Array`/`Uint8ClampedArray`, and async iterables — verified `src/builders/Attachment.ts:236`.)
- The `files` field accepts `AttachmentBuilder[] | Attachment[] | RawFile[]` (`src/common/types/write.ts:25`). It is shared by `ctx.write`, `ctx.editOrReply`, `ctx.followup`, `channel.messages.write`, webhook sends, etc.
- `ctx.write({ files: [attachment] })` / `ctx.editOrReply({ files: [attachment] })` — standard send/edit paths on `CommandContext`. v5 note: both return `void` unless you pass the response flag (`write(body, true)` / `editOrReply(body, true)`) to fetch the `Message`.
- `Embed.setImage(url?)` (`src/builders/Embed.ts:106`; the class is exported as `Embed`) — pass `attachment://<filename>` to embed an attached chart inside an embed (Discord references the file in the same `files` array by name).

External `@slipher/chartjs` API (doc-authoritative — NOT in core Seyfert, verify version in target project):

- `NapiChartjsCanvas` — constructed with a single options object (see table below).
- `canvas.renderToBuffer(configuration)` — renders a standard Chart.js config and returns a PNG `Buffer` ready for `files`. (Synchronous in the MDX example — no `await`.)
- `canvas.renderChart(configuration)` — returns the underlying Chart.js instance (with its `canvas`) for lower-level access before producing a buffer.
- Both methods force `responsive: false` and `animation: false` on the config so charts render deterministically off-screen.

### NapiChartjsCanvas options (from MDX, external)

| Option | Type | Description |
| --- | --- | --- |
| `width` | `number` | Width of the rendered chart, in pixels. |
| `height` | `number` | Height of the rendered chart, in pixels. |
| `backgroundColour` | `string` | Fill behind the chart (British spelling). Transparent if omitted. Any canvas `fillStyle`. |
| `chartCallback` | `(ChartJS) => void \| Promise<void>` | Called once with the Chart.js global, for global defaults. |
| `plugins` | `object` | Register Chart.js plugins (`modern`, `requireLegacy`, `requireChartJSLegacy`, `globalVariableLegacy`). |

## Code Examples (verified)

Core Seyfert side verified against src; `@slipher/chartjs` side is from the MDX (external). Install: `pnpm add @slipher/chartjs` (Chart.js and `@napi-rs/canvas` ship as direct deps).

### 1. Bar chart as a plain attachment (canonical MDX example)

```ts
import { NapiChartjsCanvas } from '@slipher/chartjs';
import { Command, Declare, AttachmentBuilder, type CommandContext } from 'seyfert';

// reusable canvas with a fixed size and background (module scope — build once)
const canvas = new NapiChartjsCanvas({
  width: 800,
  height: 400,
  backgroundColour: 'white',
});

@Declare({ name: 'chart', description: 'Send a sample bar chart' })
export default class ChartCommand extends Command {
  async run(ctx: CommandContext) {
    // render a standard Chart.js config to a PNG buffer
    const buffer = canvas.renderToBuffer({
      type: 'bar',
      data: {
        labels: ['January', 'February', 'March', 'April'],
        datasets: [{ label: 'Messages', data: [120, 90, 200, 150] }],
      },
    });

    // wrap the buffer as a Discord attachment (setFile('buffer', ...) verified in src)
    const file = new AttachmentBuilder().setName('chart.png').setFile('buffer', buffer);

    await ctx.write({ files: [file] });
  }
}
```

### 2. Chart inside an embed via `attachment://`

`setImage('attachment://chart.png')` references the file you put in the same `files` array by its `setName(...)` filename — the standard Discord pattern, verified against `Embed.setImage`.

```ts
import { NapiChartjsCanvas } from '@slipher/chartjs';
import { Command, Declare, AttachmentBuilder, Embed, type CommandContext } from 'seyfert';

const canvas = new NapiChartjsCanvas({ width: 900, height: 450, backgroundColour: '#2b2d31' });

@Declare({ name: 'stats', description: 'Weekly stats as an embedded chart' })
export default class StatsCommand extends Command {
  async run(ctx: CommandContext) {
    const buffer = canvas.renderToBuffer({
      type: 'line',
      data: {
        labels: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'],
        datasets: [{ label: 'Joins', data: [4, 9, 6, 12, 8, 15, 11], borderColor: '#5865f2', tension: 0.3 }],
      },
      options: { plugins: { legend: { labels: { color: '#fff' } } } },
    });

    const file = new AttachmentBuilder().setName('chart.png').setFile('buffer', buffer);

    const embed = new Embed()
      .setTitle('Weekly joins')
      .setColor('Blurple')
      .setImage('attachment://chart.png'); // filename MUST match setName above

    await ctx.write({ embeds: [embed], files: [file] });
  }
}
```

### 3. Data-driven chart with a defer for slow renders

Heavy charts (lots of datasets, big canvas) can take a moment. Defer first, then edit the deferred reply with the file. `editOrReply` returns `void` here (no response flag) — the v5 contract.

```ts
import { NapiChartjsCanvas } from '@slipher/chartjs';
import {
  Command,
  Declare,
  Options,
  createStringOption,
  AttachmentBuilder,
  type CommandContext,
} from 'seyfert';

const canvas = new NapiChartjsCanvas({ width: 800, height: 400, backgroundColour: 'white' });

const options = {
  // option keys MUST be lowercase in v5 (compile-time enforced)
  metric: createStringOption({
    description: 'Which metric to chart',
    required: true,
    choices: [
      { name: 'Messages', value: 'messages' },
      { name: 'Voice minutes', value: 'voice' },
    ] as const, // choices are readonly -> use `as const`
  }),
} as const;

@Declare({ name: 'graph', description: 'Render a metric over time' })
@Options(options)
export default class GraphCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    await ctx.deferReply(); // ack within 3s; render below

    const metric = ctx.options.metric; // typed 'messages' | 'voice'
    const series = await loadSeries(ctx.guildId!, metric); // your data layer

    const buffer = canvas.renderToBuffer({
      type: 'bar',
      data: { labels: series.labels, datasets: [{ label: metric, data: series.values }] },
    });

    const file = new AttachmentBuilder().setName(`${metric}.png`).setFile('buffer', buffer);

    await ctx.editOrReply({ content: `Here is **${metric}**:`, files: [file] });
  }
}

declare function loadSeries(guildId: string, metric: string): Promise<{ labels: string[]; values: number[] }>;
```

### 4. Reusable renderer helper (one canvas, many commands)

Factor the canvas + buffer→attachment glue into a helper so every command stays a one-liner.

```ts
import { NapiChartjsCanvas } from '@slipher/chartjs';
import { AttachmentBuilder } from 'seyfert';
import type { ChartConfiguration } from 'chart.js';

const sharedCanvas = new NapiChartjsCanvas({ width: 800, height: 400, backgroundColour: 'white' });

/** Render any Chart.js config to a named Seyfert attachment. */
export function renderChartFile(config: ChartConfiguration, name = 'chart.png'): AttachmentBuilder {
  const buffer = sharedCanvas.renderToBuffer(config);
  return new AttachmentBuilder().setName(name).setFile('buffer', buffer);
}

// usage inside a command:
// await ctx.write({ files: [renderChartFile({ type: 'pie', data: { /* ... */ } }, 'roles.png')] });
```

## Common patterns / gotchas

- Reuse ONE `NapiChartjsCanvas` at module scope across renders (fixed `width`/`height`). The MDX does this; constructing per-command wastes the native canvas allocation.
- Filename match: when embedding via `setImage('attachment://X.png')`, `X.png` MUST equal the `setName('X.png')` filename, or Discord shows a broken image.
- British spelling: the option is `backgroundColour`, not `backgroundColor`. Omitting it yields a transparent PNG (fine for embeds with a dark theme, ugly on white Discord themes).
- `renderToBuffer` is shown synchronous in the MDX (no `await`); only `chartCallback` may be async. Verify the installed version's signatures since this package is external.
- Defer before slow work: a render that risks exceeding the 3s interaction window should `await ctx.deferReply()` first, then `ctx.editOrReply({ files })`.
- `setFile('buffer', buffer)` takes a Node `Buffer` directly — no base64, no data URL. Typed arrays and `ReadableStream` also work (verified `resolveAttachmentData`).
- v5 return types: `ctx.write` / `ctx.editOrReply` return `void` unless you pass the response flag (`true`). Pass it only when you need the resulting `Message`.
- Native dependency: `@napi-rs/canvas` is a native module — ensure the deploy target (Docker image, serverless runtime) has a compatible prebuilt binary.

## Doc vs Source Corrections

- None for the core Seyfert surface. `AttachmentBuilder`, `setName`, and `setFile('buffer', buffer)` match `src/builders/Attachment.ts` exactly; `files` typing matches `src/common/types/write.ts`; `Embed.setImage` matches `src/builders/Embed.ts`. All import from the `seyfert` root barrel — no deep import needed.
- `@slipher/chartjs` itself is external and not present in `./src`, so its signatures (`NapiChartjsCanvas`, `renderToBuffer`, `renderChart`, `plugins` keys) cannot be source-verified; they are reproduced from the MDX. Confirm the installed package version/API in the target project.
- It is NOT a `createPlugin`/`definePlugins` runtime plugin — do not add it to `new Client({ plugins })`. It is a render util invoked inside commands.

## Source Anchors

- Real runtime plugins live under `src/client/plugins/*` (createPlugin) — `@slipher/chartjs` is intentionally NOT one of them.

## Agent Guidance

- Use when a bot needs to send generated charts/graphs as image attachments without spinning up a browser (server-side canvas rendering).
