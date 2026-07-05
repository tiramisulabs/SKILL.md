# Structures & Transformers

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/tips/structures
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (v5, the authoritative Seyfert source)

## Page Summary

When you fetch or receive Discord objects, Seyfert wraps the raw API payload in typed
*structure* classes (`GuildMember`, `Guild`, channel classes, `Message`, `User`, `VoiceState`, etc.)
that expose convenience methods and getters. The `Transformers` object (in
`src/client/transformers.ts`) is the single factory point for every structure: by reassigning
`Transformers.<Name>` and augmenting the `CustomStructures` interface via `declare module "seyfert"`,
you replace the shape project-wide. All these types import from the `'seyfert'` root.

## Key APIs (verified)

GuildMember (`src/structures/GuildMember.ts`):
- `member.id`, `member.guildId` (readonly), `member.user` (a `UserStructure`).
- `member.roles` is a getter returning an object literal: `keys` (`readonly string[]`, frozen,
  = `_roles.concat(guildId)` so it ALWAYS includes the `@everyone` role id), `list(force?)`,
  `add(id)`, `remove(id)`, `permissions(force?)`, `sorted(force?)`, `highest(force?)`.
- `member.voice(mode)`: overloaded — `'cache'` → `ReturnCache<VoiceStateStructure | undefined>`;
  `'rest' | 'flow'` (default `'flow'`) → `Promise<VoiceStateStructure>`. `'flow'` = cache then REST.
- `member.fetchPermissions(force = false)` → `Promise<PermissionsBitField>` (returns own `permissions`
  if present, e.g. on `InteractionGuildMember`, else computes from roles).
- `member.guild(mode)`: `'cache'` → cached `GuildStructure` or undefined; else `Promise<GuildStructure>`.
- Actions: `ban(options?: BanOptions)`, `kick(reason?)`, `edit(body, reason?)`,
  `timeout(time: number | null, reason?)` (time is MILLISECONDS in v5), `write(body)` (DM via the user),
  `dm(force?)`, `presence()`, `fetch(force?)`.
- Hierarchy gates (all async, parallel-fetch internally): `manageable(force?)`, `bannable(force?)`,
  `kickable(force?)`, `moderatable(force?)`. `bannable`/`kickable` = manageable AND bot has the perm;
  `moderatable` = target is NOT admin AND manageable AND bot has `KickMembers`.
- Getters: `displayName` (nick ?? globalName ?? username), `tag`, `name`, `username`, `globalName`,
  `bot`, `avatarURL()`, `bannerURL()`, `hasTimeout`. `InteractionGuildMember` adds a concrete
  `permissions: PermissionsBitField`.

PermissionsBitField (`src/structures/extra/Permissions.ts`, extends `BitField`):
- `has(bits)` → true if bits present OR `Administrator` present (admin override).
- `strictHas(bits)` → exact bits, NO admin override.
- `missings(bits[])` (from `BitField`) → array of missing bits (empty if none).
- `keys()` returns typed `(keyof typeof PermissionFlagsBits)[]`; `add`/`remove`/`values`/`equals` also exist.
- `bits` accepts a single bit OR an array. Static `PermissionsBitField.resolve(...)` throws
  `TypeError` on unknown strings / negative / unsafe-integer input (the old instance `resolve` is gone).

Channel type guards (`src/structures/channels.ts`, on `BaseNoEditableChannel`):
`isStage()`, `isMedia()`, `isDM()`, `isForum()`, `isThread()`, `isDirectory()`, `isVoice()`,
`isTextGuild()`, `isCategory()`, `isNews()`, `isTextable()`, `isGuildTextable()`, `isThreadOnly()`
(`ForumChannel | MediaChannel`), `isGuild()`, `isNamed()`, plus `is([...ChannelType keys])`. Each is a
`this is X` type-guard. Use `isTextable()` before `channel.messages.write(...)`. (There is NO
`isSendable()` — that name does not exist in source.)

VoiceState (`src/structures/VoiceState.ts`): boolean getters `isMuted` (mute || selfMute),
`isDeafened` (deaf || selfDeaf), `isCameraOn` (selfVideo), `isStreaming` (selfStream ?? false),
`isSuppressed` (suppress). Methods: `member(force?)`, `user(force?)`, `channel(mode)`, `setMute(mute?, reason?)`,
`setDeaf(deaf?, reason?)`, `setSuppress(suppress?)`, `requestSpeak(date?)`, `disconnect(reason?)`,
`setChannel(channelId, reason?)`, `guild(mode)`, `fetch(force?)`.

