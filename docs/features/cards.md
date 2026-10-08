0. What this phase adds

Every user can hold up to 2 standard cards (debit + credit) plus, if they qualify, a third: the Black Card. Real 16-digit numbers (Luhn-valid, real network BIN prefixes), masked-by-default display, a separate ledger for credit spend distinct from the wallet ledger, and a monthly billing cycle with realistic failure consequences (bounce fees, card blocking).

1. Assumptions and flagged decisions

State these back before building — they're reversible, but Cursor shouldn't guess on them silently:

"Account blocked" = the card is blocked, not the user's login. Building it this way — a much less destructive interpretation of what you described. Say now if you actually meant the whole account.
The 2-card limit is 1 debit + 1 credit, not "any 2 of either." The Black Card is a third card, earned, sitting outside that limit — not a replacement for one of the two.
All dollar thresholds ($100,000/year, $1,000/month, $50 bounce fee) use whatever currency the app already stores on accounts.currency — they're not hardcoded to USD or INR specifically.
CVV is stored in this project, which real PCI-DSS-compliant systems never do after issuance — flagged explicitly in §3 as a deliberate, documented demo trade-off, not an oversight.
2. Schema

Three new tables — cards, card_ledger_entries, card_billing_cycles — shown in the diagram above. Key fields not obvious from the diagram:

cards.consecutive_payment_failures — counts bounced monthly bill payments (Credit + Black). Hits 3 → card blocked.
cards.consecutive_low_spend_months — counts months a Black card fell below the $1,000 minimum spend. Hits 3 → card withdrawn. This is a separate counter from consecutive_payment_failures on purpose: missing your minimum spend and bouncing a payment are different failures with different consequences (withdrawal vs. block), and conflating them into one counter would make the business logic ambiguous.
card_ledger_entries.entry_type — CHARGE, PAYMENT, BOUNCE_FEE, REFUND.
card_ledger_entries.billing_cycle_month — which month's bill this entry belongs to (e.g. "2026-09"), so a cycle's total is a simple sum, not a date-range query.
card_billing_cycles.status — PENDING, PAID, BOUNCED, OVERDUE.

Important design point: debit card transactions do not get their own ledger. Per your own spec — "the debit card is attached to the main wallet account, all txn that happens on debit card should be there" — a debit card charge is just a normal ledger_entries row (existing wallet table), tagged with card_id for traceability. Only Credit and Black cards write to the new card_ledger_entries table, because only they represent money the user hasn't paid yet.

3. Business rules
Card limits (set at issuance, editable by admin later if you want)
	Debit	Credit	Black
Per-transaction limit	e.g. 2,000	e.g. 5,000	none / very high
Daily limit	e.g. 5,000	e.g. 15,000	none / very high
Credit limit	n/a	e.g. 50,000	e.g. 200,000

Exact numbers are yours to tune — the important part is that cards has the columns to enforce them, and every credit/black charge checks current_outstanding + amount <= credit_limit before writing.

Black Card eligibility (granting)

Trailing 365-day total spend across wallet debits + debit card + credit card charges ≥ $100,000 → eligible. Check this either on-demand (a "check eligibility" endpoint the Cards page calls) or as part of the monthly job in §3's billing section. A user who qualifies and doesn't already have a Black card gets one issued automatically.

Black Card maintenance (withdrawal)

Once granted, the same monthly job checks: did this Black card's CHARGE entries for the just-completed month total ≥ $1,000? If not, increment consecutive_low_spend_months. At 3, set status = WITHDRAWN. A single good month resets the counter to 0 — this isn't a lifetime ban, it's "you stopped using the card the way that earned it."

Credit/Black monthly billing cycle

Due date is the 5th of every month, fixed. On the 5th, a job (see the orchestration note below) does this for every active Credit/Black card:

Sum unpaid CHARGE entries for the completed month into a new card_billing_cycles row, due_date = the 5th.
Attempt to debit total_due from the user's wallet balance — reuse the exact row-locking pattern already in wallet.py (with_for_update()), since this is the same "don't let two things touch a balance at once" problem your transfer code already solves correctly.
Success: wallet debited, a ledger_entries row records the payment, cycle marked PAID, consecutive_payment_failures resets to 0, a PAYMENT entry zeroes the card's current_outstanding.
Insufficient wallet balance: cycle marked BOUNCED, a $50 BOUNCE_FEE entry is added to current_outstanding, consecutive_payment_failures increments.
Third consecutive failure: cards.status = BLOCKED. current_outstanding stays as a debt figure. A separate "pay off and reactivate" endpoint lets the user clear the negative balance and flips the card back to ACTIVE.

Where does this job actually run? You don't have a scheduler yet — that's Tier 2 of the data-engineering upgrade plan from a couple of phases back (Dagster/Airflow). This is a good, concrete reason to prioritize that: the same orchestrator that runs your Bronze→Silver→Gold pipeline can own this monthly billing job too, on a cron schedule. Until that exists, build it as a manually-triggered admin endpoint (POST /api/admin/cards/run-billing-cycle) so you can test the logic without waiting on infrastructure.

