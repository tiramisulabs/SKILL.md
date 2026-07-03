# Music Library (Kazagumo / Lavalink)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/music

Coverage reference: i18n-cache-recipes.md

Verification status: Source-verified (core) + external package (kazagumo/shoukaku — verify versions in target project)

## Page Summary

Seyfert has no built-in audio playback. The recipe wires an external Lavalink client, `kazagumo` (built on `shoukaku`), into the Seyfert `Client` so commands can search and play tracks. The only Seyfert-specific glue is the gateway voice-payload `send` callback (using `calculateShardId` + `gateway.send`), a `declare module "seyfert"` augmentation to type `client.kazagumo`, and normal slash commands that read the member's voice state and write results. All player/queue/track APIs (`Kazagumo`, `createPlayer`, `search`, `player.queue`, `player.play`) are external and target-project specific.

## Key APIs (verified)

Seyfert core APIs touched by the recipe (all importable from the `seyfert` root barrel):

- `Client` — main client class. `src/client/client.ts` (re-exported via `src/index.ts`).
- `client.gateway: ShardManager` — `gateway!: ShardManager` field on `Client`, `src/client/client.ts:41`.
- `ShardManager.calculateShardId(guildId: string): number` — `src/websocket/discord/sharder.ts:89`.
- `ShardManager.send<T extends GatewaySendPayload>(shardId: number, payload: T)` — `src/websocket/discord/sharder.ts:299`. Takes a numeric `shardId` FIRST, then the payload; this matches the recipe's `send: (guildId, payload) => client.gateway.send(client.gateway.calculateShardId(guildId), payload)`.
- `ShardManager.joinVoice(guildId, channelId, { self_deaf, self_mute })` — `src/websocket/discord/sharder.ts:254`; `ShardManager.leaveVoice(guildId)` — `:271`. Native Seyfert voice-state helpers if you ever wire voice without a library that does it for you.
- `client.start()` then `client.uploadCommands({ applicationId?, cachePath? })` — `src/client/base.ts:1056`.
- `Command`, `Declare`, `Options` decorators + `createStringOption` — `src/commands/decorators.ts` and `src/commands/applications/options.ts:146` (barrel `src/commands/index.ts`).
- `CommandContext` — generic over the options object; exposes `options`, `client`, `guildId`, `channelId`, `member`, `author`. `src/commands/applications/chatcontext.ts` (getters: guildId :236, channelId :240, author :244, member :248).
- `ctx.write(body, withResponse?)` / `ctx.editOrReply(body, withResponse?)` — `src/commands/applications/chatcontext.ts`. Return `void` unless `withResponse === true`.
- `ctx.me(mode?)` — bot's `GuildMember`; `src/commands/applications/chatcontext.ts:188`. In a guild context (`ctx.inGuild()` / `GuildCommandContext`) the non-undefined overload at :275 resolves a `GuildMemberStructure`.
- `GuildMember.voice(mode?: 'rest' | 'flow' | 'cache')` — returns a `VoiceStateStructure` (or cached `| undefined`); `src/structures/GuildMember.ts:95`. Defaults to `'flow'` (cache → rest).
- `MessageFlags.Ephemeral` — enum in `src/types/payloads/channel.ts:779`, re-exported through `src/types` → `src/index.ts`, so `import { MessageFlags } from "seyfert"` is valid.
- `Embed` builder — `src/builders/Embed.ts` (barrel `src/builders/index.ts:33`).
- `createEvent({ data: { name, once? }, run })` — `src/index.ts:68`. `run` is typed `Awaitable<unknown>`; custom/non-gateway handlers no longer receive a trailing `shardId` (v5).

External (NOT in core Seyfert — verify versions in the target project):

- `kazagumo`: `Kazagumo`, `createPlayer`, `search`, `players.get`, `player.queue.add`, `player.skip`, `player.pause`, `player.setVolume`, `player.destroy`, `player.play`, `player.playing`, `player.paused`, `player.queue.current`, result `type === "PLAYLIST"`, `result.tracks`, `result.playlistName`. Kazagumo emits `playerStart`, `playerEnd`, `playerEmpty` events.
- `shoukaku`: `Connectors.Seyfert`, `NodeOption`, `kazagumo.shoukaku.on("ready", ...)`.

## Code Examples (verified)

### index.ts — client + Kazagumo bootstrap

Core glue verified; kazagumo/shoukaku external.

