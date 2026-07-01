# Handling components

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/handling-components
Coverage reference: components.md (+ builders.md)
Verification status: Source-verified (v5, the authoritative Seyfert source) — no v5/v4 errors found in prior note; expanded with recipes.

## Page Summary

Covers static (persistent) component handlers via `ComponentCommand`. You register a `components` directory in the Seyfert config, create a file exporting `default` a class extending `ComponentCommand`, declare the `componentType` it handles, optionally filter by `customId`, and put logic in `run`. The handler engine (`executeComponent`) iterates EVERY loaded `ComponentCommand`, matches by component type + filter, and runs all that pass. This is the file-based, restart-surviving alternative to inline collectors (`message.createComponentCollector`).

## Key APIs (verified)

From `src/components/componentcommand.ts`:
- `abstract class ComponentCommand` — root import `from 'seyfert'`.
  - `type = InteractionCommandType.COMPONENT` (value `0`; modals use `1`). Both exported from the same module.
  - `abstract componentType: keyof ContextComponentCommandInteractionMap` — assigned by the subclass. Valid keys: `'Button' | 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'`.
  - `customId?: string | RegExp` — declarative match. A `string` is compared with `===`; a `RegExp` is tested with `context.customId.match(this.customId)`. Not shown in upstream docs (docs only show manual `ctx.customId === ...` inside `filter`).
  - `filter?(context): Promise<boolean> | boolean` — optional, may be async.
  - `_filter(context)` (`@internal`) — runs the `customId` check first; if `customId` matches (or is unset) and a `filter` exists, returns `filter(context)`, else `true`. This is what the handler calls.
  - `abstract run(context): any` — required handler body.
  - `get cType(): number` => `ComponentType[this.componentType]` (numeric enum value used for matching against `interaction.componentType`).
  - `middlewares: readonly (keyof ResolvedRegisteredMiddlewares)[] = []` — readonly in v5 (no `.push`).
  - `props!: ExtraProps`.
  - Optional hooks: `onBeforeMiddlewares(ctx)`, `onAfterRun(ctx, error)`, `onRunError(ctx, error)`, `onMiddlewaresError(ctx, error, metadata)`, `onInternalError(client, component, error?)` — v5 added the `component` param BEFORE `error`.

From `src/components/componentcontext.ts`:
- `class ComponentContext<Type extends keyof ContextComponentCommandInteractionMap, M = never, StringSelectValues extends string[] = string[]>` — root import `from 'seyfert'`.
  - `get customId()` => `this.interaction.customId`.
  - `get message(): MessageStructure` (v5 getter — the message the component lives on), `get author(): UserStructure`, `get member()`, `get guildId()`, `get channelId()`, `get t()` (i18n; honors `preferGuildLocale`).
  - Reply methods: `write(body, fetchReply?)`, `editOrReply(body, fetchReply?)`, `deferReply(ephemeral?, fetchReply?)`, `followup(body)`, `fetchResponse()`, `deleteResponse()`. Return `void`/`undefined` UNLESS `fetchReply === true` (then `WebhookMessageStructure`).
  - Component-specific: `update(body)` (edit the source message in place), `deferUpdate()` (ACK with no loading state, edit later), `modal(body)` / `modal(body, options)` (open a modal; with `options` returns `Promise<ModalSubmitInteraction | null>`).
  - **NO `editResponse`** — removed from `ComponentContext` in v5 (it still exists on `ModalContext`). Use `editOrReply(body, true)` for the returned Message.
  - Fetchers: `channel(mode?)`, `guild(mode?, query?)`, `me(mode?)` — modes `'cache' | 'rest' | 'flow'` (default `'flow'`); `'cache'` may be sync, others return a Promise.
  - Type guards (narrow `Type`): `isComponent()`, `isButton()`, `isStringSelectMenu()`, `isUserSelectMenu()`, `isRoleSelectMenu()`, `isMentionableSelectMenu()`, `isChannelSelectMenu()`, `inGuild()`. v5: `isButton()` (and friends) read `interaction.componentType` (was `interaction.data.componentType` in v4).
  - `interface ContextComponentCommandInteractionMap` maps each key to its interaction type (`Button -> ButtonInteraction`, `StringSelect -> StringSelectMenuInteraction<StringSelectValues>`, etc).
  - `inGuild()` narrows to `GuildComponentContext` (non-undefined `guildId`/`member`, guild-narrowed `guild()`/`me()`).

