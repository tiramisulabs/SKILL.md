# Creating Polls

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/polls
Coverage reference: components.md (+ builders.md)
Verification status: Source-verified against ./src (the authoritative Seyfert source, v5)

## Page Summary

Polls are created with the `PollBuilder` and attached to an outgoing message via the `poll` field of any message-create/edit body. Receiving votes requires the poll gateway intents (`GuildMessagePolls`, `DirectMessagePolls`) and is handled through the `messagePollVoteAdd` / `messagePollVoteRemove` events; poll completion is detected from a finalized `messageUpdate`. The `Message` and `Poll` structures expose helpers to end a poll, read results, and fetch answer voters. In v5 `PollBuilder.toJSON()` now throws if the poll is missing a question or answers (builders validate before Discord does).

## Key APIs (verified)

`PollBuilder` (src/builders/Poll.ts) — chainable builder, all setters return `this`:
- `new PollBuilder(data?: DeepPartial<RESTAPIPollCreate>)` — constructor seeds `layout_type = PollLayoutType.Default`.
- `setQuestion(data: PollMedia)` — sets question text and optional emoji.
- `setAnswers(...answers: RestOrArray<PollMedia>)` — REPLACES all answers; accepts a spread or a single array (`.setAnswers(a, b)` or `.setAnswers(arr)`).
- `addAnswers(...answers: RestOrArray<PollMedia>)` — APPENDS answers (same `RestOrArray` shape).
- `setDuration(hours: number)` — duration in HOURS (not ms). Discord max is 768 hours (32 days).
- `allowMultiselect(value = true)`.
- `toJSON(): RESTAPIPollCreate` — THROWS `SeyfertError('MISSING_POLL_QUESTION')` if no question text/emoji, and `SeyfertError('MISSING_POLL_ANSWERS')` if `answers` is empty.
- Emoji resolution: `PollMedia.emoji` is an `EmojiResolvable`; an unresolvable emoji THROWS `SeyfertError('INVALID_EMOJI')` at the time the answer/question is added (inside `resolvedPollMedia`).

`PollMedia` type (src/builders/Poll.ts): `Omit<APIPollMedia, 'emoji'> & { emoji?: EmojiResolvable }` — i.e. `{ text?: string; emoji?: EmojiResolvable }`.

Message body `poll` field (src/common/types/write.ts:29): `poll?: PollBuilder | RESTAPIPollCreate | undefined` — accepts the builder INSTANCE directly (no `.toJSON()` needed; the writer serializes it). Present on the create body and on edit bodies.

`Message` structure (src/structures/Message.ts):
- `poll?: Poll` — present when the message contains a poll (line 50; built via `Transformers.Poll`).
- `endPoll(): Promise<MessageStructure>` — delegates to `client.messages.endPoll(channelId, id)`.
- `getAnswerVoters(answerId: ValidAnswerId, checkAnswer = false): Promise<UserStructure[]>` — when `checkAnswer` is true and the message has a poll, throws if the id is not among the poll's answers.

`Poll` structure (src/structures/Poll.ts) — `class Poll extends Base` and `interface Poll extends ObjectToLower<APIPoll>`, so all `APIPoll` fields are camelCased on the instance:
- `end(): Promise<MessageStructure>` (note the asymmetry: on a Message it is `endPoll()`, on the Poll structure it is `end()`).
- `getAnswerVoters(id: ValidAnswerId, checkAnswer = false): Promise<UserStructure[]>` — throws `SeyfertError('INVALID_ANSWER_ID')` when `checkAnswer` is true and id is unknown.
- getters `expiryTimestamp` (number, `Date.parse(this.expiry)`) and `expiryAt` (`Date`).
- camelCased fields: `question` (`{ text?, emoji? }`), `answers[]` (`{ answerId, pollMedia: { text?, emoji? } }`), `results?` (`{ isFinalized, answerCounts: [{ id, count, meVoted }] }`), `allowMultiselect`, `layoutType`, `expiry`, plus `channelId` / `messageId` readonly props.

`ValidAnswerId` (src/api/Routes/channels.ts:213) = `1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10`. Answer ids are 1-based and follow insertion order (first answer added = id `1`). This is also the `id` in `results.answerCounts` and the `answerId` in vote events.

