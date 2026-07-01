# i18n, Cache, Structures, and Recipes

Source-verified against the target project installed `seyfert` package or provided Seyfert source. Import policy: every symbol below is a **root `'seyfert'` export** UNLESS explicitly flagged "deep import" (`seyfert/lib/...`) or "external" (separate npm package). v5 module augmentation lives on **one** interface: `declare module 'seyfert' { interface SeyfertRegistry { client; middlewares; langs; plugins } }` — `UsingClient`/`RegisteredMiddlewares`/`DefaultLocale`/`RegisteredPlugins` are all derived. `ParseMiddlewares` is gone (use bare `typeof middlewares`).

---

## i18n (langs)

Lang modules live in `locations.langs`. Each file's name minus extension IS the locale code (`en-US.ts` → `en-US`). Each module **default-exports** an object whose leaves are `string | nested object | array | (args) => string`. The loader (`LangsHandler.onFile`, `src/langs/handler.ts`) prefers the default export, also accepts a module with exactly one named object export (warns), and skips ambiguous/invalid modules (warns) — no more silent failures.

### Full setup

```ts
// seyfert.config.mjs — point `langs` at your locale dir
// @ts-check
import { config } from 'seyfert';
export default config.bot({
  token: process.env.BOT_TOKEN ?? '',
  intents: ['Guilds'],                       // v5: inline string intents
  locations: { base: 'dist', commands: 'commands', events: 'events', langs: 'languages' },
});
```

```ts
// languages/en-US.ts — the canonical shape (use this one as the type source)
export default {
  hello: 'Each value is a translation',
  foo: {
    bar: 'Nest objects freely',
    ping: ({ ping }: { ping: number }) => `The ping is ${ping}`,
  },
  welcome: (user: string, count: number) => `Welcome ${user}, member #${count}`,
  rules: ['Be nice', 'No spam'].join('\n'),
  // keys reused by @LocalesT / option locales for METADATA localization
  commands: { ping: { name: 'ping', description: 'Show the ping' } },
};
```

```ts
// languages/es-ES.ts — `satisfies` keeps secondary files honest at compile time
import type enUS from './en-US';
export default {
  hello: 'Hola',
  foo: { bar: 'Anida objetos', ping: ({ ping }) => `El ping es ${ping}` },
  welcome: (user, count) => `Bienvenido ${user}, miembro #${count}`,
  rules: ['Sé amable', 'No spam'].join('\n'),
  commands: { ping: { name: 'ping', description: 'Muestra la latencia' } },
} satisfies typeof enUS;
```

```ts
// index.ts — register the shape (enables typed ctx.t + typed dot-paths) and set defaults
import { Client, type ParseClient, type ParseLocales } from 'seyfert';
import type enUS from './languages/en-US';

