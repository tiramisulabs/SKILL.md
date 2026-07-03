# Shorters (`client.<resource>.*` — the first place to look)

Verification status: Source-verified against `src/common/shorters/*.ts` and `src/client/base.ts` (v5 HEAD).
Related: `pages/recipes/api-access.md` (proxy + `rest` deep dive), `pages/learn/tips/structures.md` (structure-level shorters).

## What a shorter is

A **shorter** is a typed convenience method, grouped by Discord resource and hung directly off the
client — `client.messages`, `client.channels`, `client.members`, `client.roles`, … Each one wraps one
or more Discord REST routes and does the boring work for you: transforms the request body, resolves
files/attachments, hits the cache when it makes sense, and returns a **rich Seyfert structure**
(`MessageStructure`, `GuildStructure`, …) instead of raw JSON.

- Source: every shorter class lives in `src/common/shorters/*.ts` and extends `BaseShorter`
  (`src/common/shorters/base.ts` — just `readonly client`).
- Wiring: they are instantiated as client fields in `src/client/base.ts:158-174`, so they are available
  anywhere you have a client: `ctx.client.*`, `client.*`, `interaction.client.*`.

Two flavors, same engine:

- **Client shorters** — `client.<resource>.<verb>(...)` — the catalog in this file.
- **Structure shorters** — verbs on an object you already hold: `member.ban()`, `channel.messages.write()`,
  `message.reply()`, `user.avatarURL()`, `member.guild()`. These delegate to the client shorters under the
  hood. See `pages/learn/tips/structures.md`.

> Accessor names are **plural nouns on the client**: `client.messages`, `client.channels`, `client.bans`
> — not `client.message` / `client.channel`. (Exceptions worth memorizing below: `client.voiceStates`,
> `client.soundboards`, `client.webhooks`, `client.interactions`.)

## The fallback chain — "this method seems to not exist"

When you can't find a call for something, walk these three rungs **in order**. Do NOT reach for `fetch`,
`axios`, or a hand-rolled REST client — Seyfert already gives you all three layers, fully typed.

1. **Look for a SHORTER first.** grep `src/common/shorters/` (or your `node_modules/seyfert/lib/common/shorters/`)
   and the structure classes. Most "missing" methods are just under a different accessor or a different name
   than you guessed — *send a message* is `client.messages.write(channelId, body)` (not `channels.send`);
   *fetch a member* is `client.members.fetch(guildId, userId)`. Check the catalog below and the structure
   helpers before assuming it doesn't exist.
2. **No shorter? Use `client.proxy`.** A typed JS `Proxy` mirroring the Discord REST endpoint tree with full
   autocompletion — you can call **any** route, even ones no shorter covers. It returns **raw snake_case JSON**
   (no cache, no structure transform). Path segments are properties, ids are function calls, the terminal
   verb is lowercase: `client.proxy.channels(id).threads.post({ body })`. Full rules in `pages/recipes/api-access.md`.
3. **Route not in the proxy types yet?** (bleeding-edge / undocumented endpoint) — drop to
   `client.rest.request<T>(method, url, opts)`, the low-level `ApiHandler` every shorter and the proxy funnel
   through. Type `T` yourself from the **Discord API v10 reference** (<https://discord.com/developers/docs/reference>,
   API version 10) and build the URL by hand: `client.rest.request<T>('GET', '/gateway/bot')`.

There is no rung 4. If Discord's HTTP API can do it, one of shorter → `proxy` → `rest.request` can call it.

| Layer | Call shape | Returns | Cache / transform | Reach for it when |
|---|---|---|---|---|
| Shorter | `client.messages.write(id, body)` | Seyfert structure | yes | a shorter exists (default) |
| Proxy | `client.proxy.channels(id).messages.post({ body })` | raw snake_case JSON | no | no shorter for the route |
| `rest` | `client.rest.request<T>('POST', '/channels/{id}/messages', { body })` | raw `T` you type | no | route absent from proxy types (new API) |

## Client shorter catalog

