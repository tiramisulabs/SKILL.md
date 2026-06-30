# Gateway Intents

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/tips/intents
Coverage reference: setup-runtime.md
Verification status: Source-verified (seyfert-core, branch more-qol)

## Page Summary

Gateway intents declare which events Discord delivers to your bot over the websocket gateway. You normally set them in `seyfert.config.mjs` via `config.bot({ intents })`, where `intents` accepts an array of intent names, a single bitfield number, or an array of numbers. Seyfert resolves the input to a single numeric bitfield with `resolveGatewayIntents` at config-load time. The `GatewayIntentBits` enum exposes two convenience members — `NonPrivilaged` and `OnlyPrivilaged` (both spelled without the "e", preserved exactly from source). `GuildMembers`, `GuildPresences`, and `MessageContent` are privileged and must also be enabled in the Discord Developer Portal.

In v5 you can also pass intents inline at startup via `client.start({ connection: { intents } })`, and that accepts the same `GatewayIntentInput` (names, number, or `number[]`), overriding the config value. Plugins may additionally contribute base intents, and the cache respects the resolved intents when deciding what to store.

## Key APIs (verified)

- `enum GatewayIntentBits` — `src/types/utils/index.ts:306`. Re-exported from `'seyfert'` via `export * from './types'`. Members (all `1 << n`):
  - `Guilds = 1`, `GuildMembers = 2`, `GuildModeration = 4`, `GuildExpressions = 8`, `GuildIntegrations = 16`, `GuildWebhooks = 32`, `GuildInvites = 64`, `GuildVoiceStates = 128`, `GuildPresences = 256`, `GuildMessages = 512`, `GuildMessageReactions = 1024`, `GuildMessageTyping = 2048`, `DirectMessages = 4096`, `DirectMessageReactions = 8192`, `DirectMessageTyping = 16384`, `MessageContent = 32768`, `GuildScheduledEvents = 65536`, `AutoModerationConfiguration = 1<<20`, `AutoModerationExecution = 1<<21`, `GuildMessagePolls = 1<<24`, `DirectMessagePolls = 1<<25`.
  - `NonPrivilaged = 53575421` — all non-privileged intents OR'd together. EXACT spelling (no "e").
  - `OnlyPrivilaged = 33026` — `GuildMembers | GuildPresences | MessageContent` (2 + 256 + 32768). EXACT spelling (no "e").
- `type IntentStrings = (keyof typeof GatewayIntentBits)[]` — `src/common/types/util.ts:37`. Note: this union INCLUDES the helper names `'NonPrivilaged'` and `'OnlyPrivilaged'`, so `intents: ['NonPrivilaged']` type-checks and resolves to `53575421`.
- `type GatewayIntentInput = number | IntentStrings | number[]` — `src/client/intents.ts:4`. NOT re-exported from the root barrel; needs deep import `seyfert/lib/client/intents` if you reference the type directly.
- `function resolveGatewayIntents(intents?: GatewayIntentInput): number` — `src/client/intents.ts:6`. If given a `number`, returns it as-is; otherwise reduces the array with bitwise OR (looking up string names in `GatewayIntentBits`); `undefined` resolves to `0`. NOT root-exported.
- `config.bot(data: RuntimeConfig)` — `config` exported from `'seyfert'`. Eagerly resolves `data.intents` through `resolveGatewayIntents`, storing a numeric bitfield on the returned config (`base.ts:1234`).
- `type RuntimeConfig` — `src/client/base.ts:1384`: `OmitInsert<InternalRuntimeConfig, 'intents', { intents?: GatewayIntentInput }>` — i.e. the public `config.bot(...)` shape; its `intents` field is optional `GatewayIntentInput`. Internally stored as `intents: number` (`InternalRuntimeConfig`, `base.ts:1383`). `config.http(...)` configs OMIT `intents` entirely (`InternalRuntimeConfigHTTP`, `base.ts:1375`).
- `StartOptions.connection` — `src/client/base.ts:1341`: `{ intents: GatewayIntentInput }`. Passed via `client.start({ connection: { intents } })`; overrides the config intents at runtime.

## Code Examples (verified)

Intent name array (typical gateway bot) — the canonical form:

```ts
// seyfert.config.mjs
import { config } from 'seyfert';

export default config.bot({
    token: process.env.BOT_TOKEN ?? '',
    intents: ['Guilds', 'GuildMessages', 'MessageContent', 'GuildVoiceStates'],
    locations: {
        base: 'src',
        commands: 'commands',
        events: 'events',
    },
});
```

All non-privileged intents via the enum (preferred over a hard-coded number):

```ts
import { config, GatewayIntentBits } from 'seyfert';

export default config.bot({
    token: process.env.BOT_TOKEN ?? '',
    intents: GatewayIntentBits.NonPrivilaged, // a single number is accepted
    locations: { base: 'src', commands: 'commands' },
});
```

Everything (non-privileged + privileged) by OR-ing the two helpers:

```ts
import { config, GatewayIntentBits } from 'seyfert';

export default config.bot({
    token: process.env.BOT_TOKEN ?? '',
    // NonPrivilaged | OnlyPrivilaged = truly all intents (requires the
    // privileged toggles enabled in the Developer Portal).
    intents: GatewayIntentBits.NonPrivilaged | GatewayIntentBits.OnlyPrivilaged,
    locations: { base: 'src', commands: 'commands' },
});
```

Mixed array (names and numbers are both allowed by `GatewayIntentInput`):

```ts
import { config } from 'seyfert';

export default config.bot({
    token: process.env.BOT_TOKEN ?? '',
    intents: ['Guilds', 'GuildVoiceStates'],
    locations: { base: 'src', commands: 'commands' },
});
```

Inline intents at startup (v5) — `start({ connection })` overrides the config value:

```ts
// src/index.ts
import { Client } from 'seyfert';

const client = new Client();

// Strings work inline now — no need to import GatewayIntentBits here.
// This overrides whatever `intents` is in seyfert.config.mjs.
await client.start({ connection: { intents: ['Guilds', 'GuildMessages'] } });
```

Common presets (all valid `GatewayIntentBits` names):

```ts
intents: ['Guilds']                                                       // slash-only
intents: ['Guilds', 'GuildMessages', 'MessageContent']                    // prefix commands
intents: ['Guilds', 'GuildMessages', 'GuildMembers', 'GuildModeration']   // moderation
intents: ['Guilds', 'GuildVoiceStates']                                   // music
```

## Recipes / Common patterns

### Reading message content for prefix commands

`message.content` arrives EMPTY unless `MessageContent` (privileged) is enabled — both in the array AND in the Developer Portal. The classic "my prefix bot sees blank messages" bug:

```ts
// seyfert.config.mjs — prefix commands need all three
import { config } from 'seyfert';

export default config.bot({
    token: process.env.BOT_TOKEN ?? '',
    // 'GuildMessages' delivers the messageCreate event;
    // 'MessageContent' (privileged) fills in message.content.
    intents: ['Guilds', 'GuildMessages', 'MessageContent'],
    locations: { base: 'src', commands: 'commands', events: 'events' },
});
```

The bot's own messages and messages where it is mentioned/replied-to keep `content` even without `MessageContent` — but to read arbitrary user messages you must enable it.

### Music bot — voice states

Joining/leaving voice and tracking who is in a channel requires `GuildVoiceStates`. Without it `voiceStateUpdate` never fires and `member.voice()` resolves to nothing cached:

```ts
intents: ['Guilds', 'GuildVoiceStates']
```

### Picking intents per environment

Because `connection.intents` overrides the config, you can keep a lean default in `seyfert.config.mjs` and widen it only where needed (e.g. a debug/dev run):

```ts
import { Client, GatewayIntentBits } from 'seyfert';

const client = new Client();
const intents = process.env.NODE_ENV === 'development'
    ? GatewayIntentBits.NonPrivilaged                              // everything non-privileged
    : GatewayIntentBits.Guilds | GatewayIntentBits.GuildMessages; // minimal in prod

await client.start({ connection: { intents } });
```

### The cache respects intents

The `Cache` is constructed with the resolved intents and gates what it stores via `hasIntent(...)` (`src/cache/index.ts:225`). If you do not request `Guilds`, guild/channel/role data is not cached; without `GuildMembers`, member caching is limited; etc. So missing cache entries are often a missing-intent problem, not a cache-config problem.

## Doc vs Source Corrections

