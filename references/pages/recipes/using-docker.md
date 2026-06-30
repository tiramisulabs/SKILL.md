# Docker

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/using-docker
Coverage reference: setup-runtime.md
Verification status: Source-verified (core) — Dockerfile is plain ops; all Seyfert touchpoints confirmed against ./src on branch more-qol.

## Page Summary

How to containerize a Seyfert bot with a multi-stage Docker build that compiles
TypeScript and ships a minimal Alpine production image running `node dist/index.js`
under `dumb-init`. The recipe itself is plain Docker/npm/tsc ops — **there is no
Seyfert-specific Docker API**. The only framework coupling is that the runtime needs
`seyfert.config.*` at `process.cwd()` and that `locations.base` must point at the
compiled output directory (e.g. `dist`) so command/event/component/lang loaders
resolve correctly inside the container.

Required files in a Seyfert project (per upstream MDX): `package.json`,
`seyfert.config.mjs`, `node_modules`, and `src` (compiled to `dist`).

## Key APIs (verified)

These are the framework touchpoints behind the recipe (Docker has no Seyfert API).

- `getRC()` reads `seyfert.config{.js|.mjs|.cjs|.ts|.mts|.cts}` from `process.cwd()`
  via `magicImport(join(process.cwd(), 'seyfert.config' + ext))` (using
  `Promise.any` across the extensions) and takes the default export — so the config
  file MUST be copied into the image and the working directory must be the app root.
  An `options.getRC` override (or a Cloudflare-Worker static config) short-circuits
  the file lookup. The result is cached in `_rcCache` after the first call.
  — `src/client/base.ts:1188-1239`
- If no config file resolves, `getRC()` throws a `SeyfertError`. Two distinct codes:
  `SEYFERT_CONFIG_LOAD_ERROR` (a file was found but failed to import — original error
  preserved as `cause`) and `NO_SEYFERT_CONFIG` (detail `"No seyfert.config file
  found"`). In a container, a build that drops the config file surfaces as
  `NO_SEYFERT_CONFIG` at startup. — `src/client/base.ts:1199-1215`
- Loader directories are resolved as `join(process.cwd(), locations.base, location)`
  for `commands` / `components` / `langs` / `events`. `locations.base` is used as-is
  (not re-joined). In a Docker image where you ship only `dist`, `locations.base`
  must be `dist` (or the relevant compiled subdir), NOT `src`.
  — `src/client/base.ts:1219-1229`, loaders at `1127-1168`
- `RCLocations` fields: `base` (required `string`), `commands?`, `langs?`, `events?`,
  `components?` (all optional `string`), extensible via `ExtendedRCLocations`.
  — `src/client/base.ts:1349-1355`
- `config.bot(data)` / `config.http(data)` from `'seyfert'` build the runtime config
  object placed in `seyfert.config.*`. `config.bot` resolves `intents` (accepts the
  v5 inline string form, e.g. `['Guilds']`). `config.http` defaults `port` to `8080`
  and additionally requires `publicKey` + `applicationId` (an HTTP interactions
  server needs them); `config.http` also stores the config statically when running
  under a Cloudflare Worker. Expose/publish this port in the container if running the
  HTTP interactions server. — `src/index.ts:76-103`, `src/client/base.ts:1375-1378`

## Code Examples (verified)

### Multi-stage Dockerfile (gateway bot)

The `Dockerfile` is plain ops and is correct as published. Reproduced verbatim from
upstream MDX, with notes:

```docker title="Dockerfile" copy
# [ base ] #
FROM node:<VERSION_TAG>-alpine AS base
ENV DIR /bot
WORKDIR $DIR

# [ OS packages ] #
FROM base AS pkg
RUN apk update && apk add --no-cache dumb-init

# [ project builder ] #
FROM base AS build
COPY package*.json ./
RUN npm ci
RUN npm prune --production
RUN npm i -g typescript
COPY tsconfig.json seyfert.config.mjs ./
COPY /src ./src
RUN tsc --project tsconfig.json

# [ production ready ] #
FROM base AS production
COPY --from=pkg /usr/bin/dumb-init /usr/bin/dumb-init
COPY --from=build $DIR/node_modules ./node_modules
COPY --from=build $DIR/dist ./dist
COPY --from=build $DIR/package.json ./package.json
COPY --from=build $DIR/seyfert.config.mjs ./seyfert.config.mjs
ENV NODE_ENV production
ENV USER node
USER $USER
ENTRYPOINT ["dumb-init", "node", "dist/index.js"]
```