Every shorter below is a field on the client (`src/client/base.ts:158-174`). Signatures are copied from
`src/common/shorters/*.ts` at HEAD — verify against the target project's installed `node_modules/seyfert`
if versions differ. Most read/fetch methods take a trailing `force = false` to bypass the cache; most
mutating methods take a trailing `reason?: string` (Discord audit log).

### `client.applications` — ApplicationShorter
Current application: emojis, entitlements, SKUs, activity instances, app metadata.
- `listEmojis(force?)` — GET `/applications/{app}/emojis`.
- `getEmoji(emojiId, force?)` — GET `/applications/{app}/emojis/{id}`.
- `createEmoji(raw)` — POST `/applications/{app}/emojis` (resolves the image).
- `editEmoji(emojiId, body)` — PATCH `/applications/{app}/emojis/{id}`.
- `deleteEmoji(emojiId)` — DELETE `/applications/{app}/emojis/{id}`.
- `listEntitlements(query?)` — GET `/applications/{app}/entitlements`.
- `consumeEntitlement(entitlementId)` — POST `…/entitlements/{id}/consume`.
- `createTestEntitlement(body)` / `deleteTestEntitlement(entitlementId)` — test entitlements.
- `listSKUs()` — GET `/applications/{app}/skus`.
- `fetch()` / `edit(body)` — GET/PATCH `/applications/@me`.
- `getActivityInstance(instanceId)` — GET `…/activity-instances/{id}`.

### `client.users` — UsersShorter
Users and their DM channels.
- `fetch(userId, force?): Promise<UserStructure>` — cache-first user fetch (via `raw`).
- `raw(userId, force?): Promise<APIUser>` — GET `/users/{id}`, raw payload.
- `createDM(userId, force?)` / `deleteDM(userId, reason?)` — open / close a DM channel.
- `write(userId, body): Promise<MessageStructure>` — open a DM then send a message.

### `client.channels` — ChannelShorter
Fetch/edit/delete channels, permission overwrites, pins, typing, threads, message reads.
- `fetch(id, force?): Promise<AllChannels>` — GET `/channels/{id}`, resolved structure.
- `raw(id, force?): Promise<APIChannel>` — GET `/channels/{id}`, raw payload.
- `edit(id, body, optional?)` / `delete(id, optional?)` — PATCH / DELETE `/channels/{id}`.
- `editOverwrite(channelId, overwriteId, raw, optional?)` — PUT `…/permissions/{id}` (grant/deny).
- `deleteOverwrite(channelId, overwriteId, optional?)` — DELETE `…/permissions/{id}`.
- `typing(id)` — POST `/channels/{id}/typing`.
- `pins(channelId, query?)` — GET `…/messages/pins`; `setPin(msgId, chId, reason?)` / `deletePin(msgId, chId, reason?)`.
- `thread(channelId, body, reason?)` — create a thread (delegates to `client.threads.create`).
- `fetchMessages(channelId, query?): Promise<MessageStructure[]>` — GET `/channels/{id}/messages`.
- `memberPermissions(channelId, member, checkAdmin?)` / `rolePermissions(channelId, role, checkAdmin?)` — effective channel perms from overwrites.
- `overwritesFor(channelId, member)` — buckets overwrites into everyone/roles/member.
- `setVoiceStatus(channelId, status?)` — PUT `/channels/{id}/voice-status`.

### `client.guilds` — GuildShorter
Guilds and their sub-resources (`channels`, `moderation`, `stickers`) plus member/message search.
- `fetch(id, options?)` / `raw(id, options?)` — GET `/guilds/{id}` (cache-first / raw).
- `edit(guildId, body, reason?)` — PATCH `/guilds/{id}`.
- `list(query?, force?)` — GET `/users/@me/guilds`; `leave(id)` — DELETE `/users/@me/guilds/{id}`.
- `fetchSelf(id, force?)` — the bot's own member; `widgetURL(id, style?)` — widget image URL.
- `searchMessages(guildId, query?, wait?)` — GET `…/messages/search` (retries on 202 when `wait`).
- **`guilds.channels`**: `list(guildId, force?)`, `fetch(guildId, channelId, force?)`, `create(guildId, body)`, `edit(guildId, channelId, body, reason?)`, `delete(guildId, channelId, reason?)`, `editPositions(guildId, body)`, `addFollower(channelId, webhookChannelId, reason?)`.
- **`guilds.moderation`** (auto-mod rules): `list`, `fetch`, `create`, `edit`, `delete` on `…/auto-moderation/rules`.
- **`guilds.stickers`**: `list`, `fetch`, `create` (multipart), `edit`, `delete` on `…/stickers`.