```ts
import { Client } from "seyfert";
import { Kazagumo } from "kazagumo";
import { type NodeOption, Connectors } from "shoukaku";

const client = new Client();

const nodes: NodeOption[] = [
  { name: "Node", url: "localhost:2333", auth: "youshallnotpass", secure: false },
];

client.kazagumo = new Kazagumo(
  {
    defaultSearchEngine: "youtube",
    // calculateShardId -> number, then gateway.send(shardId, payload)
    send: (guildId, payload) =>
      client.gateway.send(client.gateway.calculateShardId(guildId), payload),
  },
  new Connectors.Seyfert(client),
  nodes
);

client.kazagumo.shoukaku.on("ready", (name) =>
  console.log(`Lavalink ${name}: Ready!`)
);

declare module "seyfert" {
  interface Client {
    kazagumo: Kazagumo;
  }
}

client.start().then(() => client.uploadCommands());
```

Note: `declare module "seyfert" { interface Client { kazagumo } }` augments the `Client` class directly — this is independent of the v5 `SeyfertRegistry` augmentation (that one is for `client`/`middlewares`/`langs`/`plugins`). Adding a plain instance field to `Client` is still done this way.

### src/commands/play.ts — search & play

Core APIs verified; kazagumo calls external.

```ts
import {
  Command,
  Declare,
  Options,
  MessageFlags,
  createStringOption,
  type CommandContext,
} from "seyfert";

const options = {
  // v5: option record keys MUST be lowercase (compile-time enforced)
  query: createStringOption({
    description: "Enter a song name or url.",
    required: true,
  }),
};

@Declare({ name: "play", description: "Play music." })
@Options(options)
export default class PlayCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const { options, client, guildId, channelId, member, author } = ctx;
    const { query } = options;

    if (!guildId || !member) return;

    const voice = await member.voice();
    if (!voice)
      return ctx.write({
        content: "You must be in a voice channel to play music.",
        flags: MessageFlags.Ephemeral,
      });

    const botVoice = await ctx.me()?.voice();
    if (botVoice && botVoice.channelId !== voice.channelId)
      return ctx.write({
        content: "You must be in the same voice channel as me.",
        flags: MessageFlags.Ephemeral,
      });

    // external (kazagumo) from here down
    const player = await client.kazagumo.createPlayer({
      guildId,
      textId: channelId,
      voiceId: voice.channelId,
      volume: 100,
    });

    const result = await client.kazagumo.search(query, { requester: author });
    if (!result.tracks.length) return ctx.write({ content: "No results found!" });

    if (result.type === "PLAYLIST") player.queue.add(result.tracks);
    else player.queue.add(result.tracks[0]);

    if (!player.playing && !player.paused) player.play();

    return ctx.write({
      content:
        result.type === "PLAYLIST"
          ? `Queued ${result.tracks.length} from ${result.playlistName}`
          : `Queued ${result.tracks[0].title}`,
    });
  }
}
```

Note: `member.voice()` and `ctx.me()?.voice()` default to `'flow'` mode (cache → rest); they can throw/reject if the voice state cannot be resolved. Guard accordingly in production.

### NEW — A shared voice guard (DRY across music commands)

Pattern, not a new API: factor the "must share a voice channel + a player must exist" check so every control command stays short. Returns the player or `null` after replying.

```ts
import { MessageFlags, type GuildCommandContext } from "seyfert";

// Use GuildCommandContext: guildId is non-null and member/me() are narrowed.
export async function requirePlayer(ctx: GuildCommandContext) {
  const member = ctx.member; // non-undefined in a guild context
  const voice = await member.voice("cache"); // sync cache read; undefined if not in VC
  if (!voice?.channelId) {
    await ctx.write({
      content: "Join a voice channel first.",
      flags: MessageFlags.Ephemeral,
    });
    return null;
  }

  const player = ctx.client.kazagumo.players.get(ctx.guildId); // external
  if (!player) {
    await ctx.write({ content: "Nothing is playing.", flags: MessageFlags.Ephemeral });
    return null;
  }

  if (player.voiceId && player.voiceId !== voice.channelId) {
    await ctx.write({
      content: "You must be in my voice channel.",
      flags: MessageFlags.Ephemeral,
    });
    return null;
  }
  return player;
}
```

### NEW — skip / pause / stop / volume control commands

Each command runs inside a guild; declare with `Command` and use `ctx.inGuild()` (or the `GuildCommandContext` type) so `guildId` is non-null. kazagumo player methods are external.

```ts
import { Command, Declare, type CommandContext } from "seyfert";
import { requirePlayer } from "./shared-voice"; // the guard above

@Declare({ name: "skip", description: "Skip the current track." })
export default class SkipCommand extends Command {
  async run(ctx: CommandContext) {
    if (!ctx.inGuild()) return; // narrows ctx to GuildCommandContext
    const player = await requirePlayer(ctx);
    if (!player) return;

    if (!player.queue.current) return ctx.write({ content: "Nothing to skip." });
    const title = player.queue.current.title;
    player.skip(); // external
    return ctx.write({ content: `Skipped **${title}**.` });
  }
}
```