4. Card number, CVV, and expiry generation

Use real network BIN (bank identification number) prefixes so the numbers look authentic, then compute a correct Luhn check digit so the full 16-digit number actually passes validation — this is the same algorithm real card readers use, and it's a nice, legitimate piece of engineering to point to.

Network	Starts with
Visa	4
Mastercard	51–55 or 2221–2720
RuPay	60, 65, 81, 82, or 508

Generate: prefix + random middle digits + Luhn check digit = 16 digits total. Expiry = issuance date + 10 years. CVV = random 3 digits.

Worth saying plainly, and worth documenting in your repo: a real PCI-DSS-compliant system never stores CVV after the transaction that used it, and never stores full PAN in plaintext — this project doing both is a reasonable demo simplification (no real money, no real cardholders), but it's exactly the kind of thing to call out proactively in docs/data-governance.md next to the PII-masking work, as a "what I'd change for production" talking point. Don't let it look like an oversight.

5. UI
Reveal interactions
Card number: masked by default (•••• •••• •••• 4321). Click an eye icon to reveal all 16 digits; click again to re-mask. Persistent toggle, not hold-to-reveal.
CVV: hidden by default (•••). Press-and-hold a separate eye icon to reveal; releasing immediately re-hides it. This small difference (toggle vs. hold) mirrors how real banking apps treat the two — the card number is annoying to re-enter, the CVV is only ever needed for a few seconds.
Card visuals — one real constraint

You asked to reference real Visa/Mastercard/RuPay designs — the color language and layout conventions are completely fine to draw from, but the actual logos (Mastercard's interlocking circles, Visa's wordmark styling, RuPay's mark) are registered trademarks, so build generic cards inspired by the category, not reproductions of the mark: distinct color families per network (blue/navy for Visa-style, a warm dark gradient for Mastercard-style, green-accented for RuPay-style, matte black for the Black Card), a plain text network name, a chip icon, and the reveal-toggle number/CVV fields — same principle already established in `docs/design/design-system.md` §5.6 for merchant icons.

Where this lives

A new Cards page: one card visual per held card (stacked or carousel, similar to the "Cards Overview" pattern already in your dashboard), tap to flip and see the reveal-toggle number/CVV, and below it a tabbed view of that card's own ledger (card_ledger_entries) — separate from the main Passbook, per your spec: "there is a separate ledger for everything."

6. Payment mode integration

Every payment flow (SendMoney, Payments, bill splits) needs a mode selector: Wallet / Debit Card / Credit Card. Routing:

Wallet or Debit Card → existing ledger_entries (wallet), tagged with card_id if a debit card was used. No new logic — same balance, same lock pattern, just an extra optional column.
Credit Card → new card_ledger_entries, checked against credit_limit - current_outstanding instead of wallet balance. Does not touch the wallet at the time of purchase — that only happens at the monthly billing cycle.
7. Mock data for your ~1000 seeded users

Extend backend/scripts/seed_demo_data.py (or add seed_cards_demo.py) to generate exactly the kind of messy, uneven data your data-quality work needs:

Weighted random card ownership: most users get a debit card, fewer get a credit card, a small number get neither, a smaller number are Black-card-eligible.
For Black Card eligibility specifically, compute it from the real seeded ledger data where possible (trailing-365-day spend ≥ $100,000) rather than assigning it purely at random — it's more realistic and gives you genuine edge cases: users who should be eligible but aren't yet granted one, and users who were granted one and have since fallen below the $1,000 monthly minimum.
Deliberately inject: null CVVs, missing expiry dates, mismatched name casing/whitespace on cardholder_name, a few near-duplicate card numbers, nonsensical negative current_outstanding values, billing cycles with missing due_date. Document these as intentional fixtures for the Great Expectations work from the data-engineering phase — this is genuinely useful test data, not just noise.


8. Suggested order for Cursor
Schema: cards, card_ledger_entries, card_billing_cycles (Alembic migration).

Card issuance service: Luhn generation, BIN prefixes, limits by card type/network.

Cards page UI: card visuals, reveal toggles, per-card ledger view (build against manually-created test cards first, before wiring real business logic).

Payment-mode selector in existing payment flows, routing to wallet vs. card ledger.

Monthly billing cycle logic, exposed as a manually-triggered admin endpoint for now.

Black Card eligibility + maintenance checks (can share the same monthly job as step 5).

Mock data generation script for the 1000 seeded users.


9. upgrades 
Interest on unpaid balances, not just a flat $50 bounce fee — a real credit card compounds interest monthly on a carried balance. More realistic, and a nice small-scale "how would you model compounding" exercise.

Reward points ledger for the Black Card — instead of "benefits" being static descriptive text, an actual points balance that accrues on spend and can be redeemed. This reuses your append-only ledger pattern a third time (wallet, credit card, now points) instead of introducing a new one.

Instant freeze/unfreeze toggle — a real, cheap, high-value feature: user can freeze a card themselves (sets status = FROZEN, blocks new charges, reversible), independent of the billing-failure block logic.