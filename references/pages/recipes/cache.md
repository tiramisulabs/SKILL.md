# Cache

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/cache
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (v5 / the authoritative Seyfert source)

## Page Summary

Seyfert's cache is a global, in-memory-by-default storage layer split into per-entity `resources` (channels, users, members, etc.). Each resource can be individually disabled or given a `filter` to decide what gets stored. Storage is pluggable via the `Adapter` interface; built-ins are `MemoryAdapter`, `LimitedMemoryAdapter`, and `WorkerAdapter` (Redis lives in an external package). You can add custom resources by subclassing `BaseResource` / `GuildBasedResource` / `GuildRelatedResource`, and validate custom adapters with `client.cache.testAdapter()`.

Cache is "global": there is no per-guild partition until you read it back, at which point guild-based/related resources take a guild id (or `'*'` for all guilds).

## Key APIs (verified)

All of the following are root exports from `seyfert` (re-exported via `src/index.ts` -> `./cache` and `./collection`).

- `class Cache` — src/cache/index.ts. Fields are optional resources: `users, guilds, members, voiceStates, overwrites, roles, emojis, channels, stickers, presences, stageInstances, messages, bans`. Methods: `flush()`, `bulkGet(keys)`, `bulkSet(keys)`, `bulkPatch(keys)`, `testAdapter()`, `hasIntent(name)` plus `has*Intent`/`hasPresenceUpdates`/`hasDirectMessages`/`hasModerationIntent` getters, and `buildCache(disabledCache, client)` (rebuilds resources). Constructor: `new Cache(intents, adapter, disabledCache, client)` — you normally never call this directly; the Client builds it.
- `enum CacheFrom { Gateway = 1, Rest, Test }` — src/cache/index.ts. A value enum; passed as the FIRST arg to resource `set`/`patch`.
- `type DisabledCache` — src/cache/index.ts: `{ [P in NonGuildBased | GuildBased | GuildRelated | 'onPacket']?: boolean }` (every resource name plus `onPacket`).
- `type BulkGetKey` — src/cache/index.ts. `readonly [type, id]` for non-guild/guild-related resources, or `readonly [type, id, guildId]` for guild-based ones (`members | voiceStates | bans`). `messages` also needs its channel id in the guild slot.
- `type ReturnCache<T>` / `type InferAsyncCache` — src/cache/index.ts. Reads are SYNC with `MemoryAdapter` (`ReturnCache<T>` = `T`). They become `Promise<T>` only when you augment `interface InternalOptions { asyncCache: true }` (do this for Redis / `WorkerAdapter`).
- `interface Adapter` — src/cache/adapters/types.ts. Methods: `isAsync`, `start()`, `scan(query, keys?)`, `get/bulkGet`, `set/bulkSet`, `patch/bulkPatch`, `values(to)`, `keys(to)`, `count(to)`, `remove/bulkRemove`, `flush()`, `contains(to, key)`, `getToRelationship(to)`, `addToRelationship(to, keys)`, `bulkAddToRelationShip(data)` (note capital `S`hip), `removeToRelationship(to, keys)`, `removeRelationship(to)`. All return `Awaitable<...>`.
- `class MemoryAdapter<T>` — src/cache/adapters/default.ts. `isAsync = false`. `relationships` is a `Map<string, Set<string>>` (a `Set`, not array — only matters if you read it from a custom adapter). Constructor takes optional `{ encode, decode }` (identity by default).
- `class LimitedMemoryAdapter<T>` — src/cache/adapters/limited.ts. `isAsync = false`. Constructor takes `LimitedMemoryAdapterOptions<T>` with a `default` and per-resource `{ expire?, limit? }` buckets keyed by snake_case singular: `guild, user, ban, member, voice_state, channel, emoji, presence, role, stage_instance, sticker, overwrite, message`, plus optional `encode`/`decode`. `default.limit` is `Infinity`, `default.expire` is `undefined` (no expiry) unless set.
- `class WorkerAdapter` — src/cache/adapters/workeradapter.ts. `isAsync = true`. Used by sharded worker clients; forwards cache ops over the worker message port. NOT in upstream docs.
- `class BaseResource<T, S>` — base.ts. `namespace='base'`; `filter(data, id, from)`; `get(id)/set(from,id,data)/patch(from,id,data)/remove(id)/keys()/values()/count()/contains(id)/flush()/hashId(id)`.
- `class GuildBasedResource<T, S>` — guild-based.ts (members, voiceStates, bans). `set(from,id,guild,data)`, `get(id,guild)`, `keys/values/count('*' | guildId)`, `hashGuildId(guild,id)`, `parse(data,id,guild_id)`. `flush(guild)` has NO default — pass `'*'` to clear all.
- `class GuildRelatedResource<T, S>` — guild-related.ts (channels, roles, emojis, stickers, presences, stageInstances, overwrites, messages). Similar shape; `parse` strips `permission_overwrites`. `flush(guild = '*')` DOES default to `'*'`.
- `class LimitedCollection<K, V>` — src/collection.ts. TTL/limit map. `set(key, value, customExpire = options.expire)`, `get`, `has`, `delete`, `raw`, `keys/values/entries/rawEntries/rawValues`, `clear`, `size`, `closer`. Options: `{ limit, expire, onDelete?, resetOnDemand }`; defaults `limit=Infinity, expire=0 (never), resetOnDemand=false`. Throws `TypeError` on a `NaN` limit.
- `interface ServicesOptions['cache']` — src/client/base.ts:1390: `{ adapter?: Adapter; disabledCache?: boolean | DisabledCache | ((cacheType) => boolean) }`. Set via `client.setServices({ cache: {...} })`.

