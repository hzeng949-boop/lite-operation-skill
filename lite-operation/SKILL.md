---
name: lite-operation
description: General skill for Lite / Palmnet store-level operations. Use when the task is to create, update, inspect, or batch-manage categories, products, combo products, or tables in Lite admin. This skill defines the shared safety rules, required parameters, environment checks, auth checks, store checks, interface points, and verification rules across `lite-product-batch-management`, `lite-combos-creation`, and `lite-table-creation`.
---

# Lite Operation

Use this skill as the general operating rule for Lite / Palmnet admin work at store level.

It covers four object types:
- categories
- products
- combos
- tables

Use it to decide:
- what must be confirmed before any action
- which environment and store are in scope
- whether the task should use API or admin UI
- which parameters are required for each object type
- which checks must pass before the task is considered complete

For creating future Lite backend workflow skills, use [lite-operation-workflow](/Users/haoxian/Documents/Cursor/lite-operation-workflow/SKILL.md).

## Hard Safety Gate

Before any write, explicitly confirm all three items with the user:
- target environment
- target store
- exact change to apply

Do not send any write request and do not submit any admin form until those three items are confirmed in the current task.

Write actions include:
- `POST`
- `PATCH`
- `PUT`
- `DELETE`
- any admin UI submission that creates, updates, deletes, reorders, or uploads

If any one of the three items is missing or ambiguous, stop and ask.

## Environment Mapping

Known environments:
- development: `https://lite-dev.palmnet.co`
- production: `https://lite-es1.palmnet.co`

Do not assume the environment from memory alone.

If the user gives an admin URL, derive the base URL from it:
- `https://lite-dev.palmnet.co/admin/...` -> `https://lite-dev.palmnet.co`
- `https://lite-es1.palmnet.co/admin/...` -> `https://lite-es1.palmnet.co`

## Access Token And Admin Context

Verified auth pattern:
- login endpoint: `POST /v1/auth/login`
- current user endpoint: `GET /v1/auth/me`
- auth header: `Authorization: Bearer <token>`

Useful admin context keys:
- `localStorage["mini_pos.token"]`
- `localStorage["mini_pos.storeId"]`
- `localStorage["mini_pos.orgId"]`
- `localStorage["mini_pos.scope"]`
- `localStorage["mini_pos.viewMode"]`

Before any write, verify:
- token exists
- selected store exists
- current scope is store-level
- current store matches the user-confirmed target store

If token, store, or store-level scope is missing, stop and report what is missing.

## Store And Org Rules

All tasks covered by this skill are store-scoped.

Use these rules:
- `storeId` is required for categories, products, combos, and tables
- `orgId` is only needed for org-scoped actions such as asset upload
- do not reuse category IDs, tax IDs, product IDs, or other IDs across environments
- do not reuse IDs from an older store or an older run without re-reading the current target environment first

If the current admin context is on a different store than the user requested, stop and ask whether to switch or continue.

## Object Routing

Choose the object type first:

| Object type | Default path | Notes |
|---|---|---|
| category | API | Create or update category records for one store |
| product | API | Batch import or edit standard products |
| combo | API | Combos are created through product APIs with `kind: "combo"` |
| table | Admin UI | Default path is the tables page unless the user explicitly asks for API-oriented handling |

## Shared Required Parameters

Collect or confirm these values for every task:
- environment base URL
- target store ID or active selected store
- object type
- action type
  - `create`
  - `update`
  - `delete`
  - `inspect-only`
- exact source data or exact target change

Reject or pause on:
- missing environment
- missing store scope
- missing change scope
- data that cannot be mapped to the current store
- IDs copied from another environment without current-store revalidation

## Interface Points

Shared read/auth endpoints:
- `POST /v1/auth/login`
- `GET /v1/auth/me`

Category endpoints:
- `GET /v1/stores/{storeId}/categories`
- `POST /v1/stores/{storeId}/categories`
- `PATCH /v1/stores/{storeId}/categories/{categoryId}`
- `DELETE /v1/stores/{storeId}/categories/{categoryId}`
- `PUT /v1/stores/{storeId}/categories-order`

