# Plan versions and subscriber moves

## Versions

A plan points at its current version, and every subscription is pinned to one complete published version of its plan: prices per interval and country, trials, features and allowances, consumption rules, and the add-ons and credit packs it sells with their prices. Renewal, trial conversion, activation, feature access, purchases, and the customer portal read the subscription's own version. Published versions never change. Plan name, description, and visibility are not versioned and apply at once.

Every plan edit publishes a new version and moves no existing subscription, whether it comes from the dashboard, the API, MCP, or the CLI. Existing subscriptions keep their version's prices, features, and extras until the merchant moves them.

In the dashboard, edits go into a draft started from any version. Publishing the draft as the current version, or as an experiment, asks what happens to subscribers of earlier versions: keep them, move them at their next renewal, or move them now. API, MCP, and CLI edits publish a new current version directly and move nobody.

Market price overrides are not versioned. A Market subscription keeps its Market price through version moves, and an edited Market price reaches its subscribers at renewal.

## New subscriptions

Creation sells the plan's current version by default. `planVersionId` sells a specific published version of the plan, an earlier one included, or a version of a running experiment.

An experiment is a version line started from the current version, with its own code. Sell it with `planCode: "<plan>:<code>"`. Commet does not split traffic: the caller decides who gets the experiment. While the experiment runs, its code sells its latest version; after the experiment is made current or archived, the code sells the plan's current version. An experiment code cannot be combined with `planVersionId`. Making an experiment current makes its latest version the plan's current version without moving anyone, because its subscribers are already on it.

An unpaid checkout keeps the price of the version it was opened on. After a plan edit, creating the subscription again for that customer opens a checkout on the current version and replaces the old one.

## Moving subscribers

A subscription reaches another version only through an explicit move:

- a version change for one subscription, effective now or at the end of the period;
- a plan move for a list of subscriptions or for every subscription of a version;
- the move chosen when publishing a dashboard draft or archiving an experiment.

Every move follows the same rules:

- Only `active`, `trialing`, and `past_due` subscriptions move. A plan move leaves out only the subscriptions that cannot move by status, such as `paused` ones.
- A move goes only to a newer version. A subscription on a running experiment moves only to a later version of that experiment. Moving back to an older version is refused.
- A move never changes currency or the price the subscription pays. A destination that does not sell that price and interval, or that currency in the subscriber's country, is refused.
- A free version moves only to a free version, and a paid version only to a paid version.
- A move never prorates. A move `now` changes features, limits, and included credits or balance at once, and the price from the next renewal. A move at the end of the period changes everything at renewal.
- A subscription that does not fit the destination is refused with the reason, such as an active add-on the destination does not sell or more seats than its limit.

A plan move checks every subscriber before moving anyone. If one does not fit, nobody moves until each conflict has a change or is kept on its version. A change applies at that subscriber's renewal: lower its seats to the version's limit, stop an add-on, or switch to another interval the version sells. A preview runs the same checks and records nothing.

A move scheduled for renewal is checked again at renewal. If the subscription no longer fits, the move is canceled and the subscription renews on its version. Commet lists the subscriptions whose move could not complete, so the merchant can resolve each one with a change or keep it on its version.

Repeating the same version change changes nothing, and a new one replaces the pending move. Revoking a pending move leaves the subscription on its version. An immediate cancellation, or a change to another plan, cancels the pending move.

## Add-ons and credit packs

Each add-on and credit pack has a plan scope, `planIds`, where an empty list means every plan. Each plan version sells its own set of extras at its own prices, per country. Activation, renewal lines, credit purchases, and the portal use the subscription's version. An existing subscription cannot activate an add-on or buy a credit pack that only a newer version sells until it moves to that version.

## Reading versions and events

Subscriptions expose their `planVersion` and any `scheduledVersionChange`, and subscription lists filter by `planVersionId`.

`subscription.version_changed` is sent when a move takes effect, now or at renewal. `subscription.version_change_canceled` is sent when a pending move will not happen: the subscription no longer fit at renewal, or the merchant revoked the move (`canceled_by_merchant`). A webhook endpoint receives these events only after it lists them. Take payloads from the installed SDK types.

## Installed contract

Plan version operations, fields, and events need an installed SDK version whose `docs/manifest.json` lists them. If the installed manifest does not, report the package and version; do not substitute a source checkout or newer web documentation.

## Guard effects

Publishing a plan edit and moving subscribers are remote billing effects that change what customers pay. Before either:

1. State the exact organization and `sandbox` or `live` mode.
2. Ask the user what happens to existing subscribers: keep them on their version, move them at their next renewal, or move them now. Never choose for the user, and never assume that an edit reaches existing subscribers.
3. Preview the move and report every subscriber that does not fit, with the change it needs, before applying it.
4. Do not apply a live edit or move unless the user's request explicitly authorizes that effect.
