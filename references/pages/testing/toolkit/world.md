# World & State

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/world

Coverage reference: testing.md

Verification status: Source-verified (core) + external package

## Page Summary

The **world** is the in-memory Discord state the `@slipher/testing` mock bot serves instead of talking to Discord: guilds, channels, members, roles, messages. You seed it with `mockWorld()`, register entities, pass it to `createMockBot({ world })`, and assert on the read-only `bot.world` reader after dispatching. Crucially, seeded entities are written into the **real** client cache under `CacheFrom.Test`, so `ctx.guild()`, `ctx.channel()`, `member.voice()`, `client.cache.roles.values()` etc. resolve exactly like production cache hits — no interceptors. Recorded REST actions prove a call *happened*; world state proves *what the bot built*.

The toolkit (`mockWorld`, `createMockBot`, `apiRole`, `permissionBits`, `apiError`, `Routes`, etc.) is an EXTERNAL package NOT in core Seyfert — treat its surface as doc-authoritative and verify the installed version in the target project. The CORE seyfert APIs it dispatches against ARE verified below against ./src.

## Key APIs (verified)

External toolkit (from `@slipher/testing` — verify version in target project, NOT in core Seyfert):
- `mockWorld(): WorldBuilder` — `register*` methods mint entities with defaults, store them, and return them so you can reference their ids later.
- `createMockBot({ commands, world, onUnhandledRest?, simulateGateway? })` — deep-clones the world (safe to share a builder across tests). `await using bot = ...` for auto-teardown.
- Builder registrars (scoped — channel/role/member/emoji under a guild; thread/invite/webhook/stage under a channel): `registerGuild`, `registerChannel`, `registerThread`, `registerRole`, `registerMember`, `registerBotMember`, `registerVoiceState`, `registerMessage`, plus `registerEmoji`, `registerSticker`, `registerInvite`, `registerWebhook`, `registerAutoModRule`, `registerScheduledEvent`, `registerGuildTemplate`, `registerSoundboardSound`, `registerStageInstance`, `registerAuditLogEntry`. Registering under an unseeded id throws a descriptive `TypeError`. `registerGuild()` auto-creates `@everyone` (role id == guild id, position 0).
- `setData(key, value)` — chainable app-specific passthrough store; read back via `bot.worldData`.
- Readers: `bot.world.get.*` (exactly one or throws `WorldStateError` listing candidates), `bot.world.query.*` (one or `undefined`), `bot.world.all.*` (array). 24 entity methods (`guild`, `channel`, `thread`, `dm`, `member`, `role`, `message`, `rawMessage`, `voiceState`, `ban`, `reaction`, `pin`, `pollVote`, `threadMember`, `emoji`, `invite`, `autoModRule`, `sticker`, `scheduledEvent`, `webhook`, `guildTemplate`, `soundboardSound`, `stageInstance`, `auditLogEntry`). Each takes a narrowing query object (`{ id }`, `{ guildId, name }`, `{ guildId, userId }`, …).
- `bot.world` (`WorldStateReader`) also exposes `snapshot()` / `diff()` to capture and diff state declaratively.
- Helpers: `apiRole`, `apiGuild`, `permissionBits(names[] | bigint | string) -> wire string`, `apiError(status, code, msg)`, `Routes`.
- `bot.actions` (every call `{ seq, method, route, body, query, response }` in order), `bot.waitForAction(route)`, `bot.rest.intercept(...)`.

Core seyfert APIs the toolkit relies on (verified in ./src):
- `CacheFrom` enum — `Gateway = 1, Rest, Test` (so `CacheFrom.Test === 3`). src/cache/index.ts:1158. Toolkit writes entities under `CacheFrom.Test`.
- `GuildMember.voice(mode?)` — `'flow' | 'rest' | 'cache'`, default `'flow'`; `'cache'` returns `ReturnCache<VoiceStateStructure | undefined>`. src/structures/GuildMember.ts:95.
- `BaseCommandInteraction.channel(mode?)` / `.guild(mode?)` — `'cache' | 'rest' | 'flow'`. `'cache'` is sync. `GuildCommandContext` narrows `channel()` to `GuildCommandChannel` and `guild()` to a non-undefined guild. src/commands/applications/chatcontext.ts:172,220,269,272.
- `client.cache.roles.values(guild)` — `ReturnCache<GuildRoleStructure[]>`; `valuesRaw(guild)` returns `APIRole[]`. Accepts `'*'` for all scopes. src/cache/resources/roles.ts:35,41.
- `client.cache.voiceStates.values(guildId)` — `ReturnCache<VoiceStateStructure[]>`. src/cache/resources/voice-states.ts:42.
- `client.channels.fetchMessages(channelId, query?)` — `Promise<MessageStructure[]>`, patches the message cache; order is whatever the (mock) API returns (newest-first here). src/common/shorters/channels.ts:305.