Handler engine (`src/components/handler.ts`): `executeComponent(context)` loops `this.commands`, runs a command when `i.type === COMPONENT && i.cType === interaction.componentType && await i._filter(context)`. ALL matching handlers run (no early `break`). Files are loaded from the configured `components` dir; only the `default` export is read (`onFile` returns `file.default ? [file.default]`), and it must be a class extending `ComponentCommand`/`ModalCommand` or it is skipped with a warning.

Inline collector (`src/structures/Message.ts` + `handler.ts`): `message.createComponentCollector(options?: ListenerOptions)` returns `{ run, stop, waitFor, resetTimeouts }`:
- `run(customId, callback)` — `customId` is `string | string[] | RegExp`; `callback: (interaction, stop, refresh) => any`.
- `stop(reason?)`, `waitFor(customId, timeout?): Promise<interaction | null>`, `resetTimeouts()`.
- `ListenerOptions`: `{ timeout?, idle?, filter?, onPass?, onStop?, onError? }`. `onStop(reason, refresh)` reason is `ComponentCollectorStopReason` (`'messageDelete' | 'channelDelete' | 'guildDelete' | 'idle' | 'timeout' | string`). `ComponentCollectorStopReason` is an exported type in v5.

Builder custom IDs (matched against `customId`): `Button.setCustomId(id: string)` (`src/builders/Button.ts`); select menus `setCustomId(id)` (`src/builders/SelectMenu.ts`). Link/SKU buttons have no `setCustomId` (`ButtonLink = Omit<Button, 'setCustomId'>`). Select-menu `.setDisabled(disabled = true)` — the old `.disabed`/`.disabled` property typo was removed; use the method. `StringSelectMenu.setOptions(...)` accepts rest params OR an array (`RestOrArray`).

## Code Examples (verified)

Config — register the components directory (`seyfert.config.mjs`):

```ts
// @ts-check
import { config } from 'seyfert';

export default config.bot({
  token: process.env.BOT_TOKEN ?? '',
  intents: ['Guilds'],
  locations: {
    base: 'dist',
    commands: 'commands',
    events: 'events',
    components: 'components',
  },
});
```

Basic handler (docs style — manual filter):

```ts
import { ComponentCommand, type ComponentContext, MessageFlags } from 'seyfert';

export default class HelloWorldButton extends ComponentCommand {
  componentType = 'Button' as const;

  // optional; may return a Promise<boolean>
  filter(ctx: ComponentContext<typeof this.componentType>) {
    return ctx.customId === 'hello-world';
  }

  async run(ctx: ComponentContext<typeof this.componentType>) {
    return ctx.write({
      content: 'Hello World 👋',
      flags: MessageFlags.Ephemeral,
    });
  }
}
```

Declarative `customId` (source feature; equivalent to the filter above, also accepts a RegExp):

```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class HelloWorldButton extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'hello-world'; // or: customId = /^hello-/;

  async run(ctx: ComponentContext<typeof this.componentType>) {
    return ctx.update({ content: 'Updated the source message' });
  }
}
```

Select-menu handler with type narrowing:

```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class FavSelect extends ComponentCommand {
  componentType = 'StringSelect' as const;
  customId = 'fav-color';

  async run(ctx: ComponentContext<typeof this.componentType>) {
    if (!ctx.isStringSelectMenu()) return;
    const [value] = ctx.interaction.values; // string[]
    return ctx.update({ content: `You picked ${value}` });
  }
}
```

### Recipe — pagination buttons with a RegExp customId (encode params)

Send buttons whose ids embed state (`page:next:2`), then parse them in one handler.

```ts
// sending side (inside a command's run):
import { ActionRow, Button, ButtonStyle } from 'seyfert';

const page = 0;
const row = new ActionRow<Button>().setComponents([
  new Button().setCustomId(`page:prev:${page}`).setStyle(ButtonStyle.Secondary).setLabel('◀'),
  new Button().setCustomId(`page:next:${page}`).setStyle(ButtonStyle.Secondary).setLabel('▶'),
]);
await ctx.write({ content: renderPage(page), components: [row] });
```

