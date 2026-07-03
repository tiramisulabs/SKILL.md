# Music Library (hoshimi / Lavalink v4)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/music

Coverage reference: i18n-cache-recipes.md

Verification status: Source-verified (core) + external package **hoshimi** verified against the `hoshimi` type declarations (`dist/index.d.cts`, `hoshimi@0.3.10-dev`) and real-world Seyfert + hoshimi bot usage. Always version-verify `hoshimi` in the target project.

## Page Summary

Seyfert has no built-in audio. The v5 docs recipe wires an **external** Lavalink v4 client, [`hoshimi`](https://npmjs.com/package/hoshimi), which bundles the node connection **and** the player (a single dependency). The core glue is minimal:

1. Attach a `Hoshimi` manager to the client (a `Client` subclass holds it).
2. Forward Discord's `raw` gateway packets to `manager.updateVoiceState(...)` so hoshimi can track voice state.
3. Call `manager.init({ id, username })` once on `botReady` so it can authenticate to the node.
4. Commands create/drive players (`createPlayer` → `connect` → `search` → `queue.add` → `play`).

Requires a running **Lavalink v4** node. In v5 the manager is typed through **`SeyfertRegistry.client`** (either `ParseClient<MusicClient>` or `ParseClient<Client<true>> & { hoshimi: Hoshimi }`) — NOT the v4-style `interface Client { ... }` augmentation.

## Key APIs (verified against `hoshimi` `dist/index.d.cts`)

The entire `hoshimi` surface is external and unverifiable against core Seyfert — confirmed against `hoshimi@0.3.10-dev`; version-verify in the target project.

- `new Hoshimi(options: HoshimiOptions)` — the manager (`extends EventEmitter<HoshimiEvents>`). `createHoshimi(options)` is an equivalent factory.
- `HoshimiOptions`: `sendPayload(guildId, payload): Awaitable<void>` **(required)**, `nodes: NodeOptions[]` **(required)** — `{ host, port, password, secure }`, `client?: Partial<ClientInfo>`, `defaultSearchSource?: SearchSource` (default `SearchSources.Youtube`), plus `queueOptions?` / `nodeOptions?` / `restOptions?` / `playerOptions?`.
- Manager: `players: Collection<string, PlayerStructure>`, `ready`, `getPlayer(guildId): PlayerStructure | undefined`, `deletePlayer(guildId): boolean`, `updateVoiceState(packet): Promise<void>`, `init(info: ClientInfo): void`, `createPlayer(options: PlayerOptions): PlayerStructure`, `search(options: SearchOptions): Promise<QueryResult>`.
- `ClientInfo` = `{ id: string; username?: string }`.
- `PlayerOptions` = `{ guildId: string; voiceId: string; volume?: number (100); textId?: string; selfDeaf?: boolean (true); selfMute?: boolean (false); node?: NodeIdentifier }`. **`createPlayer` is synchronous** and returns the player; call `await player.connect()` afterwards.
- `Player` (what `createPlayer` returns, `PlayerStructure`): `connect(): Promise<this>`, `disconnect()`, `search(options: SearchOptions): Promise<QueryResult>`, `play(options?): Promise<void>`, `stop()`, `skip(options?)`, `seek(pos)`, `destroy(options?)`, `setPaused(paused?): Promise<boolean>`, `setVolume(v): Promise<void>`, `setLoop(mode: LoopMode): this`; props `queue`, `playing`, `paused`, `connected`, `destroyed`, `volume`, `loop`, `position`, `guildId`, `voiceId`, `textId`, `lyrics`.
- `player.queue` (`Queue`): `add(track | track[], position?): Promise<this>`, `current: TrackStructure | null`, `previous(remove?)`, `tracks`.
- `SearchOptions` = `SearchQuery & { requester: TrackRequester | null; node? }`; `SearchQuery` = `{ query: string; source?: SearchSource | SourceName; params? }`.
- `QueryResult` = `{ loadType: LoadType; tracks: TrackStructure[]; playlist: Playlist | null; ... }`. `track.info.title` / `track.info.uri`; `playlist.info.name` (`PlaylistInfo`).
- Enums: `LoadType` (`Track="track"`, `Playlist="playlist"`, `Search="search"`, `Empty="empty"`, `Error="error"`), `LoopMode` (`Track=1`, `Queue=2`, `Off=3`), `SearchSources` (`Youtube="ytsearch"`, `YoutubeMusic="ytmsearch"`, `Spotify="spsearch"`, `SoundCloud`, …), `EventNames`, `DebugLevels`, `DestroyReasons`.
- Raw-packet types (for the `raw` event): `VoicePacket`, `VoiceServer`, `VoiceState`, `ChannelDeletePacket` (the method accepts the internal `GatewayPackets` union).
- Manager events (`HoshimiEvents` / `EventNames`): `nodeReady (node, retries, payload)`, `playerCreate (player)`, `trackStart (player, track, payload)`, `trackEnd`, `queueEnd (player, queue)`, `playerDestroy`, `error`, `debug`, … — `track` params are `TrackStructure | null`.

## Code Examples

### index.ts — MusicClient + Hoshimi bootstrap

Core glue verified; hoshimi external.

```ts
import { Client, type ParseClient } from 'seyfert';
import { Hoshimi, SearchSources } from 'hoshimi';

// A custom client that holds the hoshimi manager
class MusicClient extends Client {
  hoshimi = new Hoshimi({
    defaultSearchSource: SearchSources.Youtube,
    sendPayload: async (guildId, payload) => {
      await this.gateway.send(this.gateway.calculateShardId(guildId), payload);
    },
    nodes: [{ host: 'localhost', port: 2333, password: 'youshallnotpass', secure: false }],
  });
}

const client = new MusicClient();

// See whether the node connected
client.hoshimi.on('nodeReady', (node) => console.log(`Lavalink ${node.id}: Ready!`));

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<MusicClient>;
  }
}

client.start().then(() => client.uploadCommands());
```

If you keep a plain `Client` and attach the manager some other way, type it with the intersection form instead:

```ts
declare module 'seyfert' {
  interface SeyfertRegistry { client: ParseClient<Client<true>> & { hoshimi: Hoshimi } }
}
```

### src/events/raw.ts — forward gateway packets

Discord delivers voice updates through the `raw` event; forward them so hoshimi can track voice state.

```ts
import type { ChannelDeletePacket, VoicePacket, VoiceServer, VoiceState } from 'hoshimi';
import { createEvent } from 'seyfert';

type AnyPacket = VoicePacket | VoiceServer | VoiceState | ChannelDeletePacket;

export default createEvent({
  data: { name: 'raw' },
  run(payload, client) {
    client.hoshimi.updateVoiceState(payload as AnyPacket);
  },
});
```

### src/events/botReady.ts — authenticate to the node

Once the bot is ready we know its id, so hoshimi can connect to the nodes.

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'botReady', once: true },
  run(user, client) {
    client.hoshimi.init({ id: user.id, username: user.username });
  },
});
```

### src/commands/play.ts — search + queue + play

Core command plumbing verified; hoshimi (`createPlayer`, `search`, `queue`, `LoadType`) external.

```ts
import { Command, Declare, Options, MessageFlags, type CommandContext, createStringOption } from 'seyfert';
import { LoadType } from 'hoshimi';

