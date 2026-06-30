# Custom Logger

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/logger
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (v5, branch more-qol)

## Page Summary

Seyfert ships a built-in `Logger` class (`src/common/it/logger.ts`) used internally and exposed per-client as `client.logger` (a `Logger` named `'[Seyfert]'`). It supports a global format override (`Logger.customize`), optional file output (`Logger.saveOnFile` / `Logger.dirname`), prototype extension for custom methods, and standard level methods. The page also demonstrates a webhook error-reporting pattern via `client.webhooks.writeMessage`. Source confirms every documented API plus several customization helpers the docs omit. In v5 the client logger is configurable via `new Client({ logger: { ... } })` and that option is now honored on worker clients too.

## Key APIs (verified)

All from `src/common/it/logger.ts` unless noted.

- `class Logger` — root export from `'seyfert'` (`src/index.ts:29` re-exports it via `./common`).
- `enum LogLevels { Debug=0, Info=1, Warn=2, Error=3, Fatal=4 }` — NOT re-exported from `'seyfert'` root; import via deep path `seyfert/lib/common` (it lives in the `src/common/index.ts:7` barrel only).
- `new Logger(options: LoggerOptions)` — constructor REQUIRES an options object (no empty constructor). `LoggerOptions = { logLevel?: LogLevels; name?: string; active?: boolean; saveOnFile?: boolean }`. Options are merged over `Logger.DEFAULT_OPTIONS` via `MergeOptions`.
- Instance methods: `debug`, `info`, `warn`, `error`, `fatal` `(...args: any[])` and `rawLog(level: LogLevels, ...args: unknown[])`.
- Instance accessors: `level` (get/set `LogLevels`), `name` (get/set string), `active` (get/set boolean), `saveOnFile` (get/set boolean, per-instance), and `readonly options: Required<LoggerOptions>`.
- `static Logger.customize(cb): () => void` — sets the global format callback AND returns a disposer that restores the *previous* callback (no-op if a newer one already replaced it). `CustomizeLoggerCallback = (self: Logger, level: LogLevels, args: unknown[]) => unknown[] | undefined`. Returning `undefined`/falsy skips that log entirely (`rawLog`: `if (!log) return`).
- `static Logger.getCustomizer(): CustomizeLoggerCallback` — returns the current callback so a new one can chain it.
- `static Logger.customizeFilename(cb: AssignFilenameCallback)` — `AssignFilenameCallback = (self: Logger) => string`; overrides log file naming (default `name-M-D-Y-epoch.log`).
- `static Logger.clearLogs(): Promise<void>` — closes streams and unlinks all files in `process.cwd()/Logger.dirname`.
- `static Logger.saveOnFile?: string[] | 'all'` — `'all'` logs every logger; an array filters by logger `name` (e.g. `['[Seyfert]']`).
- `static Logger.dirname = 'seyfert-logs'` — default output dir (relative to `process.cwd()`).
- `static Logger.DEFAULT_OPTIONS` = `{ logLevel: Debug, name: 'seyfert', active: true, saveOnFile: false }`.
- `static Logger.colorFunctions` / `Logger.prefixes` (Maps keyed by `LogLevels`), `static noColor(msg)`, and the exported free function `formatMemoryUsage(bytes)`.
- `client.logger` — a `Logger` instance named `'[Seyfert]'` (`src/client/base.ts:178`). Configurable through the client `logger` option (`ClientOptions.logger?: LoggerOptions`, `base.ts:1300`) applied in `configureLogger` (`base.ts:281`, sets `active`, `logLevel`, `name`, `saveOnFile`). Worker clients get the same wiring named `'[Worker #id]'` (`workerclient.ts:167`).
- `client.webhooks.writeMessage(webhookId, token, payload)` — `src/common/shorters/webhook.ts:87`; payload is `{ body, ...rest }`. Returns `WebhookMessageStructure` (or `null` when `wait` is false).

Behavior: `rawLog` returns early if `!this.active` or `level < this.level`. File writes happen in `__write` when instance `saveOnFile` is true, OR static `saveOnFile === 'all'`, OR the static array includes the logger `name`; the written line is color-stripped (`stripColor`).

## Code Examples (verified)

### 1. Override the default format

The callback may return `undefined` to drop a line entirely:

```ts
import { Logger } from 'seyfert';

const restore = Logger.customize((logger, level, args) => {
    const timestamp = new Date().toISOString();
    const mem = (process.memoryUsage().heapUsed / 1024 / 1024).toFixed(1);
    return [`[${timestamp}]`, `[${mem}MB]`, `${logger.name ?? ''} >`, ...args];
});
// later: restore(); // reverts to the previous callback (e.g. in tests)
```

### 2. File-based logging

```ts
import { Logger } from 'seyfert';

Logger.saveOnFile = 'all';            // every logger -> file
Logger.saveOnFile = ['[Seyfert]'];    // only loggers whose name matches
Logger.dirname = 'logs';              // default is 'seyfert-logs'
```

### 3. Custom methods via prototype extension