Message (`src/structures/Message.ts`): `reply(body, fail = true)`, `edit(body)`, `delete(reason?)`,
`crosspost(reason?)`, `createComponentCollector(options?)`, plus
`id/content/author/guildId/channelId/createdTimestamp`.

Transformers / CustomStructures:
- `Transformers` (`src/client/transformers.ts`) is a plain object whose keys are factory functions
  (default = `(...args) => new Structure(...args)`) for every structure. Reassign a key to override.
- `CustomStructures` is an (empty) interface declared in `src/commands/applications/shared.ts`,
  re-exported from `'seyfert'`. `InferCustomStructure<T, N>` = `CustomStructures extends Record<N, infer P> ? P : T`.
  Augment it through `declare module "seyfert"` so your custom shape flows everywhere.
- Shorter raw access for override examples: `client.users.raw(userId, force?)`
  (`src/common/shorters/users.ts`) returns the raw `APIUser`.

## Code Examples (verified)

Member basics:
```ts
import { type GuildCommandContext } from 'seyfert';
declare const ctx: GuildCommandContext;

const member = ctx.member;          // GuildMember (non-undefined in a GuildCommandContext)
member.id;                          // user id
member.guildId;                     // guild id
member.user;                        // UserStructure
member.roles.keys;                  // readonly string[] (includes @everyone)
await member.roles.list();          // GuildRoleStructure[]

const voice = await member.voice('cache');    // VoiceStateStructure | undefined
const voiceFlow = await member.voice('flow'); // cache -> REST fallback

const perms = await member.fetchPermissions();
perms.has(['Administrator']);                   // boolean (admin override applies)
perms.missings(['SendMessages', 'EmbedLinks']); // missing bits, [] if none

await member.ban({ deleteMessageSeconds: 0, reason: 'Reason' });
await member.kick('Reason');
await member.edit({ nick: 'New Nickname' });
await member.timeout(60_000, 'cooldown');       // MILLISECONDS (60s), optional reason
await member.write({ content: 'DM message' });  // DM the member
```

Channel guard before sending:
```ts
import { type UsingClient } from 'seyfert';
declare const client: UsingClient;

const channel = await client.channels.fetch('channelId');
if (channel.isTextable()) await channel.messages.write({ content: 'Hello' });
```

Custom transformer + type augmentation (replaces the shape — built-in methods are lost unless re-added):
```ts
import { Transformers } from 'seyfert';
import type { APIUser } from 'seyfert';

interface MyUser {
  username: string;
  isAdmin: boolean;
  raw(): Promise<APIUser>;
}

Transformers.User = (client, data) => ({
  username: data.username,
  isAdmin: ['123456789012345678'].includes(data.id),
  raw: () => client.users.raw(data.id),
});

declare module 'seyfert' {
  interface CustomStructures {
    User: MyUser; // ctx.author, member.user, etc. all become MyUser now
  }
}
```

## Recipes / Common patterns

Safe moderation with hierarchy gates (never ban above the bot, never act on admins):
```ts
import { type GuildCommandContext } from 'seyfert';
declare const ctx: GuildCommandContext;
declare const target: import('seyfert').GuildMember;

// bannable/kickable/moderatable parallel-fetch the bot's roles + guild internally.
if (!(await target.bannable())) {
  await ctx.editOrReply({ content: 'I cannot ban this member (role hierarchy / ownership).' });
} else {
  await target.ban({ deleteMessageSeconds: 7 * 24 * 60 * 60, reason: `by ${ctx.author.tag}` });
  await ctx.editOrReply({ content: `Banned ${target.tag}.` });
}

// Manual gate when you want a custom message per check:
const me = await ctx.me();            // bot member, if available
const botPerms = await me?.fetchPermissions();
if (!botPerms?.strictHas(['ManageRoles'])) {
  // strictHas: an admin bot does NOT auto-pass this — you really need the bit.
}
```