const client = new Client();
client.setServices({
  langs: {
    default: 'en-US',          // -> langs.defaultLang (fallback locale); ALWAYS set one
    preferGuildLocale: true,   // ctx.t resolves guild locale before the user's
    aliases: { 'en-US': ['en-GB'], 'es-ES': ['es-419'] }, // serve one file to many codes
  },
});
client.start();

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    langs: ParseLocales<typeof enUS>;  // ParseLocales<T> = T (identity, router.ts:62)
  }
}
```

### Runtime access — `ctx.t` / `client.t(locale)`

`ctx.t` returns the `SeyfertLocale` proxy bound to the resolved locale; every leaf has `.get(locale?)`. **Call function leaves first, THEN `.get()`.**

```ts
ctx.t.hello.get();                       // string in the resolved locale
ctx.t.foo.ping({ ping: 12 }).get();      // function leaf: call, then .get()
ctx.t.welcome('Alice', 42).get();        // multi-arg function leaf
ctx.t.rules.get();                       // array/string leaf
ctx.t.hello.get('es-ES');                // force a specific locale
const t = ctx.t.get('es-ES');            // grab a resolved object: leaves are plain -> t.foo.ping({ ping })
```

- `SeyfertLocale` (`router.ts:60`) = `__InternalParseLocale<DefaultLocale> & { get(locale?): DefaultLocale }`. Kept a **named alias** (commit 7411731) so inferred `ctx.t` references `DefaultLocale` instead of collapsing to `{}`.
- `DefaultLocale` (`shared.ts:26`) = registered langs shape, or `{}` if unregistered (then dot-paths/`ctx.t` are untyped — register first).
- Resolution (`chatcontext.ts:72`, mirrored in menu/entry/component/modal): `preferGuildLocale ? (guildLocale ?? locale ?? defaultLang ?? 'en-US') : (locale ?? defaultLang ?? 'en-US')`. `LangRouter.get` internally falls back to `defaultLang` on lookup failure (try/catch, `router.ts:23`) — so a `default` prevents `UNDEFINED_LOCALE`/`INTERNAL_ERROR` throws.
- Outside a context (events/services), pick a locale yourself: `client.t(client.langs.defaultLang ?? 'en-US').welcome(name, n).get()`.
- `client.langs` is a `LangsHandler`: `get(locale)`, `getLocale(alias)` (alias→canonical), `getKey(lang, dotPath)` (returns the value only when it is a `string`), `reload(lang)`, `reloadAll(stopIfFail=true)`.

### Localizing command METADATA (load-time, not runtime strings)

Two families — **dot-path** (resolve `FlatObjectKeys<DefaultLocale>` against lang files) and **raw-tuple** (inline `[LocaleString, string][]`). Don't mix shapes on one decorator.

```ts
import { Command, Declare, LocalesT, Locales, GroupsT, Group, defineGroups, createStringOption } from 'seyfert';

@Declare({ name: 'ping', description: 'Show the ping' })
@LocalesT('commands.ping.name', 'commands.ping.description') // dot-paths
export default class Ping extends Command {}

@Declare({ name: 'ping', description: 'Pong' })
@Locales({ name: [['es-ES', 'latencia']], description: [['es-ES', 'Muestra la latencia']] }) // raw tuples
class PingRaw extends Command {}

// Group localization — only defaultDescription is required
const groups = defineGroups({ config: { defaultDescription: 'Configuration', name: 'cmd.config.name' } });
@Declare({ name: 'admin', description: 'Admin tools' })
@GroupsT(groups)
@Group(groups, 'config') // 'config' is type-checked against defineGroups keys
class Admin extends Command {}

// Per-option locales = OBJECT of dot-paths; per-CHOICE locales = a single dot-path STRING
const options = {
  topic: createStringOption({           // option keys MUST be lowercase (v5 compile-time)
    description: 'Pick a topic',
    locales: { name: 'opt.topic.name', description: 'opt.topic.desc' },
    choices: [{ name: 'General', value: 'general', locales: 'choice.general' }] as const, // readonly -> as const
  }),
};
```

**Gotchas:** files MUST be named to Discord locale codes (or wired via `aliases`). Metadata resolution skips (logs a warning) when a dot-path lands on a non-string — it never throws. `reload`/`reloadAll` throw `SeyfertError('RELOAD_NOT_SUPPORTED')` under Cloudflare Workers. Langs config is on `setServices`, NOT the `Client` constructor (`BaseClientOptions` has no `langs` key).

---

## Cache

Global, in-memory-by-default, split into per-entity **resources**: `users, guilds, members, voiceStates, overwrites, roles, emojis, channels, stickers, presences, stageInstances, messages, bans`. Each is **optional/`undefined` when disabled** — always null-check (`client.cache.users?.…`).

```ts
import { Client, ChannelType, LimitedMemoryAdapter } from 'seyfert';
const client = new Client();

// Disable resources: object / boolean (also kills onPacket) / predicate
client.setServices({ cache: { disabledCache: { bans: true } } });
client.setServices({ cache: { disabledCache: { onPacket: true } } });   // stop ALL gateway writes
client.setServices({ cache: { disabledCache: true } });                 // disable everything
client.setServices({ cache: { disabledCache: t => t === 'presences' } });

