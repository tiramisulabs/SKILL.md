# Shorters and Proxy (Direct API Access)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/api-access
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (branch more-qol)

## Page Summary

Three layers exist for talking to Discord, from highest to lowest:

1. **Shorters** — typed convenience methods on the client (`client.messages.write`, `client.channels.fetch`, `client.reactions.add`) and on structures (`member.guild()`, `user.avatarURL()`). They transform bodies, resolve files, hit the cache when appropriate, and return rich Seyfert structures.
2. **`client.proxy`** — a typed JS `Proxy` mirroring the Discord REST endpoint tree. Call ANY route (even ones with no shorter) with full TS autocompletion. Returns RAW snake_case JSON, no caching, no transform.
3. **`client.rest`** (`ApiHandler`) — the low-level handler everything funnels through. Use `client.rest.request<T>(...)` directly only when you need a custom typed call, and `client.rest.observe(...)` to instrument REST traffic.

CDN URLs are a separate proxy exposed as `client.rest.cdn`.

## Key APIs (verified)

- `client.proxy` — getter returning `this.rest.proxy` (`src/client/base.ts:289`). Lazily built by `ApiHandler.proxy` (`src/api/api.ts:139-141`) as `(this._proxy_ ??= new Router(this).createProxy())`.
- `client.rest: ApiHandler<this>` — the low-level handler (`src/client/base.ts:155`, default token `'INVALID'` until `client.start`).
- `Router.createProxy(route = [])` (`src/api/Router.ts:29`) — recursive `Proxy` over a no-op fn. `get` of a non-method key appends a path segment (`createProxy([...route, key])`); the `apply` trap (calling a segment, e.g. `channels(id)`) appends its args as segments; a terminal HTTP-method key returns a function that calls `rest.request(KEY.toUpperCase(), '/'+route.join('/'), ...options)`.
- HTTP method keys are lowercase, from `enum ProxyRequestMethod = delete | get | patch | post | put` (`src/api/Router.ts:12-18`). Any other key is a path segment. Internally uppercased to `HttpMethods = 'GET' | 'DELETE' | 'PUT' | 'POST' | 'PATCH'` (`src/api/shared.ts:42`).
- `ApiHandler.request<T = unknown>(method: HttpMethods, url: \`/${string}\`, request: ApiRequestOptions = {}): Promise<T>` (`src/api/api.ts:295`).
- `ApiRequestOptions` (`src/api/shared.ts:30-40`): `{ body?, query?, files?, auth?, reason?, route?, unshift?, appendToFormData?, token? }`. `reason` → `X-Audit-Log-Reason` header; `query` → querystring; `files: RawFile[]` → multipart attachments.
- `RawFile` (`src/api/shared.ts:23-28`): `{ data: ArrayBuffer | Buffer | Uint8Array | ... | string; filename: string; contentType?; key? }`.
- `ApiHandler.observe(observer: RestObserver, opts?): RestObserverDisposer` (`src/api/api.ts:119-129`) — register stacking REST observers; returns a disposer. Hooks: `onRequest`, `onSuccess`, `onFail`, `onRatelimit`, each given a readonly payload (`src/api/shared.ts:44-78`). Observer errors are isolated and logged, never thrown.
- `ApiHandler.cdn = CDNRouter.createProxy()` (`src/api/api.ts:66`) — accessed as `client.rest.cdn`. Terminal key is `get`, returning a URL string (NOT part of `client.proxy`). `parseCDNURL(route, options)` handles `a_`-animated detection, `forceStatic`, extension, and `size` (`src/api/Router.ts:45-92`).
- `client.messages.write(channelId, body): Promise<MessageStructure>` (`src/common/shorters/messages.ts:16`) — resolves files, transforms the body, calls `client.proxy.channels(channelId).messages.post(...)`, caches, then `Transformers.Message(...)`.
- `client.reactions.add(messageId, channelId, emoji): Promise<void>` (`src/common/shorters/reactions.ts:9`) — example of a shorter wrapping a `PUT …/reactions/{emoji}/@me`.
- `member.guild(mode?)` (`src/structures/GuildMember.ts:61-73`) — overloaded: default/`'rest'`/`'flow'` return `Promise<GuildStructure<'cached' | 'api'>>`; `'cache'` returns `ReturnCache<GuildStructure<'cached'> | undefined>`. Default is `'flow'` (cache, then fetch); `'rest'` forces a network fetch. Same overload shape on `channels`, `Emoji`, `GuildRole`, `GuildBan`, `AutoModerationRule`.
- `user.avatarURL(options?)` / `member.avatarURL()` / `guild.iconURL()` etc. — structure CDN helpers built on `client.rest.cdn` (`src/structures/User.ts:41-56`).
- Channel narrowing guards used before route access: `channel.isThreadOnly(): this is ForumChannel | MediaChannel` (`src/structures/channels.ts:168`), plus `isThread()` (128), `isVoice()` (136), `isTextGuild()` (140).

