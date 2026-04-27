---
name: "Setup: Discount Rules"
description: Configures automatic discount rules using the eCommerce Discount Rules API. Covers percentage and fixed-amount discounts, scope targeting (catalog-wide, specific collections, or individual products), and scheduling active periods.
layer: L3
---
# Setup Discount Rules

## Prerequisites

- Wix Stores (or another eCommerce business solution) installed on the site
- At least one product in the catalog

## Required APIs

- [Create Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/create-discount-rule)
- [Get Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/get-discount-rule)
- [Query Discount Rules](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/query-discount-rules)
- [Update Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/update-discount-rule)
- [Delete Discount Rule](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/delete-discount-rule)

---

## Step 1: Query existing discount rules

Before creating new rules, check what already exists to avoid conflicts.

**Endpoint**: `POST https://www.wixapis.com/ecom/v1/discount-rules/query`

**Request**:
```json
{
  "query": {
    "paging": {
      "limit": 100
    }
  }
}
```

**Response**:
```json
{
  "discountRules": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "revision": "1",
      "name": "Summer Sale 10%",
      "active": true,
      "activeTimeInfo": {
        "start": "2026-06-01T00:00:00.000Z",
        "end": "2026-08-31T23:59:59.000Z"
      },
      "discounts": [
        {
          "discount": {
            "discountType": "PERCENTAGE",
            "percentage": 10
          },
          "scope": {
            "id": "catalog",
            "type": "CATALOG"
          }
        }
      ]
    }
  ],
  "pagingMetadata": {
    "count": 1,
    "hasNext": false
  }
}
```

Note existing rules and their scopes to avoid stacking conflicts.

---

## Step 2: Create a percentage discount rule

**Endpoint**: `POST https://www.wixapis.com/ecom/v1/discount-rules`

**Request** — 20% off all products:
```json
{
  "discountRule": {
    "name": "Flash Sale 20% Off",
    "active": true,
    "activeTimeInfo": {
      "start": "2026-05-01T00:00:00.000Z",
      "end": "2026-05-03T23:59:59.000Z"
    },
    "discounts": [
      {
        "discount": {
          "discountType": "PERCENTAGE",
          "percentage": 20
        },
        "scope": {
          "id": "catalog",
          "type": "CATALOG"
        }
      }
    ]
  }
}
```

**Response**:
```json
{
  "discountRule": {
    "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "revision": "1",
    "name": "Flash Sale 20% Off",
    "active": true,
    "activeTimeInfo": {
      "start": "2026-05-01T00:00:00.000Z",
      "end": "2026-05-03T23:59:59.000Z"
    },
    "discounts": [
      {
        "discount": {
          "discountType": "PERCENTAGE",
          "percentage": 20
        },
        "scope": {
          "id": "catalog",
          "type": "CATALOG"
        }
      }
    ]
  }
}
```

**Request** — 15% off a specific collection:
```json
{
  "discountRule": {
    "name": "Summer Collection Sale",
    "active": true,
    "discounts": [
      {
        "discount": {
          "discountType": "PERCENTAGE",
          "percentage": 15
        },
        "scope": {
          "id": "collection-uuid-here",
          "type": "COLLECTION"
        }
      }
    ]
  }
}
```

---

## Step 3: Create a fixed-amount discount rule

**Request** — $5 off specific products:
```json
{
  "discountRule": {
    "name": "$5 Off Selected Items",
    "active": true,
    "discounts": [
      {
        "discount": {
          "discountType": "FIXED_AMOUNT",
          "fixedAmount": "5.00"
        },
        "scope": {
          "id": "product-uuid-here",
          "type": "SPECIFIC_PRODUCTS"
        }
      }
    ]
  }
}
```

---

## Step 4: Update a discount rule

**Endpoint**: `PATCH https://www.wixapis.com/ecom/v1/discount-rules/{discountRuleId}`

**Request**:
```json
{
  "discountRule": {
    "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "revision": "1",
    "name": "Extended Flash Sale 25% Off",
    "discounts": [
      {
        "discount": {
          "discountType": "PERCENTAGE",
          "percentage": 25
        },
        "scope": {
          "id": "catalog",
          "type": "CATALOG"
        }
      }
    ]
  }
}
```

The `revision` field is required and must match the current revision.

---

## Step 5: Deactivate or delete a discount rule

To deactivate without deleting:
```json
{
  "discountRule": {
    "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "revision": "2",
    "active": false
  }
}
```

To delete permanently:

**Endpoint**: `DELETE https://www.wixapis.com/ecom/v1/discount-rules/{discountRuleId}`

---

## Key field rules

| Field | Required | Notes |
|---|---|---|
| `name` | Yes | Internal name for the rule |
| `active` | Yes | Whether the rule is currently applied |
| `activeTimeInfo.start` | No | ISO 8601 start time. Omit for immediate activation |
| `activeTimeInfo.end` | No | ISO 8601 end time. Omit for no expiration |
| `discounts[].discount.discountType` | Yes | `PERCENTAGE` or `FIXED_AMOUNT` |
| `discounts[].discount.percentage` | If PERCENTAGE | Integer 1-100 |
| `discounts[].discount.fixedAmount` | If FIXED_AMOUNT | Decimal string (e.g., `"5.00"`) |
| `discounts[].scope.type` | Yes | `CATALOG`, `COLLECTION`, or `SPECIFIC_PRODUCTS` |
| `discounts[].scope.id` | Yes | `"catalog"` for CATALOG type, or the collection/product UUID |

## Scope types

| Scope Type | `scope.id` value | Description |
|---|---|---|
| `CATALOG` | `"catalog"` | Applies to all products in the store |
| `COLLECTION` | Collection UUID | Applies to all products in a specific collection |
| `SPECIFIC_PRODUCTS` | Product UUID | Applies to a single product (use multiple discount entries for multiple products) |

## Error Handling

| Error | Cause | Fix |
|---|---|---|
| `DISCOUNT_RULE_NOT_FOUND` | The discount rule ID doesn't exist | Re-query discount rules to get current IDs |
| `REVISION_MISMATCH` | The `revision` doesn't match the current version | Re-fetch the rule to get the latest revision, then retry |
| `INVALID_DISCOUNT_TYPE` | Unsupported discount type | Use `PERCENTAGE` or `FIXED_AMOUNT` |

## References

- [Discount Rules API](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/introduction)
