---
name: "Recommend: Discount Strategy"
description: Proactive discount recommendation skill — gathers site data, classifies merchant intent into 4 business goals, analyzes catalog, and generates up to 3 actionable discount recommendations across different strategies.
layer: R
references:
  - name: "Goal: Increase Average Order Value"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/goal-increase-aov
    load: true
  - name: "Goal: Clear Slow-Moving Inventory"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/goal-clear-inventory
    load: false
  - name: "Goal: Capitalize on Seasonal Events"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/goal-seasonal-revenue
    load: false
  - name: "Goal: Drive Cross-Sells"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/goal-drive-cross-sells
    load: false
  - name: "Guardrail: Discount Conflicts"
    url: https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/skills/guardrail-discount-conflicts
    load: false
---
# Recommend: Discount Strategy

Use this skill when the merchant asks for discount recommendations, or proactively when analyzing a store's discount opportunities. Follow the steps below in order.

---

## Step 1: Validate the request

Before doing anything, check if the merchant's request is within scope. **Reject** these and explain why:

| Unsupported request | Why | Suggest instead |
|---|---|---|
| Free shipping | Not a discount — it's a shipping configuration | Use the "Flow: Add Free Shipping" skill |
| Buy one get one (BOGO) | Not supported by the Discount Rules API | Explain limitation |
| Fixed-price bundles ("3 for $100") | Requires custom pricing logic, not discount rules | Explain limitation |
| Unrelated to discounts | Out of scope | Decline politely |

If the request is valid, continue to Step 2.

---

## Step 2: Gather site data

Call `getSiteData` with these parameters:
- **fields**: `country`, `businessType`, `industry`, `visitors`, `revenue`, `ordersCount`, `currency`, `language`
- **include**: `currentDiscounts`, `AOV`, `discountMargin`

You need these values for the rest of the skill:

| Value | Where it comes from | What it's used for |
|---|---|---|
| AOV | revenue / ordersCount | Setting minSubTotal thresholds |
| discountMargin | Site setting, default 25% | Maximum allowed discount percentage |
| currentDiscounts | Active discount rules | Conflict detection |
| country | Site settings | Holiday detection, localization |
| currency | Site settings | Price formatting in recommendation names |
| language | Site settings | Translating recommendation names and descriptions |

**Validation**: If `country`, `industry`, or `revenue` are missing or null, stop and report the issue. You cannot generate reliable recommendations without these.

---

## Step 3: Classify the merchant's intent

Determine which business goal best matches what the merchant wants:

| Goal | Trigger phrases | What it optimizes |
|---|---|---|
| **UPSELL_BOOST** | "increase AOV", "spend more", "upsell", "order value", "bigger orders" | Average order value |
| **BUNDLE_AND_SAVE** | "bundle", "cross-sell", "buy together", "multi-buy", "product discovery" | Items per order |
| **STOCK_MOVER** | "clear inventory", "overstock", "dead stock", "clearance", "old inventory" | Inventory turnover |
| **SEASONAL** | Holiday names, date references, "seasonal", "sale event", "Black Friday" | Event-driven revenue |

**Rules**:
- If the request clearly matches one goal, use that goal as the primary strategy
- If the request is ambiguous or generic (e.g., "give me discount ideas"), default to `UPSELL_BOOST`
- If it's a follow-up request ("try again", "give me another"), reuse the previous goal but generate a different recommendation

Also extract from the merchant's input:
- **Keywords**: Product names, brand names, specific terms (e.g., "t-shirts", "electronics")
- **Category suggestions**: Only if the merchant explicitly says "category" (e.g., "discount on the Shoes category")
- **Date range**: Start/end dates for time-limited campaigns. Map holidays to actual dates. If no dates specified but a major holiday is within 30 days, detect it proactively.

---

## Step 4: Determine discount mechanism — Automatic Discount or Coupon

Before analyzing the catalog, decide whether to create an **automatic discount** or a **coupon**. These are two different Wix features with different APIs and behavior.

| Mechanism | How it works | Best for |
|---|---|---|
| **Automatic Discount** | Applies at checkout without customer action | Sales, seasonal promotions, upsell thresholds, site-wide discounts |
| **Coupon** | Requires customer to enter a code | Email campaigns, influencer partnerships, loyalty rewards, targeted offers |

### Decision logic