- Docs example uses `intents: 32767, // All intents` -> SOURCE: 32767 is only bits 0-14; it is NOT all intents (it omits `MessageContent` at bit 15, `GuildScheduledEvents`, the AutoModeration and Polls bits). Prefer `GatewayIntentBits.NonPrivilaged` (53575421), or `NonPrivilaged | OnlyPrivilaged` for truly everything (`src/types/utils/index.ts:306`).
- Docs example sets `locations.base: 'dist'`; not wrong, but `'src'` is the more common layout when running source directly. `base` is just the compiled/source root — either is accepted.
- Docs only present the string-array and single-number forms -> SOURCE: `GatewayIntentInput` also accepts a `number[]`, and arrays may mix names and numbers (`src/client/intents.ts:4`).
- Docs don't mention inline intents -> SOURCE (v5): `client.start({ connection: { intents } })` accepts `GatewayIntentInput` and overrides the config value (`base.ts:1341`, `client.ts:151`).
- Docs table is a subset -> SOURCE additionally defines: `GuildExpressions`, `GuildIntegrations`, `GuildWebhooks`, `GuildInvites`, `GuildMessageTyping`, `DirectMessageReactions`, `DirectMessageTyping`, `AutoModerationConfiguration`, `AutoModerationExecution`, `GuildMessagePolls`, `DirectMessagePolls`, plus the `NonPrivilaged`/`OnlyPrivilaged` helper members (`src/types/utils/index.ts:306`).

## Source Anchors

- `src/client/intents.ts:4,6` — `GatewayIntentInput` type, `resolveGatewayIntents` function.
- `src/types/utils/index.ts:306-330` — `GatewayIntentBits` enum (incl. `NonPrivilaged = 53575421`, `OnlyPrivilaged = 33026`).
- `src/common/types/util.ts:37` — `IntentStrings`.
- `src/client/base.ts:1234` — config load resolves `intents` via `resolveGatewayIntents`.
- `src/client/base.ts:1336-1384` — `StartOptions.connection.intents` (`GatewayIntentInput`), `RuntimeConfig`/`InternalRuntimeConfig`/`*HTTP` (HTTP omits `intents`).
- `src/client/base.ts:458` — `resolvePluginGatewayIntents` (plugins contribute base intents).
- `src/client/client.ts:145-193` — `start()` resolves `connection.intents ?? configIntents`, passes the number to `ShardManager`.
- `src/cache/index.ts:157,225` — `Cache.intents` and `hasIntent(...)` gate caching by intent.
- `src/index.ts` — `config.bot` / `config.http`; `export * from './types'` re-exports the enum.

## Agent Guidance

- Set intents in `seyfert.config.mjs` inside `config.bot({ ... })`. They are resolved to a number at config-load time, so passing names, a number, or `number[]` all work identically downstream.
- Import the enum from the root: `import { GatewayIntentBits } from 'seyfert'`. Watch the misspellings — it is `NonPrivilaged` / `OnlyPrivilaged` (no "e"); `NonPrivileged` will NOT compile.
- For the full non-privileged set use `GatewayIntentBits.NonPrivilaged`; for absolutely everything use `NonPrivilaged | OnlyPrivilaged`. Do NOT assume `32767` means "all intents" (it omits bit 15 `MessageContent` and everything above it).
- `resolveGatewayIntents` and the `GatewayIntentInput` type are internal helpers — not on the root barrel. App code rarely needs them, since `config.bot` (and `client.start`) call the resolver for you. Deep-import `seyfert/lib/client/intents` only if you genuinely need them.
- Privileged intents (`GuildMembers`, `GuildPresences`, `MessageContent` = `OnlyPrivilaged`) require enabling toggles in the Discord Developer Portal (Bot > Privileged Gateway Intents) or the gateway connection is rejected. `MessageContent` is required to read `message.content` for prefix commands.
- Runtime override: `client.start({ connection: { intents } })` takes the same `GatewayIntentInput` and wins over the config value. Useful for per-environment intent sets.
- HTTP-only bots (`config.http`) do not use intents at all — there is no gateway connection (`RuntimeConfigHTTP` omits the field).
- If cache entries are missing, check intents first: the cache only stores what your intents permit (`hasIntent`).
