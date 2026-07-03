# Modals (Writing Tests)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/modals
Coverage reference: testing.md
Verification status: Source-verified (core) + external package (`@slipher/testing`)

## Page Summary

Covers testing the modal flow with the `@slipher/testing` mock bot. A modal is a two-step interaction: a command/component handler opens it via `ctx.modal(modal, { waitFor })`, then the user submits field values. The mock drives the whole open -> resolve -> settle handshake: dispatch the opener, then chain `.fillModal(customId, values)` to submit, or `.timeoutModal()` to exercise the timeout branch. `@slipher/testing` is NOT part of core Seyfert (verify its version in the target project); the Seyfert builders and the `modal()` callback API it dispatches against ARE verified below.

Two modal architectures exist in Seyfert, and tests differ slightly:
1. **Awaited (inline)**: `await ctx.modal(modal, { waitFor })` returns the submit interaction (or `null` on timeout) in the same handler. Test with `.fillModal(...)` / `.timeoutModal()`.
2. **Decoupled (registered handler)**: open one-shot with `ctx.modal(modal)` (returns `undefined`), and a separate `ModalCommand` class (matched by `customId`) handles the submit. Test by filling the modal so the registered handler runs through the real pipeline.

## Key APIs (verified)

Core Seyfert (all importable from root `'seyfert'`):
- `Modal` builder — `src/builders/Modal.ts`. Methods: `setCustomId(id)`, `setTitle(title)`, `addComponents(...comps)` (rest OR array), `setComponents(comps[])`, `run(callback)`, `toJSON()`. `toJSON()` THROWS `SeyfertError('MISSING_MODAL_CUSTOM_ID')` / `('MISSING_MODAL_TITLE')` if customId/title unset. Constructor accepts `Partial<APIModalInteractionResponseCallbackData>`.
- `Label` builder — `src/builders/Label.ts`. Methods: `setLabel(label)`, `setDescription(description)`, `setComponent(component)`, `toJSON()`. `toJSON()` THROWS `SeyfertError('MISSING_COMPONENT')` if no inner component. Label is the v5 wrapper for modal inputs (replaces the old `ActionRow<TextInput>` form). Accepts a wide component union: `TextInput | BuilderSelectMenus | FileUpload | Checkbox | CheckboxGroup | RadioGroup` (`Label.ts:9`).
- `TextInput` builder — `src/builders/Modal.ts` (exported via builders barrel). Methods: `setCustomId(id)`, `setStyle(style)`, `setPlaceholder(p)`, `setValue(v)`, `setRequired(bool=true)`, `setLength({ min, max })`.
- `TextInputStyle` enum — re-exported from root. Values: `Short`, `Paragraph`.
- `ctx.modal(body, options)` — context method on every command/component context (`src/commands/applications/chatcontext.ts:99-105`, plus menu/entry/component contexts). Overloaded:
  - `modal(body)` (no options) -> `Promise<undefined>`: just opens the modal (one-shot reply). Use with a decoupled `ModalCommand`.
  - `modal(body, { waitFor })` -> `Promise<ModalSubmitInteraction | null>`: opens AND awaits the submit; resolves with the submit interaction, or `null` after `waitFor` ms. Verified at `src/structures/Interaction.ts:507-538`.
  - On a **prefix/message context with no interaction**, `ctx.modal(...)` THROWS `SeyfertError('CANNOT_USE_MODAL')` -> "Cannot use modal without an interaction." (`chatcontext.ts:102`, `common/it/error.ts:101`). Modals are interaction-only.
