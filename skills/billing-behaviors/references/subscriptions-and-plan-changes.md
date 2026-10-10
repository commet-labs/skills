# Subscriptions and plan changes

## Lifecycle and access

Current subscription statuses are `draft`, `pending_payment`, `trialing`, `active`, `past_due`, `paused`, and `canceled`.

Feature access is granted for `active`, `trialing`, and `past_due`. `pending_payment` does not grant access before the first payment succeeds, and `paused` does not grant access.

A customer has at most one open subscription (`draft` through `paused`) per organization, including under concurrent creates. Repeating creation while the open subscription is `pending_payment` reuses it when plan, price, interval, and requested promotion match; omitting a previously requested promo code or direct Offer replaces it and invalidates the old checkout.

Free subscriptions activate without a paid checkout. Paid and trial flows route through the organization's configured payment provider.

When a trial reaches its first paid invoice, a retryable provider decline enters `past_due` recovery. Missing-payment-method and customer-action-required outcomes remain `pending_payment`.

## Introductory Offers

An automatic Introductory Offer is an independent Offer placed on one base plan price. Subscription creation applies it when the customer is eligible and no explicit override is supplied.

Current eligibility excludes customers with an `active`, `past_due`, or `paused` subscription in the organization. A direct `offerId` is an explicit override. An immediate plan change ends the existing Offer Application and may accept a new `offerId`; a scheduled change does not accept one. Credit for the old plan uses the effective price actually paid.

## Plan and interval changes

Normal portal changes require both plans to belong to the same plan group. Classification is not based on comparing prices:

- an interval upgrade is immediate;
- an interval downgrade is scheduled;
- a lower `sortOrder` in the group is a scheduled plan downgrade;
- paid-to-free is scheduled;
- other valid changes are immediate;
- free-to-free is immediate and creates no charge.

Immediate paid changes restart the billing period. Commet credits the unused portion of the old effective plan price, charges the full new plan price, applies available customer credits, clears the prior discount, resets the new plan allocation, and records a new feature snapshot.

Scheduled changes take effect at trial end or current-period end.

## Cancellation and reactivation

Cancellation can be immediate or scheduled. Free, `pending_payment`, `past_due`, and `paused` subscriptions cancel immediately; one-time subscriptions are not cancelable. A scheduled trial cancellation ends at trial end without converting to paid or charging. A scheduled paid cancellation ends at the current paid period boundary, extended only through contiguous accepted free phases (a zero fixed price); later paid Offer phases do not extend it. Canceling a past-due subscription stops dunning and voids unpaid invoices.

A full or partial refund does not change the subscription. Cancel it separately when access should end.

Cancellation does not reset or delete the persisted subscription balance.

Reactivation charges first and restores state only after payment succeeds. It does not automatically reset an already initialized balance.

## Pause and resume

Only paid recurring subscriptions in `active` or `trialing` can be paused, and not while a cancellation, plan change, or plan-change checkout is pending.

- `immediate` stops access now and preserves the unused paid or trial time. Resuming restores that remaining time without a charge, and renewal moves by the paused duration.
- `period_end` keeps access until the current period or trial ends and skips the next renewal. Resuming charges a new period with the saved payment method; access returns after settlement.

A positive duration schedules an automatic resume attempt measured from the effective pause time; no duration means the pause lasts until a manual resume. Paused subscriptions do not generate renewal invoices, but a period-end pause can still invoice usage, overage, or seat adjustments from the period that ended.

A declined resume charge keeps the subscription `paused` and its invoice outstanding while the normal retry schedule runs; exhausting it cancels the subscription. Canceling a paused subscription discards its preserved time; a scheduled pause must be revoked before cancellation. Use resume for `paused` and reactivation for `canceled`.