### `client.messages` — MessageShorter
Send/edit/delete/fetch messages, crosspost, bulk purge, threads, polls.
- `write(channelId, body): Promise<MessageStructure>` — POST `…/messages` (resolves files, caches).
- `edit(messageId, channelId, body)` — PATCH `…/messages/{id}`.
- `delete(messageId, channelId, reason?)` — DELETE `…/messages/{id}`.
- `fetch(messageId, channelId, force?)` / `raw(...)` — GET a single message (structure / raw).
- `list(channelId, query?)` — GET `…/messages` (list); `purge(messages[], channelId, reason?)` — bulk delete.
- `crosspost(messageId, channelId, reason?)` — publish an announcement message.
- `thread(channelId, messageId, options)` — start a thread from a message.
- `endPoll(channelId, messageId)` — expire a poll; `getAnswerVoters(channelId, messageId, answerId)` — voters for an answer.

### `client.members` — MemberShorter
Guild members: fetch/search/edit, roles, ban/kick, timeouts, voice, presence.
- `fetch(guildId, memberId, force?)` / `raw(...)` — GET a member (structure / raw).
- `list(guildId, query?, force?)` / `search(guildId, query)` — list / search members.
- `resolve(guildId, resolvable)` — resolve from mention/id/display name.
- `edit(guildId, memberId, body, reason?)` — PATCH member (nick, roles, etc.).
- `add(guildId, memberId, body)` — PUT (add member via OAuth token).
- `ban(guildId, memberId, options?)` / `unban(guildId, memberId, reason?)` / `kick(guildId, memberId, reason?)`.
- `addRole(guildId, memberId, roleId)` / `removeRole(guildId, memberId, roleId)`.
- `listRoles(guildId, memberId, force?)` / `sortRoles(...)` — member's roles (sorted by position).
- `permissions(guildId, memberId, force?)` — computed `PermissionsBitField`.
- `timeout(guildId, memberId, time|null, reason?)` — time is **ms**; `hasTimeout(member)` — ms left or `false`.
- `voice(guildId, memberId|'@me', force?)` — member voice state; `presence(memberId)` — cached presence.

### `client.webhooks` — WebhookShorter
Webhooks and the messages sent through them.
- `create(channelId, body)` / `fetch(webhookId, token?)` / `edit(webhookId, body, options)` / `delete(webhookId, options)`.
- `writeMessage(webhookId, token, payload)` — execute the webhook (send).
- `editMessage(webhookId, token, payload)` / `deleteMessage(payload)` / `fetchMessage(payload)`.
- `listFromGuild(guildId)` / `listFromChannel(channelId)`.

### `client.templates` — TemplateShorter
Guild templates.
- `fetch(code)` — GET `/guilds/templates/{code}`.
- `list(guildId)` / `create(guildId, body)` / `sync(guildId, code)` / `edit(guildId, code, body)` / `delete(guildId, code)`.

### `client.roles` — RoleShorter
Guild roles.
- `create(guildId, body, reason?)` — POST `…/roles`.
- `fetch(guildId, roleId, force?)` / `raw(...)` — a single role.
- `list(guildId, force?)` / `listRaw(...)` — all roles.
- `edit(guildId, roleId, body, reason?)` / `delete(guildId, roleId, reason?)`.
- `editPositions(guildId, body)` — reorder roles.
- `memberCounts(guildId)` — role id → member count.

### `client.reactions` — ReactionShorter
Message reactions.
- `add(messageId, channelId, emoji)` — PUT own reaction (handles emoji encoding).
- `delete(messageId, channelId, emoji, userId?)` — remove a reaction (default `@me`).
- `fetch(messageId, channelId, emoji, query?)` — users who reacted.
- `purge(messageId, channelId, emoji?)` — remove all reactions (or all of one emoji).

