---
name: "Setup: Discount Constraints"
description: Configures the constraint system for discount rules — margin requirements, global discount caps, scope mutual exclusivity rules, ID format validation, and catalog query limits.
layer: L3
---
# Discount Constraints

## Minimum Margin

The default `minMarginPct` is **15%**. Never recommend discounts that violate this threshold UNLESS user input explicitly overrides it.

## Maximum Discount

The global `discountMargin` cap defaults to **25%**. Respect this limit unless user input requests a higher value.

## User Input Override Protocol

`user_input` has **ABSOLUTE PRIORITY** and overrides ALL constraints -- margin, discount caps, naming, percentages, product selection.

Only hard limits that cannot be overridden:
- `maxItems: 5` for `productIds`
- `maxItems: 3` for `categoryIds`

## Scope Mutual Exclusivity

Scopes are mutually exclusive. NEVER populate both `categoryIds` and `productIds` simultaneously.

| Scope | `categoryIds` | `productIds` |
|---|---|---|
| `SITE` | `[]` (empty) | `[]` (empty) |
| `CATEGORY` | `[...GUIDs...]` | `[]` (MUST BE EMPTY) |
| `ITEMS` | `[]` (MUST BE EMPTY) | `[...GUIDs...]` |

Mixed scope ban: NEVER populate both arrays. If both are provided, the request is invalid.

## ID Format

- All IDs must be valid GUIDs.
- Copy IDs EXACTLY from tool output. NEVER alter, append, or truncate IDs.
- Category names are NOT valid IDs -- always use the GUID.

## Numerical Rounding

- Round discount percentages to **5/10/15/20/25%** increments (unless the user specifies an exact value).
- Round monetary values to clean numbers.