// Filter what gets stored. Guild signature: (data, id, guild_id, from); BaseResource.filter: (data, id, from)
client.cache.channels!.filter = (c, id, guildId, from) =>
  ![ChannelType.DM, ChannelType.GroupDM].includes(c.type);

// Cap RAM + TTL. Per-resource buckets are snake_case singular; `default` covers the rest.
client.setServices({ cache: { adapter: new LimitedMemoryAdapter({
  default: { limit: 1000 },
  message: { limit: 100, expire: 60_000 }, // 100 messages, 60s TTL
  presence: { limit: 0 },                  // effectively don't keep presences
}) } });
```

### Reading (sync with MemoryAdapter)

```ts
import { Command, type CommandContext } from 'seyfert';
export default class extends Command {
  async run(ctx: CommandContext) {
    const me = ctx.client.cache.users?.get(ctx.client.botId);          // sync, no await
    if (ctx.guildId) {
      const member = ctx.client.cache.members?.get(ctx.author.id, ctx.guildId);
      const count = ctx.client.cache.members?.count(ctx.guildId) ?? 0;
      await ctx.write({ content: `${member?.user.username} — ${count} cached` });
    }
    // Typed bulkGet: shape is inferred from the key tuples. Guild-based keys (members/voiceStates/bans)
    // and messages need their guild/channel id 3rd. bulkGet is always async.
    const { users, members } = await ctx.client.cache.bulkGet([
      ['users', ctx.author.id],
      ['members', ctx.author.id, ctx.guildId!],
      ['bans', ctx.author.id, ctx.guildId!],   // bans is GUILD-BASED in v5
    ]);
  }
}
```

- `enum CacheFrom { Gateway = 1, Rest, Test }`. **Resource `set`/`patch` take `CacheFrom` FIRST** (`set(from, id, data)` / guild-based `set(from, id, guild, data)`) — the #1 stale-code break.
- Cross-guild: `members.values('*')` / `count('*')` span all guilds; pass a guild id to scope. `GuildRelatedResource.flush(guild='*')` defaults to `'*'`; `GuildBasedResource.flush(guild)` has NO default — pass `'*'` to clear all.
- Reads are SYNC with `MemoryAdapter`/`LimitedMemoryAdapter`. They return Promises only after `declare module 'seyfert' { interface InternalOptions { asyncCache: true } }` (required for Redis / `WorkerAdapter`). Use `.raw()`/`.valuesRaw()`/`.bulkRaw()` for the plain API payload.
- Adapters (root): `MemoryAdapter` (default, `isAsync=false`, `relationships: Map<string, Set<string>>`), `LimitedMemoryAdapter`, `WorkerAdapter` (`isAsync=true`, sharded workers). Redis = external (`tiramisulabs/extra`); must satisfy the `Adapter` interface (`src/cache/adapters/types.ts`, note `bulkAddToRelationShip` capital S).
- `client.cache.testAdapter()` **throws `SeyfertError`** (not boolean) on the first mismatch; needs `users`+`members`+`channels`+`overwrites` enabled.
- `LimitedCollection<K,V>` (`src/collection.ts`) is a standalone TTL/limit map: `set(key, val, ttlMs?)`, defaults `limit=Infinity, expire=0 (never), resetOnDemand=false`; `values()/entries()` yield plain data, `rawValues()/rawEntries()` carry metadata; throws on a `NaN` limit.

### Custom resource (subclass + Cache augmentation)

`Cache` is a runtime class, not a `SeyfertRegistry` key. For manually attached resources, augment `interface Cache` so TypeScript can see the property; for plugin-owned resources prefer `api.cache.resource(...)` and cast or augment where application code reads the dynamic resource.

```ts
import { BaseResource, GuildBasedResource, CacheFrom, Client, type ParseClient } from 'seyfert';