| Merchant says | Mechanism | Why |
|---|---|---|
| "sale", "promotion", "discount for everyone" | Automatic Discount | Applies to all customers |
| "coupon", "code", "promo code", "voucher" | Coupon | Explicitly requested code-based |
| "discount for subscribers", "influencer code", "loyalty reward" | Coupon | Needs attribution or audience targeting |
| "20% off electronics" (no code/coupon mention) | **Ask the merchant** | Intent is ambiguous |

**If intent is unclear, ask**: "Would you like this to apply automatically to everyone at checkout, or as a coupon code that customers enter? Automatic discounts are great for site-wide sales; coupons work better for targeted campaigns where you want to track which channel drove the purchase."

### Impact on recommendations

- If **Automatic Discount**: Use the Discount Rules API. Set `advice.action` to `apply_discount`. Include in `advice.params`: `mechanism: "AUTOMATIC"`.
- If **Coupon**: Use the Coupons API. Set `advice.action` to `apply_coupon`. Include in `advice.params`: `mechanism: "COUPON"`, plus `code` (suggested coupon code), `usageLimit` (total uses), and `limitPerCustomer`.

### Stacking warning

If the store already has active automatic discounts AND you're creating a coupon (or vice versa), warn the merchant: "You have active automatic discounts. A coupon will stack on top of them — customers using the code will get both discounts applied."

---

## Step 5: Analyze the catalog

Run these two calls **concurrently** (in parallel):

**Call 1 — getCatalogAnalytics**:

Use aggregates based on the primary goal:

| Goal | Aggregates to request |
|---|---|
| UPSELL_BOOST | `count`, `quantiles([0.5,0.75,0.9], price)`, `avg(profitMargin)` |
| BUNDLE_AND_SAVE | `min(price)`, `max(price)`, `avg(profitMargin)`, `count` |
| STOCK_MOVER | `sum(quantity)`, `sum(ordersCount)`, `avg(profitMargin)` |
| SEASONAL | `sum(ordersCount)`, `quantiles([0.5,0.9], price)`, `avg(profitMargin)` |

**Call 2 — getProductCatalogData**:

| Goal | Sort order | Max items |
|---|---|---|
| UPSELL_BOOST | price DESC, ordersCount DESC | 30 |
| BUNDLE_AND_SAVE | price DESC, ordersCount DESC | 30 |
| STOCK_MOVER | quantity DESC, ordersCount ASC | 30 |
| SEASONAL | ordersCount DESC | 30 |

Pass `keywords` to the `query` parameter and `categorySuggestions` to `categoryNames` if the merchant specified them. Always exclude "All Products" from category filters.

**If both calls fail**: Skip to Step 5 using the low-data fallback path.
**If only getProductCatalogData fails**: Proceed with analytics data. Use category names from analytics to call `getCategoryIds`.

---

## Step 6: Generate up to 3 recommendations

Generate **up to 3 recommendations**. Each one MUST use a **different strategy** — do not repeat the same approach.

### How to pick the 3 strategies

Look at the data and find the best opportunities:

1. **If AOV data is available** → include an UPSELL_BOOST recommendation (minSubTotal above AOV)
2. **If inventory shows slow movers** (high quantity, low ordersCount) → include a STOCK_MOVER recommendation
3. **If a holiday is within 30 days** OR merchant mentioned seasonal context → include a SEASONAL recommendation
4. **If catalog has many low-priced items** → include a BUNDLE_AND_SAVE recommendation
5. **If none of the above stand out** → include a conservative SITE-scope recommendation

Return fewer than 3 if the data doesn't support more. Do not pad with weak recommendations.

### Scope selection for each recommendation

1. **CATEGORY** (preferred): When analytics clearly identify a high-performing or high-opportunity category. **You must call `getCategoryIds`** to convert the category name to a GUID before using it.
2. **ITEMS** (specific): When particular products stand out (slow movers, high-margin outliers). Maximum 5 product IDs.
3. **SITE** (broad): When the store needs overall traffic conversion or for store-wide seasonal events.

### Low-data fallback

When data is sparse (few orders, limited analytics):
1. First try: CATEGORY scope with 10-15% discount on the highest-margin category
2. If no category data: SITE scope with 5-10% discount
3. Use catalog price quantiles as AOV proxy if revenue data is unavailable

### Performance-based signals

