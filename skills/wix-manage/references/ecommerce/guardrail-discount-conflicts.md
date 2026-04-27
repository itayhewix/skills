---
name: "Guardrail: Discount Conflicts"
description: Validation rules for detecting and preventing discount stacking conflicts, coupon overlap, and unintended deep discounts before applying new promotions. Covers automatic discount rules, coupon interactions, and safe discount limits.
layer: L5
---
# Guardrail: Discount Conflicts

## When to use this guardrail

Run these checks **before** creating or updating any discount rule. Discount conflicts are one of the most common merchant mistakes — two overlapping discounts can silently stack and give customers a much deeper discount than intended.

---

## Check 1: Existing active discount rules on the same scope

**Why:** Wix eCommerce stacks automatic discount rules. If a 20% catalog-wide discount and a 15% collection discount both apply to the same product, the customer may get both applied.

**How to check:**

1. Query all active discount rules:

**Endpoint**: `POST https://www.wixapis.com/ecom/v1/discount-rules/query`

**Request**:
```json
{
  "query": {
    "filter": {
      "active": true
    },
    "paging": {
      "limit": 100
    }
  }
}
```

2. For each existing active rule, compare its scope against the new rule:
   - If both target `CATALOG` scope: **conflict** — both apply to all products
   - If new rule targets `CATALOG` and existing targets `COLLECTION`: **conflict** — catalog-wide includes that collection
   - If both target the same `COLLECTION` ID: **conflict** — same products affected
   - If both target the same `SPECIFIC_PRODUCTS` ID: **conflict** — same product

3. If a conflict is found, warn the merchant:
   > "There's already an active discount '{existingRuleName}' ({existingDiscount}%) that applies to the same products. Adding this new {newDiscount}% discount may stack, giving customers a combined discount. Would you like to deactivate the existing rule first, or proceed with both?"

---

## Check 2: Discount percentage sanity

**Why:** A discount above 50% is unusual and may indicate a typo (the merchant meant 15% not 50%). A discount of 100% makes the product free.

**Rules:**
- Discount > 50%: Warn — "This discount is {percentage}% off. Are you sure? This means a $100 product would sell for ${100 - percentage}."
- Discount = 100%: Block unless explicitly confirmed — "This would make the product free. Please confirm this is intentional."
- Discount > 100%: Block — "A discount cannot exceed 100%."

---

## Check 3: Time overlap with existing promotions

**Why:** Two promotions running simultaneously on overlapping products cause stacking during the overlap period.

**How to check:**

1. For the new rule's `activeTimeInfo` (start/end), check if any existing active rule has an overlapping time window on the same scope.
2. Overlap exists when: `existingStart < newEnd AND existingEnd > newStart`
3. If overlap found on the same scope: warn the merchant about the overlap period.

---

## Check 4: Coupon interaction

**Why:** Coupons (manual codes) and automatic discount rules can stack. A customer with a 20% coupon code buying during a 20% automatic sale gets ~36% off total.

**How to check:**

1. Query active coupons (if available via API)
2. If coupons exist that target the same products/collections as the new discount rule, warn:
   > "There are active coupon codes that apply to the same products. Customers using these coupons during the sale will get both the coupon discount and the automatic discount stacked."

---

## Check 5: Minimum profit margin

**Why:** Deep discounts on low-margin products can result in selling at a loss.

**Rules:**
- If the merchant has provided cost/margin information, verify that the discount doesn't reduce the price below cost
- If no cost information is available, warn for discounts > 40% as a general safety threshold

---

## Summary: Decision matrix

| Scenario | Action |
|---|---|
| No conflicts found | Proceed with creating the discount rule |
| Scope overlap with existing active rule | Warn merchant, ask to deactivate existing or confirm stacking |
| Discount > 50% | Warn merchant, ask for confirmation |
| Discount = 100% | Block unless explicitly confirmed |
| Time overlap on same scope | Warn about overlap period |
| Coupon stacking risk | Inform merchant about potential stacking |

## References

- [Discount Rules API](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/introduction)
