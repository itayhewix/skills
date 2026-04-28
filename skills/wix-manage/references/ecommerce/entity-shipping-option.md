---
name: "Entity: Shipping Option"
description: Shipping option data model — rate structure types, condition logic, multiplyByQuantity anti-pattern, region linkage, and orphaned option detection.
layer: L2
---
# Entity: Shipping Option

## Rate Structure

A shipping option's rate structure is determined by its `conditions[]`, not a separate type field. Identify the structure by inspecting what conditions are present:

| Structure | Conditions Pattern |
|---|---|
| **Flat rate** | `conditions[]` is empty. A single fixed `amount`. |
| **Free shipping** | `amount="0"`. May optionally include a `BY_TOTAL_PRICE` GTE condition to set a minimum order threshold. |
| **Weight-based** | Conditions use `BY_TOTAL_WEIGHT`. |
| **Price-based** | Conditions use `BY_TOTAL_PRICE`. |
| **Quantity-based** | Conditions use `BY_TOTAL_QUANTITY`. |
| **Tiered pricing** | Multiple rate entries with different condition ranges (e.g., 0-5kg at $5, 5-10kg at $8). |

---

## Condition Logic

### Operators

Conditions support these operators: `EQ`, `GT`, `GTE`, `LT`, `LTE`.

### Evaluation Rules

- Multiple conditions within the **same rate** use AND logic — all must match for the rate to apply.
- At runtime, up to **one rate matches** per shipping option. If multiple rates match, the **lowest** amount is returned.

---

## multiplyByQuantity Anti-Pattern

When `multiplyByQuantity=true`, the shipping amount is multiplied by the number of line items in the cart.

**Example**: A $5 flat rate with multiplyByQuantity=true on a 5-item order charges $25 for shipping.

**Why this is always flagged**:
- Discourages larger carts, directly conflicting with AOV growth goals.
- Creates unpredictable shipping costs that customers only discover at checkout.
- Penalizes customers who buy multiple items, the opposite of the desired incentive.

**Always flag multiplyByQuantity=true regardless of the rate amount.**

---

## Region Linkage

Each shipping option has a `deliveryRegionIds[]` array linking it to specific delivery regions:

- Links are **immutable after creation** — to change the region association, the option must be deleted and recreated.
- A single option can link to **up to 50 regions**.
- Region linkage determines where the option is available. If a region has no linked shipping options, customers shipping to that region's destinations see nothing at checkout.

---

## Orphaned Options

An orphaned shipping option is one whose `deliveryRegionIds[]` references regions that have been **deleted or no longer exist** in any delivery profile.

Orphaned options:
- Exist in the system and can be queried.
- Are **never shown** to customers at checkout.
- Consume configuration space and create confusion in the dashboard.
- Are detected by cross-referencing `deliveryRegionIds` against all regions in all delivery profiles.

---

## Options on Inactive Regions

Shipping options linked to regions with `active=false` are **invisible at checkout**, even if the options themselves are correctly configured.

This is the **most commonly missed issue**: the merchant sees the options listed in their dashboard and believes they are live, but customers shipping to those destinations see nothing. Always cross-check region active state when investigating missing shipping options.

---

## Tier Gaps

In weight-based, price-based, or quantity-based options with tiered pricing, gaps between tiers create blind spots where **no rate matches**.

**Example**: Tier 1 is LTE 5kg, Tier 2 is GT 10kg. A cart weighing 7kg matches neither tier and gets no shipping rate for this option.

**Detection**: Sort rates by their condition boundaries and check for discontinuities. Any gap means some orders will silently lose this shipping option at checkout.