## Code Examples (verified)

Seeding a world (toolkit external; dispatch hits core cache):

```ts
import { createMockBot, mockWorld } from '@slipher/testing';

const world = mockWorld();
const guild = world.registerGuild({ name: 'Slipher Lab' });
world.registerChannel(guild.id);
world.registerMember(guild.id, { nick: 'soc' });

await using bot = await createMockBot({ commands: [WhereCommand], world });
await bot.slash({ name: 'where', guildId: guild.id });
```

Voice states — `member.voice()` then resolves from the cache like production:

```ts
const world = mockWorld();
const guild = world.registerGuild();
const channel = world.registerChannel(guild.id, { name: 'General' });
const member = world.registerMember(guild.id);
world.registerVoiceState(guild.id, { userId: member.user.id, channelId: channel.id });
// In the command: const vs = await ctx.member!.voice('cache'); vs?.channelId === channel.id
```

Permissions — defaults are permissive: bot `app_permissions` = `DEFAULT_PERMISSIONS` (every bit), invoking member = `DEFAULT_MEMBER_PERMISSIONS` (realistic non-admin). Pass `memberPermissions: 'all'` for an admin invoker, or restrict to trigger `onPermissionsFail` / `onBotPermissionsFail`:

```ts
await bot.slash({ name: 'ban', memberPermissions: [] }); // member fails -> onPermissionsFail
await bot.slash({ name: 'ban', permissions: [] });       // bot fails    -> onBotPermissionsFail
await bot.slash({ name: 'ban', memberPermissions: 'all' }); // admin invoker
```

```ts
import { apiRole, permissionBits } from '@slipher/testing';

const mod = apiRole({ id: 'mod', permissions: permissionBits(['BanMembers']) });
await bot.slash({ name: 'ban', memberRoles: [mod] });
```

Discord-like computed permissions (mock runs the real algorithm: owner short-circuit, `Administrator`, overwrite layering). Role `position`s land in the role cache, readable via `client.cache.roles.values(guildId)`:

```ts
const world = mockWorld();
const guild = world.registerGuild({
  id: 'guild',
  ownerId: 'owner-user', // apiGuild otherwise mints a random owner id
  everyonePermissions: ['SendMessages'],
});
const mod = world.registerRole(guild.id, { permissions: ['BanMembers'], position: 5 });
const member = world.registerMember(guild.id, { roles: [mod.id] });
const denied = world.registerChannel(guild.id, {
  overwrites: [{ id: mod.id, type: 'role', deny: ['BanMembers'] }],
});
world.registerBotMember(guild.id, { roles: [mod.id] });

await using bot = await createMockBot({ commands: [BanCommand], world });
await bot.slash({ name: 'ban', guildId: guild.id, channel: denied, user: member.user });
```

With a seeded bot member, moderation REST routes (ban/kick/bulk-ban/edit-member/add-role/remove-role) enforce computed bot perms + hierarchy and return Discord's `403 50013`. Without a bot member they stay permissive. To drive an error branch directly:

```ts
bot.rest.intercept(Routes.ban, () => apiError(403, 50013, 'Missing Permissions'));
```

Querying world state (entities you seeded + everything the bot wrote: channels, messages, replies, edits, followups, DMs, bans, role changes, timeouts, overwrites):