| What you observe | What to include |
|---|---|
| High visitors, low ordersCount | A site-wide recommendation to test broad conversion |
| Low visitors, low revenue | Conservative site-wide (5-10%) to avoid margin erosion |
| High AOV, few items per order | Prioritize a BUNDLE_AND_SAVE recommendation |
| Many products with high stock + low orders | Prioritize a STOCK_MOVER recommendation |
| Holiday within 30 days | Include a SEASONAL recommendation |

---

## Step 7: Validate before returning

Before finalizing, run these checks on each recommendation:

1. **Conflict check**: Query both active discount rules AND active coupons. If any existing promotion targets the same scope, warn about stacking risk — especially cross-mechanism stacking (automatic + coupon).
2. **Margin check**: No recommendation should exceed the discountMargin cap (default 25%) unless the merchant explicitly asked for a higher value.
3. **Strategy uniqueness**: Each of the 3 recommendations must use a different strategy type.
4. **Mechanism consistency**: Verify the mechanism (automatic/coupon) matches the merchant's intent from Step 4.
5. **ID validity**: All category IDs must be GUIDs from `getCategoryIds`, not category names. All product IDs must come from `getProductCatalogData`.
6. **Discount values**: Round to clean increments (5%, 10%, 15%, 20%, 25%) unless the merchant specified an exact value.

---

## Output format

Return a JSON object with a `recommendations` array. Each recommendation follows this structure:

```json
{
  "recommendations": [
    {
      "title": "15% Off Electronics — Orders Over $200",
      "reasoning": "AOV is $165. Setting $200 threshold incentivizes adding one more item. Electronics has 42% margin — ideal for this discount.",
      "domain": "discounts",
      "urgency": "HIGH | MEDIUM | LOW",
      "advice": {
        "action": "apply_discount",
        "params": {
          "mechanism": "AUTOMATIC | COUPON",
          "scope": "SITE | CATEGORY | ITEMS",
          "categoryIds": [],
          "productIds": [],
          "name": "Spend More, Save More",
          "why": "Rewards orders above your $165 average with 15% off, driving higher cart values.",
          "discountType": "PERCENTAGE",
          "discount": 15,
          "code": "",
          "usageLimit": 0,
          "limitPerCustomer": 0,
          "conditions": {
            "minItemQuantity": 0,
            "minSubTotal": 200,
            "startDate": "",
            "endDate": ""
          }
        },
        "success_criteria": "15% discount applied to Electronics for orders above $200"
      }
    }
  ]
}
```

### Field rules

| Field | Rule |
|---|---|
| `title` | Short, actionable. Max 200 chars. Combine scope + discount + goal. Always English. |
| `reasoning` | Data-backed explanation. Include specific numbers. Always English. |
| `domain` | Always `"discounts"` |
| `urgency` | `HIGH` (merchant explicitly asked, or high-revenue store), `MEDIUM` (good opportunity), `LOW` (optimization) |
| `name` | Marketing headline, 2-5 words. **Translate to the site's `language`** if not English. |
| `why` | 1-2 sentences explaining the business opportunity. Include specific data points (AOV, margin %, stock levels). **Translate to the site's `language`** if not English. |
| `mechanism` | `AUTOMATIC` (Discount Rules API) or `COUPON` (Coupons API). Determined in Step 4. |
| `discountType` | `PERCENTAGE`, `FIXED_AMOUNT`, or `FIXED_PRICE` |
| `discount` | Integer for percentage (1-100), decimal string for fixed amounts |
| `code` | Coupon code string. Only for `mechanism: "COUPON"`. Empty string for automatic. Suggest a memorable, brand-relevant code (e.g., "SUMMER25", "SAVE15"). |
| `usageLimit` | Total number of times the coupon can be used. Only for coupons. `0` = unlimited. |
| `limitPerCustomer` | Max uses per customer. Only for coupons. `0` = unlimited. |
| `conditions` | Set to `0` or `""` for fields that don't apply to this recommendation |
| `scope` + IDs | Mutually exclusive: SITE = both empty, CATEGORY = categoryIds only (max 3), ITEMS = productIds only (max 5) |

---

## Constraints

- Maximum 3 recommendations per invocation
- Each recommendation must use a different strategy
- Do not recommend discounts on scopes that already have active discounts (unless merchant explicitly wants to stack)
- Respect discountMargin cap (default 25%) unless merchant overrides
- All category IDs must be GUIDs — never output category names as IDs
- Catalog queries are limited to 30 items
