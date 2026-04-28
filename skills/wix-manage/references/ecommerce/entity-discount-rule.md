---
name: "Entity: Discount Rule"
description: Discount rule data model — scope types, discount types, stacking behavior, scheduling, and revision-based updates. Annotates relationships and invariants not explicit in the API docs.
layer: L2
---
# Entity: Discount Rule

A discount rule is an automatic promotion that applies at checkout without requiring a coupon code. Understanding the data model's constraints and relationships is critical for avoiding the most common merchant errors.

---

## Scope types

Each discount entry targets a scope that determines which products receive the discount.

| Scope type | `scope.id` value | What it targets |
|---|---|---|
| `CATALOG` | `"catalog"` (literal string) | All products in the store |
| `COLLECTION` | Collection UUID (from `getCategoryIds`) | All products in a specific collection |
| `SPECIFIC_PRODUCTS` | Product UUID | A single product |

### Scope mutual exclusivity

The scope maps to the merchant's intent as follows:

- **SITE-wide** (CATALOG scope): Both `categoryIds` and `productIds` arrays must be empty. The `scope.id` is the literal string `"catalog"`.
- **CATEGORY-targeted** (COLLECTION scope): Only `categoryIds` is populated (max 3 categories). `productIds` must be empty.
- **ITEMS-targeted** (SPECIFIC_PRODUCTS scope): Only `productIds` is populated (max 5 products). `categoryIds` must be empty.

These modes are mutually exclusive. You cannot mix category and product targeting in a single discount entry.

### ID format requirement

All IDs must be valid GUIDs (e.g., `"a1b2c3d4-e5f6-7890-abcd-ef1234567890"`). Category names like "Adidas" or "Summer Collection" are NOT valid IDs. You must call `getCategoryIds` to convert human-readable category names into their UUID equivalents before constructing the discount rule payload.

---

## Discount types

| Type | Field | Value format | Example |
|---|---|---|---|
| `PERCENTAGE` | `percentage` | Integer 1-100 | `"percentage": 20` |
| `FIXED_AMOUNT` | `fixedAmount` | Decimal string | `"fixedAmount": "5.00"` |
| `FIXED_PRICE` | `fixedPrice` | Decimal string | `"fixedPrice": "29.99"` |

- **PERCENTAGE**: Reduces the price by a percentage. Must be an integer between 1 and 100 inclusive.
- **FIXED_AMOUNT**: Subtracts a fixed monetary amount from the price (e.g., "$5 off"). Value is a decimal string like `"5.00"`.
- **FIXED_PRICE**: Sets the product price to an exact amount regardless of its original price. Use with care as it overrides the catalog price entirely.

---

## Stacking behavior

This is the single most important invariant that causes merchant mistakes:

- Automatic discount rules **stack with each other**. Two active rules targeting overlapping products combine their discounts.
- Automatic discount rules **also stack with manual coupon codes**. A customer using a coupon during a sale period receives both discounts.
- Example: A 20% catalog-wide rule + a 15% collection rule on the same product = combined discount on that product. A customer also applying a 10% coupon code receives all three.

**Always query existing active rules before creating new ones.** Warn the merchant about stacking whenever overlapping scopes are detected.

---

## Scheduling

Discount rules use `activeTimeInfo` for time-bounded promotions:

```json
{
  "activeTimeInfo": {
    "start": "2026-06-01T00:00:00.000Z",
    "end": "2026-06-30T23:59:59.000Z"
  }
}
```

- Both `start` and `end` are ISO 8601 datetime strings.
- Omitting `start` means the rule is active immediately upon creation (if `active: true`).
- Omitting `end` means the rule has no expiration.

### No native auto-deactivation

There is no native scheduling system that automatically deactivates a rule when the end time passes. If the discount is time-limited, the agent must remind the merchant to manually deactivate the rule after the promotion period, or the agent should plan to deactivate it.

### Active flag

The `active` field (boolean) controls the immediate state of the rule:

- `active: true` — the rule applies at checkout (subject to `activeTimeInfo` window).
- `active: false` — the rule is dormant regardless of time window.

---

## Revision system

Discount rules use optimistic concurrency via a `revision` field:

1. Every discount rule response includes a `revision` string (e.g., `"1"`, `"2"`).
2. Update and delete requests **must** include the current `revision` value.
3. On successful update, the response returns an incremented revision.
4. If the provided revision does not match the server's current value, the API returns a `REVISION_MISMATCH` error. The fix is to re-fetch the rule to get the latest revision, then retry the update.

This prevents lost updates when multiple clients modify the same rule concurrently.

---

## References

- [Discount Rules API](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/introduction)
- [Create Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/create-discount-rule)
- [Update Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/update-discount-rule)