Events (src/events/hooks/message.ts:63-69): `MESSAGE_POLL_VOTE_ADD` / `MESSAGE_POLL_VOTE_REMOVE` both `return toCamelCase(data)`, so the handler `data` is camelCase — `userId`, `messageId`, `channelId`, `answerId`, `guildId?`. Client event names are `messagePollVoteAdd` / `messagePollVoteRemove`.

Intents `GuildMessagePolls` / `DirectMessagePolls` are required to receive vote events.

Root exports (from `'seyfert'`): `PollBuilder`, `createEvent`, `Command`, `SubCommand`, `Declare`, `Options`, `AutoLoad`, `MessageFlags`, `createStringOption`, `createNumberOption`, `CommandContext`, `OKFunction` (`src/builders/index.ts` re-exports `./Poll`).

## Code Examples (verified)

Config intents (seyfert.config.mjs):
```js
export default config.bot({
  // ...other options
  intents: ['GuildMessagePolls', 'DirectMessagePolls'],
});
```

Vote events (`events/addVote.ts`):
```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'messagePollVoteAdd' },
  // data is camelCase: userId, messageId, channelId, answerId, guildId?
  run: (data) => {
    console.log(`User ${data.userId} voted answer ${data.answerId} on poll ${data.messageId}`);
  },
});
```

Detect poll end via finalized messageUpdate:
```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'messageUpdate' },
  // payload is [newMessage, oldMessage]
  run: ([newMessage]) => {
    if (newMessage.poll?.results?.isFinalized) {
      console.log(`Poll ${newMessage.id} has ended`);
    }
  },
});
```

Start a poll (subcommand):
```ts
import {
  type CommandContext,
  Declare,
  type OKFunction,
  Options,
  PollBuilder,
  SubCommand,
  createStringOption,
  createNumberOption,
  MessageFlags,
} from 'seyfert';

const options = {
  question: createStringOption({ description: 'Poll question.', required: true }),
  answers: createStringOption({
    description: 'Poll answers separated by commas.',
    required: true,
    value: ({ value }, ok: OKFunction<string[] | null>) => {
      const answers = value.split(', ');
      if (!answers.length) return ok(null);
      return ok(answers);
    },
  }),
  hours: createNumberOption({ description: 'The duration of the poll (hours).' }),
};

@Declare({ name: 'start', description: 'Create a new poll.' })
@Options(options)
export default class StartPoll extends SubCommand {
  async run(ctx: CommandContext<typeof options>) {
    const { question, answers } = ctx.options;
    const channel = await ctx.channel('rest');
    if (!channel.isTextGuild()) return;
    if (!answers)
      return ctx.editOrReply({ flags: MessageFlags.Ephemeral, content: 'Answers need commas!' });

    const hours = ctx.options.hours ?? 1;

    const newPoll = new PollBuilder()
      .allowMultiselect(true)
      .setQuestion({ text: question })
      .setAnswers(
        // Discord limit: 10 answers (NOT enforced by the builder)
        answers.map((text) => ({ text, emoji: '🐐' })),
      )
      .setDuration(hours);

    await channel.messages.write({ content: 'New poll started.', poll: newPoll });
    await ctx.editOrReply({
      content: `Poll started in ${channel.toString()}`,
      flags: MessageFlags.Ephemeral,
    });
  }
}
```

End a poll (subcommand):
```ts
import {
  type CommandContext,
  Declare,
  Options,
  SubCommand,
  createStringOption,
  MessageFlags,
} from 'seyfert';

const options = {
  message: createStringOption({ description: 'The message id of the poll', required: true }),
};

@Declare({ name: 'end', description: 'End a poll.' })
@Options(options)
export default class EndPoll extends SubCommand {
  async run(ctx: CommandContext<typeof options>) {
    const channel = await ctx.channel('rest');
    if (!channel.isTextGuild()) return;

    const message = await channel.messages.fetch(ctx.options.message);
    if (!message?.poll)
      return ctx.editOrReply({ content: 'Not a poll.', flags: MessageFlags.Ephemeral });

    await message.endPoll(); // Message#endPoll()
    await ctx.editOrReply({ content: 'Poll ended.', flags: MessageFlags.Ephemeral });
  }
}
```

## Recipes (verified)