const options = {
  query: createStringOption({ description: 'Enter a song name or url.', required: true }),
};

@Declare({ name: 'play', description: 'Play music.' })
@Options(options)
export default class PlayCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const { client, guildId, channelId, member, author } = ctx;
    const { query } = ctx.options;
    if (!guildId || !member) return;

    const voice = await member.voice();
    if (!voice.channelId)
      return ctx.write({ content: 'You must be in a voice channel to play music.', flags: MessageFlags.Ephemeral });

    const me = await ctx.me();
    const botVoice = await me?.voice();
    if (botVoice && botVoice.channelId !== voice.channelId)
      return ctx.write({ content: 'You must be in the same voice channel as me.', flags: MessageFlags.Ephemeral });

    const player = client.hoshimi.createPlayer({
      guildId,
      textId: channelId,
      voiceId: voice.channelId,
      volume: 100,
    });
    await player.connect();

    const { loadType, tracks, playlist } = await player.search({ query, requester: author });
    if (loadType === LoadType.Empty || loadType === LoadType.Error)
      return ctx.write({ content: 'No results found!' });

    if (loadType === LoadType.Playlist) await player.queue.add(tracks);
    else await player.queue.add(tracks[0]);

    if (!player.playing) await player.play();

    return ctx.write({
      content:
        loadType === LoadType.Playlist
          ? `Queued ${tracks.length} tracks from ${playlist?.info.name}`
          : `Queued ${tracks[0].info.title}`,
    });
  }
}
```

### NEW — player controls (skip / pause / volume / loop)

Narrow to guild, fetch the live player with `getPlayer(guildId)`, then drive it — all hoshimi:

```ts
import { LoopMode } from 'hoshimi';

const player = client.hoshimi.getPlayer(ctx.guildId); // PlayerStructure | undefined
if (!player) return ctx.write({ content: 'Nothing is playing.' });

await player.skip();                     // next track
await player.setPaused(!player.paused);  // toggle pause
await player.setVolume(80);              // 0–1000 (Lavalink range)
player.setLoop(LoopMode.Queue);          // Track=1 | Queue=2 | Off=3
await player.destroy();                  // leave + clear
```

### NEW — now playing (from the queue)

```ts
import { Embed } from 'seyfert';

const player = client.hoshimi.getPlayer(ctx.guildId);
const track = player?.queue.current; // TrackStructure | null
if (!track) return ctx.write({ content: 'Nothing is playing.' });

