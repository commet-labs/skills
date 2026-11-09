# Catalog, pricing, Offers, and add-ons

## Plans and prices

Paid plans have one or more public plan-price resources. Free plans have no price row. Hiding a plan controls discovery; archiving it does not automatically cancel or migrate subscriptions.

Regional pricing sets the currency and prices each country pays. Configuring a local currency for a plan applies its prices and overage values to every supported country that uses that currency, derived from an exchange rate or fixed. A plan version can also price countries one by one, each with its own currency, USD included outside the United States, and each price set by hand or following the USD base at the country's rate. Markets, available to Enterprise organizations, map reusable, non-overlapping country groups to explicit price and currency overrides. A non-USD Market price requires the plan's regional pricing for that currency first.

The request country at checkout selects the price: a Market override first, then the plan version's price for that country, then the base USD price. The billing address is tax and payer data and never selects pricing. Server-to-server subscription creation has no request country, so the currency is decided when the buyer opens checkout.

A price variant points to one base price through `inheritsFromPriceId` and overrides selected markets. Every unselected market inherits the base price. A caller selects a variant with `priceId`; omitting `priceId` keeps default-price resolution.

A subscription persists the selected price identity and renews at the amount its plan version sells for it. Plan edits publish a new version and do not reach existing subscriptions until they are moved, while an unpaid checkout follows the latest version of what was requested; see [plan-versions-and-moves.md](plan-versions-and-moves.md). Market overrides are not versioned, so an edited Market price reaches its subscribers at renewal. Archiving a price hides it from new selection while existing subscriptions continue using it. A referenced market group cannot be deleted.

## Offers

An Offer is an independent reusable sequence of phases:

- free trial, measured in days, only as the first phase;
- percentage discount;
- amount off with explicit currency amounts;
- fixed price with explicit currency amounts. A zero fixed price is a free cycle.

Non-final discount phases have a bounded number of cycles. The final phase may be indefinite, in which case the customer never returns to the normal price.

The Offer does not contain a purpose or price association. Attach a compatible Offer to one base price for automatic introductory selection, pass `offerId` for direct Promotional selection, reference it from a Promo Code, or distribute it to eligible card BINs through a Card Promotion on Enterprise organizations. Commet does not assign Offer or plan experiment variants; the caller chooses them, and sells a plan experiment with `planCode: "<plan>:<code>"`. An introductory placement accepts at most a leading free trial and one bounded `percentage` or `amount_off` phase.

Accepted offer terms are snapshotted in an Offer Application for billing and audit. Updating or archiving the catalog Offer does not rewrite an existing application. The plan price itself comes from the subscription's plan version, not from the Offer Application.

Promo codes reference an Offer instead of owning separate discount economics. A compatible code Offer has at least one discount phase, optionally preceded by a free trial; a trial-only Offer cannot back a promo code. A code that resolves to a zero first charge saves the payment method and activates without charging. Checkout resolves one active selection source.

## Add-ons

Add-ons and credit packs are sold per plan version. Each has a plan scope, `planIds`, where an empty list means every plan, and each version sells its own set at its own prices, per country. Activation, renewal lines, and credit purchases use the subscription's version; an add-on or pack that only a newer version sells is refused until the subscription moves to that version.

An add-on is compatible when its consumption model is `boolean` or matches the plan's consumption model. Mid-period activation charges a prorated amount of the version's price and grants the feature after successful payment. Deactivation marks it inactive without refunding the current period and revokes a boolean add-on's access unless the plan or a plan grant also provides it.

Recurring billing includes active paid add-ons on billing-month invoices. Discounts do not apply to add-on lines.