```ts
import { Command, Declare, Options, createNumberOption, type CommandContext } from "seyfert";
import { requirePlayer } from "./shared-voice";

const options = {
  // lowercase key (v5); numeric option -> autocomplete/choices must be numbers
  percent: createNumberOption({
    description: "Volume 0-200",
    required: true,
    min_value: 0,
    max_value: 200,
  }),
};

@Declare({ name: "volume", description: "Set playback volume." })
@Options(options)
export default class VolumeCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    if (!ctx.inGuild()) return;
    const player = await requirePlayer(ctx);
    if (!player) return;

    player.setVolume(ctx.options.percent); // external
    return ctx.write({ content: `Volume set to ${ctx.options.percent}%.` });
  }
}
```

```ts
import { Command, Declare, type CommandContext } from "seyfert";
import { requirePlayer } from "./shared-voice";

@Declare({ name: "stop", description: "Stop and leave the channel." })
export default class StopCommand extends Command {
  async run(ctx: CommandContext) {
    if (!ctx.inGuild()) return;
    const player = await requirePlayer(ctx);
    if (!player) return;

    player.destroy(); // external: clears queue + leaves voice
    return ctx.write({ content: "Stopped and left the channel." });
  }
}
```

### NEW — `nowplaying` with an Embed

Builds a Seyfert `Embed` (core) from the kazagumo current track (external).

```ts
import { Command, Declare, Embed, MessageFlags, type CommandContext } from "seyfert";

@Declare({ name: "nowplaying", description: "Show the current track." })
export default class NowPlayingCommand extends Command {
  async run(ctx: CommandContext) {
    if (!ctx.inGuild()) return;

    const player = ctx.client.kazagumo.players.get(ctx.guildId); // external
    const track = player?.queue.current;
    if (!track)
      return ctx.write({ content: "Nothing is playing.", flags: MessageFlags.Ephemeral });

    const embed = new Embed()
      .setTitle(track.title)
      .setURL(track.uri)
      .setDescription(`Requested by ${track.requester}`)
      .setThumbnail(track.thumbnail ?? undefined)
      .addFields({ name: "Queue", value: `${player.queue.length} track(s) waiting` })
      .setColor("Blurple");

    return ctx.write({ embeds: [embed] });
  }
}
```

### NEW — kazagumo player events → post to the text channel

kazagumo's own emitter (external) drives "now playing" / "queue ended" messages. Use the Seyfert `client.messages.write` shorter to send to the stored `textId`. Wire this once, near the bootstrap.

```ts
// in index.ts after the Kazagumo instance is created
client.kazagumo.on("playerStart", (player, track) => {
  if (player.textId)
    client.messages.write(player.textId, { content: `Now playing: **${track.title}**` });
});

client.kazagumo.on("playerEmpty", (player) => {
  if (player.textId)
    client.messages.write(player.textId, { content: "Queue ended. Leaving." });
  player.destroy(); // external: leave when idle
});
```

`client.messages.write(channelId, body)` is core (`MessageShorter`, `src/structures/`); the `playerStart`/`playerEmpty` events and `track` shape are kazagumo's (external).

### NEW — auto-leave when the channel empties (Seyfert event)

A core `voiceStateUpdate` event listener that destroys the player if the bot is left alone. Uses `createEvent` — note v5 custom-event handlers have no trailing `shardId`, but gateway events like this still pass `(payload, client)`.

```ts
// src/events/voiceLeave.ts
import { createEvent } from "seyfert";

export default createEvent({
  data: { name: "voiceStateUpdate" },
  async run([newState, oldState], client) {
    const guildId = newState?.guildId ?? oldState?.guildId;
    if (!guildId) return;

    const player = client.kazagumo.players.get(guildId); // external
    if (!player?.voiceId) return;

    // Count non-bot members in the bot's voice channel (cache read).
    const states = await client.cache.voiceStates?.values(guildId);
    const humans = (states ?? []).filter(
      (s) => s.channelId === player.voiceId,
    );
    if (humans.length <= 1) player.destroy(); // only the bot remains
  },
});
```

`voiceStateUpdate` fires `[VoiceState, VoiceState | undefined]` (new, then old). `client.cache.voiceStates` is core but requires the `GuildVoiceStates` intent AND voice-state caching enabled; the exact `.values()` shape depends on your cache adapter — verify in-project.

## Doc vs Source Corrections