### `client.emojis` — EmojiShorter
Guild custom emojis.
- `list(guildId, force?)` / `fetch(guildId, emojiId, force?)`.
- `create(guildId, body)` (resolves image) / `edit(guildId, emojiId, body, reason?)` / `delete(guildId, emojiId, reason?)`.

### `client.threads` — ThreadShorter
Thread lifecycle, membership, and listing.
- `create(channelId, body, reason?)` — POST `…/threads`; `fromMessage(channelId, messageId, options)` — thread from a message.
- `edit(threadId, body, reason?)` / `lock(threadId, locked?, reason?)`.
- `join(threadId)` / `leave(threadId)` / `addMember(threadId, memberId)` / `removeMember(threadId, memberId)`.
- `fetchMember(threadId, memberId, withMember)` / `listMembers(threadId, query?)`.
- `listArchived(channelId, 'public'|'private', query?)` / `listJoinedArchivedPrivate(channelId, query?)` / `listGuildActive(guildId, force?)`.

### `client.bans` — BanShorter
Guild ban entries.
- `create(guildId, memberId, options?)` — PUT ban; `remove(guildId, memberId, reason?)` — unban.
- `bulkCreate(guildId, body, reason?)` — POST `…/bulk-bans`.
- `fetch(guildId, userId, force?)` / `list(guildId, query?, force?)`.

### `client.interactions` — InteractionShorter
Interaction responses, callbacks, and followups (used by contexts under the hood).
- `reply(id, token, body, withResponse?)` — POST the initial callback.
- `fetchResponse(token, messageId, threadId?)` / `fetchOriginal(token)`.
- `editMessage(token, messageId, body)` / `editOriginal(token, body)`.
- `deleteResponse(token, messageId)` / `deleteOriginal(token)`.
- `followup(token, body)` — POST a followup webhook message.

### `client.voiceStates` — VoiceStateShorter
The bot's own voice state in a guild.
- `requestSpeak(guildId, date)` — PATCH `…/voice-states/@me` request-to-speak.
- `setSuppress(guildId, suppress)` — PATCH suppress flag.

### `client.soundboards` — SoundboardShorter
Soundboard sounds (default + guild) and sending them.
- `getDefaults()` — GET `/soundboard-default-sounds`.
- `send(channelId, body)` — POST `…/send-soundboard-sound` to a voice channel.
- `list(guildId)` / `get(guildId, soundId)` / `create(guildId, body)` / `edit(guildId, soundId, body)` / `delete(guildId, soundId, reason?)`.

### `client.invites` — InvitesShorter
Invites, their target users, plus channel/guild invite listings.
- `get(code)` — GET `/invites/{code}`; `delete(code, reason?)` — revoke.
- `getTargetUsers(code)` / `updateTargetUsers(code, targetIds)` / `jobStatus(code)` — invite target-user list (CSV).
- **`invites.channels`**: `create({ channelId, reason, ...body })` / `list(channelId)`.
- **`invites.guilds`**: `list(guildId)` — a guild's invites.

## Structure-level shorters (delegate to the above)

You usually already hold a structure — use its verbs instead of re-fetching. They call the client shorters
for you. Non-exhaustive; see `pages/learn/tips/structures.md` for the full surface.

- **Message**: `message.reply(body)`, `message.edit(body)`, `message.delete(reason?)`, `message.crosspost()`,
  `message.react(emoji)`, `message.createComponentCollector()`.
- **GuildMember**: `member.ban(opts?)`, `member.kick(reason?)`, `member.edit(body, reason?)`,
  `member.timeout(ms, reason?)`, `member.roles.add(id)` / `.remove(id)` / `.list()`, `member.guild(mode?)`,
  `member.voice(mode?)`, `member.fetchPermissions()`, `member.avatarURL()`, gates `member.bannable()` /
  `kickable()` / `moderatable()`.
- **Channels**: `channel.messages.write(body)` (after an `isTextable()` guard), `channel.edit()`,
  `channel.delete()`, `channel.fetch()`, voice `channel.members()`.
- **User**: `user.fetch()`, `user.avatarURL()`, `user.dm()` / `user.write(body)`.
- **Guild / Role / others**: `guild.fetch()`, `guild.members.*`, `role.edit()`, `role.delete()`, etc.