Note the deep import for `LogLevels` — it is not on the `'seyfert'` root:

```ts
import { Logger } from 'seyfert';
import { LogLevels } from 'seyfert/lib/common';

Logger.prototype.success = function (...args: unknown[]) {
    this.rawLog(LogLevels.Info, ...args);
};

declare module 'seyfert' {
    interface Logger {
        success(...args: unknown[]): void;
    }
}
```

Use the custom method (`botReady` is a Seyfert custom event; signature verified):

```ts
import { createEvent } from 'seyfert';

export default createEvent({
    data: { name: 'botReady', once: true },
    run(user, client) {
        client.logger.info(`${user.username} is online!`);
    },
});
```

### 4. Configure the client logger at construction (v5)

In v5 you set the level/name/file flags directly when building the `Client` — no need to mutate `client.logger` after the fact. Set `logLevel` to surface `Debug` lines:

```ts
import { Client } from 'seyfert';
import { LogLevels } from 'seyfert/lib/common';

const client = new Client({
    logger: {
        name: '[MyBot]',
        logLevel: process.env.NODE_ENV === 'production' ? LogLevels.Info : LogLevels.Debug,
        active: true,
        saveOnFile: process.env.NODE_ENV === 'production',
    },
});
```

The same `logger` option is honored by `WorkerClient` and `HttpClient`.

### 5. Toggle verbosity at runtime

`level` and `active` are live accessors — flip them from a command, a SIGHUP handler, etc.:

```ts
import { LogLevels } from 'seyfert/lib/common';

// Crank up detail while debugging an incident:
client.logger.level = LogLevels.Debug;
client.logger.debug('cache snapshot', client.cache.guilds?.count('*'));

// Silence a noisy subsystem temporarily:
client.logger.active = false;
```

### 6. A dedicated subsystem logger

Most app logging should reuse `client.logger`, but a standalone module (worker pool, scheduler, payment gateway) can own a named instance:

```ts
import { Logger } from 'seyfert';
import { LogLevels } from 'seyfert/lib/common';

export const dbLogger = new Logger({
    name: '[DB]',
    logLevel: LogLevels.Info,
    saveOnFile: true, // this instance writes to file regardless of the static flag
});

dbLogger.info('pool connected', { size: 10 });
dbLogger.error('query failed', err);
```

### 7. Chain the existing customizer instead of clobbering it

`getCustomizer()` returns whatever callback is currently installed, so plugins/libraries can layer on top without losing the default format:

```ts
import { Logger } from 'seyfert';

const previous = Logger.getCustomizer();
Logger.customize((logger, level, args) => {
    // forward to whatever was there before, then append your own tag
    const base = previous(logger, level, args);
    if (!base) return; // honor a prior decision to drop the line
    return [...base, '#prod'];
});
```

### 8. Mirror REST traffic into the logger (v5 `client.rest.observe`)

v5 lets app code register stacking REST observers (`api/api.ts:119`); a great place to log requests/failures. Payload fields are `client`, `method`, `url`, `request`, plus `response`/`error` depending on the hook (`api/shared.ts:44`):

```ts
const dispose = client.rest.observe({
    onSuccess({ method, url, response }) {
        client.logger.debug(`REST ${method} ${url} -> ${response.status}`);
    },
    onFail({ method, url, error }) {
        client.logger.error(`REST ${method} ${url} failed`, error);
    },
    onRatelimit({ method, url }) {
        client.logger.warn(`REST ratelimited: ${method} ${url}`);
    },
});
// dispose(); // unsubscribe when you no longer need it
```

### 9. Webhook error reporting

`Embed.setColor` accepts a number via `ColorResolvable`:

```ts
import { Embed, type Client } from 'seyfert';

const WEBHOOK_ID = process.env.LOG_WEBHOOK_ID!;
const WEBHOOK_TOKEN = process.env.LOG_WEBHOOK_TOKEN!;

async function reportError(client: Client, error: Error, context?: string) {
    const embed = new Embed()
        .setTitle('Error Report')
        .setDescription(`\`\`\`\n${error.stack ?? error.message}\n\`\`\``)
        .setColor(0xff0000)
        .setTimestamp();
    if (context) embed.addFields({ name: 'Context', value: context });

    await client.webhooks.writeMessage(WEBHOOK_ID, WEBHOOK_TOKEN, {
        body: { embeds: [embed] },
    });
}
```

### 10. Wire reporting into a command error hook

`onRunError(context, error)` verified in `src/commands/applications/chat.ts:353`:

```ts
import { Command, type CommandContext } from 'seyfert';

