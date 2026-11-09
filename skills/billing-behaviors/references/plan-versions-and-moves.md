# Plan versions and subscriber moves

## Versions

A plan points at its current version, and every subscription is pinned to one complete published version of its plan: prices per interval and country, trials, features and allowances, consumption rules, and the add-ons and credit packs it sells with their prices. Renewal, trial conversion, activation, feature access, purchases, and the customer portal read the subscription's own version. Published versions never change. Plan name, description, and visibility are not versioned and apply at once.

Every plan edit publishes a new version and moves no existing subscription, whether it comes from the dashboard, the API, MCP, or the CLI. Existing subscriptions keep their version's prices, features, and extras until the merchant moves them. Commet never schedules a move on its own. An unpaid checkout is not a subscriber yet and follows the latest version of what was requested; see [New subscriptions](#new-subscriptions).

In the dashboard, edits go into a draft started from any version, shown in the plan header. Publishing the draft as the current version, or as an experiment, asks what happens to subscribers of earlier versions: keep them, move them at their next renewal, or move them now. Drafts published through the API take the same choice. Other API, MCP, and CLI edits publish a new current version directly and move nobody. A draft's or version's changes record unchecking a feature as Disabled; only deleting it is recorded as removed.

The dashboard plan page shows versions with their drafts, experiments, and moves in separate Versions, Experiments, and Moves tabs. Its Subscribers tab shows each subscriber's version, a pending move with its date, and the subscribers that need a change.

Market price overrides are not versioned. A Market subscription keeps its Market price through version moves, and an edited Market price reaches its subscribers at renewal.

## New subscriptions

Creation sells the plan's current version by default. `planVersionId` sells a specific published version of the plan, an earlier one included, or a version of a running experiment.

An experiment is a version line started from the current version, with its own code. Sell it with `planCode: "<plan>:<code>"`. Commet does not split traffic: the caller decides who gets the experiment. While the experiment runs, its code sells its latest version; after the experiment is made current or archived, the code sells the plan's current version. An experiment code cannot be combined with `planVersionId`. Making an experiment current makes its latest version the plan's current version without moving anyone, because its subscribers are already on it.

An unpaid checkout (`draft` or `pending_payment`) follows the latest version of what was requested, and its checkout link charges that version without a new request:

- requested with the plan code, it follows the plan's current version;
- requested with an experiment code, it follows that experiment's latest version, never the plan's main line;
- requested with `planVersionId`, it stays on that version.

Repeating creation with the same request for that customer reuses the unpaid checkout: the same subscription and checkout link, answered with 200 instead of 201, charging that latest version. A different request replaces it: another plan, experiment, version, interval, or price, the plan code after an experiment code, or `planVersionId` after a code, or the other way round, even for the same version. The old subscription is canceled, its invoice voided, and its link expired.

## Moving subscribers

A subscription reaches another version only through an explicit move:

- a version change for one subscription, effective now or at the end of the period;
- a plan move for a list of subscriptions or for every subscription of a version;
- the move chosen when publishing a draft or archiving an experiment.

A version change for one subscription is a move of that one subscriber. Like every move, it is listed in the plan's moves, in the dashboard Moves tab and through the API, and a subscriber that needs a change always names the move it could not complete. A subscription has at most one pending move.

Every move follows the same rules:

- Only `active`, `trialing`, and `past_due` subscriptions move. A plan move leaves out the subscriptions that cannot move by status, such as `paused` ones, and listed subscriptions already on the destination.
- A one-time subscription never renews, so it moves only now. A version change at the end of the period is refused with `invalid_subscription_status` (409), and a plan move at renewal leaves it out.
- A move goes only to a newer version. A subscription on a running experiment moves only to a later version of that experiment. Moving back to an older version is refused.
- A move never changes currency or the price the subscription pays. A destination that does not sell that price and interval, or that currency in the subscriber's country, is refused.
- A free version moves only to a free version, and a paid version only to a paid version.
- A move never prorates. A move `now` deducts usage already tracked, then changes features, limits, and included credits or balance at once, and the price from the next renewal. A move at the end of the period leaves the current period untouched, including balance, usage checks, features, and access, and changes everything at renewal.
- A subscription that does not fit the destination is refused with the reason, such as an active add-on the destination does not sell or more seats than its limit.

