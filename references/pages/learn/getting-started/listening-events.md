# Listening to Events

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/listening-events

Coverage reference: setup-runtime.md

Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

Event modules are files that default-export `createEvent({ data: { name, once? }, run })`. Seyfert auto-loads them from the folder set at `locations.events` in `seyfert.config`, but only gateway/worker clients dispatch Discord gateway events — pure HTTP apps skip this entirely. Beyond Discord gateway events you can declare custom events via the `CustomEvents` module-augmentation interface and fire them with `client.events.runCustom(name, ...args)` (alias `emit`). Note `client.start()` loads and runs the bot but does NOT upload commands (that is the separate `client.uploadCommands()`).

## Key APIs (verified)

- `createEvent<E extends ClientNameEvents | CustomEventsKeys>(data: { data: { name: E; once?: boolean }; run: (...args: ResolveEventParams<E>) => Awaitable<unknown> })` — root export `seyfert` (src/index.ts:68). Mutates `once` to default `false` if omitted (`data.data.once ??= false`), then returns the object unchanged. `run` is typed `Awaitable<unknown>` in v5.
- `config.bot(data: RuntimeConfig)` / `config.http(data: RuntimeConfigHTTP)` — `seyfert` (src/index.ts:76,83). `config.bot` resolves `intents` via `resolveGatewayIntents` (src/index.ts:86). `RuntimeConfigHTTP.locations` Omits `events` (HTTP has no gateway events) (src/client/base.ts).
- `locations.events?: string` — optional sub-path under `locations.base`; resolved to a full path at load time (src/client/base.ts).
- `Client.events: EventHandler` (non-optional) (src/client/client.ts:49); `WorkerClient.events: EventHandler` (src/client/workerclient.ts:87). `BaseClient` (and thus `HttpClient`) only has `events: CustomEventRunner = new CustomEventHandler(this)` — custom events work but NO gateway dispatch (src/client/base.ts:197).
- `client.loadEvents(dir?)` — defaults `dir` to `locations.events`; called by `Client.start()` (src/client/client.ts).
- `events.runCustom<T extends CustomEventsKeys>(name, ...args)` and its alias `events.emit(name, ...args)` — fire a custom event; `args` are `ResolveEventRunParams<T> = Parameters<CustomEvents[T]>` only (the `UsingClient` is appended automatically before calling `run`) (src/events/handler.ts:95-99, 118/125).
- `events.reload(name)` / `events.reloadAll(stopIfFail?)` — hot-reload by file path; throws `SeyfertError('RELOAD_NOT_SUPPORTED')` on Cloudflare Workers (src/events/handler.ts:430).
- Type helpers (deep import `seyfert/lib/events` if needed): `ClientNameEvents`, `CustomEventsKeys`, `CustomEvents`, `ResolveEventParams`, `ResolveEventRunParams`, `EventContext`, `GatewayEvents` (src/events/event.ts, src/events/handler.ts).

### Event run() parameters (verified)
- Gateway/client events: `run(payload, client, shardId)` where `payload = Awaited<ClientEvents[name]>`, `client: UsingClient`, `shardId: number` (src/events/event.ts:22-23; called as `Event.run(hook, client, shardId)` at src/events/handler.ts:410,417). Gateway handlers KEEP `shardId` in v5.
- Custom events: `run(...customArgs, client)` — your declared args followed by the client; NO trailing `shardId` (src/events/handler.ts:118,125; type src/events/event.ts:25).
- `ClientEvents[name] = ReturnType<HOOK>` and `payload = Awaited<...>` (src/events/hooks/index.ts:25). Most hooks return a transformed structure; a few return tuples (see gotchas).

### Built-in custom events (ship by default, no augmentation)
`CustomEvents` already declares three events (src/events/event.ts:7-13): `commandsLoaded` (`PluginLoadedMetadata`), `componentsLoaded`, `uploadCommands` (`PluginUploadCommandsMetadata`). You can `createEvent`/listen to these without augmenting anything.