Gotcha in the recipe as published: it runs `npm prune --production` in the `build`
stage and then `tsc`. `typescript` is installed globally (`npm i -g typescript`), so
the prune does not break compilation — but if your build relies on any local
devDependencies (custom tsc plugins, `tsc-alias`/path resolvers, code generators)
they will have been pruned before `tsc` runs. If so, move the prune AFTER the build,
or run it in a separate dependency stage (see the recipe below).

### `seyfert.config.mjs` (gateway) — loaders point at `dist`

```js
// seyfert.config.mjs
import { config } from 'seyfert';

export default config.bot({
  token: process.env.BOT_TOKEN ?? '',
  intents: ['Guilds'],        // v5 inline string intents
  locations: {
    base: 'dist',             // compiled output COPYed into the image (used as-is)
    commands: 'commands',     // -> <cwd>/dist/commands
    events: 'events',         // -> <cwd>/dist/events
    components: 'components',  // -> <cwd>/dist/components
    langs: 'langs',           // -> <cwd>/dist/langs
  },
});
```

### Recipe — split deps stage so devDependencies survive the build

Cleaner ordering when your build needs devDependencies but production must not ship
them. Build with the full tree, then COPY a pruned tree into production.

```docker title="Dockerfile" copy
FROM node:22-alpine AS base
ENV DIR=/bot
WORKDIR $DIR

# Full install (dev + prod) used only to compile.
FROM base AS build
COPY package*.json tsconfig.json ./
RUN npm ci
COPY seyfert.config.mjs ./
COPY ./src ./src
RUN npx tsc --project tsconfig.json   # uses local devDeps (tsc, plugins)

# Production-only node_modules, isolated from the build tree.
FROM base AS deps
COPY package*.json ./
RUN npm ci --omit=dev

FROM base AS pkg
RUN apk add --no-cache dumb-init

FROM base AS production
COPY --from=pkg  /usr/bin/dumb-init /usr/bin/dumb-init
COPY --from=deps  $DIR/node_modules ./node_modules
COPY --from=build $DIR/dist ./dist
COPY --from=build $DIR/package.json ./package.json
COPY --from=build $DIR/seyfert.config.mjs ./seyfert.config.mjs
ENV NODE_ENV=production
USER node
ENTRYPOINT ["dumb-init", "node", "dist/index.js"]
```

### Recipe — HTTP interactions bot (expose the port)

When you run the HTTP interactions adapter instead of the gateway, use
`config.http` and publish the port. `config.http` defaults `port` to `8080` and
requires `publicKey` + `applicationId`.

```js
// seyfert.config.mjs  (HTTP interactions)
import { config } from 'seyfert';

export default config.http({
  token: process.env.BOT_TOKEN ?? '',
  publicKey: process.env.BOT_PUBLIC_KEY ?? '',
  applicationId: process.env.BOT_APP_ID ?? '',
  port: Number(process.env.PORT ?? 8080),
  locations: {
    base: 'dist',
    commands: 'commands',
    components: 'components',
    langs: 'langs',
    // NOTE: an HTTP server config has no `events` location (gateway-only).
  },
});
```

Add to the production stage of the Dockerfile so Docker documents/publishes the port:

```docker
# [ production ready ] #  (HTTP interactions)
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["dumb-init", "node", "dist/index.js"]
```

`EXPOSE` is documentation only — still publish at run time
(`docker run -p 8080:8080 ...`) and point your Discord "Interactions Endpoint URL"
at the reachable host.

### Recipe — `.dockerignore` (smaller, safer context)

Keep secrets and bulky/local artifacts out of the build context. Crucially, do NOT
ship a local `dist` — let the `build` stage produce it fresh.

```gitignore title=".dockerignore" copy
node_modules
dist
.git
.env
.env.*
*.log
Dockerfile
.dockerignore
```

### Recipe — `docker-compose.yml` (inject secrets at runtime)

Secrets stay out of image layers; they arrive as runtime env. `restart` keeps the
bot up across crashes/redeploys.

```yaml title="docker-compose.yml" copy
services:
  bot:
    build: .
    restart: unless-stopped
    environment:
      BOT_TOKEN: ${BOT_TOKEN}
      NODE_ENV: production
    # For an HTTP interactions bot, also expose the port:
    # ports:
    #   - "8080:8080"
    # environment:
    #   BOT_PUBLIC_KEY: ${BOT_PUBLIC_KEY}
    #   BOT_APP_ID: ${BOT_APP_ID}
```