export default class MyCommand extends Command {
    async onRunError(ctx: CommandContext, error: unknown) {
        ctx.client.logger.error(`/${ctx.fullCommandName} threw`, error);
        await ctx.editOrReply({ content: 'Something went wrong!' });
        if (error instanceof Error) {
            await reportError(ctx.client, error, `Command: /${ctx.fullCommandName}`);
        }
    }
}
```

## Common patterns / gotchas

- **Reuse `client.logger`.** Inside commands/events/components reach for `ctx.client.logger.{debug,info,warn,error,fatal}` — do not `new Logger(...)` for general app logging; the client already owns one named `'[Seyfert]'`.
- **`LogLevels` needs the deep import.** It is exported from `seyfert/lib/common`, not the `'seyfert'` root. Same for `LoggerOptions`, `CustomizeLoggerCallback`, `AssignFilenameCallback`. `Logger` itself IS on the root.
- **Levels gate output, not just formatting.** A log is dropped when `level < logger.level` or `active === false`. To see `Debug` you must lower `logLevel` (via the client `logger` option or `logger.level = LogLevels.Debug`).
- **Static vs instance file logging.** `Logger.saveOnFile`, `Logger.dirname`, `Logger.customizeFilename`, `Logger.clearLogs` are STATIC (process-wide). Per-instance file output is the boolean `logger.saveOnFile` / the `saveOnFile` constructor option — either path triggers a write.
- **`customize` returns a disposer.** Keep it if you ever need to revert (tests, hot-reload). Returning `undefined` from the callback suppresses that line — useful for filtering noisy levels.
- **`customize` is last-write-wins, global.** It replaces the single global callback for ALL loggers. To extend rather than replace, snapshot `getCustomizer()` and call it inside your new callback (see example 7).
- **Color is stripped only in files.** Console output keeps ANSI colors from `colorFunctions`; file writes go through `stripColor`.
- **No empty `new Logger()`.** The constructor requires an options object; pass at least `{}` is NOT valid — give it a `name` (TS requires the argument). For app subsystems always set `name`.
- **`clearLogs()` only touches `Logger.dirname`.** It reads that directory under `process.cwd()`, closes any open streams, and unlinks files — safe to call on a schedule for log rotation.

## Doc vs Source Corrections

- Docs show `Logger.dirname = 'logs'` as the value -> src default is `'seyfert-logs'` (`logger.ts:33`); the example merely reassigns it, which is valid.
- Doc import `import { LogLevels } from 'seyfert/lib/common'` is correct and required: `LogLevels` is NOT in the root `'seyfert'` named exports (`index.ts:29` exports `Logger` only); the root barrel forwards selected names from `./common`.
- Docs omit that `Logger.customize` RETURNS a disposer `() => void` (`logger.ts:74`).
- Docs omit `Logger.getCustomizer()`, `Logger.customizeFilename()`, and `Logger.clearLogs()`.
- Docs omit that the customize callback may return `undefined` to suppress a log line (`rawLog`: `if (!log) return`, `logger.ts:186`).
- Docs omit the v5 client `logger` option and that it is now honored on worker clients (`base.ts:1300`, `workerclient.ts:167`).

## Source Anchors

- `src/common/it/logger.ts` — `Logger` class, `LogLevels`, `LoggerOptions`, callbacks, file logging, `formatMemoryUsage`.
- `src/common/index.ts:7` — barrel exporting `Logger`, `LogLevels`, `LoggerOptions`, `CustomizeLoggerCallback`, `AssignFilenameCallback`.
- `src/index.ts:29` — root exports (`Logger` present; `LogLevels` absent → needs deep import).
- `src/client/base.ts:178/281/1300` — `client.logger` instance (`'[Seyfert]'`), `configureLogger`, `logger` client option.
- `src/client/workerclient.ts:167` — worker client logger wiring.
- `src/api/api.ts:119` + `src/api/shared.ts:44` — `client.rest.observe(...)` and observer payload shapes.
- `src/common/shorters/webhook.ts:87` — `writeMessage` overloads/signature.
- `src/builders/Embed.ts` + `src/common/types/resolvables.ts` — `setColor`/`setTimestamp`/`addFields`, `ColorResolvable` (`number` allowed).
- `src/commands/applications/chat.ts:353` — `onRunError` signature.

## Agent Guidance

- Reach for `client.logger.{debug,info,warn,error,fatal}` inside commands/events; do not construct a new `Logger` for app logging — the client already owns one. Use a named `new Logger({ name })` only for isolated subsystems.
- Configure level/name/file flags via `new Client({ logger: { ... } })` (v5) rather than mutating after start; the same option flows to worker clients.
- To change format globally, call `Logger.customize` once at startup; keep the returned disposer if you need to revert. Chain via `Logger.getCustomizer()` to preserve existing behavior.
- `Logger.saveOnFile`, `Logger.dirname`, `Logger.customizeFilename`, and `Logger.clearLogs` are STATIC (process-wide); per-instance file logging is the boolean `logger.saveOnFile` / the `saveOnFile` constructor option.
- Always import `LogLevels` (and `LoggerOptions`, callback types) from `seyfert/lib/common` — they are not on the root entrypoint.
- Logs below `logger.level` or when `active === false` are silently dropped; set the client `logger.logLevel` option to surface `Debug`.
- Webhook reporting and REST-traffic logging are just core API usage (`client.webhooks.writeMessage` + `Embed`; `client.rest.observe`); no external package required.
