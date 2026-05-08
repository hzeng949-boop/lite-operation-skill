---
name: lite-combos-creation
description: Create or update combo products in Lite / Palmnet admin through store-level APIs instead of the UI. Use when the user wants to add, edit, or inspect combo payloads for `https://lite-dev.palmnet.co/admin/combos`, `https://lite-es1.palmnet.co/admin/combos`, or the same Lite combo admin page in another environment. This skill is specifically for combo products with variant groups, selectable items, min/max rules, and item-level upcharges.
---

# Lite Combos Creation

Use this skill for store-level combo work in Lite.

Shared operating rules live in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any submission
- verify token, store scope, and current admin context
- do not reuse IDs from another environment or another store

This skill only adds combo-specific rules.

Prefer API writes over UI entry.

## Hard Safety Gate

Before any write operation, explicitly confirm all three items with the user:
- target environment
- target store
- exact change to apply

Do not send `POST`, `PATCH`, `PUT`, or `DELETE` until the user has confirmed those three items in the current task.

If any one of them is missing or ambiguous, stop and ask.

## Required Input

Collect or infer these values before writing:
- environment base URL
- target store ID or the currently selected store
- action type
  - `create`
  - `update`
  - `delete`
  - `inspect-only`
- combo name
- combo category
- tax rate
- one or more variants
- for each variant:
  - visible variant name, or blank if intentionally blank
  - price
  - one or more combo groups
- for each combo group:
  - group name
  - `minSelect`
  - `maxSelect`
  - item list
- for each item:
  - product ID
  - optional variant ID
  - `extraCents`

Reject or pause on:
- missing store scope
- missing tax rate
- missing item resolution
- decimal price text that has not yet been converted to cents
- duplicated or conflicting group rules
- any write request that the user has not confirmed

## Shared Interface Points

Use the shared auth, product, category, tax, and optional asset endpoints defined in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Combo products do not use a separate `/combos` API path. They are created through the product endpoint with `kind: "combo"`.

## Required Combo Parameters

Top-level payload fields:
- `kind`
- `name`
- `taxRateId`
- `categoryLinks`
- `variants`

Top-level optional fields often safe to omit unless needed:
- `sku`
- `brief`
- `description`

Required rules:
- `kind` must be `"combo"`
- combo category must resolve to a valid category ID, usually `Combos`
- `name._base` is the visible combo name
- `taxRateId` must match the current environment and store setup
- prices must be integer cents

Variant rules:
- at least one variant is required
- each variant needs `priceCents`
- each variant may keep `name._base` blank if that is the intended admin behavior
- each variant can contain `comboGroups`

Combo group rules:
- each group needs `name._base`
- each group needs numeric `minSelect`
- each group needs numeric `maxSelect`
- `items` must not be empty
- `minSelect` must be less than or equal to `maxSelect`

Item rules:
- `productId` required
- `variantId` optional
- `extraCents` required as integer cents

## Verified Create Payload Shape

```json
{
  "categoryLinks": [
    { "categoryId": "COMBOS_CATEGORY_ID" }
  ],
  "sku": "",
  "kind": "combo",
  "name": {
    "_base": "Test Combo 9.99 API"
  },
  "taxRateId": "TAX_RATE_ID",
  "variants": [
    {
      "name": {
        "_base": ""
      },
      "priceCents": 999,
      "comboGroups": [
        {
          "name": { "_base": "Primero" },
          "minSelect": 1,
          "maxSelect": 1,
          "items": [
            { "productId": "PRODUCT_ID_1", "extraCents": 0 },
            { "productId": "PRODUCT_ID_2", "extraCents": 0 }
          ]
        }
      ]
    }
  ]
}
```

## Normalization Rules

Before sending:
- convert visible prices like `9.99` to `999`
- convert item upcharges like `1 EUR` to `100`
- resolve category names to category IDs
- resolve tax labels like `VAT (19%)` to a valid `taxRateId`
- resolve product names to product IDs

Do not guess across environments:
- category IDs are environment-specific
- tax rate IDs are environment-specific
- product IDs are environment-specific

Never reuse IDs copied from another environment without re-reading the target environment first.

Use the shared tax resolution rules from [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

## Recommended Workflow

1. Confirm environment, store, and exact change with the user.
2. Read current admin token and selected store context.
3. Read categories, tax rates, and products from the target store.
4. Resolve combo category ID, tax rate ID, and every combo item product ID.
5. Build the final payload.
6. Recheck the payload against the user request.
7. Send the create or update request.
8. Read back the created or updated combo and verify the final structure.

## Update Guidance

For updates:
- read the current combo first
- compare old and new structure before sending `PATCH`
- keep untouched fields unchanged
- tell the user exactly what will change before sending the write

If the update is destructive, such as removing groups, removing items, or changing selection rules, require explicit confirmation again even if the user already confirmed the general task.

## Verification

A combo write is complete only when all of these are true:
- the API returned success
- the combo exists in the target environment
- the combo is in the intended store
- name matches
- tax rate matches
- price matches
- variant count matches
- each group name matches
- each group `minSelect` and `maxSelect` match
- each expected item is present
- each expected `extraCents` value matches

Minimum read-back check:
- fetch products for the target store
- locate the combo by returned ID
- inspect the stored variant and group structure

## Response Style

Report only:
- environment used
- store used
- whether the run used API or UI
- whether the action was create, update, delete, or inspect-only
- what changed
- which verification points passed
- any real mismatch that still needs attention
