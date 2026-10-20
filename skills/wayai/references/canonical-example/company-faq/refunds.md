# Refund Policy

## Eligibility window
- Standard plans: refundable within **14 days** of the original charge.
- Annual plans: refundable within **30 days** of the original charge.
- Add-ons (one-time purchases): refundable within **7 days** if unused.

## Non-refundable
- Usage-based charges already consumed (e.g., metered API operations).
- Subscriptions cancelled after the eligibility window — service continues until the period ends.

## Process
1. Confirm the order ID and charge date with the customer.
2. Verify the charge falls inside the eligibility window.
3. If eligible: confirm with the customer, then escalate to `Tier 2 Support` with `ticket_intake.urgency: medium`. Tier 2 issues the refund in Stripe.
4. If not eligible: explain the window, offer account credit (Tier 2 can approve).

## Edge cases
- Duplicate charges: always refundable, regardless of window.
- Service outage refunds: case-by-case — escalate to Tier 2.