const embed = new Embed().setTitle('Now Playing').setDescription(`[${track.info.title}](${track.info.uri})`);
return ctx.write({ embeds: [embed] });
```

### NEW — manager events → post to the text channel

hoshimi's emitter (external) drives "now playing" / "queue ended". Use the core `client.messages.write` shorter to send to the player's stored `textId`. Wire once, near the bootstrap. Event `track` params can be `null`.

```ts
// in index.ts after the Hoshimi instance is created
client.hoshimi.on('trackStart', (player, track) => {
  if (player.textId && track) client.messages.write(player.textId, { content: `Now playing: ${track.info.title}` });
});
client.hoshimi.on('queueEnd', (player) => {
  if (player.textId) client.messages.write(player.textId, { content: 'Queue ended.' });
});
```

`client.messages.write(channelId, body)` is core (`MessageShorter`, `src/structures/`); the `trackStart` / `queueEnd` events and `track` shape are hoshimi's (external).

## Common patterns / gotchas

- **`GuildVoiceStates` intent is mandatory** — without it `member.voice()` never resolves and hoshimi can't track voice. `member.voice()` defaults to `'flow'` (cache→REST) and may reject; prefer `member.voice('cache')` inside guards.
- **Narrow to guild early.** Music is guild-only — `if (!ctx.inGuild()) return;` (or type the handler as `GuildCommandContext`) so `guildId`/`member`/`ctx.me()` are non-undefined; no `?.` sprinkling.
- **`createPlayer` is sync, `connect` is async.** `const player = client.hoshimi.createPlayer({...}); await player.connect();` — then `search`/`queue.add`/`play`.
- **`sendPayload` needs the shard id first.** `this.gateway.send(this.gateway.calculateShardId(guildId), payload)` — `calculateShardId` maps the guild to its shard; passing the guild id directly is wrong.
- **Type the manager on `SeyfertRegistry.client`, not `interface Client`.** The v4/kazagumo pattern (`interface Client { kazagumo }`) is gone; use the `Client` subclass + `ParseClient<MusicClient>` or the `& { hoshimi: Hoshimi }` intersection so `client.hoshimi` is typed in every command/event.
- **Pin the version.** hoshimi is pre-1.0 (`0.3.x-dev`); options/enums/events shift across builds. Verify `SearchSources`, `LoadType`, event names, and player methods against the installed package.

## Doc vs Source Corrections

- **`SearchEngines` / `defaultSearchEngine` do not exist.** The upstream MDX imports `SearchEngines` and passes `defaultSearchEngine: SearchEngines.Youtube`, but hoshimi's type exports have **no `SearchEngines`** — the enum is `SearchSources` and the option is `defaultSearchSource` (`HoshimiOptions.defaultSearchSource?: SearchSource`, default `SearchSources.Youtube`). Confirmed against `dist/index.d.cts` (the `Hoshimi` constructor docstring itself uses `defaultSearchSource: SearchSources.Youtube`). Use `SearchSources` / `defaultSearchSource`.
- **Augmentation moved to `SeyfertRegistry.client`.** The old recipe augmented `interface Client { kazagumo: Kazagumo }` directly; the v5 hoshimi setup types the manager through `SeyfertRegistry.client` (subclass or intersection). Augmenting a bare `interface Client` no longer wires `client.hoshimi` typing in v5.
- **`SearchSources` values are Lavalink search prefixes** (`Youtube="ytsearch"`, `Spotify="spsearch"`, …) and depend on server-side plugins (youtube-source, lava-src) being installed on the Lavalink node.

## Source Anchors

- `hoshimi` `dist/index.d.cts` (`hoshimi@0.3.10-dev`): `Hoshimi` class (`updateVoiceState`, `init`, `createPlayer`, `getPlayer`, `search`), `HoshimiOptions`, `PlayerOptions`, `Player`/`Queue`, `SearchOptions`/`SearchQuery`/`QueryResult`, enums `LoadType`/`LoopMode`/`SearchSources`/`EventNames`, packet types `VoicePacket`/`VoiceServer`/`VoiceState`/`ChannelDeletePacket`.
- Real-world Seyfert + hoshimi bot usage: a `Hoshimi` subclass with options, `raw`-event forwarding to `updateVoiceState`, a `play` command (`createPlayer` → `connect` → `search` → `queue.add` → `play`), and event listeners keyed by `EventNames`.
- Core Seyfert: `src/structures/` (`client.messages.write` `MessageShorter`), `member.voice()` on the member structure, `ctx.me()` / `ctx.inGuild()` on the command context.

## Agent Guidance

- Use when adding music/voice playback to a Seyfert bot. Seyfert only provides the gateway + command plumbing; hoshimi is the Lavalink v4 client and needs a running Lavalink node (`localhost:2333` in examples).
- Type `client.hoshimi` via `SeyfertRegistry.client` (subclass `ParseClient<MusicClient>` or `ParseClient<Client<true>> & { hoshimi: Hoshimi }`). Requires the declare-module setup to be loaded — see `learn/getting-started/declare-module`.
- Import runtime values (`Hoshimi`, `SearchSources`, `LoadType`, `LoopMode`, `EventNames`) and types (`PlayerStructure`, `TrackStructure`, `QueryResult`, packet types) from `'hoshimi'` — none of these are on `'seyfert'`.
- Do NOT copy the docs' `SearchEngines` / `defaultSearchEngine`; they don't exist — use `SearchSources` / `defaultSearchSource`.
- Always `player.destroy()` when the channel empties or on `queueEnd` if you don't idle; forward the `raw` event and call `init()` on `botReady`, or nothing plays.