- Import style: the MDX splits imports into two `from "seyfert"` statements and lists `MessageFlags` separately. Source confirms a single barrel works — `MessageFlags`, the decorators, `createStringOption`, `Embed`, `createEvent`, and `CommandContext` are all exported from the `seyfert` root. Consolidated in the examples. (cosmetic; doc not wrong, just split)
- `send` callback signature: doc is correct against source — `ShardManager.send(shardId: number, payload)` takes the numeric shard id first (`src/websocket/discord/sharder.ts:299`), which is why `calculateShardId(guildId)` is called inside. Flagged because it is easy to misread as `send(guildId, payload)`.
- v5 deltas applied to the added examples (NOT in the original MDX, which only shows `play`): lowercase option keys are now compile-time enforced; numeric options (`createNumberOption`) require numeric `choices`/autocomplete values; `ctx.inGuild()` narrows to `GuildCommandContext` so `guildId`/`member`/`me()` are non-undefined; `createEvent`'s `run` is `Awaitable<unknown>` and custom (non-gateway) handlers lost the trailing `shardId`.
- The kazagumo/shoukaku surface (`Kazagumo`, `Connectors.Seyfert`, `createPlayer`, `search`, `players.get`, `player.skip/destroy/setVolume`, events, queue/track shapes) is external and unverifiable against core Seyfert — verify against the installed package versions.

## Source Anchors

- `src/index.ts` (root barrel: re-exports `./commands`, `./types`, `./builders`; defines `createEvent`)
- `src/client/client.ts` (`Client`, `gateway` field)
- `src/websocket/discord/sharder.ts` (`calculateShardId`, `send`, `joinVoice`, `leaveVoice`)
- `src/client/base.ts` (`uploadCommands`)
- `src/commands/applications/options.ts` (`createStringOption`, `createNumberOption`)
- `src/commands/applications/chatcontext.ts` (`CommandContext`, `me`, `write`, `inGuild`, getters)
- `src/structures/GuildMember.ts` (`voice`)
- `src/builders/Embed.ts` (`Embed`)
- `src/types/payloads/channel.ts` (`MessageFlags`)

## Agent Guidance

- Use when adding music/voice playback to a Seyfert bot. Seyfert itself only provides the gateway and command plumbing; pick a Lavalink client (kazagumo here, or lavalink-client / poru) and run a Lavalink server (the `localhost:2333` node).
- The single mandatory Seyfert integration point is the `send` callback: `(guildId, payload) => client.gateway.send(client.gateway.calculateShardId(guildId), payload)`. Get this wrong and voice connections silently never establish.
- Add `declare module "seyfert" { interface Client { kazagumo: Kazagumo } }` so `client.kazagumo` is typed everywhere (commands, events). This augments the `Client` class directly and is separate from the v5 `SeyfertRegistry` augmentation. Requires the declare-module setup to be loaded — see learn/getting-started/declare-module.
- Ensure the `GuildVoiceStates` gateway intent is enabled (in `seyfert.config`/client intents) or `member.voice()`, the bot's voice state, and the auto-leave cache read will not resolve.

## Common patterns / gotchas

- **Narrow to guild early.** Music is guild-only. Call `if (!ctx.inGuild()) return;` (or type the handler as `GuildCommandContext`) so `guildId` is `string` and `member`/`ctx.me()` are non-undefined — no `!guildId || !member` boilerplate, no `?.` everywhere.
- **`voice()` mode choice.** `member.voice('cache')` is a synchronous cache read (`VoiceStateStructure | undefined`); `member.voice()` defaults to `'flow'` (cache → REST) and returns a Promise that can reject. Prefer `'cache'` in hot paths/guards; use `'flow'` only when you must hit REST.
- **`write`/`editOrReply` return `void`** unless you pass `withResponse: true` (v5). Don't assign the result expecting a `Message` without the flag.
- **Lowercase option keys + typed numeric choices** are compile-time errors in v5. `createNumberOption`/`createIntegerOption` autocomplete and `choices` values must be numbers; mark `choices`/`channel_types` arrays `as const`.
- **Player lifecycle is the library's, not Seyfert's.** Creating a player joins voice via your `send` callback; `player.destroy()` leaves and clears the queue. If you ever bypass the library, Seyfert exposes `client.gateway.joinVoice(...)` / `leaveVoice(...)` directly.
- **Always `player.destroy()` on empty/idle** (via `playerEmpty` or a `voiceStateUpdate` auto-leave) so the bot doesn't linger in empty channels.
- **Pin versions.** `kazagumo` + `shoukaku` APIs (`createPlayer`, `search`, result `type`/`playlistName`, event names) change across majors and are not guaranteed by core Seyfert. Verify them against the installed packages in the target project.