### Common event payload shapes (verified hooks)
- `botReady`: payload is the bot `ClientUser` (`me.username`) (src/events/hooks/custom.ts:6 `BOT_READY`).
- `guildDelete`: payload is `GuildStructure<'cached'> | APIUnavailableGuild`; check `.unavailable` to distinguish a real removal from an outage (src/events/hooks/custom.ts:25-30 `RAW_GUILD_DELETE`).
- `messageCreate`: payload is a `MessageStructure` (src/events/hooks/message.ts:16). Needs the `MessageContent` intent to read `.content`.
- `interactionCreate`: payload is the resolved interaction (src/events/hooks/interactions.ts:5). Seyfert routes commands/components itself — only use this for raw inspection.
- `guildMemberAdd`: payload is a `GuildMemberStructure` (src/events/hooks/guild.ts:86). Needs the `GuildMembers` privileged intent.
- `voiceStateUpdate`: payload is a TUPLE `[state: VoiceStateStructure, old?: VoiceStateStructure]` — destructure the first param (src/events/hooks/voice.ts:16-22).
- `voiceChannelStatusUpdate` (`VOICE_CHANNEL_STATUS_UPDATE`): payload is `[status, channel: VoiceChannel | undefined]` — channel is resolved, not a raw cache promise (src/events/hooks/voice.ts:28-36).

## Code Examples (verified)

seyfert.config.mjs — declare the events folder:

```ts
import { config } from 'seyfert';

export default config.bot({
  token: process.env.BOT_TOKEN ?? '',
  intents: ['Guilds'], // v5: inline strings; resolved via resolveGatewayIntents
  locations: {
    base: 'dist',
    commands: 'commands',
    events: 'events', // resolves to <base>/events at runtime
  },
});
```

src/events/botReady.ts:

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  // `once: true` runs the handler a single time.
  data: { once: true, name: 'botReady' },
  run(user, client) {
    client.logger.info(`${user.username} is ready`);
  },
});
```

src/events/guildDelete.ts:

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'guildDelete' }, // once defaults to false
  run(unguild, client) {
    if (unguild.unavailable) return; // outage, not a real removal
    client.logger.info(`I was removed from: ${unguild.id}`);
  },
});
```

### messageCreate — react to plain messages (needs `MessageContent`)

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'messageCreate' },
  // payload is a MessageStructure; shardId is available as the 3rd arg.
  run(message, client) {
    if (message.author.bot) return;
    if (message.content === '!ping') {
      message.reply({ content: 'Pong!' });
    }
  },
});
```

### guildMemberAdd — welcome new members (needs `GuildMembers` intent)

```ts
import { createEvent, Formatter } from 'seyfert';

export default createEvent({
  data: { name: 'guildMemberAdd' },
  async run(member, client) {
    const channel = await client.channels.fetch(/* welcome channel id */ '123');
    if (!channel.isTextGuild()) return;
    await channel.messages.write({
      content: `Welcome ${Formatter.userMention(member.id)} to the server!`,
    });
  },
});
```

### voiceStateUpdate — payload is a tuple `[state, old?]`

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'voiceStateUpdate' },
  // First param is destructured: the new state, plus the previous (cached) one.
  run([state, old], client) {
    if (!old?.channelId && state.channelId) {
      client.logger.info(`${state.userId} joined ${state.channelId}`);
    } else if (old?.channelId && !state.channelId) {
      client.logger.info(`${state.userId} left ${old.channelId}`);
    }
  },
});
```

### Using shardId (3rd gateway arg)

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { once: true, name: 'botReady' },
  run(user, client, shardId) {
    client.logger.info(`Ready as ${user.username} (shard #${shardId})`);
  },
});
```

### Custom events — declare the type (module augmentation)

```ts
// index.ts
declare module 'seyfert' {
  interface CustomEvents {
    ourEvent: (text: string) => void;
    levelUp: (userId: string, level: number) => void;
  }
}
```

Custom event module (placed in the events folder, auto-loaded):

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'ourEvent', once: false },
  // run receives your declared args followed by the client (no shardId).
  run: (text, client) => {
    client.logger.info(text);
  },
});
```

Firing the custom event (from anywhere with a client reference):

```ts
import { Client } from 'seyfert';

const client = new Client();

(async () => {
  await client.start();
  // runCustom/emit args do NOT include the client; it is injected into run().
  client.events.runCustom('ourEvent', 'Hello, world!');
})();
```

### Fire a custom event from inside a command