Product and combo endpoints:
- `GET /v1/stores/{storeId}/products`
- `POST /v1/stores/{storeId}/products`
- `PATCH /v1/stores/{storeId}/products/{productId}`
- `DELETE /v1/stores/{storeId}/products/{productId}`
- `PUT /v1/stores/{storeId}/products/{productId}/image`
- `PUT /v1/stores/{storeId}/products/{productId}/variants/{variantId}/image`

Tax lookup endpoint:
- `GET /v1/stores/{storeId}/tax-rates`

Table endpoints:
- `GET /v1/stores/{storeId}/tables`
- `POST /v1/stores/{storeId}/tables`
- `PATCH /v1/stores/{storeId}/tables/{tableId}`
- `DELETE /v1/stores/{storeId}/tables/{tableId}`
- `PUT /v1/stores/{storeId}/tables-order`

Optional asset endpoint:
- `POST /v1/orgs/{orgId}/assets`

## Object-Specific Parameters

### Categories

Required:
- category name
- target store
- action type

Checks:
- create missing categories before products that depend on them
- keep category names exactly as used in the target store

### Products

Required:
- product name
- category name
- price
- tax rate

Verified payload points for standard products:
- `kind: "standard"`
- `name._base`
- `taxRateId`
- `categoryLinks`
- `variantsEnabled`
- `variants[].priceCents`

Checks:
- convert all prices to integer cents before write
- resolve category names to current-store category IDs
- resolve tax to a current-store `taxRateId`
- default behavior is to skip exact normalized duplicates unless the user explicitly asks for updates

### Combos

Required:
- combo name
- combo category
- tax rate
- one or more variants

For each variant:
- visible name, or confirmed intentional blank
- `priceCents`
- one or more combo groups

For each combo group:
- `name._base`
- `minSelect`
- `maxSelect`
- non-empty item list

For each item:
- `productId`
- optional `variantId`
- `extraCents`

Verified payload points for combos:
- `kind: "combo"`
- `name._base`
- `taxRateId`
- `categoryLinks`
- `variants[].priceCents`
- `variants[].comboGroups`

Checks:
- combos do not use a separate `/combos` API path
- create or update combos through product endpoints
- `minSelect` must be less than or equal to `maxSelect`
- every item must resolve inside the current target environment
- all prices and upcharges must be integer cents

### Tables

Required:
- target tables page or confirmed table scope
- one or more groups

For each group:
- `area`
- `prefix`
- `start`
- `end`
- numbering format, including zero padding if present

Checks:
- expand compact ranges into the full table list before creation
- preserve the exact numbering style the user wrote
- reject mixed padding or duplicate ranges
- default path is admin UI creation on the tables page

## Tax Resolution Rules

Never hardcode a tax ID from a previous store or previous run.

Preferred order:
1. read current-store tax rates
2. map visible labels and percents
3. match the source data against the current store
4. if more than one match exists, stop and ask
5. if no match exists, stop and ask

If the current store has only one active tax rate and the source data gives no tax hint, using that one active rate is acceptable.

## Data Normalization Rules

Before write:
- convert decimal prices to integer cents
- convert item upcharges to integer cents
- resolve names to current-environment IDs
- preserve accented characters
- preserve store-specific category naming

Do not guess:
- missing category mappings
- missing tax mappings
- missing product resolution
- conflicting numbering rules
- conflicting combo group rules

## Required Confirmation Before Submission

Immediately before any final submit, re-check all of these:
- environment confirmed by the user
- target store confirmed by the user
- exact change confirmed by the user
- current admin context matches the confirmed store
- token exists
- required IDs were re-read from the target environment
- payload or table list matches the user-confirmed source

If any one of the above is not true, do not submit.

## Completion Standards

The task is complete only when:
- the intended write path finished without unresolved errors
- the result exists in the correct environment
- the result exists in the correct store
- sampled verification points pass
- no unresolved mismatch remains hidden

Minimum verification by object type:

| Object type | Minimum verification |
|---|---|
| category | requested categories exist in the target store |
| product | total count is consistent and sampled products show correct name, category, and price |
| combo | combo exists and sampled name, tax, price, groups, and items are correct |
| table | total count matches the expanded list and the last item of each range exists with the correct area |

## Response Style

Report only:
- object type and path used
  - API
  - admin UI
- what changed
- count created, updated, deleted, or skipped
- which verification points passed
- any real mismatch that still needs attention

Do not report hidden assumptions as if they were facts.
