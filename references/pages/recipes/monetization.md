# Monetization

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/monetization
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (re-verified against the target project installed `seyfert` package or provided Seyfert source)

## Page Summary

Seyfert supports Discord monetization: entitlements, SKUs, premium buttons, and entitlement gateway events. An entitlement represents that a user or guild has access to a premium offering. You receive `entitlementCreate` / `entitlementUpdate` / `entitlementDelete` events, build premium buttons with `ButtonStyle.Premium` + a SKU id, and read active entitlements off any interaction (`ctx.interaction.entitlements`). Monetization must first be enabled in the Discord developer portal and SKUs created there; that setup is external to Seyfert.

## Key APIs (verified)

- `Entitlement` structure — `src/structures/Entitlement.ts`. `interface Entitlement extends ObjectToLower<APIEntitlement>`; class extends `DiscordBase<APIEntitlement>`. Fields are camelCase: `id`, `skuId`, `userId?`, `guildId?`, `applicationId`, `type`, `deleted`, `startsAt` (string|null), `endsAt` (string|null), `consumed?`.
  - Getters: `startsAtTimestamp` / `endsAtTimestamp` → `number | null` (`Date.parse` of the ISO strings, else null).
  - Method: `consume()` → `client.applications.consumeEntitlement(this.id)` (for consumable items).
- `EntitlementStructure` — `src/client/transformers.ts`, `InferCustomStructure<Entitlement, 'Entitlement'>`. This is what events and interactions actually deliver (custom-structure aware), NOT the raw `Entitlement` class. Built via `Transformers.Entitlement(client, apiEntitlement)`.
- Entitlement events — `src/events/hooks/entitlement.ts`. Hooks `ENTITLEMENT_CREATE/UPDATE/DELETE` each transform `APIEntitlement` → `EntitlementStructure`. `createEvent` names: `entitlementCreate`, `entitlementUpdate`, `entitlementDelete`. Handler signature `(entitlement: EntitlementStructure, client) => Awaitable<unknown>`.
- `ctx.interaction.entitlements` — `src/structures/Interaction.ts:110,135`. Typed `EntitlementStructure[]`, built per-interaction via `interaction.entitlements.map(e => Transformers.Entitlement(...))`. Present on command and component interactions (`declare entitlements` at lines 441, 668).
- `Button` premium API — `src/builders/Button.ts:78`: `setSKUId(skuId: string)` (capital S-K-U) sets `sku_id`. `setStyle(ButtonStyle.Premium)` (`src/types/payloads/components.ts:188`). Premium buttons OMIT `custom_id`, `emoji`, `label` (the type `Omit<...,'custom_id'|'emoji'|'label'>` at `components.ts:167`). They do NOT emit component interactions when clicked — Discord handles the purchase flow.
- `ApplicationShorter` (`client.applications`) — `src/common/shorters/application.ts`:
  - `listEntitlements(query?: RESTGetAPIEntitlementsQuery): Promise<EntitlementStructure[]>` (query keys are snake_case, e.g. `user_id`, `guild_id`, `sku_ids`, `exclude_ended`).
  - `consumeEntitlement(entitlementId: string)`
  - `createTestEntitlement(body: RESTPostAPIEntitlementBody): Promise<EntitlementStructure>`
  - `deleteTestEntitlement(entitlementId: string)`
  - `listSKUs()` → raw `APISKU[]` from the API (not transformed).
- Monetization types — `src/types/payloads/monetization.ts`: `APIEntitlement`, `EntitlementType` (1=`Purchase`, 2=`PremiumSubscription`, 3=`DeveloperGift`, 4=`TestModePurchase`, 5=`FreePurchase`, 6=`UserGift`, 7=`PremiumPurchase`, 8=`ApplicationSubscription`), `APISKU`, `SKUType` (Durable=2, Consumable=3, Subscription=5, SubscriptionGroup=6), `SKUFlags`, `APISubscription`, `SubscriptionStatus`.

## Code Examples (verified)

### Entitlement events

Handlers receive the transformed `EntitlementStructure`.

```ts
// src/events/entitlementCreate.ts — fires on a new purchase/subscription
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'entitlementCreate' },
  async run(entitlement, client) {
    if (!entitlement.userId) return; // guild-scoped entitlement: no userId
    const user = await client.users.fetch(entitlement.userId);
    await client.messages.write('LOG_CHANNEL_ID', {
      content: `${user.globalName} (${user.id}) subscribed to ${entitlement.skuId}`,
    });
  },
});
```