// Global (non-guild) resource — note set(from, id, data)
interface CooldownData { id: string; lastDrip: number }
export class CooldownResource extends BaseResource<CooldownData, Omit<CooldownData, 'lastDrip'> & Partial<CooldownData>> {
  namespace = 'cooldowns';
  override set(from: CacheFrom, id: string, data: Omit<CooldownData, 'lastDrip'> & Partial<CooldownData>) {
    return super.set(from, id, { ...data, lastDrip: data.lastDrip ?? Date.now() });
  }
}

// Guild-scoped resource — set(from, id, guild, data); query with '*' | guildId
interface WarnData { id: string; guild_id: string; count: number }
export class WarnsResource extends GuildBasedResource<WarnData> { namespace = 'warns'; }

const client = new Client();
client.cache.cooldown = new CooldownResource(client.cache, client);
client.cache.warns = new WarnsResource(client.cache, client);
client.cache.cooldown.set(CacheFrom.Rest, 'user-1', { id: 'user-1' }); // write/read it yourself
client.cache.warns.set(CacheFrom.Rest, 'user-1', 'guild-1', { count: 1 });

declare module 'seyfert' {
  interface Cache { cooldown: CooldownResource; warns: WarnsResource }
  interface SeyfertRegistry { client: ParseClient<Client> }
}
```

Manual `client.cache.x = ...` works but is cleared when `buildCache` re-runs (e.g. on `setServices({ cache })`). Prefer a plugin's `api.cache.resource(...)` contribution — it survives rebuilds and can add required gateway intents.

---

## Structures & Transformers

Discord payloads are wrapped in typed structure classes. `Transformers` (`src/client/transformers.ts`) is the single factory point.

- `member.roles` getter → `{ keys, list(force?), add(id), remove(id), permissions(force?), sorted(force?), highest(force?) }`. **`roles.keys` always includes `@everyone`** (the guild id) — filter it out for assigned-only.
- `member.voice(mode)` / `member.guild(mode)` (also `channels`/`Emoji`/`GuildRole`/`GuildBan`): `'cache'` → `ReturnCache<X | undefined>` (sync); `'rest' | 'flow'` (default `'flow'` = cache→REST) → `Promise<X>`.
- `member.fetchPermissions(force=false)` → `Promise<PermissionsBitField>`. `perms.has(bits)` applies an **Administrator override**; `strictHas(bits)` is the exact bit (no override); `missings(bits[])` lists missing. `keys()` returns typed permission names. `PermissionsBitField.resolve(...)` (static) throws on unknown/negative/unsafe input; instances also inherit `resolve()` from `BitField`.
- Moderation: `member.timeout(ms, reason?)` is **milliseconds** in v5; `member.ban({ deleteMessageSeconds, reason })` (camelCase, reason in the object). Gate first with `bannable()`/`kickable()`/`moderatable()`/`manageable()` (async, parallel-fetch).
- Channel guards on `BaseChannel` (`this is X`): `isTextable()`, `isGuildTextable()`, `isDM()`, `isThread()`, `isThreadOnly()` (Forum|Media), `isVoice()`, `isTextGuild()`, `isStage()`, `isMedia()`, `isForum()`, `isNews()`, `isCategory()`, `isDirectory()`, `isGuild()`, `isNamed()`, plus multi-type `is(['GuildText', ...])`. Use `isTextable()` before `channel.messages.write(...)`. **There is NO `isSendable()`.**
- `VoiceState` boolean getters: `isMuted`, `isDeafened`, `isCameraOn`, `isStreaming`, `isSuppressed`.

```ts
import { type GuildCommandContext } from 'seyfert';
import { PermissionsBitField } from 'seyfert/lib/structures/extra/Permissions';
declare const ctx: GuildCommandContext;           // member non-undefined, channel() narrowed to guild
declare const target: import('seyfert').GuildMember;

