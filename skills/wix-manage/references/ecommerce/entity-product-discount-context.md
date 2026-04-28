---
name: "Entity: Product Discount Context"
description: How products relate to discounts — margin data, pricing quantiles, inventory velocity, category membership, and catalog query limits.
layer: L2
---
# Entity: Product Discount Context

This entity reference describes the data relationships between products and discounts. It covers the analytics, inventory, and catalog data needed to make informed discounting decisions, along with system limits and goal-specific query parameterization.

---

## Margin data

Profit margin information comes from `getCatalogAnalytics` and is returned as an average per category.

- **`profitMargin`**: Average profit margin for products in the queried scope.
- **`minMarginPct` baseline**: 15%. Discounts should not push the effective margin below this threshold unless the merchant explicitly overrides it.
- **Global `discountMargin` cap**: 25%. This is the maximum discount percentage applied by default. The cap is overridable by explicit user input (e.g., the merchant says "I want 40% off").

When proposing a discount, verify that `discountPercentage <= profitMargin - minMarginPct` to avoid selling at or below cost.

---

## Pricing quantiles

`getCatalogAnalytics` returns price distribution quantiles when requested:

```
quantiles([0.5, 0.75, 0.9], price)
```

These quantiles serve multiple purposes:

- **Upsell tier calculation**: The 75th and 90th percentile define "premium" price tiers for upsell targeting.
- **Rate strategy**: Quantile spread determines whether a flat percentage or tiered discount is more appropriate. Wide spread favors tiered; narrow spread favors flat.
- **AOV sanity checking**: The median (50th percentile) price provides a baseline for average order value estimates. Proposed discounts that would drop AOV below the median warrant a warning.

---

## Inventory velocity

Inventory velocity is derived from the ratio of `ordersCount` to `quantity` returned by `getProductCatalogData`:

- **High `quantity` + low `ordersCount`** = slow-moving stock. These are candidates for clearance or STOCK_MOVER discount strategies.
- **Low `quantity` + high `ordersCount`** = fast-selling items. Discounting these further is usually unnecessary and erodes margin.
- **High `quantity` + high `ordersCount`** = high-volume staples. Discount only strategically (e.g., bundle incentives).

The velocity ratio (`ordersCount / quantity`) is the primary signal for the STOCK_MOVER strategy.

---

## Category membership

Products belong to categories (also called collections in the Wix UI).

- `getCategoryIds` converts human-readable category names into GUIDs required by the discount rules API.
- **"All Products" exclusion**: The "All Products" category is a system default that contains every product. It must always be excluded from category filters when targeting specific segments, otherwise the filter effectively targets the entire catalog.
- Category GUIDs are used as `scope.id` values in discount rules with `scope.type: "COLLECTION"`.

---

## Catalog query limit

```
system.CATALOG_LIMIT = 30
```

`getProductCatalogData` returns a maximum of 30 items per call. For catalogs larger than 30 products, the agent must work within this limit by:

- Filtering by category or other criteria to reduce the result set.
- Accepting that analysis is based on a representative sample rather than the full catalog.
- Prioritizing the most relevant products for the discount goal (e.g., highest quantity for STOCK_MOVER, highest price for UPSELL_BOOST).

---

## Tool parameterization by goal

Different discount strategies require different analytics aggregates and sort orders from `getCatalogAnalytics` and `getProductCatalogData`. The table below defines the correct parameterization for each goal.

### UPSELL_BOOST

Targets high-value products to incentivize customers to trade up.

- **Analytics aggregates**: `count`, `quantiles([0.5, 0.75, 0.9], price)`, `avg(profitMargin)`
- **Product sort order**: `price DESC`, `ordersCount DESC`
- **Rationale**: Surface the most expensive, best-selling products first. Quantile data defines the upsell tiers.

### STOCK_MOVER

Clears slow-moving inventory through aggressive discounting.

- **Analytics aggregates**: `sum(quantity)`, `sum(ordersCount)`, `avg(profitMargin)`
- **Product sort order**: `quantity DESC`, `ordersCount ASC`
- **Rationale**: Surface products with the most stock and fewest sales. Margin data sets the discount ceiling.

### SEASONAL

Promotes products during seasonal events or time-limited campaigns.

- **Analytics aggregates**: `sum(ordersCount)`, `quantiles([0.5, 0.9], price)`, `avg(profitMargin)`
- **Product sort order**: `ordersCount DESC`
- **Rationale**: Lead with proven sellers to maximize seasonal campaign revenue. Price quantiles inform tiered discount structures.

### BUNDLE_AND_SAVE

Encourages multi-product purchases through bundle pricing.

- **Analytics aggregates**: `min(price)`, `max(price)`, `avg(profitMargin)`, `count`
- **Product sort order**: `price DESC`, `ordersCount DESC`
- **Rationale**: Price range data (min/max) determines viable bundle combinations. Count establishes catalog breadth for bundle composition.

---

## References

- [Discount Rules API](https://dev.wix.com/docs/api-reference/business-solutions/e-commerce/extensions/discounts/discount-rules/introduction)
- [Products V3 API](https://dev.wix.com/docs/api-reference/business-solutions/stores/catalog-v3/products-v3/introduction)