```ts
// src/events/entitlementUpdate.ts — fires on renewal (and on cancel scheduling)
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'entitlementUpdate' },
  run: async (entitlement, client) => {
    if (!entitlement.userId) return;
    const user = await client.users.fetch(entitlement.userId);
    await client.messages.write('LOG_CHANNEL_ID', {
      content: `Subscription (${entitlement.skuId}) renewed by ${user.globalName}`,
    });
  },
});
```

```ts
// src/events/entitlementDelete.ts — refund / manual deletion ONLY (never expiry)
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'entitlementDelete' },
  run(entitlement, client) {
    void client.messages.write('LOG_CHANNEL_ID', {
      content: `Entitlement removed (sku ${entitlement.skuId}) [type ${entitlement.type}]`,
    });
  },
});
```

### Premium button

```ts
import { Button, ButtonStyle } from 'seyfert';

new Button()
  .setSKUId('STORE_ITEM_SKU_ID')
  .setStyle(ButtonStyle.Premium);
// premium buttons need no customId / label / emoji; clicking opens Discord's checkout
```

### Command gating on entitlements

```ts
import { Declare, Command, type CommandContext, ActionRow, Button, ButtonStyle } from 'seyfert';

@Declare({ name: 'premium', description: 'Premium command' })
export class PremiumCommand extends Command {
  async run(ctx: CommandContext) {
    const isPremium = ctx.interaction.entitlements.length > 0;

    if (!isPremium) {
      const row = new ActionRow<Button>().setComponents([
        new Button().setSKUId('STORE_ITEM_SKU_ID').setStyle(ButtonStyle.Premium),
      ]);
      return ctx.editOrReply({
        content: 'Click to subscribe and get access to this command!',
        components: [row],
      });
    }

    // premium-only logic here
    await ctx.editOrReply({ content: 'Thanks for subscribing!' });
  }
}
```

### Recipe: gate by a specific SKU + non-expired check

`entitlements.length` only tells you the user has *something*. To gate a feature behind one SKU, filter by `skuId` and verify the window with the timestamp getters.

```ts
import { Declare, Command, type CommandContext } from 'seyfert';

const PRO_SKU = 'PRO_TIER_SKU_ID';

function hasActiveSku(ctx: CommandContext, skuId: string) {
  const now = Date.now();
  return ctx.interaction.entitlements.some(
    e =>
      e.skuId === skuId &&
      !e.deleted &&
      // null endsAt = perpetual; otherwise must still be in the future
      (e.endsAtTimestamp === null || e.endsAtTimestamp > now),
  );
}

@Declare({ name: 'pro', description: 'Pro-tier feature' })
export class ProCommand extends Command {
  async run(ctx: CommandContext) {
    if (!hasActiveSku(ctx, PRO_SKU)) {
      return ctx.editOrReply({ content: 'This feature requires the Pro tier.' });
    }
    await ctx.editOrReply({ content: 'Welcome, Pro member!' });
  }
}
```

### Recipe: consumable purchase flow (deliver, then consume)

For consumable SKUs you grant the item and then mark the entitlement consumed so Discord lets the user buy it again. Find the unconsumed entitlement on the interaction (or via `listEntitlements`), deliver, then call `consume()`.

```ts
import { Declare, Command, type CommandContext } from 'seyfert';

const GEM_PACK_SKU = 'GEM_PACK_SKU_ID';

@Declare({ name: 'claim', description: 'Redeem a purchased gem pack' })
export class ClaimCommand extends Command {
  async run(ctx: CommandContext) {
    const pending = ctx.interaction.entitlements.find(
      e => e.skuId === GEM_PACK_SKU && e.consumed === false,
    );
    if (!pending) {
      return ctx.editOrReply({ content: 'No unredeemed gem packs found.' });
    }

    // 1) deliver the goods in your own DB first (idempotent on entitlement id)
    await grantGems(ctx.author.id, 100);

    // 2) tell Discord it's been consumed so it can be repurchased
    await pending.consume(); // === client.applications.consumeEntitlement(pending.id)

    await ctx.editOrReply({ content: '100 gems added to your balance!' });
  }
}

declare function grantGems(userId: string, amount: number): Promise<void>;
```

### Recipe: server-side audit / reconciliation

Outside an interaction (e.g. a cron job) you don't have `ctx.interaction.entitlements`; query the REST API. `listEntitlements` query keys are snake_case.

```ts
// all active (non-ended) entitlements for one user
const active = await client.applications.listEntitlements({
  user_id: 'USER_ID',
  exclude_ended: true,
});

// everyone entitled to a specific SKU
const subs = await client.applications.listEntitlements({ sku_ids: 'SKU_ID' });

// map your SKUs to display names
import { SKUType } from 'seyfert';
const skus = await client.applications.listSKUs(); // raw APISKU[]
const subscriptions = skus.filter(s => s.type === SKUType.Subscription);
```