## Object resolution vs id-based shorters

Many Discord libraries make you materialize an intermediate gateway object (`Channel`, `Guild`,
`GuildMember`) before you can act on it. Seyfert shorters collapse those multi-step REST dances into a
single id-based call. Concrete before/after:

**Send / log to a channel by id.** The object-resolution style fetches the channel and narrows its type
just to reach `.send`; the shorter writes straight to the snowflake already in config (no extra REST
round-trip):

```ts
// object resolution: fetch → narrow → send
const channel = await client.channels.fetch(config.channels.reports);
if (channel?.type !== ChannelType.GuildText) return;
await channel.send({ embeds: [embed], components: [row] });

// seyfert: one id-based call
await client.messages.write(config.channels.reports, { embeds: [embed], components: [row] });
```

**Fetch a member by id.** The object-resolution style needs a materialized `interaction.guild` first; the shorter takes ids directly:

```ts
// object resolution
const { guild } = interaction;
if (!guild) return;
const member = await guild.members.fetch({ user }).catch(() => null);

// seyfert
if (!ctx.guildId) return;
const member = await ctx.client.members.fetch(ctx.guildId, user.id).catch(() => null);
```

**List a guild's roles.** No live `Guild` object, no `guild.roles.everyone` (everyone === `guildId`), no `PermissionsBitField` import:

```ts
// object resolution: read the cache off a live guild
const roles = guild.roles.cache
  .filter(r => !r.permissions.has(PermissionsBitField.Flags.Administrator) && !r.managed && r !== guild.roles.everyone)
  .sort((a, b) => b.position - a.position);

// seyfert: by id, returns an array
const roles = (await client.roles.list(guildId))
  .filter(r => !r.permissions.has('Administrator') && !r.managed && r.id !== guildId)
  .sort((a, b) => b.position - a.position);
```

More spots where an intermediate object collapses to a single id-based shorter call:

- **DM a user** — `user.send(...)` → `user.write({ content })` (structure helper opens the DM implicitly).
- **Edit a channel overwrite** — `channel.permissionOverwrites.edit(id, {...})` → `client.channels.editOverwrite(channelId, targetId, { type, allow: [...] }, { guildId })` (no fetched `GuildChannel` needed).
- **Create a channel** — `guild.channels.create({...})` → `client.guilds.channels.create(guildId, {...})` (no live `Guild`).
- **Fetch a message** — `guildChannel.messages.fetch(id)` → `client.messages.fetch(messageId, channelId)` (both ids, no channel object).

> Pattern: the object-resolution style makes you resolve an intermediate gateway object (`Channel`, `Guild`, `GuildMember`)
> and call a method on *it*; Seyfert shorters take the raw snowflake ids you already have. Fewer lines,
> one fewer round-trip, no `ChannelType` narrowing just to satisfy a `.send`.

## Agent guidance

- **Shorter first, always.** Before writing any REST call, look for the client shorter (catalog above) or
  the structure method. They cache, transform, resolve files, and return typed structures — raw proxy/`rest`
  calls give you none of that.
- **Then `client.proxy`, then `client.rest.request<T>`** — never `fetch`/`axios`/manual REST. Walk the
  fallback chain in order and stop at the first rung that works.
- **Accessors are plural** on the client (`client.messages`, `client.bans`, `client.voiceStates`). If an
  accessor "doesn't exist", you probably singularized it or guessed the resource — check the catalog.
- **Prefer the structure verb ONLY when you already hold the object** (`member.ban()` over
  `client.bans.create(guildId, member.id)`): fewer ids to thread, same engine underneath. But **do not
  fetch an object just to reach its verb** — if all you have are ids, call the client shorter directly
  (`client.bans.create(guildId, userId)`), don't `members.fetch(...)` first just to call `member.ban()`.
- **`force` and `reason`**: pass `force: true` to skip the cache on fetch/read shorters; pass `reason` on
  mutating shorters to write the Discord audit log.
- Proxy returns **raw snake_case JSON** with no caching or transform — only drop to it when no shorter fits.