if (!(await target.bannable())) {
  await ctx.editOrReply({ content: 'I cannot ban this member (hierarchy/ownership).' });
} else {
  await target.ban({ deleteMessageSeconds: 7 * 24 * 60 * 60, reason: `by ${ctx.author.tag}` });
  await ctx.editOrReply({ content: `Banned ${target.tag}.` });
}
const perms = await target.fetchPermissions();
const missing = perms.missings(['ManageChannels', 'ManageRoles']);
if (missing.length) { const names = new PermissionsBitField(missing).keys(); /* typed names */ }
```

Custom structure = reassign `Transformers.<Name>` AND augment `CustomStructures` (both required; types/runtime decoupled). Prefer **subclassing** to keep built-ins:

```ts
import { Transformers, GuildMember } from 'seyfert';
class ExtendedMember extends GuildMember {
  get isStaff() { return this.roles.keys.includes('STAFF_ROLE_ID'); }
}
Transformers.GuildMember = (...args) => new ExtendedMember(...args);
declare module 'seyfert' { interface CustomStructures { GuildMember: ExtendedMember } }
```

A plain-object transformer **fully replaces** the shape — keep a `raw()` delegate if you still need API data: `Transformers.User = (client, data) => ({ /* ... */ raw: () => client.users.raw(data.id) })`.

---

## Recipe: Database (plugin pattern)

DB-agnostic. Idiomatic v5 = a `createPlugin` that owns the connection, exposes it via the `client` map (`client.db`) and per-interaction `ctx` helpers.

```ts
import { createPlugin, createMiddleware, Client, definePlugins } from 'seyfert';

const requireAccount = createMiddleware<void>(async ({ context, next, stop }) => {
  if (!(await context.client.db.findUser(context.author.id)))
    return stop('You need an account first. Run /register.'); // stop('reason') = deny; stop() = silent skip
  return next();
});

export const databasePlugin = createPlugin({
  name: 'database',
  client: { db: () => db },                                  // -> client.db, typed everywhere
  ctx: { user: i => async () => db.findUser(i.user.id) },    // factory is SYNC; return an async fn to await
  register(api) { api.middlewares.add('requireAccount', requireAccount, { global: true }); }, // register is SYNC
  async setup(client) { await client.db.connect(); },        // open here, NOT in register
  async teardown(client) { await client.db.disconnect(); },  // close here (runs in reverse order)
});

const plugins = definePlugins(databasePlugin);
declare module 'seyfert' { interface SeyfertRegistry { plugins: typeof plugins } }
const client = new Client({ plugins });
```

```ts
// Using it in a command — ctx.user() resolves the async factory
import { Command, Declare, type CommandContext } from 'seyfert';
@Declare({ name: 'balance', description: 'Check your balance' })
export default class extends Command {
  async run(ctx: CommandContext) {
    const user = await ctx.user();           // factory returned an async fn -> await it
    await ctx.write({ content: `Balance: ${user?.balance ?? 0}` });
  }
}
```

- Lifecycle: `register` (**sync**) → `setup` (async, before the bot handles anything) → `teardown` (async, reverse). A throwing `setup` tears down completed plugins and raises `SeyfertPluginError`/`SeyfertPluginAggregateError`.
- The **`ctx` factory runs synchronously** before `run()` — for async reads it MUST return a function (`i => async () => {...}`), then `await ctx.user()`. Cache the read in the closure to avoid double DB hits.
- For per-bot config, use `createPluginFactory({ defaults, validate?, factory })` → `(options?) => plugin`.
- Resource names are guarded (`assertSafePluginResourceName`) and throw on reserved-name collisions. Under sharding/workers each process gets its own `client.db` — point them at the same server.

---

## Recipe: Logger

`client.logger` is a `Logger` named `'[Seyfert]'`. Use `client.logger.{debug,info,warn,error,fatal}` — don't construct a new one for app logging.

```ts
import { Client, Logger, LogLevels } from 'seyfert';