## Code Examples (verified)

### Disabling resources

```ts
import { Client } from 'seyfert';

const client = new Client();

client.setServices({ cache: { disabledCache: { bans: true } } });
// stop gateway packets from writing to the cache at all:
client.setServices({ cache: { disabledCache: { onPacket: true } } });
```

`disabledCache` also accepts a boolean (disable EVERYTHING, including `onPacket`) or a predicate:

```ts
client.setServices({ cache: { disabledCache: true } }); // disable all resources
client.setServices({ cache: { disabledCache: type => type === 'presences' } });
```

### Filtering a resource

The `filter` callback for guild-related/based resources is `(data, id, guild_id, from)`; `BaseResource.filter` is `(data, id, from)`. `from` is a `CacheFrom`. Existing 3-arg usage still compiles.

```ts
import { Client, ChannelType } from 'seyfert';

const client = new Client();

client.cache.channels!.filter = (channel, id, guildId, from) => {
  return ![ChannelType.DM, ChannelType.GroupDM].includes(channel.type);
};
```

### Reading from the cache (inside a command)

Resources return Seyfert STRUCTURES (`UserStructure`, `GuildMemberStructure`, …). Use `.raw()` / `.valuesRaw()` / `.bulkRaw()` when you want the plain API object. With the default `MemoryAdapter` these are synchronous; with an async adapter they return Promises (see the gotcha below).

```ts
import { Command, type CommandContext } from 'seyfert';

export default class extends Command {
  async run(ctx: CommandContext) {
    // Sync read with MemoryAdapter — no await needed.
    const me = ctx.client.cache.users?.get(ctx.client.botId);

    if (ctx.guildId) {
      const member = ctx.client.cache.members?.get(ctx.author.id, ctx.guildId);
      const memberCount = ctx.client.cache.members?.count(ctx.guildId) ?? 0;
      await ctx.write({ content: `${member?.user.username} — ${memberCount} cached members` });
    }
  }
}
```

### Typed `bulkGet`

`Cache.bulkGet(...)` infers its result shape from the key tuples. Guild-based keys (`members`, `voiceStates`, `bans`) and `messages` need their guild/channel id as the 3rd tuple element. The result is keyed by resource name; each value is an array. `bulkGet` is always async (returns a Promise).

```ts
const { users, members } = await ctx.client.cache.bulkGet([
  ['users', ctx.author.id],
  ['users', ctx.client.botId],
  ['members', ctx.author.id, ctx.guildId!], // guild-based -> needs guildId
  ['bans', someUserId, ctx.guildId!],        // bans is guild-based in v5
]);
// users: UserStructure[] | undefined ; members: GuildMemberStructure[] | undefined
```

