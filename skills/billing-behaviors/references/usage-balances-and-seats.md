# Usage, balances, and seats

## Consumption models

A plan uses one consumption model for usage features:

- `metered` aggregates usage and invoices allowed overage;
- `credits` consumes plan credits and then purchased credits;
- `balance` deducts rate-scale value and either blocks or permits negative overage according to `blockOnExhaustion`.

Feature types are `boolean`, `usage`, `seats`, and `quota`. `metered` is a consumption model, not a persisted feature type.

Use `featureAccess.get()` for current feature state. Use `usage.check()` before a prospective consumption, then `usage.track()` to record it. `usage.set()` replaces a metered feature's current-period usage with an exact value; protect retries with an idempotency key. Usage inputs use `featureCode`; the logical body identity is `eventId`.

## Resets and purchases

- Credits reset `planCredits` and preserve `purchasedCredits`.
- Balance renewal resets `currentBalance`; top-ups are part of the amount replaced by that reset.
- Credit-pack fulfillment increments `purchasedCredits`.
- Balance top-up increments `currentBalance`.
- Cancellation has no special balance mutation.
- Plan-change fulfillment resets the model-owned allocation to the new plan.
- A version move `now` first deducts usage already tracked, then adds or subtracts the difference in included plan credits or plan balance: an increase first pays any plan-part debt, a decrease never creates new debt, and purchased credits and balance are untouched. Scheduling a move at renewal changes nothing in the current period, including balance and `usage.check()`; the new version's allocation applies at that renewal's reset.

## Quota

Quota is a durable held-count feature, not per-period usage. `quota.add`, `set`, and `remove` mutate the held count; `quota.get` and `getAll` read allowances.

## Seats

Seats use advance plus true-up. Incremental additions above the paid or included high-water mark are prorated from the increase time to period end.

Decreases are recorded but do not refund the current period. Re-increasing below the already paid high-water mark does not charge twice.