Run with `docker compose --env-file .env up --build` (the `.env` lives next to the
compose file and is read by Compose, NOT baked into the image).

## Common patterns / gotchas

- **`locations.base` must match what you ship.** A bot that "starts but registers no
  commands" in a container almost always means `locations.base` still points at `src`
  (which you did not COPY) instead of `dist`. The loaders resolve
  `join(cwd, locations.base, <location>)` — `base` itself is used verbatim.
- **Config file must be in the image and at the WORKDIR.** `getRC()` reads from
  `process.cwd()`. The Dockerfile copies `seyfert.config.mjs` and runs the entrypoint
  from `$DIR` (`/bot`), so cwd is correct. If you forget to COPY it you get
  `NO_SEYFERT_CONFIG`; if it imports but throws (e.g. a bad top-level import you also
  forgot to ship) you get `SEYFERT_CONFIG_LOAD_ERROR` with the cause attached.
- **Compile the config too, or ship the `.mjs`.** Shipping `seyfert.config.mjs` (plain
  ESM) avoids needing to compile it. If you write `seyfert.config.ts`, make sure tsc
  emits it into `dist` and copy that compiled file instead.
- **Never bake secrets into layers.** Inject `BOT_TOKEN` (and `publicKey`/`appId` for
  HTTP) via env/secrets at run time, never via `ENV BOT_TOKEN=...` or a committed
  config. `.dockerignore` your `.env`.
- **Pin the Node version.** `node:<VERSION_TAG>` is a placeholder — pin a concrete
  major (e.g. `node:22-alpine`) and verify it satisfies the target project's engines.
- **`dumb-init` is the PID-1 reaper.** Keep it as the entrypoint wrapper so SIGTERM
  reaches Node and the bot shuts down cleanly (gateway disconnect, plugin teardown).
- **HTTP server config has no `events`.** `InternalRuntimeConfigHTTP` omits the
  `events` location (events are a gateway concept). Don't expect event handlers under
  the HTTP adapter.

## Doc vs Source Corrections

- Upstream MDX/draft do not state the concrete framework requirement -> source shows
  the hard requirement is `seyfert.config.*` present at `process.cwd()` and
  `locations.base` pointing at the compiled dir (`src/client/base.ts:1188-1229`).
- Draft listed a single generic "No seyfert.config" error -> source distinguishes two
  codes: `SEYFERT_CONFIG_LOAD_ERROR` (import failed, `cause` preserved) and
  `NO_SEYFERT_CONFIG` (`src/client/base.ts:1207-1214`).
- MDX/draft do not mention the `npm prune --production` ordering caveat -> flagged
  above (no src impact; ops note). Added a split-deps recipe that avoids it.
- Otherwise the Dockerfile and required-files list are accurate; the bot loads from
  the `dist` output via `getRC().locations`.

## Source Anchors

- `src/client/base.ts:1127-1168` — `loadCommands`/`loadComponents`/`loadLangs` default
  their dir from `getRC().locations.*`
- `src/client/base.ts:1188-1239` — `getRC()` reads `seyfert.config` from cwd (across
  6 extensions), error codes, resolves+joins locations, caches in `_rcCache`
- `src/client/base.ts:1349-1365` — `RCLocations` / `RC` shape (`base` required)
- `src/client/base.ts:1375-1378` — `InternalRuntimeConfigHTTP` requires
  `publicKey`/`port`/`applicationId`, omits `events` location
- `src/index.ts:76-103` — `config.bot` / `config.http` helpers (http `port` default
  `8080`; Cloudflare static-config stash)

## Agent Guidance

- This is an ops recipe; treat the Dockerfile as project-owned. The only things to
  verify against Seyfert: the config file is copied in, `WORKDIR`/`ENTRYPOINT` run
  from the app root, and `locations.base` matches the directory you actually ship
  (`dist`).
- For HTTP interactions bots, switch the config helper to `config.http`, supply
  `publicKey`/`applicationId`, and `EXPOSE`/publish the port (default `8080`).
- Recommend a `.dockerignore` (drop `node_modules`, `dist`, `.env`) and runtime secret
  injection (env/compose/secrets) — never `ENV`-baked tokens.
- When debugging "no commands registered" in a container, check `locations.base` and
  that `dist/commands` (etc.) actually exists in the image.