```ts
const channel = bot.world.query.channel({ guildId: guild.id, name: 'acme-s1' });
expect(channel?.lastMessage?.content).toContain('Welcome');
expect(channel?.lastMessage?.component('Approve')).toMatchObject({ customId: 'approve' });
expect(channel?.lastMessage?.embeds[0]).toMatchObject({
  title: 'Acme S1',
  fields: [{ name: 'Budget', value: '$5,000' }],
});

// Channels & roles resolve by id alone (Discord keys them globally):
expect(bot.world.query.channel({ id: channel.id })?.name).toBe('general');
expect(bot.world.query.role({ id: role.id })?.permissions).toBe('4'); // BanMembers
expect(bot.world.query.dm({ userId: user.id })?.lastMessage?.content).toBe('Check your inbox');

// `get` throws on 0 / many — use it to assert uniqueness loudly:
expect(bot.world.get.guild({ id: guild.id }).name).toBe('Slipher Lab');
```

Seeded message history via core `fetchMessages` (no interceptor; mock returns newest-first):

```ts
world.registerMessage(channel.id, { content: 'old' });
world.registerMessage(channel.id, { content: 'new' });
expect(await bot.client.channels.fetchMessages(channel.id)).toMatchObject([
  { content: 'new' },
  { content: 'old' },
]);
```

Recorded actions and REST stubs:

```ts
import { Routes } from '@slipher/testing';

const edit = await bot.waitForAction(Routes.editOriginalResponse);
expect(edit.body).toMatchObject({ content: 'done' });
bot.actions; // every call, in order

// Stub a route the command READS from so it doesn't hit the unhandled-REST trap:
bot.rest.intercept('GET', '/guilds/:guildId', (_action, params) => ({ id: params.guildId, name: 'Stubbed' }));
// Unmatched REST throws by default; relax with createMockBot({ onUnhandledRest: 'warn' | 'silent' }).
```

## Recipes / more worked examples

### A full ban command + permission test (core command, external harness)

```ts
// ban.command.ts — pure seyfert v5
import { Command, Declare, Options, createUserOption } from 'seyfert';

const options = {
  user: createUserOption({ description: 'Who to ban', required: true }),
};

@Declare({ name: 'ban', description: 'Ban a member', defaultMemberPermissions: ['BanMembers'] })
@Options(options)
export default class BanCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const target = ctx.options.user;
    await ctx.client.members.ban(ctx.guildId!, target.id, { deleteMessageSeconds: 0, reason: 'test' });
    await ctx.write({ content: `Banned ${target.username}` });
  }
}
```

```ts
// ban.test.ts — assert the ban landed in world state AND the reply was sent
const world = mockWorld();
const guild = world.registerGuild({ id: 'g1' });
const target = world.registerMember(guild.id, { nick: 'spammer' });
world.registerBotMember(guild.id, { permissions: ['BanMembers'] });

await using bot = await createMockBot({ commands: [BanCommand], world });
await bot.slash({ name: 'ban', guildId: guild.id, options: { user: target.user } });

// world state proves WHAT happened:
expect(bot.world.get.ban({ guildId: guild.id, userId: target.user.id })).toBeDefined();
// recorded action proves the reply fired:
const reply = await bot.waitForAction(Routes.editOriginalResponse);
expect(reply.body).toMatchObject({ content: `Banned ${target.user.username}` });
```

### Snapshot / diff — assert mutations declaratively

Instead of field-by-field point queries, capture before/after and diff:

```ts
const before = bot.world.snapshot();
await bot.slash({ name: 'setup', guildId: guild.id });
const changes = bot.world.diff(before); // entities created/updated/removed since the snapshot
expect(changes.channel.created).toHaveLength(3);
```

### setData / worldData — app-specific passthrough

For domain state the mock should never interpret (e.g. a fixture id your command reads from your own store):

```ts
const world = mockWorld().setData('premiumGuildId', 'g1');
await using bot = await createMockBot({ commands: [PremiumCommand], world });
expect(bot.worldData.premiumGuildId).toBe('g1'); // read it back
```

### simulateGateway — stateful writes emit the matching event

With `simulateGateway` on (the default), a member edit / channel create / message create through REST also emits the gateway event (`GUILD_MEMBER_UPDATE`, `CHANNEL_CREATE`, `MESSAGE_CREATE`, …), so event handlers and the world stay consistent with a real bot. Disable it to test pure REST in isolation:

```ts
await using bot = await createMockBot({ commands, world, simulateGateway: false });
```

### Reading the direct interaction reply

When a channel view is not the clearest surface, the interaction reply is also available directly:

```ts
const result = await bot.slash({ name: 'ping', guildId: guild.id });
expect(result.reply?.body.data).toMatchObject({ content: 'Pong!' });
```

## Common patterns / gotchas

- **Build worlds in a factory, not at module scope.** `createMockBot` deep-clones on entry, but a shared *mutable* builder reused across files before cloning is a footgun. A helper that returns a fresh `mockWorld()` per test is safest.
- **Seed `registerBotMember` whenever bot authorization matters.** Without it, moderation REST routes stay permissive (good for unit-style tests). With it, they enforce computed bot perms + hierarchy and return `403 50013`.
- **Defaults are deliberately permissive** so bare dispatches sail past perm guards. Restrict (`memberPermissions: []`) to exercise denial branches; `memberPermissions: 'all'` for admin.
- **Unregistered user in a seeded guild → one-time warning.** Either `registerMember(...)` that user or pass explicit `memberPermissions` to bypass the computation.
- **Reader cardinality:** `get` (throws on 0/many — use for uniqueness asserts), `query` (`undefined`-able — optional chaining), `all` (array). Pick `get` so failures are loud.
- **`CacheFrom.Test` writes mean no interceptors for cache reads.** `ctx.guild('cache')`, `member.voice('cache')`, `client.cache.roles.values()` all resolve from seeded state. Only REST *reads* the command makes need `bot.rest.intercept(...)` (or `onUnhandledRest`).
- **Voice is state-only** — no real connections/audio. Seed `registerVoiceState` and read via `member.voice('cache')`.
- **Channels/roles query by id alone** (globally keyed); members/voice states need `{ guildId, userId }`.

## Doc vs Source Corrections

- None for the core APIs. `CacheFrom.Test`, `GuildMember.voice()`, `BaseCommandInteraction.guild()/channel()` (with `GuildCommandContext` narrowing), `client.cache.roles.values()`, `client.cache.voiceStates.values()`, and `client.channels.fetchMessages()` all exist and behave as the docs describe (verified against the target project installed `seyfert` package or provided Seyfert source). The entire toolkit surface (`mockWorld`, `createMockBot`, builders, readers, `snapshot`/`diff`, `apiRole`, `permissionBits`, `apiError`, `Routes`, `setData`/`worldData`) lives in the EXTERNAL `@slipher/testing` package and could not be verified against core Seyfert — verify its version/signatures in the target project.
- Note for v5: `member.ban(...)` / `client.members.ban(...)` now take `{ deleteMessageSeconds, reason }` (camelCase, reason inside the object) — reflected in the ban recipe above (changelog: Moderation breaking change).

## Source Anchors

- src/cache/index.ts:1158 — `CacheFrom` enum (`Gateway`, `Rest`, `Test`)
- src/structures/GuildMember.ts:95 — `voice(mode?)` overloads (`'cache'` is sync)
- src/commands/applications/chatcontext.ts:172,220,269,272 — `channel()` / `guild()` cache reads + `GuildCommandContext` narrowing
- src/cache/resources/roles.ts:35,41 — `roles.values('*' | guildId)` / `valuesRaw()`
- src/cache/resources/voice-states.ts:42 — `voiceStates.values()`
- src/common/shorters/channels.ts:305 — `channels.fetchMessages()`

## Agent Guidance

- Use this page when writing integration-style tests for commands/middleware that READ Discord state (permissions, voice, channels, message history), not just assert a REST call fired. World state answers "what did the bot build"; `bot.actions` answers "what calls happened".
- The toolkit is EXTERNAL (`@slipher/testing`). Before relying on any `register*` / reader signature, check the installed version — names and options may drift from these docs. The seyfert CORE APIs it dispatches against are verified here.
- Seed `registerBotMember` whenever the bot-authorization contract matters; otherwise moderation routes stay permissive.
- Reach for `get` (throws on 0/many) when asserting uniqueness; `query` for optional; `all` for arrays.
- Entities are written under `CacheFrom.Test`, so production cache reads resolve without interceptors. Only REST reads the command performs need stubbing (`bot.rest.intercept`) or `onUnhandledRest: 'warn' | 'silent'`.
- When generating v5 bot code in these tests, use the v5 moderation shape (`ban({ deleteMessageSeconds, reason })`, `timeout(ms, reason)`) and root imports from `'seyfert'`.
