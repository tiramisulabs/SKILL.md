# Message Component Collectors

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/collectors
Coverage reference: components.md (+ builders.md)
Verification status: Source-verified (the authoritative Seyfert source). No v5 deltas needed — message collector API is unchanged from the doc; this note only ADDS source-confirmed detail and recipes the MDX omits.

## Page Summary

Message component collectors are an in-process, message-scoped way to handle component interactions (buttons, select menus) coming from one specific message, while keeping the surrounding command context (`ctx.author`, closure variables, etc.) in scope. You create one from a message with `message.createComponentCollector(options?)`, then register handlers with `collector.run(customId, callback)`. Collectors live ONLY in the process that created them: if that process stops (restart, crash, different worker/shard), the collector dies and interactions go unhandled — Discord just shows a failed interaction. Modals are not message components, so they use a separate mechanism: `new Modal().run(callback)` (callback keyed by the submitting user's id), or the inline `ctx.modal(modal, { waitFor })`.

## Key APIs (verified)

- `Message.createComponentCollector(options?: ListenerOptions): CreateComponentCollectorResult` — src/structures/Message.ts:76. Delegates to `client.components.createComponentCollector(this.id, this.channelId, this.guildId, options)`. Inherited by `BaseMessage`.
- `CreateComponentCollectorResult` — src/components/handler.ts:48-59:
  - `run<T extends CollectorInteraction>(customId, callback): void` — register a handler.
  - `stop(reason?: string): void` — stop the collector; fires `onStop(reason ?? undefined, refresh)` (the dispatch-time `stop` defaults to `'stop'`; this returned `stop` forwards `undefined` when called with no arg).
  - `waitFor<T>(customId, timeout?): Promise<T | null>` — resolves with the next matching interaction, or `null` on timeout / if the collector was already cleared. NOT in docs.
  - `resetTimeouts(): void` — refresh idle+timeout timers manually. NOT in docs.
- `ListenerOptions` (collector options) — src/builders/types.ts:70-77:
  - `timeout?: number` — absolute lifetime in ms (0/undefined = none).
  - `idle?: number` — inactivity timeout in ms; refreshed on each matched+filtered interaction.
  - `filter?: (interaction) => any` — truthy to accept the interaction.
  - `onPass?: (interaction) => any` — called when `filter` rejects. NOT in docs. (Note: it gets the interaction but no `stop`/`refresh`.)
  - `onStop?: (reason, refresh) => any` — `ComponentOnStopCallback`.
  - `onError?: (interaction, error, stop, refresh) => any` — called when a `run` callback throws; if absent, errors are logged via `client.logger.error`. NOT in docs.
- `collector.run` callback `ComponentCallback<T> = (interaction, stop, refresh) => any` — src/builders/types.ts:23-25. Callback receives `stop` and `refresh` as 2nd/3rd args (docs only use `interaction`). `T` defaults to `CollectorInteraction = ComponentInteraction | StringSelectMenuInteraction` (src/components/handler.ts:46).
- `customId` (`UserMatches`) accepts `string | string[] | RegExp` — src/components/handler.ts:32,76-80. Docs only show a single string. String = exact equality; array = `includes`; RegExp = `.test()`.
- `ComponentCollectorStopReason` — src/builders/types.ts:30-37: `'messageDelete' | 'channelDelete' | 'guildDelete' | 'idle' | 'timeout' | (string & {}) | undefined`. Built-in idle/timeout pass `'idle'`/`'timeout'`; lifecycle deletions pass the `*Delete` reasons.
- `refresh()` in `onStop` (and the 3rd `run` arg, which calls `resetTimeouts`) — note these differ: the `refresh` passed to `onStop` RE-CREATES the collector with the same registered handlers (src/components/handler.ts:100-101,132-134,185-186); the `refresh` passed to a `run` callback just resets the idle/timeout timers (src/components/handler.ts:190-192).
- One collector per message id — `values` is `Map<messageId, COMPONENTS>` (handler.ts:63). Re-creating for the same message id OVERWRITES the previous collector.
- Modal handling — src/builders/Modal.ts:91-94: `new Modal().setCustomId(id).setTitle(t).run(callback)`; `run(func: ModalSubmitCallback): this` stores `__exec`. `ctx.modal(modal)` replies with the modal and, when `__exec` is set, registers the callback by user id (`client.components.modals.set(user.id, __exec)` — src/structures/Interaction.ts:269). Default modal expiry is 10 minutes (src/components/handler.ts:65); Discord sends no event on modal cancel, so the entry just expires.
- Inline modal await — `ctx.modal(modal, { waitFor })` overload returns `Promise<ModalSubmitInteraction | null>` (src/structures/Interaction.ts:507-538); resolves `null` after `waitFor` ms.

## Code Examples (verified)

### 1. Basic collector with filter, idle refresh, and the extra callback args

```ts
import { Button, ActionRow, Command, Declare, ButtonStyle, type CommandContext } from 'seyfert';

@Declare({ name: 'hello', description: 'I will send you a hello world message' })
export default class HelloWorldCommand extends Command {
  async run(ctx: CommandContext) {
    const button = new Button().setCustomId('hello').setLabel('Hello').setStyle(ButtonStyle.Primary);
    const row = new ActionRow<Button>().setComponents([button]);

    // second arg `true` => fetchReply, returns the Message so we can attach a collector
    const message = await ctx.write(
      { content: 'Do you want a hello world? Click the button below.', components: [row] },
      true,
    );

    const collector = message.createComponentCollector({
      filter: (i) => i.user.id === ctx.author.id && i.isButton(),
      onStop(reason, refresh) {
        if (reason === 'idle') return refresh(); // keep collecting after idle
      },
      idle: 1e3, // 1000ms inactivity window
    });

    // customId may be a string, string[], or RegExp
    collector.run('hello', async (i, stop, refresh) => {
      if (i.isButton()) await i.write({ content: 'Hello World 👋' });
      // stop('done') to end early; refresh() to reset idle/timeout timers
    });
  }
}
```

### 2. Confirm / cancel with onPass guarding other users

`filter` gates who may interact; `onPass` lets you ack rejected users instead of silently dropping. `stop()` ends the flow once a choice is made.

```ts
import { Button, ActionRow, Command, Declare, ButtonStyle, MessageFlags, type CommandContext } from 'seyfert';

@Declare({ name: 'ban', description: 'Confirm before banning' })
export default class ConfirmCommand extends Command {
  async run(ctx: CommandContext) {
    const confirm = new Button().setCustomId('confirm').setLabel('Confirm').setStyle(ButtonStyle.Danger);
    const cancel = new Button().setCustomId('cancel').setLabel('Cancel').setStyle(ButtonStyle.Secondary);
    const row = new ActionRow<Button>().setComponents([confirm, cancel]);

    const message = await ctx.write({ content: 'Are you sure?', components: [row] }, true);

    const collector = message.createComponentCollector({
      filter: (i) => i.user.id === ctx.author.id,
      // someone else clicked — reply ephemerally, don't end the collector
      onPass: (i) => i.write({ content: 'This is not your confirmation.', flags: MessageFlags.Ephemeral }),
      timeout: 15_000, // hard 15s lifetime
      onStop: (reason) => {
        if (reason === 'timeout') return ctx.editOrReply({ content: 'Timed out.', components: [] });
      },
    });

    collector.run(['confirm', 'cancel'], async (i, stop) => {
      const ok = i.customId === 'confirm';
      await i.update({ content: ok ? 'Confirmed ✅' : 'Cancelled ❌', components: [] });
      stop('done'); // end the collector; reason forwarded to onStop
    });
  }
}
```

### 3. Pagination with closure state + waitFor loop

`waitFor` returns the next matching interaction (or `null` on timeout) so you can drive a linear flow without registering persistent handlers. Page index lives in a plain closure variable.

```ts
import { Button, ActionRow, Command, Declare, ButtonStyle, type CommandContext, type ButtonInteraction } from 'seyfert';

const pages = ['Page 1', 'Page 2', 'Page 3'];

@Declare({ name: 'pages', description: 'Browse pages' })
export default class PagesCommand extends Command {
  async run(ctx: CommandContext) {
    let index = 0;
    const buildRow = () =>
      new ActionRow<Button>().setComponents([
        new Button().setCustomId('prev').setLabel('◀').setStyle(ButtonStyle.Secondary).setDisabled(index === 0),
        new Button().setCustomId('next').setLabel('▶').setStyle(ButtonStyle.Secondary).setDisabled(index === pages.length - 1),
      ]);

    const message = await ctx.write({ content: pages[index], components: [buildRow()] }, true);
    const collector = message.createComponentCollector({ filter: (i) => i.user.id === ctx.author.id });

    while (true) {
      const i = await collector.waitFor<ButtonInteraction>(['prev', 'next'], 60_000);
      if (!i) {
        await ctx.editOrReply({ components: [] }); // timed out — strip buttons
        break;
      }
      index += i.customId === 'next' ? 1 : -1;
      await i.update({ content: pages[index], components: [buildRow()] });
    }
  }
}
```

### 4. String select menu collector

`run`'s callback interaction is `ComponentInteraction | StringSelectMenuInteraction`; narrow with `isStringSelectMenu()` to read the typed `values` array.

```ts
import { StringSelectMenu, StringSelectOption, ActionRow, Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'pick', description: 'Pick a color' })
export default class PickCommand extends Command {
  async run(ctx: CommandContext) {
    const menu = new StringSelectMenu()
      .setCustomId('color')
      .setPlaceholder('Pick one')
      .addOption(
        new StringSelectOption().setLabel('Red').setValue('red'),
        new StringSelectOption().setLabel('Blue').setValue('blue'),
      );
    const row = new ActionRow<StringSelectMenu>().setComponents([menu]);

    const message = await ctx.write({ content: 'Choose:', components: [row] }, true);
    const collector = message.createComponentCollector({
      filter: (i) => i.user.id === ctx.author.id,
      idle: 30_000,
    });

    collector.run('color', async (i, stop) => {
      if (!i.isStringSelectMenu()) return;
      await i.update({ content: `You picked: ${i.values[0]}`, components: [] });
      stop('picked');
    });
  }
}
```

### 5. Modal via the builder `run` (modals are not message components)

```ts
import { Modal, TextInput, Label, Command, Declare, TextInputStyle, type ModalSubmitInteraction, type CommandContext } from 'seyfert';

@Declare({ name: 'feedback', description: 'Send feedback' })
export default class FeedbackCommand extends Command {
  async run(ctx: CommandContext) {
    const modal = new Modal()
      .setCustomId('feedback')
      .setTitle('Feedback')
      .setComponents([
        new Label().setLabel('Your message').setComponent(
          new TextInput().setCustomId('msg').setStyle(TextInputStyle.Paragraph).setRequired(true),
        ),
      ])
      .run(this.handleModal); // callback keyed by submitting user id

    await ctx.modal(modal);
  }

  async handleModal(i: ModalSubmitInteraction) {
    const msg = i.getInputValue('msg', true); // required => string
    return i.write({ content: `Thanks! You said: ${msg}` });
  }
}
```

### 6. Inline modal await with `ctx.modal(modal, { waitFor })`

Avoids a separate `.run` method when you want the submission inline. Resolves `null` if the user does not submit within `waitFor` ms.

```ts
async run(ctx: CommandContext) {
  const modal = new Modal()
    .setCustomId('name')
    .setTitle('Your name')
    .setComponents([
      new Label().setLabel('Name').setComponent(new TextInput().setCustomId('name').setStyle(TextInputStyle.Short)),
    ]);

  const submitted = await ctx.modal(modal, { waitFor: 60_000 }); // ModalSubmitInteraction | null
  if (!submitted) return; // timed out / cancelled
  await submitted.write({ content: `Hi, ${submitted.getInputValue('name', true)}!` });
}
```

## Recipes / Common patterns and gotchas

- ALWAYS set `idle` and/or `timeout`. Without them the collector lives until the process dies, lingering in `client.components.values` keyed by message id. Strip the components in `onStop`/after `waitFor` so the message no longer shows clickable-but-dead buttons.
- `run` vs `waitFor`: use `run` for long-lived multi-button flows (pagination with persistent handlers, dashboards); use `waitFor` for linear "ask one thing, await it" steps. They share the same collector and `filter`.
- `i.update(...)` (edit the message the component is on) vs `i.write(...)` (a fresh reply). For pagination/confirm flows you almost always want `update`. `i.deferUpdate()` acks without changing anything.
- `filter` runs BEFORE handlers; a rejected interaction goes to `onPass` (not `onError`) and does NOT refresh idle. Errors thrown inside a `run` callback go to `onError` (or are logged if absent) — they do not crash the collector.
- The two `refresh` semantics differ (see Key APIs): `onStop`'s `refresh` re-creates the collector (resurrects it after idle/timeout); the `run` callback's `refresh` only resets timers on the still-alive collector.
- One collector per message id. If you re-render a message and call `createComponentCollector` again, you replace the old collector — re-register handlers.
- Durability: collectors are NOT durable. They die on restart and do not work across shards/workers (the collector lives only in the process that built the message). For persistent buttons, restart-safe flows, or cross-process handling, use static `ComponentCommand` / `ModalCommand` handlers with encoded custom-id state instead. See components.md.

## Doc vs Source Corrections

- Docs show only `collector.run` + `filter`/`onStop`/`idle`/`timeout`. Source also exposes `stop()`, `waitFor()`, `resetTimeouts()` on the result, and `onPass`/`onError` options.
- Docs: `run` callback is `(i) => ...`. Source: `(interaction, stop, refresh) => any` — two extra args.
- Docs: `run('hello', ...)` uses a single string id. Source: `customId` accepts `string | string[] | RegExp`.
- Docs `onStop` only mentions `timeout`/`idle`. Source adds lifecycle reasons `'messageDelete' | 'channelDelete' | 'guildDelete'`, and `stop(reason)` / manual stop pass any string (the dispatch-time stop defaults to `'stop'`).
- v5 check: this page's API is unchanged in v5. Do NOT confuse it with the EVENT-LEVEL `Collectors` class (next note) — that one renamed its param type `CollectorRunPameters` -> `CollectorRunParameters`, uses camelCase client event names, and runs every matching collector (no first-match `break`). The message collector here is untouched by those deltas.

## Bonus: event-level Collectors (contrast, src-verified)

Different API, same "collector" word. `client.collectors.create({ event, filter, run, ... })` (src/client/collectors.ts:43) listens to ANY client event, not one message. Verified v5 shape — note the `stop` arg in `run`, camelCase event name, and the corrected `CollectorRunParameters` type:

```ts
client.collectors.create({
  event: 'messageCreate', // camelCase client event name (NOT 'MESSAGE_CREATE')
  filter: (message) => message.author.id === someUserId,
  run: (message, stop) => {
    if (message.content === 'stop') return stop('done');
    return message.reply({ content: 'got it' });
  },
  idle: 30_000,
  timeout: 60_000,
  onStop: (reason) => console.log('collector stopped:', reason),
  onStopError: (reason, error) => console.error(reason, error),
  onRunError: (arg, error, stop) => console.error(error),
});
```

v5 facts (src/client/collectors.ts): option type is `CollectorRunParameters<T>` (old `CollectorRunPameters` typo removed); `run`/`filter`/`onRunError` receive `(arg, stop)`-style args; `onStop(reason)` has NO `refresh`; `run('messageCreate', ...)` internally normalizes raw gateway names and now runs EVERY collector whose filter matches (the early `break` is gone — tighten filters if you relied on first-match). Exported root names: `Collectors`, `CollectorRunParameters`, `AllClientEvents`, `ParseClientEventName`.

## Source Anchors

- src/structures/Message.ts:76 (createComponentCollector entry, inherited by BaseMessage)
- src/components/handler.ts:48-59 (CreateComponentCollectorResult), :82-172 (createComponentCollector + run/stop/waitFor/resetTimeouts), :174-207 (onComponent dispatch, filter/onPass/onError), :63-65 (values map + modal registry/expiry)
- src/builders/types.ts:23-41,70-77 (ComponentCallback, ComponentCollectorStopReason, ListenerOptions, ModalSubmitCallback)
- src/builders/Modal.ts:91-94 (Modal.run / __exec), :100-124 (toJSON validation)
- src/structures/Interaction.ts:269 (modal __exec registration by user id), :507-538 (modal() reply + waitFor overload), :1024-1026 (getInputValue), :716-728 (StringSelectMenuInteraction.values typing)
- src/client/collectors.ts (separate event-level Collectors class — contrast only)

## Agent Guidance

- Use message component collectors for short-lived, in-process flows tied to one message (pagination, confirm/cancel, wizards). They do NOT survive restarts or cross shards/workers — for durable/multi-process flows use static `ComponentCommand`/`ModalCommand` with encoded custom-id state.
- Always set `idle` and/or `timeout`, and strip components on stop so dead buttons don't linger.
- Gate with `filter`; ack rejected users with `onPass`; catch handler throws with `onError`.
- Prefer `i.update(...)` over `i.write(...)` for in-place flows; use `i.deferUpdate()` to ack silently.
- `waitFor(customId, timeout?)` is the cleanest way to await a single (or, in a loop, sequential) interaction inline; resolves `null` on timeout or if the collector was cleared.
- For modals: builder `.run(cb)` for a reusable handler, or `ctx.modal(modal, { waitFor })` for an inline await. Modals expire after 10 minutes; Discord emits no cancel event.
- All root imports come from `'seyfert'` (Button, ActionRow, StringSelectMenu, StringSelectOption, Modal, TextInput, Label, Command, Declare, ButtonStyle, TextInputStyle, MessageFlags; types CommandContext / ModalSubmitInteraction / ButtonInteraction). No deep imports needed.
