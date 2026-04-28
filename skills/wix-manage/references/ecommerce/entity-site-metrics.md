---
name: "Entity: Site Metrics"
description: Site-level business metrics — AOV sanity checking, effective AOV derivation, delivery step conversion, and revenue impact calculation.
layer: L2
---
# Entity: Site Metrics

## Site Data Fields

Retrieved via the `getSiteData` tool with `fields` and `include` params:

| Field | Description |
|---|---|
| `country` | Site's primary country |
| `businessType` | Type of business |
| `industry` | Industry vertical |
| `visitors` | Total visitor count (period-dependent) |
| `revenue` | Total revenue |
| `ordersCount` | Number of completed orders |
| `currency` | Store currency code |
| `language` | Site language |

**AOV** (Average Order Value) = `revenue / ordersCount`.

---

## AOV Sanity Check

**This check is MANDATORY before any threshold calculation that references AOV.** Raw AOV can be misleading due to data issues, unit-count orders, or bulk purchases.

### Procedure

1. Extract `price_p25` and `price_p50` from `catalog_stats` (use the "All Products" category group).

2. Evaluate AOV against catalog price distribution:

| Condition | Interpretation | Action |
|---|---|---|
| AOV < price_p25 | **Anomalous** — likely a data or unit issue. AOV is below the 25th percentile of product prices, meaning the average order is cheaper than 75% of products. | Override: use `price_p50` as base. Note the override in reasoning. |
| AOV > price_p90 | **Possible bulk/combo orders** — AOV exceeds 90th percentile of product prices. | Still use AOV but note the discrepancy. May reflect legitimate high-value orders or bundled purchases. |
| price_p25 <= AOV <= price_p90 | **Reasonable** — AOV aligns with catalog pricing. | Use AOV as-is. |

3. Store the result as `effective_aov`. Use `effective_aov` everywhere AOV would otherwise be referenced (backup rate calculations, shipping cost thresholds, free shipping threshold recommendations).

---

## Delivery Step Conversion

Measures how many customers who begin the delivery/shipping step in checkout actually complete it.

| Metric | Definition |
|---|---|
| `delivery_started` | Number of checkout sessions that reached the shipping step |
| `delivery_finished` | Number of checkout sessions that completed the shipping step |
| `delivery_step_cvr` | `(delivery_finished / delivery_started) * 100` (percentage) |

**Benchmark**: 65% delivery step conversion rate.

### Interpretation

| CVR Range | Interpretation |
|---|---|
| >= 90% | Excellent. Focus on best practices and maintenance, not conversion optimization. |
| 65-89% | Acceptable. Minor improvements may help. |
| < 65% | Below benchmark. Shipping configuration issues are likely contributing to drop-off. Investigate actively. |

---

## Revenue Impact Formula

Estimates the revenue lost due to shipping-related checkout abandonment.

```
revenue_impact = ((benchmark_delivery_cvr - delivery_step_cvr) / 100) * total_checkouts * aov
```

### When to Calculate

- **Only** when `delivery_step_cvr < benchmark` AND shipping configuration issues have been detected.
- If `delivery_step_cvr >= 90%` → note excellent performance. Do not calculate a revenue impact; focus recommendations on best practices rather than conversion recovery.
- If no shipping issues are found → do not attribute checkout drop-off to shipping, even if CVR is below benchmark.

### Variables

| Variable | Source |
|---|---|
| `benchmark_delivery_cvr` | 65 (constant) |
| `delivery_step_cvr` | Calculated from delivery_started and delivery_finished |
| `total_checkouts` | Total checkout sessions (delivery_started) |
| `aov` | Use `effective_aov` (after sanity check) |

---

## Traffic Tiers

Site traffic volume determines the amplification factor of any shipping issue.

| Tier | Condition | Implication |
|---|---|---|
| **High traffic** | `visitors > 30000` (approximately 1000+/day) | Issues are amplified — even a small percentage drop in conversion affects many orders. Prioritize fixes. |
| **Standard** | `visitors <= 30000` | Normal priority. Issues still matter but affect fewer potential orders. |

When reporting revenue impact, always note the traffic tier. A 5% conversion drop on a high-traffic site has a materially different business impact than the same drop on a standard site.