```ts
// components/pagination.ts
import { ActionRow, Button, ButtonStyle, ComponentCommand, type ComponentContext } from 'seyfert';

export default class Pagination extends ComponentCommand {
  componentType = 'Button' as const;
  customId = /^page:(prev|next):\d+$/;

  async run(ctx: ComponentContext<typeof this.componentType>) {
    const [, dir, raw] = ctx.customId.split(':');
    const next = dir === 'next' ? Number(raw) + 1 : Number(raw) - 1;
    const page = Math.max(0, next);

    const row = new ActionRow<Button>().setComponents([
      new Button().setCustomId(`page:prev:${page}`).setStyle(ButtonStyle.Secondary).setLabel('◀'),
      new Button().setCustomId(`page:next:${page}`).setStyle(ButtonStyle.Secondary).setLabel('▶'),
    ]);
    // update() edits the original message — no new reply, no "thinking" state
    return ctx.update({ content: renderPage(page), components: [row] });
  }
}

declare function renderPage(page: number): string;
```

### Recipe — button opens a modal, then handle the submit elsewhere

`ctx.modal(builder)` opens the modal. Two ways to read the result:
1. Fire-and-forget: open the modal here, handle the submit in a separate `ModalCommand` file (matched by the modal's `customId`).
2. Inline: pass `{ idle }`/`{ timeout }` options to await the `ModalSubmitInteraction` right here.

```ts
// components/openProfile.ts
import { ComponentCommand, type ComponentContext, Modal, Label, TextInput, TextInputStyle } from 'seyfert';

export default class OpenProfile extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'open-profile';

  async run(ctx: ComponentContext<typeof this.componentType>) {
    const modal = new Modal()
      .setCustomId('profile-modal')
      .setTitle('Your profile')
      .setComponents([
        new Label()
          .setLabel('Bio')
          .setComponent(new TextInput().setCustomId('bio').setStyle(TextInputStyle.Paragraph)),
      ]);

    // Inline await: returns ModalSubmitInteraction | null (null on timeout)
    const submit = await ctx.modal(modal, { waitFor: 60_000 });
    if (!submit) return; // user closed / timed out
    const bio = submit.getInputValue('bio', true);
    return submit.write({ content: `Saved: ${bio}` });
  }
}
```

### Recipe — gate a component with middlewares

`ComponentCommand.middlewares` is a readonly tuple of registered middleware names. A middleware that calls `stop('reason')` routes to `onMiddlewaresError`; `stop()` with no args skips silently (v5 — `pass()` was removed).

```ts
// components/adminPanel.ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class AdminPanel extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'admin-action';
  middlewares = ['isAdmin'] as const; // names from your SeyfertRegistry middlewares

  async run(ctx: ComponentContext<typeof this.componentType>) {
    return ctx.update({ content: 'Action performed.' });
  }

  onMiddlewaresError(ctx: ComponentContext<typeof this.componentType>, error: string) {
    return ctx.write({ content: error || 'Not allowed.', flags: 64 });
  }
}
```

### Recipe — inline collector (short-lived, in-memory)

For per-message flows that do NOT need to survive a restart, attach a collector to the sent message instead of a file handler.

```ts
import { ActionRow, Button, ButtonStyle } from 'seyfert';

const row = new ActionRow<Button>().setComponents([
  new Button().setCustomId('confirm').setStyle(ButtonStyle.Success).setLabel('Confirm'),
  new Button().setCustomId('cancel').setStyle(ButtonStyle.Danger).setLabel('Cancel'),
]);

const message = await ctx.write({ content: 'Proceed?', components: [row] }, true); // fetchReply: true

const collector = message.createComponentCollector({
  idle: 30_000,
  // only let the original author interact
  filter: i => i.user.id === ctx.author.id,
  onStop: reason => reason === 'idle' && message.edit({ components: [] }),
});

collector.run('confirm', async i => {
  await i.update({ content: 'Confirmed ✅', components: [] });
  collector.stop('done');
});
collector.run('cancel', async i => {
  await i.update({ content: 'Cancelled ❌', components: [] });
  collector.stop('done');
});

// Or await a single interaction instead of registering callbacks:
// const picked = await collector.waitFor('confirm', 30_000); // interaction | null
```

## Doc vs Source Corrections

- Upstream docs handle `customId` only manually inside `filter` (`ctx.customId === 'hello-world'`). Source adds a declarative `customId?: string | RegExp` property on `ComponentCommand` whose check runs in `_filter` BEFORE any user `filter` (`componentcommand.ts`). Prefer it for simple matches; use a RegExp to encode params.
- Docs say "set the type ... (`Buttons` or whichever type of `SelectMenu`)". The actual literals are singular: `'Button'`; selects are `'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'` (`ContextComponentCommandInteractionMap`).
- Docs imply one handler runs. Source runs EVERY loaded `ComponentCommand` whose type + filter match (`executeComponent` loops all `commands`, no `break`) — keep `customId` filters specific to avoid double-handling.
- Not in docs: `update`/`deferUpdate` for editing the source message, `modal(...)` to open a modal, `ctx.message`, and the `is*SelectMenu()`/`isButton()`/`inGuild()` guards (`componentcontext.ts`).
- v5 fixes baked in above: `ComponentContext.editResponse` REMOVED (use `editOrReply(body, true)`); `isButton()` reads `interaction.componentType`; `onInternalError(client, component, error?)` gained the `component` param; middleware `pass()` removed (use `stop()`); select-menu `.disabled` property typo removed (use `setDisabled`).

## Source Anchors

- `src/components/componentcommand.ts` — `ComponentCommand`, `componentType`, `customId`, `filter`, `_filter`, `cType`, `InteractionCommandType`, hooks.
- `src/components/componentcontext.ts` — `ComponentContext`, getters (`message`, `customId`, `author`...), reply/update/modal methods, guards, `ContextComponentCommandInteractionMap`, `GuildComponentContext`.
- `src/components/handler.ts` — `executeComponent`/`execute`, `createComponentCollector`, loader (`load`/`set`/`onFile`/`callback`), default-export requirement.
- `src/components/index.ts` — barrel exports (`componentcommand`, `componentcontext`, `modalcommand`, `modalcontext`).
- `src/builders/Button.ts`, `src/builders/SelectMenu.ts`, `src/builders/types.ts` — `setCustomId`, `setDisabled`, `ButtonLink` omit, `ComponentCallback`, `ListenerOptions`, `ComponentCollectorStopReason`.
- `src/structures/Message.ts` — `createComponentCollector(options?)`.

## Agent Guidance

- Use `ComponentCommand` for persistent components that must survive restarts (collectors live only in memory). For short-lived per-message flows use `message.createComponentCollector` instead.
- The file MUST `export default` the class; the loader skips non-class / non-default exports and warns (`... doesn't export the class by export default <ComponentCommand>`).
- Match the builder's `setCustomId(...)` value with the handler's `customId` (string or RegExp). Link/Premium buttons can't have a custom id and won't trigger handlers.
- `componentType` selects the interaction kind; `customId`/`filter` narrow which custom ids. Encode params in a RegExp `customId` (e.g. `vote:123`) and parse inside `run`.
- Use `ctx.update`/`ctx.deferUpdate` to mutate the originating message; `ctx.write`/`ctx.deferReply` create a new response. `editOrReply`/`write` return `void` unless you pass `true` as the second arg.
- Always call exactly one ack per interaction (`update`, `deferUpdate`, `write`, or `deferReply`) or Discord shows "interaction failed".
- Errors in `run` route to `onRunError` then `onAfterRun(ctx, error)`; uncaught/internal failures route to `onInternalError(client, component, error)`. Middleware denials route to `onMiddlewaresError(ctx, error, metadata)` where `metadata.middleware` names the denying middleware.

## Common patterns / gotchas

- Multiple handlers fire: if two `ComponentCommand`s match the same interaction's type AND filter, BOTH run. Make `customId` specific (string equality or anchored RegExp `^...$`).
- `ctx.interaction.values` is only available after a guard narrows the type — call `ctx.isStringSelectMenu()` (or the matching guard) first, otherwise `values` is not on the union.
- Prefer the declarative `customId` over a `filter` for simple equality — it's checked first and reads cleaner; reserve `filter` for async/role checks.
- A `RegExp` `customId` is matched with `String.match`, so it does NOT need to be anchored to "match anywhere", but anchor it (`^vote:\d+$`) to avoid accidental overlap with other handlers.
- Inline collectors: set `idle`/`timeout` and an `onStop` to clean up components, and a `filter` to restrict who can click. `stop(reason)` ends it; `waitFor` resolves `null` on timeout.
- Builders gotcha: select menus and modals now validate on `toJSON()` — a `Modal` without a title or `StringSelectOption` without required fields throws at serialize time, not at the Discord API.