Root imports (`createEvent`, etc.) come from `'seyfert'`. `client.proxy` / `client.rest` are accessed off the client instance, so no deep import is needed for normal use.

## Code Examples (verified)

### Shorter — send a message by channel id (no cache/object lookup)

```ts
import { createEvent } from 'seyfert';

const db = new Map<string, string>();

export default createEvent({
	data: { name: 'guildMemberAdd' },
	run: async (member, client) => {
		const channelId = db.get(member.guildId);
		if (!channelId) return;

		await client.messages.write(channelId, {
			content: `Welcome ${member} :wave:`,
		});
	},
});
```

### Structure shorter — `member.guild()` (default `'flow'`: cache first, then REST)

```ts
import { createEvent } from 'seyfert';

const db = new Map<string, string>();

export default createEvent({
	data: { name: 'guildMemberAdd' },
	run: async (member, client) => {
		const channelId = db.get(member.guildId);
		if (!channelId) return;

		const guild = await member.guild(); // pass 'rest' to force a direct API fetch
		await client.messages.write(channelId, {
			content: `Welcome ${member} to ${guild.name} :wave:`,
		});
	},
});
```

### Proxy — call a raw endpoint that has no shorter (create a thread in a forum/media channel)

```ts
import { createEvent } from 'seyfert';

export default createEvent({
	data: { name: 'channelCreate' },
	run: async (channel, client) => {
		if (!channel.isThreadOnly()) return; // narrows to ForumChannel | MediaChannel

		// path mirrors Discord endpoints: POST /channels/{id}/threads
		await client.proxy.channels(channel.id).threads.post({
			body: {
				name: 'First thread!',
				message: { content: 'Seyfert >' },
			},
			reason: "I'm always the first", // -> X-Audit-Log-Reason header
		});
	},
});
```

### Proxy shape cheatsheet

```ts
// GET  /channels/{id}
await client.proxy.channels(channelId).get();
// PATCH /guilds/{id}  with audit reason
await client.proxy.guilds(guildId).patch({ body: { name: 'New name' }, reason: 'rename' });
// bulk overwrite global app commands
await client.proxy.applications(appId).commands.put({ body: [] });
// query params -> ?limit=50
await client.proxy.channels(channelId).messages.get({ query: { limit: 50 } });
// PUT with no body: add a reaction (encode the emoji segment yourself)
await client.proxy
	.channels(channelId)
	.messages(messageId)
	.reactions(encodeURIComponent('🔥'))('@me')
	.put();
// DELETE with audit reason
await client.proxy.channels(channelId).delete({ reason: 'cleanup' });
```

> The proxy returns RAW Discord JSON (snake_case). The equivalent shorter `client.reactions.add(messageId, channelId, '🔥')` handles encoding for you and returns `void`.

### Attachments — shorter (resolves files) vs raw proxy (`RawFile[]`)

```ts
import { createEvent, AttachmentBuilder } from 'seyfert';
import { readFileSync } from 'node:fs';

export default createEvent({
	data: { name: 'messageCreate' },
	run: async (message, client) => {
		// Shorter: pass builders/paths; resolveFiles + multipart handled for you.
		await client.messages.write(message.channelId, {
			content: 'Here you go',
			files: [new AttachmentBuilder({ filename: 'note.txt' }).setFile('buffer', Buffer.from('hi'))],
		});

		// Raw proxy: you build the multipart parts yourself as RawFile[].
		await client.proxy.channels(message.channelId).messages.post({
			body: { content: 'raw upload', attachments: [{ id: '0', filename: 'data.bin' }] },
			files: [{ filename: 'data.bin', data: readFileSync('./data.bin') }],
		});
	},
});
```

### Typed raw request — `client.rest.request<T>` for niche/unwrapped routes

```ts
import type { RESTGetAPIGatewayBotResult } from 'seyfert/lib/types';

// The proxy already infers per-route types from APIRoutes, but request<T> lets
// you annotate a hand-built URL (e.g. dynamic route() strings) explicitly.
const info = await client.rest.request<RESTGetAPIGatewayBotResult>('GET', '/gateway/bot');
console.log(info.session_start_limit.remaining);
```

### REST observers — stack instrumentation without clobbering (`client.rest.observe`)

```ts
// In a setup/start hook. observe() returns a disposer; observers STACK
// (v5 replaced the single overwritable client.rest.onSuccessRequest callback).
const dispose = client.rest.observe({
	onRequest({ method, url }) {
		client.logger.debug(`REST -> ${method} ${url}`);
	},
	onSuccess({ method, url, response }) {
		metrics.increment('rest.success', { method, url, status: response.status });
	},
	onFail({ method, url, error }) {
		client.logger.warn(`REST x ${method} ${url}`, error);
	},
	onRatelimit({ url }) {
		client.logger.warn(`Ratelimited on ${url}`);
	},
});

// later, when the feature unloads:
dispose();
```

### CDN URLs — `client.rest.cdn` and structure helpers

