---
name: "Flow: Run Flash Sale"
description: Orchestrates a time-limited flash sale by creating discount rules, validating against conflicts, and applying visual indicators to selected products. Multi-step flow that chains discount setup, guardrail validation, and product updates.
layer: L4
references:
  - name: "Guardrail: Discount Conflicts"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/guardrail-discount-conflicts
    load: true
  - name: "Setup: Discount Rules"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/setup-discount-rules
    load: true
  - name: "Query Products"
    url: https://dev.wix.com/docs/api-reference/business-solutions/stores/skills/query-products
    load: false
  - name: "Discount Rules API"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/introduction
    load: false
---
# Flow: Run Flash Sale

A flash sale is a time-limited promotion with deep discounts to drive urgency and volume. This flow creates the discount rule, validates it against existing promotions, and optionally adds visual indicators (ribbons) to sale products.

## Prerequisites

- Wix Stores installed on the site
- Products exist in the catalog
- Discount rules configuration understood (see referenced Setup: Discount Rules skill)

## Required APIs

- [Create Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/create-discount-rule)
- [Query Discount Rules](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/query-discount-rules)
- [Query Products V3](https://dev.wix.com/docs/api-reference/business-solutions/stores/catalog-v3/products-v3/query-products)
- [Create Ribbon](https://dev.wix.com/docs/api-reference/business-solutions/stores/catalog-v3/ribbons/create-ribbon)
- [Update Product](https://dev.wix.com/docs/api-reference/business-solutions/stores/catalog-v3/products-v3/update-product)

---

## Step 1: Gather sale parameters from the merchant

Ask the merchant for:
- **Scope**: Which products? Options: all products, a specific collection, or specific product IDs
- **Discount**: What percentage or fixed amount off?
- **Duration**: Start and end date/time for the flash sale
- **Visual indicator**: Add a "Sale" ribbon to products? (recommended)

Example merchant input: "Run a 24-hour flash sale with 25% off my Summer Collection starting tomorrow at 9 AM"

---

## Step 2: Validate against discount conflicts

**IMPORTANT: Run the Guardrail: Discount Conflicts checks before proceeding.**

1. Query existing active discount rules
2. Check for scope overlap with the planned sale
3. Check if the discount percentage is reasonable
4. Check for time overlap with other promotions
5. Check for coupon stacking risks

If conflicts are found, present them to the merchant and get confirmation before proceeding.

---

## Step 3: Create the discount rule

Use the Setup: Discount Rules instructions to create the rule.

**Endpoint**: `POST https://www.wixapis.com/ecom/v1/discount-rules`

**Request** — 25% off a collection for 24 hours:
```json
{
  "discountRule": {
    "name": "Flash Sale - Summer Collection 25% Off",
    "active": true,
    "activeTimeInfo": {
      "start": "2026-05-01T09:00:00.000Z",
      "end": "2026-05-02T09:00:00.000Z"
    },
    "discounts": [
      {
        "discount": {
          "discountType": "PERCENTAGE",
          "percentage": 25
        },
        "scope": {
          "id": "summer-collection-uuid",
          "type": "COLLECTION"
        }
      }
    ]
  }
}
```

Save the returned `id` and `revision` for later management.

---

## Step 4: Add visual indicators (optional but recommended)

Adding a "Sale" ribbon helps shoppers identify discounted products in the catalog.

### Step 4a: Create or find a "Sale" ribbon

**Endpoint**: `POST https://www.wixapis.com/stores/v3/ribbons`

**Request**:
```json
{
  "ribbon": {
    "name": "Flash Sale"
  }
}
```

**Response**:
```json
{
  "ribbon": {
    "id": "r1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "Flash Sale"
  }
}
```

### Step 4b: Apply ribbon to sale products

Query the products in the sale scope, then update each with the ribbon.

**Query products in collection**:

**Endpoint**: `POST https://www.wixapis.com/stores/v3/products/query`

**Request**:
```json
{
  "query": {
    "filter": {
      "categoryIds": { "$hasSome": ["summer-collection-uuid"] }
    },
    "paging": {
      "limit": 100
    },
    "fields": ["ID", "NAME", "RIBBONS", "REVISION"]
  }
}
```

**Update each product with the ribbon**:

**Endpoint**: `PATCH https://www.wixapis.com/stores/v3/products/{productId}`

**Request**:
```json
{
  "product": {
    "ribbons": [
      {
        "id": "r1b2c3d4-e5f6-7890-abcd-ef1234567890"
      }
    ]
  },
  "fieldMask": {
    "paths": ["ribbons"]
  }
}
```

> **Note**: Process products in batches of up to 100 per query page. For large catalogs, paginate through all products.

---

## Step 5: Verify the sale is live

1. Query discount rules to confirm the new rule exists and is `active: true`
2. Query a product in the sale scope to confirm the ribbon is applied
3. Report to the merchant:
   > "Flash sale is live: 25% off Summer Collection for 24 hours (May 1 9AM - May 2 9AM). {N} products have the 'Flash Sale' ribbon. The discount will apply automatically at checkout."

---

## Step 6: End the sale

When the sale period expires or the merchant requests early termination:

1. **Deactivate the discount rule**:

**Endpoint**: `PATCH https://www.wixapis.com/ecom/v1/discount-rules/{discountRuleId}`

**Request**:
```json
{
  "discountRule": {
    "id": "discount-rule-uuid",
    "revision": "1",
    "active": false
  }
}
```

2. **Remove ribbons from products**: Update each product to remove the "Flash Sale" ribbon (set `ribbons` to empty array or previous ribbons)

3. **Report**: "Flash sale ended. Discount deactivated and ribbons removed from {N} products."

---

## Branching logic

| Merchant intent | Scope | Notes |
|---|---|---|
| "Put my summer collection on sale" | `COLLECTION` with collection UUID | Use existing collection |
| "20% off everything" | `CATALOG` with `id: "catalog"` | Applies store-wide |
| "Discount these 3 products" | `SPECIFIC_PRODUCTS` with product UUIDs | One discount entry per product |
| "Buy 2 get 1 free" | Not covered by discount rules | Requires different approach (coupon or custom logic) |

## Error Handling

| Error | Cause | Fix |
|---|---|---|
| `DISCOUNT_RULE_NOT_FOUND` | Rule ID doesn't exist | Re-query discount rules for current IDs |
| `REVISION_MISMATCH` | Revision doesn't match | Re-fetch rule for latest revision, then retry |
| `PRODUCT_NOT_FOUND` | Product ID doesn't exist | Re-query products to get current IDs |
| `RIBBON_NOT_FOUND` | Ribbon ID doesn't exist | Re-query or create the ribbon |

## References

- [Discount Rules API](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/introduction)
- [Products V3 API](https://dev.wix.com/docs/api-reference/business-solutions/stores/catalog-v3/products-v3/introduction)
- [Ribbons API](https://dev.wix.com/docs/api-reference/business-solutions/stores/catalog-v3/ribbons/introduction)