// Configure at construction (v5; honored on Worker/Http clients too)
const client = new Client({ logger: {
  name: '[MyBot]',
  logLevel: process.env.NODE_ENV === 'production' ? LogLevels.Info : LogLevels.Debug,
  active: true,
  saveOnFile: process.env.NODE_ENV === 'production',
} });

// Global format override -> returns a disposer; returning undefined drops that line
const restore = Logger.customize((logger, level, args) => [`[${new Date().toISOString()}]`, logger.name, ...args]);

// Chain instead of clobber (libraries/plugins): snapshot the current customizer first
const prev = Logger.getCustomizer();
Logger.customize((l, lvl, a) => { const base = prev(l, lvl, a); return base && [...base, '#prod']; });

// Custom level method via prototype extension
Logger.prototype.success = function (...a: unknown[]) { this.rawLog(LogLevels.Info, ...a); };
declare module 'seyfert' { interface Logger { success(...a: unknown[]): void } }
```

- `Logger`, `LogLevels`, `LoggerOptions`, and the logger callback types are all **root exports** of `seyfert`.
- Static (process-wide): `Logger.saveOnFile = 'all' | string[]`, `Logger.dirname` (default `'seyfert-logs'`), `Logger.customizeFilename(cb)`, `Logger.clearLogs()`. Per-instance file logging = the boolean `logger.saveOnFile`. Files are color-stripped; console keeps ANSI.
- Logs below `logger.level` or when `active === false` are silently dropped — lower `logLevel` to see `Debug`.

```ts
// Mirror REST traffic into the logger via stacking observers (v5)
const dispose = client.rest.observe({
  onSuccess({ method, url, response }) { client.logger.debug(`REST ${method} ${url} -> ${response.status}`); },
  onFail({ method, url, error }) { client.logger.error(`REST ${method} ${url} failed`, error); },
  onRatelimit({ url }) { client.logger.warn(`Ratelimited: ${url}`); },
});
// dispose(); // unsubscribe later

// Webhook error reporting is plain core API
import { Embed } from 'seyfert';
async function report(err: Error) {
  const embed = new Embed().setTitle('Error').setDescription(`\`\`\`\n${err.stack}\n\`\`\``).setColor(0xff0000).setTimestamp();
  await client.webhooks.writeMessage(process.env.LOG_WEBHOOK_ID!, process.env.LOG_WEBHOOK_TOKEN!, { body: { embeds: [embed] } });
}
```

---

## Recipe: API access (shorters, proxy, rest)

Three layers, high→low: **shorters** (typed, cached, return structures) → **`client.proxy`** (raw REST tree, snake_case JSON) → **`client.rest.request<T>`** (low-level).

```ts
// Shorters — prefer these
await client.messages.write(channelId, { content: 'hi' });           // MessageStructure
await client.reactions.add(messageId, channelId, '🔥');               // encodes for you, returns void

// Proxy — pass dynamic path parts as CALL args; terminal call is lowercase get/post/put/patch/delete
await client.proxy.channels(channelId).get();                                  // GET /channels/{id}
await client.proxy.channels(id).threads.post({ body: {...}, reason: 'x' });    // no-shorter route; reason -> X-Audit-Log-Reason
await client.proxy.channels(id).messages.get({ query: { limit: 50 } });       // ?limit=50
await client.proxy.channels(id).messages(mid).reactions(encodeURIComponent('🔥'))('@me').put(); // encode yourself

// Typed low-level call for a hand-built URL
import type { RESTGetAPIGatewayBotResult } from 'seyfert/lib/types';
const info = await client.rest.request<RESTGetAPIGatewayBotResult>('GET', '/gateway/bot');

