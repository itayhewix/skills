---
name: "Entity: Delivery Profile"
description: Delivery profile data model — profile-region-carrier hierarchy, active/inactive states, external vs Wix-managed carriers, backup rates, and destination rules.
layer: L2
---
# Entity: Delivery Profile

## Hierarchy

DeliveryProfile → DeliveryRegion[] (up to 100) → DeliveryCarrier[] (up to 25).

The first delivery profile is auto-created when Wix Stores is installed and **cannot be deleted**. Additional profiles may exist but the default profile always remains.

Each profile contains an ordered list of regions. Each region contains an ordered list of carriers. This three-level hierarchy is the foundation of all shipping configuration.

---

## Region States

Each `DeliveryRegion` has an `active` boolean:

- **active=true** — Region is available at checkout. Customers shipping to matching destinations see the associated shipping options.
- **active=false** — Region is configured but disabled. Customers shipping to those destinations see **no shipping options** and cannot complete checkout.

A fully configured but inactive region is often a forgotten configuration — the merchant set it up, disabled it temporarily, and never re-enabled it. This is the single most common cause of "customers can't check out" complaints for specific countries.

---

## Destinations

Each region specifies a `destinations[]` array:

- **countryCode** — ISO 3166-1 alpha-2 (e.g., `US`, `GB`, `DE`).
- **subdivisions[]** — ISO 3166-2 codes (e.g., `US-CA`, `US-NY`). An empty subdivisions array means the entire country is covered.
- **Empty destinations[]** — If the destinations array is empty, the region represents a **"Rest of World"** catch-all. This region matches any destination not covered by other regions in the same profile.

---

## Carrier Management

Each `DeliveryCarrier` is identified by an `appId` that determines which application provides shipping rates.

### Known Carrier App IDs

| Carrier Type | appId |
|---|---|
| Wix-native (built-in rates) | `45c44b27-ca7b-4891-8c0d-1747d588b835` |
| Shippo (external) | `2b1943e2-3fc2-47bc-be56-3d402e5966d7` |

Any appId not matching the Wix-native ID is considered an external carrier.

---

## External vs Wix-Managed Classification

Classify each region by examining the appIds of ALL its carriers:

| Scenario | Classification | Action |
|---|---|---|
| ALL carriers have an external appId | Externally managed | **DO NOT** generate recommendations for this region. External carriers manage their own rates. |
| SOME carriers are external, some Wix-native | Hybrid | Only analyze the Wix-native carriers. Leave external carriers untouched. |
| NO carriers are external (all Wix-native) | Fully Wix-managed | Analyze all carriers and generate recommendations. |

---

## Backup Rates

Each carrier has a `backupRate` object with an `active` boolean and an `amount`:

- **backupRate.active=false** and carrier fails to return a rate → the shipping option **silently disappears** at checkout. The customer sees fewer options (or none) with no explanation.
- **backupRate.active=true** but amount is very high → **sticker shock**. The customer sees an unexpectedly expensive shipping option and abandons.

**Recommended backup rate**: 5-10% of effective_aov. This provides a safety net without surprising customers.

---

## Additional Charges

Carriers may have hidden surcharges (`additionalCharges[]`) that are added to the combined shipping price. These are not always visible in the dashboard summary.

**Threshold**: If total additional charges push the combined shipping cost above 10% of AOV, flag as excessive. Additional charges are the most commonly overlooked contributor to high shipping costs.