- `ModalCreateOptions` — `src/common/types/write.ts:78`: `{ waitFor?: number }` (milliseconds). Timeout branch only arms when `waitFor > 0` (`Interaction.ts:527`).
- `ModalCreateBodyRequest` — `src/common/types/write.ts:76`: `APIModalInteractionResponse['data'] | Modal`.
- `ModalSubmitInteraction` — `src/structures/Interaction.ts:866`. `.write(body)` to reply, `.getInputValue(customId, required?)` -> `string | string[] | undefined` (`:1024-1026`), plus `getChannels/getRoles/getUsers/getMentionables/getRadioValues/getCheckbox/getCheckboxValues/getFiles`.
- `ModalCommand` (decoupled handler) — `src/components/modalcommand.ts`. Abstract: `run(ctx: ModalContext)`. Optional `customId?: string | RegExp` and `filter?(ctx)` decide whether it handles a submit (`_filter`, `modalcommand.ts:17-25`). Lifecycle hooks: `onBeforeMiddlewares`, `onAfterRun`, `onRunError`, `onMiddlewaresError(ctx, error, metadata)`, `onInternalError(client, modal, error?)` (modal instance BEFORE error — v5 signature). `middlewares` is a `readonly` tuple.
- `ModalContext` — `src/components/modalcontext.ts:36`. Same accessor set as `ModalSubmitInteraction` plus `write`, `editOrReply`, `editResponse` (STILL exists on modal context — unlike component context), `update(...)`, `deferUpdate()`.

External (doc-authoritative, verify version in target project): `@slipher/testing`
- `createMockBot({ commands, components, modals, ... })` -> disposable mock bot (`await using`). Register decoupled `ModalCommand` classes via the relevant option so `fillModal` can route to them.
- Dispatch chain returns a thenable that MUST be chained with a modal helper when the dispatch opens a modal:
  - `.fillModal(customId, values)` — submits the modal as the same user; `values` keyed by inner-component `customId`.
  - `.timeoutModal()` — resolves the modal's `waitFor` with `null` (no fake-timer setup needed), forcing the handler's timeout branch.
- Awaiting an opener dispatch directly (without `.fillModal`/`.timeoutModal`) fails loud by design.

## Code Examples (verified)

### 1. Command opener + awaited submit (builders verified vs `src/builders/Modal.ts`, `Label.ts`)

```ts
// src/commands/feedback.ts
import { Command, Declare, Label, Modal, TextInput, TextInputStyle, type CommandContext } from 'seyfert';

@Declare({ name: 'feedback', description: 'Send feedback' })
export class FeedbackCommand extends Command {
	async run(ctx: CommandContext) {
		const modal = new Modal()
			.setCustomId('feedback-modal')
			.setTitle('Feedback')
			.addComponents(
				new Label()
					.setLabel('Rating')
					.setComponent(new TextInput().setCustomId('rating').setStyle(TextInputStyle.Short)),
			);

		// waitFor (ms) -> resolves with the submit interaction, or null on timeout
		const submit = await ctx.modal(modal, { waitFor: 60_000 });
		if (!submit) return ctx.write({ content: 'timed out' });
		await submit.write({ content: 'thanks' });
	}
}
```

Submit path — external toolkit API (verify version in target project):

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { FeedbackCommand } from '../src/commands/feedback';

test('feedback modal replies with thanks', async () => {
	await using bot = await createMockBot({ commands: [FeedbackCommand] });

	// `user` must be the SAME identity across opener + fillModal so the submit correlates
	const modal = await bot
		.clickButton('open-feedback', { user })
		.fillModal('feedback-modal', { rating: '5' }); // values keyed by inner customId

	expect(modal.content).toBe('thanks');
});
```

Timeout path:

```ts
const timedOut = await bot
	.clickButton('open-feedback', { user })
	.timeoutModal();

expect(timedOut.content).toBe('timed out');
```

### 2. Multi-field modal + reading several values in the awaited handler

`Label` accepts more than `TextInput` — select menus, checkboxes, radios, file uploads. Read each by its inner `customId`. (Component union verified at `Label.ts:9`; accessor return types at `modalcontext.ts` / `Interaction.ts`.)

```ts
// src/commands/report.ts
import {
	Command, Declare, Label, Modal, TextInput, TextInputStyle,
	ChannelSelectMenu, type CommandContext,
} from 'seyfert';