// CDN is a SEPARATE proxy (client.rest.cdn); terminal key `get` returns a URL string. Prefer structure helpers.
const url = user.avatarURL({ size: 256, extension: 'png' });
```

**Gotchas:** `client.proxy.channels(id)` (call) not `.channels.id` (literal segment). Proxy returns raw snake_case JSON — no cache/transform. Raw uploads need `files: RawFile[]` AND a matching `attachments` array in `body`; shorters resolve `AttachmentBuilder`/paths for you. Observers stack and are isolated (plugin observers → app `observe(...)` → legacy callbacks); a throwing observer is caught, never bubbled.

---

## Recipe: Monetization

- `Entitlement` (`src/structures/Entitlement.ts`): camelCase `id, skuId, userId?, guildId?, applicationId, type, startsAt, endsAt, deleted, consumed?`; getters `startsAtTimestamp`/`endsAtTimestamp` (→ `number | null`); `consume()`. Events/interactions deliver **`EntitlementStructure`** (custom-structure-aware), not raw `Entitlement`.
- Events `entitlementCreate` / `entitlementUpdate` (renewal) / `entitlementDelete` (**refund/manual ONLY — never expiry**); handler `(entitlement: EntitlementStructure, client)`.
- `ctx.interaction.entitlements` → `EntitlementStructure[]` (per-request snapshot; no `cache.entitlements`).
- Premium button: `new Button().setSKUId('SKU').setStyle(ButtonStyle.Premium)` — capital `SKU`; OMITS `customId`/`label`/`emoji`; clicking opens Discord checkout (no `ComponentContext`).
- `client.applications`: `listEntitlements(query?)` (snake_case keys), `consumeEntitlement(id)`, `createTestEntitlement(body)`, `deleteTestEntitlement(id)`, `listSKUs()` (raw `APISKU[]`).

```ts
import { Declare, Command, type CommandContext, ActionRow, Button, ButtonStyle } from 'seyfert';
const PRO_SKU = 'PRO_TIER_SKU_ID';