Narrowing a fetched channel to the right type, then using type-specific API:
```ts
import { type UsingClient } from 'seyfert';
declare const client: UsingClient;

const channel = await client.channels.fetch('channelId'); // AllChannels union

if (channel.isThread()) {
  // channel is ThreadChannel here
  await channel.messages.write({ content: 'inside a thread' });
} else if (channel.isVoice()) {
  // channel is VoiceChannel — voice-only members helper
  const members = await channel.members();     // GuildMemberStructure[]
  console.log(`${members.length} in voice`);
} else if (channel.is(['GuildText', 'GuildAnnouncement'])) {
  // multi-type guard via is([...]) — narrows to TextGuildChannel | NewsChannel
  await channel.messages.write({ content: 'announcement-capable text channel' });
}
```

Reacting to voice with the boolean getters (no more `state.mute || state.selfMute`):
```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'voiceStateUpdate' },
  run([newState, oldState], client) {
    // newState.isStreaming etc. collapse the mute/selfMute, deaf/selfDeaf pairs for you.
    if (newState.isStreaming && !oldState?.isStreaming) {
      client.logger.info(`${newState.userId} started streaming in ${newState.channelId}`);
    }
    if (newState.isCameraOn) { /* selfVideo === true */ }
  },
});
```

Custom transformer that EXTENDS the built-in class (keeps every native method/getter):
```ts
import { Transformers, GuildMember } from 'seyfert';

// Subclass so you don't lose ban()/roles/fetchPermissions(), just add behavior.
class ExtendedMember extends GuildMember {
  get isStaff() {
    return this.roles.keys.includes('123456789012345678'); // some staff role id
  }
}

Transformers.GuildMember = (...args) => new ExtendedMember(...args);

declare module 'seyfert' {
  interface CustomStructures {
    GuildMember: ExtendedMember; // ctx.member.isStaff is now typed everywhere
  }
}
```

Permission diffing for a clear "missing perms" reply:
```ts
import { PermissionsBitField } from 'seyfert/lib/structures/extra/Permissions';
declare const member: import('seyfert').GuildMember;

const required = ['ManageChannels', 'ManageRoles'] as const;
const perms = await member.fetchPermissions();
const missing = perms.missings([...required]);          // bigint[] of the ones lacking
if (missing.length) {
  const names = new PermissionsBitField(missing).keys(); // typed permission-name strings
  // -> e.g. ['ManageRoles']
}
```

## Doc vs Source Corrections

- Docs comment `member.voice('cache')` as "Voice channel ID or null"; src returns a
  `VoiceStateStructure | undefined` (you then read `.channelId`). `'flow'` (the default) is
  cache-then-REST, not "API fallback" only.
- Docs imply `fetchPermissions()` takes no args; src signature is `fetchPermissions(force = false)`
  and returns a `Promise<PermissionsBitField>`.
- Docs show only `has`/`missings`; src `PermissionsBitField.has()` applies an `Administrator`
  override (always true for admins). Use `strictHas()` when you need the exact bit without override.
- Docs list only a subset of channel guards; src also has `isDM()`, `isTextGuild()`,
  `isGuildTextable()`, `isThreadOnly()`, `isGuild()`, `isNamed()`, and the multi-type `is([...])`
  (all `this is X` guards). There is NO `isSendable()`.
- Docs reference `MyUser` in the transformer example without showing it; the interface must be
  defined (as above) for the augmentation to type-check.
- `member.timeout(...)` takes MILLISECONDS in v5 (was seconds in v4) and accepts an optional reason.
- `member.ban(...)` takes `{ deleteMessageSeconds, reason }` (camelCase, reason inside the object) —
  no raw `delete_message_seconds` / positional reason.
- Otherwise the doc Member/Guild/Channel/Message/Shorter/Interaction-guard examples match src.

## Source Anchors

- `src/structures/extra/Permissions.ts` + `src/structures/extra/BitField.ts`
  (`has`/`strictHas`/`missings`/`keys`/`resolve`)
- `src/structures/Message.ts`, `src/structures/User.ts`

## Agent Guidance

- Reach for structure methods/getters over raw payload poking; they delegate to the right shorter.
- To customize a structure you MUST do both: reassign `Transformers.<Name>` AND augment
  `CustomStructures` in `declare module "seyfert"` — types and runtime are decoupled.
- Prefer EXTENDING the built-in class (subclass + `Transformers.X = (...a) => new Sub(...a)`) when you
  only want to ADD behavior; a plain object literal transformer REPLACES the shape and drops every
  native method unless you re-add/delegate (keep a `raw()` to the API object when you still need it).