A fixed two-option poll, no free-text parsing (the simplest copy-paste flow):
```ts
import { Command, Declare, PollBuilder, type CommandContext } from 'seyfert';

@Declare({ name: 'deploy', description: 'Ask the team to approve a deploy.' })
export default class DeployPoll extends Command {
  async run(ctx: CommandContext) {
    const channel = await ctx.channel('rest');
    if (!channel.isTextGuild()) return;

    const poll = new PollBuilder()
      .setQuestion({ text: 'Ship to production?' })
      // emoji can be unicode, a custom-emoji name:id, or an EmojiResolvable
      .setAnswers({ text: 'Yes', emoji: '✅' }, { text: 'No', emoji: '❌' })
      .setDuration(2); // hours

    await channel.messages.write({ content: 'Vote below:', poll });
    await ctx.write({ content: 'Poll posted.' });
  }
}
```

Read final results and fetch the voters for the winning answer. `results.answerCounts`
is camelCase (`{ id, count, meVoted }`) and `getAnswerVoters` takes the 1-based answer id:
```ts
import { createEvent, Formatter } from 'seyfert';

export default createEvent({
  data: { name: 'messageUpdate' },
  run: async ([message]) => {
    const poll = message.poll;
    if (!poll?.results?.isFinalized) return;

    // Sort answers by votes, pick the winner.
    const counts = [...poll.results.answerCounts].sort((a, b) => b.count - a.count);
    const top = counts[0];
    if (!top) return;

    // Map the winning answer id back to its label.
    const winning = poll.answers.find((a) => a.answerId === top.id);
    const label = winning?.pollMedia.text ?? `answer ${top.id}`;

    // top.id is a ValidAnswerId (1..10); fetch the users who picked it.
    const voters = await message.getAnswerVoters(top.id as 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10);

    await message.edit({
      content: `Poll closed. Winner: **${label}** (${top.count} votes). Voters: ${voters
        .map((u) => Formatter.userMention(u.id))
        .join(', ') || 'none'}`,
    });
  },
});
```

Live tally that edits the poll message on every vote (in-memory map; reset on restart):
```ts
import { createEvent } from 'seyfert';

// messageId -> answerId -> Set<userId>
const tallies = new Map<string, Map<number, Set<string>>>();

function bump(messageId: string, answerId: number, userId: string, add: boolean) {
  const byAnswer = tallies.get(messageId) ?? new Map<number, Set<string>>();
  const set = byAnswer.get(answerId) ?? new Set<string>();
  add ? set.add(userId) : set.delete(userId);
  byAnswer.set(answerId, set);
  tallies.set(messageId, byAnswer);
  return byAnswer;
}

function render(byAnswer: Map<number, Set<string>>) {
  return [...byAnswer.entries()]
    .map(([id, voters]) => `Answer ${id}: ${voters.size}`)
    .join('\n');
}

export const onAdd = createEvent({
  data: { name: 'messagePollVoteAdd' },
  run: async (data, client) => {
    const byAnswer = bump(data.messageId, data.answerId, data.userId, true);
    await client.messages.edit(data.messageId, data.channelId, {
      content: `Live results:\n${render(byAnswer)}`,
    });
  },
});

export const onRemove = createEvent({
  data: { name: 'messagePollVoteRemove' },
  run: async (data, client) => {
    const byAnswer = bump(data.messageId, data.answerId, data.userId, false);
    await client.messages.edit(data.messageId, data.channelId, {
      content: `Live results:\n${render(byAnswer)}`,
    });
  },
});
```

End a poll when you hold the `Poll` structure (e.g. read from cache/fetch), using `Poll#end()`:
```ts
const message = await ctx.client.messages.fetch(messageId, channelId);
if (message.poll) {
  await message.poll.end();          // Poll#end()  -> resolves the updated Message
  // equivalent when you only have the Message: await message.endPoll();
}
```

## Common patterns / gotchas