@Declare({ name: 'pro', description: 'Pro feature' })
export class Pro extends Command {
  async run(ctx: CommandContext) {
    const now = Date.now();
    const active = ctx.interaction.entitlements.some(e =>
      e.skuId === PRO_SKU && !e.deleted && (e.endsAtTimestamp === null || e.endsAtTimestamp > now));
    if (!active) {
      const row = new ActionRow<Button>().setComponents([new Button().setSKUId(PRO_SKU).setStyle(ButtonStyle.Premium)]);
      return ctx.editOrReply({ content: 'Subscribe to unlock this.', components: [row] });
    }
    await ctx.editOrReply({ content: 'Welcome, Pro member!' });
  }
}
```

**Gotchas:** gate by `skuId` (+ `endsAtTimestamp`/`deleted`), not just `.length`. For consumables: deliver in your DB first, THEN `entitlement.consume()`. Guild-scoped entitlements have `guildId` but no `userId` — null-check before `client.users.fetch`. Enable monetization + create SKUs in the dev portal (external).

---

## Recipe: Music (EXTERNAL: lavalink client, e.g. `kazagumo`/`shoukaku`)

Seyfert has no built-in audio. Only core glue: the gateway voice `send` callback, a `declare module` to type `client.kazagumo`, and normal commands reading voice states.

```ts
import { Client } from 'seyfert';
const client = new Client();
client.kazagumo = new Kazagumo({
  defaultSearchEngine: 'youtube',
  // ShardManager.send(shardId: NUMBER, payload) — shard id FIRST, hence calculateShardId
  send: (guildId, payload) => client.gateway.send(client.gateway.calculateShardId(guildId), payload),
}, new Connectors.Seyfert(client), nodes);
declare module 'seyfert' { interface Client { kazagumo: Kazagumo } } // augments Client directly (separate from SeyfertRegistry)
```

```ts
// Guild-narrow early so guildId/member/me() are non-undefined; voice('cache') is a sync read
import { MessageFlags, type GuildCommandContext } from 'seyfert';
export async function requirePlayer(ctx: GuildCommandContext) {
  const voice = await ctx.member.voice('cache');
  if (!voice?.channelId) { await ctx.write({ content: 'Join a voice channel first.', flags: MessageFlags.Ephemeral }); return null; }
  const player = ctx.client.kazagumo.players.get(ctx.guildId); // external
  if (!player) { await ctx.write({ content: 'Nothing is playing.', flags: MessageFlags.Ephemeral }); return null; }
  if (player.voiceId && player.voiceId !== voice.channelId) { await ctx.write({ content: 'Join my channel.', flags: MessageFlags.Ephemeral }); return null; }
  return player;
}
```

- Enable the `GuildVoiceStates` intent or `member.voice()` won't resolve. `member.voice()` defaults to `'flow'` (cache→REST) and may reject — prefer `'cache'` in guards.
- `write`/`editOrReply` return `void` unless you pass the response flag `true`. Always `player.destroy()` on `playerEmpty`/empty channel. `Kazagumo`, `createPlayer`, `search`, queue/track APIs are external — version-verify.

---

## Recipe: Yuna (EXTERNAL: `yunaforseyfert`)

Advanced prefix/text-command parser. **v5 ships it as a plugin** — the old `setServices`/`HandleCommand`-subclass path is STALE; do not recommend it.

```ts
import { Client, type ParseClient, definePlugins } from 'seyfert';
import { Yuna } from 'yunaforseyfert';
const plugins = definePlugins(Yuna.plugin({ parser: { syntax: { namedOptions: ['-', '--'] } }, resolver: {} }));
const client = new Client({
  plugins,
  commands: { prefix: async msg => (msg.guildId ? await db.getPrefixes(msg.guildId) : ['!']), reply: () => true },
});
declare module 'seyfert' {
  interface SeyfertRegistry { client: ParseClient<Client<true>>; plugins: typeof plugins }
  interface InternalOptions { withPrefix: true }   // type-level switch for prefix commands
}
```

- **Two independent requirements:** `InternalOptions { withPrefix: true }` (type switch) AND a `commands.prefix` function (runtime). The plugin sets NEITHER.
- `commands.prefix?: (msg: MessageStructure) => Awaitable<string[]>` and `commands.reply?: (ctx) => Awaitable<boolean>` live on the CLIENT, not plugin options. Always return a non-empty array (DMs/uncached guilds).
- A single `Command` class serves slash AND prefix (Yuna parses the text form). `Yuna.plugin`, `@Watch`, `Yuna.watchers`, `client.yuna`/`ctx.yuna` are external — version-verify.

---

## Review Checklist

- Lang shape registered once via `ParseLocales<typeof lang>` under `SeyfertRegistry.langs`; a `default` locale set; files named to Discord codes (or aliased)?
- Runtime strings via `ctx.t.path.get()`; function leaves CALLED then `.get()`? Metadata via `@LocalesT`/`@Locales`/`@GroupsT`/`defineGroups`/option `locales` (object) / choice `locales` (string)?
- Langs/cache config via `client.setServices({...})` (NOT the constructor for langs)?
- Cache: resource access null-checked? `set`/`patch` passing `CacheFrom` FIRST? `disabledCache` form correct (bool/object/predicate)? `bans` keyed with `guildId`? `asyncCache` augmented for Redis/Worker?
- Custom resource: manual `interface Cache` augmentation or access-site cast? Plugin resource registered via `api.cache.resource(...)`? Custom structure: both `Transformers.X` AND `CustomStructures` (subclass to keep built-ins)?
- DB plugin: connection in `setup`/`teardown` (not `register`); async `ctx` helper returns a function?
- Imports from root `'seyfert'` (including `LogLevels`/`LoggerOptions`), except type-only `seyfert/lib/types`? External packages (yuna, kazagumo, redis) version-verified?
- v5 hygiene: lowercase option keys; readonly `choices`/`channel_types` as `as const`; `timeout` in ms; `ban({ deleteMessageSeconds, reason })`; `stop()`/`stop('reason')` not `pass()`; no `isSendable()`; `PermissionsBitField.resolve` throws on bad input?
- `reload`/`reloadAll` avoided under Cloudflare Workers? Monetization gated by `skuId` (+ `endsAtTimestamp`/`deleted`), SKUs configured in the portal?