```ts
import { Command, type CommandContext } from 'seyfert';

export default class Level extends Command {
  async run(ctx: CommandContext) {
    // ctx.client.events is the same EventHandler; emit is an alias of runCustom.
    ctx.client.events.emit('levelUp', ctx.author.id, 5);
    await ctx.write({ content: 'Leveled up!' });
  }
}
```

### Listen to a built-in custom event (no augmentation needed)

```ts
import { createEvent } from 'seyfert';

// Fires after Seyfert (or a plugin) uploads application commands.
export default createEvent({
  data: { name: 'uploadCommands', once: false },
  run(metadata, client) {
    client.logger.info('Commands uploaded', metadata);
  },
});
```

### Multiple events in one file (loader flattens an array)

```ts
import { createEvent } from 'seyfert';

// A default-exported ARRAY of createEvent(...) is flattened on load.
export default [
  createEvent({ data: { name: 'guildCreate' }, run: (g, c) => c.logger.info(`+${g.id}`) }),
  createEvent({ data: { name: 'guildDelete' }, run: (g, c) => c.logger.info(`-${g.id}`) }),
];
```

(src/events/handler.ts:460-462 — `onFile` returns `Array.isArray(default) ? default : [default]`.)

## Common patterns / gotchas

- One handler per event `name`: the loader stores by normalized name in `values`, so a second file with the same name OVERWRITES the first (src/events/handler.ts:262,285). Combine logic into one handler instead of registering twice.
- `voiceStateUpdate` (and `voiceChannelStatusUpdate`) deliver a TUPLE as the first arg — destructure `[state, old]`, do not treat it as a single object.
- Gateway handlers receive `shardId` as the 3rd arg; CUSTOM handlers do NOT — they get `(...yourArgs, client)`. Don't expect a shardId in custom events.
- Throwing inside a `once` handler resets `fired` so it can fire again next time (src/events/handler.ts:411-414); a failing once-handler is not permanently consumed.
- HTTP-only bots (`HttpClient`) have no gateway dispatch — `createEvent` files for Discord events won't fire. Custom events via `runCustom`/`emit` still work on the base `CustomEventHandler`.
- Custom events need `CustomEvents` augmentation for typing (except the three built-ins); put the `declare module 'seyfert'` block somewhere that gets compiled (e.g. index.ts).
- Don't pass the client to `runCustom`/`emit` yourself — it is appended automatically before `run` executes. Passing it is a common mistake.
- Enable the intents an event needs: `messageCreate` content needs `MessageContent`, `guildMemberAdd`/voice needs `GuildMembers`/`GuildVoiceStates`. Intents are resolved from the `config.bot({ intents })` array.
- `client.start()` does NOT upload application commands — call `client.uploadCommands({ applicationId })` separately when needed.
- Collectors run alongside events: `client.collectors.run('messageCreate', ...)` uses the camelCase client name (not the raw gateway name), and in v5 EVERY matching collector runs (the old first-match `break` was removed).

## Doc vs Source Corrections

- Docs call `client.events?.runCustom(...)` with optional chaining. On `Client`/`WorkerClient`, `events` is non-optional (`EventHandler`), so `client.events.runCustom(...)` is fine; the `?.` is only meaningful where the value may be undefined (it is not for these classes) (src/client/client.ts:49).
- Docs custom-event handler `run: (text) => {...}` omits the trailing client param. That is valid (you may ignore it), but the client IS passed as the last argument at call time (src/events/handler.ts:118,125).
- Docs do not mention `once` defaults to `false` — confirmed it is filled in by `createEvent` (src/index.ts:72).
- Docs do not mention the built-in `CustomEvents` (`commandsLoaded`, `componentsLoaded`, `uploadCommands`) or the `emit` alias for `runCustom` (src/events/event.ts:7-13, src/events/handler.ts:95).
- Docs do not surface that some hook payloads are tuples (`voiceStateUpdate`, `voiceChannelStatusUpdate`); destructure accordingly (src/events/hooks/voice.ts).
- Otherwise the doc examples are accurate against source.

## Source Anchors

- src/client/httpclient.ts (no gateway events)

## Agent Guidance

- Event files must `export default createEvent(...)` (or an array of them); a file missing a `run` function is logged and skipped (src/events/handler.ts:258-261, 277-283).