### Cross-guild queries and flushing

Guild-based/related `keys/values/count` accept `'*'` to span every guild.

```ts
const allMembers = ctx.client.cache.members?.values('*');   // every cached member
const totalRoles = ctx.client.cache.roles?.count('*') ?? 0; // across all guilds

// flush(): GuildRelatedResource defaults to '*', GuildBasedResource does NOT.
ctx.client.cache.roles?.flush();        // GuildRelated -> defaults to '*' (all)
ctx.client.cache.members?.flush('*');   // GuildBased   -> '*' required
ctx.client.cache.members?.flush(ctx.guildId!); // or scope to one guild
```

### LimitedMemoryAdapter (cap RAM + TTL)

Per-resource buckets use snake_case singular names; `default` applies to anything not listed.

```ts
import { Client, LimitedMemoryAdapter } from 'seyfert';

const client = new Client();
client.setServices({
  cache: {
    adapter: new LimitedMemoryAdapter({
      default: { limit: 1000 },                 // cap everything at 1000 entries
      message: { limit: 100, expire: 60_000 },  // keep 100 messages, 60s TTL
      presence: { limit: 0 },                   // effectively don't keep presences
    }),
  },
});
```

### LimitedCollection (standalone TTL map)

```ts
import { LimitedCollection } from 'seyfert';

const cache = new LimitedCollection<string, { score: number }>({ limit: 500 });

cache.set('user-123', { score: 42 }, 60_000); // 3rd arg = TTL ms; omit for options.expire (0 = never)
const item = cache.get('user-123');           // undefined if missing/expired
cache.has('user-123');
// rawValues()/rawEntries() carry expiration metadata; values()/entries() yield plain data.
```

### Custom resource (subclass `BaseResource`)

Note `set(from, id, data)` — the first arg is `CacheFrom`.

```ts
// resource.ts
import { BaseResource, CacheFrom } from 'seyfert';

interface CooldownData { id: string; lastDrip: number }
type CooldownInput = Omit<CooldownData, 'lastDrip'> & Partial<Pick<CooldownData, 'lastDrip'>>;

export class CooldownResource extends BaseResource<CooldownData, CooldownInput> {
  namespace = 'cooldowns';
  override set(from: CacheFrom, id: string, data: CooldownInput) {
    return super.set(from, id, { ...data, lastDrip: data.lastDrip ?? Date.now() });
  }
}
```

Register it (manual assignment + module augmentation). Seyfert never touches a custom resource unless your code does:

```ts
import { Client, CacheFrom, type ParseClient } from 'seyfert';
import { CooldownResource } from './resource';

const client = new Client();
client.cache.cooldown = new CooldownResource(client.cache, client);

// write/read it yourself
client.cache.cooldown.set(CacheFrom.Rest, 'user-1', { id: 'user-1' });
const cd = client.cache.cooldown.get('user-1');

declare module 'seyfert' {
  interface Cache { cooldown: CooldownResource }
  interface SeyfertRegistry { client: ParseClient<Client> }
}
```

### Custom guild-scoped resource (subclass `GuildBasedResource`)

```ts
import { GuildBasedResource, CacheFrom } from 'seyfert';

interface WarnData { id: string; guild_id: string; count: number }

export class WarnsResource extends GuildBasedResource<WarnData> {
  namespace = 'warns';
}
// usage: warns.set(CacheFrom.Rest, userId, guildId, { count: 1 })
//        warns.get(userId, guildId) ; warns.count('*') ; warns.flush(guildId)
```

### Custom adapter + testing

Key format is `<resource>.<id>` (base) / `<resource>.<guild>.<id>` (guild-based); `scan` queries use `*` as a wildcard segment and must match segment count.

```ts
import { Client } from 'seyfert';

const client = new Client();
client.setServices({ cache: { adapter: new MyAdapter() } });
await client.cache.testAdapter(); // throws SeyfertError on any mismatch; needs users+members+channels+overwrites enabled
```

