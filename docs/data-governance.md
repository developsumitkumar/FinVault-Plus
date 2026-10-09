# Data governance notes

Portfolio / demo project notes on sensitive data handling. These are deliberate trade-offs for a non-production FinVault build — call them out in interviews rather than treating them as oversights.

## Card PAN and CVV (demo only)

This project stores **full 16-digit PANs in plaintext** and **stores CVV after issuance** so the Cards UI can support reveal / hold-to-reveal interactions.

**What a PCI-DSS-compliant production system would do instead:**

- Never store CVV after the authorization that used it (PCI DSS requirement).
- Store PAN only in truncated (`last4`) or tokenized form; full PAN in a dedicated vault / HSM with strict access controls.
- Transmit card data only over TLS; restrict logging so PAN/CVV never land in application logs or analytics events.
- Prefer network tokenization (Visa/Mastercard/RuPay tokens) for recurring charges.

FinVault keeps plaintext PAN + CVV because there is no real money and no real cardholders. Treat any production port as a rebuild of this surface, not a config flip.

## Related PII

User emails, KYC document numbers, and uploaded KYC files are also sensitive. Prefer:

- Masking in analytics / Bronze→Silver pipelines (see `docs/data-quality.md` and pipeline docs).
- Role-scoped admin access for KYC review.
- Avoid committing real identity documents or production dumps to git.

## Intentional bad card fixtures

`backend/scripts/seed_cards_demo.py` injects messy rows (empty CVV, missing expiry, negative outstanding, null billing `due_date`, near-duplicate PANs) for Great Expectations / data-quality demos. Document those fixtures in DQ suites as expected fail cases, not as silent corruption.