A plan move checks every subscriber before moving anyone. If one does not fit, nobody moves until each conflict has a change or is kept on its version. A change applies at that subscriber's renewal: lower its seats to the version's limit, stop an add-on, or switch to another interval the version sells. A preview runs the same checks and records nothing.

Repeating the same version change changes nothing, and a new move replaces the pending one. Revoking a pending move, directly on the subscription or by canceling its plan move, leaves the subscription on its version and sends `subscription.version_change_canceled` with `canceled_by_merchant`. With nothing pending, a revoke changes nothing and sends nothing.

## Moves at renewal

A move at the end of the period waits for the subscription's next renewal, and `scheduledVersionChange.effectiveAt` is that renewal:

- A trial converts on the destination version. The move is dated at the trial end, and nothing changes during the trial.
- A `past_due` subscription keeps renewing once it pays, and the move applies at the renewal after the one being collected. Until then the move is dated at the end of the period that the outstanding renewal pays for.
- After an immediate pause and resume, or an interval change, the move applies at the next renewal that actually happens, and its date follows it.

At renewal the move is checked again. If it still fits, the new period is invoiced on the destination version, with one `subscription.version_changed`. If the subscription no longer fits, the move is canceled with its reason, `subscription.version_change_canceled` is sent, and the subscription renews on its version and appears in Needs a change, so the merchant can resolve it with a change or keep it on its version.

A subscription that ends before that renewal ends on its version, and its move stops waiting: `scheduledVersionChange` becomes `null` and no version event is sent. This applies to a cancellation at period or trial end, exhausted dunning, the suspension of the organization, and an immediate cancellation. While a cancellation or downgrade is only scheduled, the move stays visible with its date, and revoking the cancellation lets it apply at renewal.

A change to another plan, now or at that renewal, including a scheduled downgrade or a cancellation into the plan group's default plan, lands on that plan's current version, drops the move, and sends no `subscription.version_changed`.

## Reactivation

Reactivating a canceled subscription returns it on the latest version of what was bought: the plan's current version when bought with the plan code, the experiment's latest version when bought with an experiment code while the experiment runs, and the same version when bought with `planVersionId`. After the experiment is made current or archived, a subscription bought with its code returns on the plan's current version. Active add-ons that version does not sell are deactivated, and a version that does not sell the subscription's currency in its country refuses the reactivation with `missing_regional_price`. A `past_due` subscription that recovers stays on its version.

## Add-ons and credit packs

Each add-on and credit pack has a plan scope, `planIds`, where an empty list means every plan. Each plan version sells its own set of extras at its own prices, per country. Activation, renewal lines, credit purchases, and the portal use the subscription's version. An existing subscription cannot activate an add-on or buy a credit pack that only a newer version sells until it moves to that version.

## Reading versions and events

Subscriptions expose their `planVersion` and any `scheduledVersionChange` with the date it applies, and subscription lists filter by `planVersionId`.

`subscription.version_changed` is sent when a move takes effect, now or at renewal. It is not sent when an experiment is made current or when the subscription changes to another plan. `subscription.version_change_canceled` is sent when a pending move will not happen: the subscription no longer fit or the move failed at renewal, or the merchant revoked the scheduled version change or canceled the plan move (`canceled_by_merchant`). A move replaced by another, a move made now, a change to another plan, and a subscription that ends send none. A webhook endpoint receives these events only after it lists them. Take payloads from the installed SDK types.

## Installed contract

Plan version operations, fields, and events need an installed SDK version whose `docs/manifest.json` lists them. If the installed manifest does not, report the package and version; do not substitute a source checkout or newer web documentation.

## Guard effects

Publishing a plan edit and moving subscribers are remote billing effects that change what customers pay. Before either:

1. State the exact organization and `sandbox` or `live` mode.
2. Ask the user what happens to existing subscribers: keep them on their version, move them at their next renewal, or move them now. Never choose for the user, and never assume that an edit reaches existing subscribers.
3. Preview the move and report every subscriber that does not fit, with the change it needs, before applying it.
4. Do not apply a live edit or move unless the user's request explicitly authorizes that effect.