```ts
// Structure helper (preferred): handles default avatars + animated detection.
const url = user.avatarURL({ size: 256, extension: 'png' });

// Direct CDN proxy: terminal key is `get`, returns a URL string (not a request).
const guildIconUrl = client.rest.cdn.icons(guildId).get(iconHash, { size: 512 });
const emojiUrl = client.rest.cdn.emojis(emojiId).get({ extension: 'gif' });
```

## Common patterns / gotchas

- **Call segments, don't index them.** `client.proxy.channels(id)` (function call, via the `apply` trap) is correct; `client.proxy.channels.id` adds a literal `"id"` path segment and produces `/channels/id`.
- **Terminal call is lowercase.** End every proxy chain with `get/post/put/patch/delete` (lowercase). Each takes a single optional `ApiRequestOptions` object.
- **Proxy ≠ structures.** Proxy responses are raw snake_case JSON with no caching/transform. Prefer a shorter when one exists; drop to the proxy only for routes with no shorter or bleeding-edge API versions.
- **Encode dynamic emoji/path parts yourself** when using the raw proxy (`encodeURIComponent('🔥')`); shorters like `client.reactions.add` do it for you.
- **`reason` everywhere.** Any proxy call (and most shorters) accept `reason` for the audit log. `query` for querystrings; `files: RawFile[]` for attachments (raw proxy needs a matching `attachments` array in the body).
- **`member.guild('cache')` is the cache-only read** returning `ReturnCache<... | undefined>` (sync on a sync adapter, a Promise on an async one). Default `'flow'` falls back to REST; `'rest'` forces the network.
- **Observers stack and are isolated** (v5): plugin observers run first, then app `observe(...)` registrations, then any legacy callbacks. A throwing observer is caught and logged, never bubbled.

## Doc vs Source Corrections

- The MDX comment frames `member.guild()` as "a fetch request to cache (force it if you want a direct api fetch)." Source is more precise: default mode is `'flow'` (read cache, fall back to REST); pass `'rest'` to force a network fetch or `'cache'` for a cache-only `ReturnCache` read (`src/structures/GuildMember.ts:61-73`).
- CDN access is exposed as `client.rest.cdn` (an `ApiHandler` field, `src/api/api.ts:66`), NOT a top-level `client.cdn`. It is a distinct proxy from `client.proxy`; its terminal key is `get` and returns a URL string.
- v5 REST instrumentation: the old single `client.rest.onSuccessRequest = …` (last-assignment-wins) is superseded by `client.rest.observe({...})`, which stacks and returns a disposer. App code (not just plugins) may register observers.
- Otherwise the doc examples (`client.messages.write`, `client.proxy.channels(id).threads.post`, `isThreadOnly`) match source exactly. No API renames.

## Source Anchors

- `src/api/Router.ts` (proxy construction, `ProxyRequestMethod`, `CDNRouter`, `parseCDNURL`)
- `src/api/api.ts` (`ApiHandler`, `proxy` getter :139, `cdn` :66, `request` :295, `observe` :119)
- `src/api/shared.ts` (`ApiRequestOptions`, `HttpMethods`, `RawFile`, `RestObserver*`, `ApiHandlerOptions`)
- `src/api/Routes/channels.ts` (`threads` route presence)
- `src/client/base.ts` (`rest` :155, `proxy` getter :289)
- `src/common/shorters/messages.ts` (`MessageShorter.write` :16), `src/common/shorters/reactions.ts` (`add` :9)
- `src/structures/GuildMember.ts` (`guild()` overloads :61), `src/structures/channels.ts` (`isThreadOnly` :168), `src/structures/User.ts` (CDN URL helpers :41)

## Agent Guidance

- Prefer a shorter when one exists (`client.messages.*`, `client.channels.*`, `client.reactions.*`, structure helpers); they handle file resolution, body transformation, caching, and return rich structures. Drop to `client.proxy` only for endpoints with no shorter or new/in-development API versions.
- Proxy returns RAW Discord API payloads (snake_case JSON), NOT Seyfert structures — no caching, no transform. The terminal call is one of `get/post/put/patch/delete` (lowercase) and takes a single `ApiRequestOptions` object.
- IDs and dynamic path parts MUST be passed as function arguments: `client.proxy.channels(id)`, not `client.proxy.channels.id`.
- Use `reason` for audit-log entries, `query` for querystrings, `files` (`RawFile[]`) for attachments. `body` is sent as JSON unless files force multipart (then mirror them in `body.attachments`).
- For typed return values, the proxy already infers per-route types from `APIRoutes`; supply a generic only on the low-level handler: `client.rest.request<T>(method, url, opts)`.
- Instrument REST with `client.rest.observe({...})` and keep the returned disposer for teardown — do not overwrite handler callbacks.
- CDN URLs come from `client.rest.cdn.<segment>(...).get(...)` or, preferably, structure helpers (`user.avatarURL()`, `guild.iconURL()`).
