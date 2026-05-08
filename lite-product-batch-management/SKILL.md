---
name: lite-product-batch-management
description: Batch-create categories and products in the Lite / Palmnet admin for a selected store. Use when the user wants to import many menu categories or products into `https://lite-es1.palmnet.co/admin/categories`, `https://lite-es1.palmnet.co/admin/products`, or the same Lite admin screens, especially from Excel exports, CSV-style lists, or compact menu lists. Prefer store-level API import, and only fall back to UI entry when the API path is blocked.
---

# Lite Product Batch Management

Use this skill for store-level category and standard product work in Lite.

Shared operating rules live in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any submission
- verify token, store scope, and current admin context
- do not reuse IDs from another environment or another store

This skill only adds category-specific and product-specific rules.

Prefer API writes over manual page entry.

## Hard Safety Gate

Before any write, explicitly confirm all three items with the user:
- target environment
- target store
- exact change to apply

Do not send `POST`, `PATCH`, `PUT`, or `DELETE`, and do not submit any admin form, until those three items are confirmed in the current task.

If any one of them is missing or ambiguous, stop and ask.

## Required Input

Collect or infer these values before writing anything:
- environment base URL
- target admin context and active store
- action type
  - `create`
  - `update`
  - `delete`
  - `inspect-only`
- category list to create, if categories do not exist yet
- product source data, usually Excel rows or a structured list
- product name rule: capitalize each word unless the user says otherwise
- category name for each product
- numeric price for each product
- target tax rate

Reject ambiguous data instead of guessing:
- missing category for a product
- missing price
- missing tax mapping
- duplicate product rows that cannot be safely deduplicated
- unclear capitalization rules that conflict with user instructions

## Shared Interface Points

Use the shared auth, category, product, and tax endpoints defined in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

For this skill, the main write path is:
- categories through store category endpoints
- standard products through store product endpoints with `kind: "standard"`

Create categories before products when the source data references categories that do not yet exist.

Modifier-group relations also exist, but skip them unless the user explicitly asks for modifier import.

## Verified Request Shape

For a standard single-price product, the verified create payload is:

```json
{
  "kind": "standard",
  "name": { "_base": "Patatas Bravas" },
  "brief": null,
  "description": null,
  "taxRateId": "TAX_RATE_ID",
  "categoryLinks": [{ "categoryId": "CATEGORY_ID" }],
  "variantsEnabled": false,
  "variants": [
    {
      "name": {},
      "priceCents": 700
    }
  ]
}
```

Rules:
- `name._base` is the visible product name
- `priceCents` is integer cents, not decimal text
- `categoryLinks` must contain the resolved category ID
- `taxRateId` must be a valid current-store tax rate ID

## Normalization Rules

Before import:
- title-case product names if the user requested initial capitals
- preserve accented characters
- keep category names exactly as used in Lite
- convert `7`, `7.0`, `7.00` into `700`
- skip products that already exist by exact normalized name unless the user asks for updates

For title case, use this rule:
- split on spaces
- uppercase the first character of each token
- keep the remaining characters unchanged

Example:
- `patatas bravas` -> `Patatas Bravas`
- `coulant de tortilla de patatas` -> `Coulant De Tortilla De Patatas`

## Tax Rate Handling

Use the shared tax resolution rules from [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

Product-specific matching rule:
- prefer an explicit tax label from the source data, such as `IVA (10%)`
- if the source data gives only a percent, match by percent within the current store
- if multiple current-store tax rates match the same label or percent, stop and ask the user
- if no current-store tax rate matches, stop and ask the user

If the current store has only one active tax rate and the user did not provide a tax hint, it is acceptable to use that single current-store tax rate.

## Preferred Workflow

1. Confirm environment, store, and exact change with the user.
2. Confirm the current store context.
3. Read auth and store-scoped data.
4. Create missing categories first.
5. Read back categories and map category names to category IDs.
6. Normalize product names to title case if required.
7. Convert prices to integer cents.
8. Resolve the correct current-store tax rate.
9. Create or update products through the store product endpoint.
10. Refresh the products page and verify totals plus sample rows.

Use the UI only when the API path is unavailable or returns blocking validation errors that require manual correction.

## Update Guidance

For updates:
- read the current product first
- compare old and new values before sending `PATCH`
- keep untouched fields unchanged
- tell the user exactly what will change before sending the write

If the update is destructive, such as deleting categories, deleting products, or moving products across categories unexpectedly, require explicit confirmation again even if the general task was already confirmed.

## Verification

A batch import is complete only when all of these are true:
- requested categories exist
- final product total matches created plus previously existing items
- no API create or update errors remain
- sample products show the correct name, category, and price on the admin page

Minimum checks:
- refresh `商品` page and confirm total count
- confirm at least one newly created or updated product from the first part of the source data
- confirm at least one newly created or updated product from the last part of the source data
- confirm one product in a wine or beverage category if those were included

## Response Style

Report only:
- whether the run used API or UI
- environment used
- store used
- how many categories and products were created, updated, deleted, or skipped
- which verification points passed
- any real mismatch that still needs attention