- Pass the `PollBuilder` instance straight into `poll:` — do NOT call `.toJSON()` yourself; the writer serializes it. Calling `.toJSON()` early only matters if you want eager validation.
- `toJSON()` (and therefore sending) THROWS in v5 if there is no question or no answers (`MISSING_POLL_QUESTION` / `MISSING_POLL_ANSWERS`). Always `setQuestion(...)` plus at least one `setAnswers`/`addAnswers` before sending.
- A bad emoji throws `INVALID_EMOJI` at the moment the answer/question is added (inside `setAnswers`/`addAnswers`/`setQuestion`), not at send time.
- `setDuration` is in HOURS, not milliseconds or seconds. Discord allows up to 768 hours (32 days).
- Answer ids are 1-based and follow insertion order; the first answer you add is id `1`. Use these ids with `getAnswerVoters(id)` and to read `results.answerCounts[].id`.
- Discord caps polls at 10 answers — the builder does NOT enforce this, so validate your inputs.
- `setAnswers` REPLACES the answer list; `addAnswers` APPENDS. Both accept either a rest spread or a single array.
- Polls cannot be edited after creation — there is no "edit poll" API; you can only end it (`endPoll()` / `Poll#end()`).
- To receive votes you MUST add `GuildMessagePolls` and/or `DirectMessagePolls` intents; without them the vote events never fire.
- Vote-event `data` is camelCase (`userId`, `messageId`, `channelId`, `answerId`) because the hooks call `toCamelCase(data)`. There is no trailing `shardId` for these (and custom non-gateway events lost their trailing `shardId` in v5).
- For completion, do not rely on a timer: listen to `messageUpdate` and check `newMessage.poll?.results?.isFinalized`. `results` is only meaningful once finalized; intermediate counts may be unstable (Discord caveat).

## Doc vs Source Corrections

- Builder usage, setter names, intents, event names, `Message#endPoll()`, and the start/end command examples all match source — no signature drift from the upstream MDX.
- Clarification (not an error): the vote-event handler reads `data.userId` / `data.messageId` / `data.answerId`. Correct, because the event hooks run `toCamelCase(data)` before dispatch (raw gateway payload is snake_case).
- Asymmetry the docs omit: on a `Message` the method is `endPoll()`, but on the `Poll` structure it is `end()`.
- v5 addition the docs omit: `toJSON()` / the send path throw `SeyfertError` (`MISSING_POLL_QUESTION`, `MISSING_POLL_ANSWERS`, `INVALID_EMOJI`). Builders validate before Discord does now.
- `setDuration` is HOURS — the doc example is correct; do not pass milliseconds.
- `results.answerCounts` field is `meVoted` (camelCase of `me_voted`), not `me_voted`, on the structure.

## Source Anchors

- src/builders/Poll.ts (PollBuilder, PollMedia, toJSON validation throws, emoji resolution)
- src/structures/Poll.ts (Poll structure: end, getAnswerVoters, expiry getters, camelCased APIPoll)
- src/structures/Message.ts:50,192-200 (Message#poll, endPoll, getAnswerVoters)
- src/common/types/write.ts:29 (message body `poll?: PollBuilder | RESTAPIPollCreate`)
- src/events/hooks/message.ts:63-69 (MESSAGE_POLL_VOTE_ADD/REMOVE -> toCamelCase)
- src/common/shorters/messages.ts:131-142 (messages.endPoll, getAnswerVoters)
- src/api/Routes/channels.ts:213 (ValidAnswerId = 1..10)
- src/types/payloads/poll.ts:84-110 (APIPollResults: is_finalized, answer_counts[{ id, count, me_voted }])
- src/builders/index.ts (barrel re-export of Poll)

## Agent Guidance

- Build polls with `PollBuilder` and pass the instance straight into the `poll` field of `messages.write` / `editOrReply` — no manual `.toJSON()`.
- Always `setQuestion` and at least one answer before sending; a missing question or empty answers THROWS in v5. A bad emoji throws as soon as it is added.
- `setDuration` takes hours (max 768). Discord caps answers at 10; the builder will not stop you, so validate inputs.
- Answer ids are 1-based and in insertion order; use them for `getAnswerVoters(id)` and `results.answerCounts[].id`.
- To receive votes, add `GuildMessagePolls` and/or `DirectMessagePolls` intents.
- For poll completion, listen to `messageUpdate` and check `newMessage.poll?.results?.isFinalized`; read `results.answerCounts` ({ id, count, meVoted }) for totals.
- Ending a poll: `Message#endPoll()` (when you hold the message) or `Poll#end()` (when you hold the `Poll` structure). Voter lists: `getAnswerVoters(answerId, checkAnswer?)` exists on both.