@Declare({ name: 'report', description: 'File a report' })
export class ReportCommand extends Command {
	async run(ctx: CommandContext) {
		const modal = new Modal()
			.setCustomId('report-modal')
			.setTitle('Report')
			.addComponents(
				new Label()
					.setLabel('Summary')
					.setComponent(
						new TextInput().setCustomId('summary').setStyle(TextInputStyle.Paragraph).setRequired(),
					),
				new Label()
					.setLabel('Where did it happen?')
					.setComponent(new ChannelSelectMenu().setCustomId('where')),
			);

		const submit = await ctx.modal(modal, { waitFor: 120_000 });
		if (!submit) return ctx.write({ content: 'Report cancelled (timed out).' });

		const summary = submit.getInputValue('summary', true); // required -> string
		const channels = submit.getChannels('where');          // AllChannels[] | void

		await submit.write({ content: `Logged: ${summary} (${channels?.length ?? 0} channel(s))` });
	}
}
```

```ts
test('report modal logs summary + channel', async () => {
	await using bot = await createMockBot({ commands: [ReportCommand] });

	const res = await bot
		.clickButton('start-report', { user })
		.fillModal('report-modal', { summary: 'spam in chat', where: ['123'] });

	expect(res.content).toContain('Logged: spam in chat');
});
```

### 3. Decoupled flow — one-shot open + a registered `ModalCommand` handler

The command opens the modal with NO `waitFor` (returns `undefined`), and a separate `ModalCommand` matched by `customId` handles the submit. This is the pattern for persistent modals that may be submitted long after the open.

```ts
// src/commands/nickname.ts
import { Command, Declare, Label, Modal, TextInput, TextInputStyle, type CommandContext } from 'seyfert';

@Declare({ name: 'nickname', description: 'Change your nickname' })
export class NicknameCommand extends Command {
	async run(ctx: CommandContext) {
		const modal = new Modal()
			.setCustomId('nickname-modal')
			.setTitle('New nickname')
			.addComponents(
				new Label()
					.setLabel('Nickname')
					.setComponent(new TextInput().setCustomId('nick').setStyle(TextInputStyle.Short).setRequired()),
			);
		await ctx.modal(modal); // one-shot: no waitFor, resolves to undefined
	}
}
```

```ts
// src/components/nickname-modal.ts
import { ModalCommand, type ModalContext } from 'seyfert';

export default class NicknameModal extends ModalCommand {
	customId = 'nickname-modal'; // string OR RegExp; gates which submits this runs for

	async run(ctx: ModalContext) {
		const nick = ctx.getInputValue('nick', true);
		await ctx.write({ content: `Nickname set to ${nick}` });
	}
}
```

```ts
import { createMockBot } from '@slipher/testing';
import { expect, test } from 'vitest';
import { NicknameCommand } from '../src/commands/nickname';
import NicknameModal from '../src/components/nickname-modal';