### Recipe: simulating purchases in development

`createTestEntitlement` / `deleteTestEntitlement` let you exercise the whole flow with no real payment. The test entitlement also triggers `entitlementCreate`.

```ts
// grant a test entitlement to a user
const test = await client.applications.createTestEntitlement({
  sku_id: 'SKU_ID',
  owner_id: 'USER_ID',
  owner_type: 2, // 1 = guild subscription, 2 = user subscription
});

// later, clean it up
await client.applications.deleteTestEntitlement(test.id);
```

## Doc vs Source Corrections

- Docs type the event parameter as `Entitlement` → src delivers `EntitlementStructure` (custom-structure-aware transformer; `src/client/transformers.ts`, `src/events/hooks/entitlement.ts`). Fields are identical; only the type name differs.
- Docs (and the original draft) omit `Entitlement.consume()` and the `startsAtTimestamp` / `endsAtTimestamp` getters → both present in `src/structures/Entitlement.ts`.
- Docs omit the `client.applications` helpers (`listEntitlements`, `consumeEntitlement`, `createTestEntitlement`, `deleteTestEntitlement`, `listSKUs`) → present in `src/common/shorters/application.ts`.
- Method name is `setSKUId` (capital S-K-U), matching docs; do NOT write `setSkuId`.
- `listSKUs()` returns raw `APISKU[]`, NOT a transformed structure (no `Transformers` call in the source).

## Common patterns / gotchas

- `ctx.interaction.entitlements` is a per-request snapshot Discord sends with the interaction — it is NOT a live cache and there is no `cache.entitlements`. For background jobs use `client.applications.listEntitlements(...)`.
- `entitlementDelete` is NOT an expiry signal — it fires only on refund or manual deletion. Subscriptions that simply lapse never emit it. Detect lapses via `endsAt`/`endsAtTimestamp` or `entitlementUpdate`.
- Premium buttons (`ButtonStyle.Premium`) do not produce a `ComponentContext` interaction; do not register a component handler for them. Discord owns the checkout UI.
- Guild-scoped entitlements have `guildId` but no `userId` (and vice-versa). Always null-check `entitlement.userId` before `client.users.fetch`.
- Event `run` is typed `Awaitable<unknown>` in v5 — returning a value is fine but ignored; custom (non-gateway) handlers no longer receive a trailing `shardId` (entitlement events are gateway events, so their signature is `(entitlement, client)`).
- `consume()` only makes sense for consumable SKUs (`SKUType.Consumable`); calling it on a subscription entitlement is meaningless. Deliver the item in your own store first, then consume, so a failed delivery doesn't lose the purchase.
- `EntitlementType.ApplicationSubscription` (8) is the value for app-subscription purchases; `PremiumSubscription` (2) is Discord Nitro, not your SKU. Gate on `skuId` rather than `type` when you can.

## Source Anchors

- `src/structures/Entitlement.ts` (consume, startsAtTimestamp/endsAtTimestamp)
- `src/client/transformers.ts` (EntitlementStructure, Transformers.Entitlement)
- `src/events/hooks/entitlement.ts` (ENTITLEMENT_CREATE/UPDATE/DELETE)
- `src/structures/Interaction.ts:110,135,441,668` (entitlements field)
- `src/builders/Button.ts:78` (setSKUId / setStyle)
- `src/types/payloads/monetization.ts` (APIEntitlement, EntitlementType, SKU/Subscription types)
- `src/types/payloads/components.ts:167,188` (ButtonStyle.Premium; premium button omits custom_id/emoji/label)
- `src/common/shorters/application.ts:92-132` (entitlement/SKU shorters)

## Agent Guidance

- Use `entitlement.userId` / `entitlement.guildId` to know who/what is entitled; gate premium features by checking `ctx.interaction.entitlements` and filtering by `skuId` (and `endsAtTimestamp`/`deleted`) rather than only `.length`.
- For consumable purchases, deliver the item, then call `entitlement.consume()` (or `client.applications.consumeEntitlement(id)`).
- During development, use `createTestEntitlement` / `deleteTestEntitlement` to simulate purchases without real payments.
- `entitlementDelete` is NOT expiry — it fires only on refund/manual deletion. Track `endsAt` or use `entitlementUpdate` for lapses.
- Root imports (`Button`, `ButtonStyle`, `ActionRow`, `Command`, `Declare`, `CommandContext`, `createEvent`, `SKUType`, `EntitlementType`) all come from `'seyfert'`. No deep import needed.
- External prerequisite: monetization must be enabled in the Discord developer portal and SKUs created there before any of this works; that step is outside Seyfert.