The upstream async `Adapter` skeleton in the MDX is accurate against `src/cache/adapters/types.ts` (method names, the `bulkAddToRelationShip` casing, and the `scan(query, keys?)` overloads all match). The built-in `MemoryAdapter.scan` additionally requires the query's segment count to equal each key's.

## Doc vs Source Corrections

- Docs list adapters as only `MemoryAdapter` and `LimitedMemoryAdapter` -> src also ships `WorkerAdapter` (src/cache/adapters/workeradapter.ts), used by worker/sharded clients. (Redis remains external.)
- Docs show `disabledCache` only as an object -> src `ServicesOptions.cache.disabledCache` is `boolean | DisabledCache | ((cacheType) => boolean)` (base.ts:1392; the boolean path also disables `onPacket`, base.ts:318-346).
- Docs `filter` example uses `(channel, id, guildId)` -> src guild signature has a 4th param `from: CacheFrom`; `BaseResource.filter` is `(data, id, from)`. Existing 3-arg usage still compiles.
- Docs resource table label "stageInstances -> StageChannel" -> the cached entity is a stage instance (namespace `stage_instance`), not the channel.
- v5 `bans` is a guild-BASED resource: its `bulkGet` key is `['bans', userId, guildId]` (was `['bans', userId]` in older code).
- Otherwise the MDX code examples (disable, filter, LimitedCollection, custom resource, adapter skeleton) match src.

## Common patterns / gotchas

- `set`/`patch` on resources require a `CacheFrom` FIRST argument (`CacheFrom.Gateway` from packets, `CacheFrom.Rest` when seeding from REST, `CacheFrom.Test` internal). Forgetting it is the #1 stale-code break.
- Resource accessors are OPTIONAL (`client.cache.users?`): a disabled resource is `undefined` at runtime. Always null-check, or use `!` only when you know it's enabled.
- Reads are SYNC with `MemoryAdapter`/`LimitedMemoryAdapter` — no `await`. They become Promises only after you augment `interface InternalOptions { asyncCache: true }` (required for Redis / `WorkerAdapter`). Mismatching this is a common type error.
- Resources hand back structures (`.get()`, `.values()`); use `.raw()` / `.valuesRaw()` / `.bulkRaw()` for the plain API payload.
- Guild-based `keys/values/count/flush` take `'*' | guildId`. `GuildBasedResource.flush` requires the arg; `GuildRelatedResource.flush` defaults to `'*'`. `BaseResource` (users/guilds) has no guild arg.
- Configure cache via `client.setServices({ cache: { adapter, disabledCache } })`, not by reassigning `client.cache` wholesale. (Direct `client.cache.adapter = ...` also works — that is what `setServices` does.)
- `testAdapter()` throws a `SeyfertError` (not a boolean) on the first inconsistency, requires `users`/`members`/`channels`/`overwrites` enabled, and flushes the adapter before and after.
- For custom resources, prefer a plugin's `cacheResources` contribution if you have a plugin (`buildCache` wires `client.pluginRegistry.cacheResources` and survives a cache rebuild); manual `client.cache.x = ...` works but is cleared whenever `buildCache` re-runs (e.g. on `setServices({ cache: { disabledCache } })`).
- `LimitedCollection` default `expire` is `0` = never expire; pass a per-call TTL or set `expire` in options. `resetOnDemand` (re-extends TTL on `get`) defaults to `false`. A `NaN` limit throws.

## Source Anchors

- src/cache/resources/default/{base,guild-based,guild-related}.ts (resource bases, flush defaults)
- src/cache/resources/{users,members}.ts (structure vs raw return shapes)
- src/commands/applications/shared.ts:52 (InternalOptions, augment asyncCache)

## Agent Guidance

- Module augmentation uses `interface SeyfertRegistry { client; middlewares; … }` in v5 — NOT `UsingClient`/`RegisteredMiddlewares` (derived). Augment `interface Cache { ... }` to add custom resources.
- Redis adapter is an external package (tiramisulabs/extra redis-adapter) — verify its version in the target project; it must satisfy the local `Adapter` interface, and you must set `InternalOptions.asyncCache = true`.
