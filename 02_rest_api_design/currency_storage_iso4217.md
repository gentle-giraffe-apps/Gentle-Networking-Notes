
# Currency Storage & ISO 4217 Best Practices for Mobile APIs

## Why Currency Modeling Is a Systems Design Problem

Money is not just a number. It is:

- Regulated
- Jurisdictional
- Rounding-sensitive
- Dispute-prone

On mobile, mistakes propagate silently across retries, offline states, and partial sync windows.

---

## Never Use Floating Point as Source of Truth

`Double` / `Float` introduce rounding drift:

- 9.99 becomes 9.989999
- Serialization differs across platforms
- Reconciliation becomes impossible

---

## Canonical Representation: Minor Units + ISO 4217

```json
{
  "amount_minor": 1999,
  "currency": "USD"
}
```

- `amount_minor` is integer
- `currency` is ISO 4217 code

### Zero-minor-unit currencies
| Currency | Example |
|---------|---------|
| JPY | 500 |
| KRW | 12000 |

The exponent is derived from ISO 4217.

---

## When You Need Sub‑Cent Precision

Use **decimal-as-string** or **microunits**.

### Decimal-as-string

```json
{
  "amount": "0.004731",
  "currency": "USD"
}
```

- Server stores as DECIMAL
- Client parses or treats as string

### Microunits

```json
{
  "amount_micros": 4731,
  "currency": "USD",
  "micros_per_unit": 1000000
}
```

Used for usage-based billing and ledgers.

---

## Reference Prices vs Charge Prices

Mobile apps often display a price hours before checkout.

### Display Price

```json
{
  "reference_price": { "amount_minor": 39727, "currency": "USD" },
  "rate_timestamp": "2025-12-29T18:20:00Z"
}
```

### Checkout Quote

At checkout, the server recalculates:

```json
{
  "quote_id": "q_8192",
  "final_price": { "amount_minor": 39748, "currency": "USD" },
  "tax": { "amount_minor": 21, "currency": "USD" },
  "expires_at": "2025-12-29T18:30:00Z"
}
```

This protects against:

- Tax law changes
- FX drift
- Pricing rule updates

---

## Why Finalization Must Be Server‑Side

Clients cannot be trusted to:

- Apply tax rules correctly
- Use up‑to‑date FX
- Handle rounding modes consistently

The server owns the **billing boundary**.

---

## Handling “Price Changed” UX

Never silently change prices.

| Scenario | UX |
|--------|----|
| Reference ≠ Final | Display delta |
| Quote expired | Force refresh |
| Tax changed | Explain jurisdictional change |

---

## Rounding Rules

Define explicitly:

- When rounding happens
- Rounding mode (half-even recommended)
- Whether tax rounds per-line or per-invoice

Store rounding decisions in code, not tribal knowledge.

---

## Key Concepts

> “We model money using ISO 4217 minor units, keep all intermediate math in high‑precision decimals or microunits, and only finalize rounding at the server billing boundary with expiring quotes.”