test('decoupled nickname modal runs the registered handler', async () => {
	// register BOTH the command and the modal handler so fillModal can route to it
	await using bot = await createMockBot({ commands: [NicknameCommand], modals: [NicknameModal] });

	const res = await bot
		.clickButton('open-nickname', { user })
		.fillModal('nickname-modal', { nick: 'Glorp' });

	expect(res.content).toBe('Nickname set to Glorp');
});
```

## Common patterns / gotchas

- **Same `user` across open + submit.** If a modal flow hangs in a test, the usual cause is a different `user` between the opener dispatch and `.fillModal()`. The submit only correlates if the identity matches.
- **Always chain a modal helper.** Awaiting a modal-opener dispatch directly (no `.fillModal`/`.timeoutModal`) fails loud. In real Seyfert it would stall on the `waitFor` timer and silently take the timeout branch.
- **`waitFor` must be `> 0` to ever time out.** The timeout branch only arms when `waitFor > 0` (`Interaction.ts:527`). The one-arg overload `ctx.modal(modal)` never resolves to `null` — it returns `undefined` immediately and you handle the submit elsewhere.
- **Awaited vs decoupled is a real choice.** Awaited keeps state in the closure (great for short flows); decoupled `ModalCommand` survives restarts and long delays but needs the `customId` registered. Match your test (`{ commands }` only vs `{ commands, modals }`).
- **`customId` can be a RegExp.** `ModalCommand.customId` accepts `string | RegExp` (`modalcommand.ts:20`) — use a pattern for dynamic ids like `report-123`, then parse the suffix from `ctx.customId`.
- **Builders throw early.** `Modal.toJSON()` throws on missing customId/title; `Label.toJSON()` throws on a missing inner component. Partial builders blow up at serialize time (inside `ctx.modal`), which surfaces as a test failure rather than a silent bad payload — a feature, not a bug.
- **Modals are interaction-only.** Opening a modal from a prefix/message command throws `CANNOT_USE_MODAL`. Don't offer a modal flow on a context that may run as a prefix command.
- **`fillModal` keys = inner component customIds**, NOT the modal's customId. Text inputs take a string; select/channel/role/user menus take arrays; checkboxes take booleans/arrays — match the accessor you read with (`getInputValue`/`getChannels`/`getCheckbox`...).

## Doc vs Source Corrections

- None on core APIs. The MDX's `Modal`/`Label`/`TextInput`/`TextInputStyle` usage and the `ctx.modal(modal, { waitFor })` -> `submit | null` contract all match `./src` exactly.
- Clarification (not in docs): the `waitFor` timeout only arms when `waitFor > 0` (`Interaction.ts:527`); passing `0`/omitting it on the awaited overload would never auto-resolve to `null`.
- Clarification (not in docs): `ctx.modal()` throws `CANNOT_USE_MODAL` on a prefix context with no interaction (`chatcontext.ts:102`).
- The doc says "It calls `ctx.modal(...)` (or `interaction.modal(...)`)". Confirmed: both the context wrapper (`chatcontext.ts:99-105`) and the underlying `interaction.modal` (`Interaction.ts:507`) share the identical overload set.
- `@slipher/testing` helpers (`createMockBot`, `clickButton`, `fillModal`, `timeoutModal`) cannot be verified in core Seyfert — external package, doc-authoritative.

## Source Anchors

- `src/builders/Modal.ts` (Modal + TextInput builders; toJSON validation `:100-124`)
- `src/builders/Label.ts` (Label builder; component union `:9`; toJSON throw `:40`)
- `src/structures/Interaction.ts:507-538` (`modal()` overloads, waitFor timer `:527`), `:866` (ModalSubmitInteraction), `:1024-1026` (getInputValue)
- `src/commands/applications/chatcontext.ts:99-105` (context `modal()` wrapper; CANNOT_USE_MODAL throw `:102`)
- `src/components/modalcommand.ts` (ModalCommand: customId/filter `:12-25`, hooks `:31-35`, readonly middlewares `:27`)
- `src/components/modalcontext.ts:36` (ModalContext accessors `:72-140`, editResponse exists)
- `src/common/types/write.ts:76-80` (`ModalCreateBodyRequest`, `ModalCreateOptions`)
- `src/common/it/error.ts:101` (`CANNOT_USE_MODAL` message)
- `src/index.ts` (root re-exports; Modal/Label/TextInput/TextInputStyle/ModalCommand all resolve from 'seyfert')

## Agent Guidance

- For an awaited modal flow, ALWAYS use the two-arg `ctx.modal(modal, { waitFor })` overload — the one-arg `ctx.modal(modal)` only opens the modal and returns `undefined`; pair it with a registered `ModalCommand` to handle the submit.
- Handle the `null` (timeout) result explicitly before calling `submit.write(...)` — the awaited overload returns `ModalSubmitInteraction | null`.
- Read submitted fields with `ctx.getInputValue(customId)` (and `getChannels`/`getRoles`/`getCheckbox`/... for non-text inputs); in tests supply those values via `.fillModal(customId, { field: value })`, keyed by the INNER component customId.
- Build modal inputs with `Label` + `setComponent(...)` (v5 form), not the legacy `ActionRow`. Label takes selects, checkboxes, radios, and file uploads too — not only `TextInput`.
- Register decoupled `ModalCommand` classes in `createMockBot({ modals: [...] })` so `fillModal` can route to the real handler.
- Because `@slipher/testing` is external, pin and verify its version in the consuming project; its chainable helper names may drift from this note.
