---
name: lite-operation-workflow
description: General workflow guidance for Lite / Palmnet store-level operations. Use this first for any Lite backend workflow, then call the relevant workflow-specific skill such as products, combos, or tables. This skill defines shared safety rules, environment and store checks, auth checks, interface points, verification rules, workflow naming rules, and guidance reference rules.
---

# Lite Operation Workflow

Use this skill as the only shared entry for Lite / Palmnet admin work.

All Lite backend work should follow this order:
1. read this workflow skill first
2. confirm shared operating conditions
3. choose the matching workflow-specific skill
4. apply only that skill's unique rules

This skill covers:
- shared operating rules
- environment and store confirmation
- auth and admin context checks
- common interface points
- completion and verification rules
- naming rules for future workflow skills
- guidance reference rules for future expansion

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

Environment-specific login accounts currently documented by the user:

| Environment | Base URL | Email | Password |
|---|---|---|---|
| development | `https://lite-dev.palmnet.co` | `admin@palmnet.co` | `Palmnet2026!` |
| production | `https://lite-es1.palmnet.co` | `admin@palmnet.co` | `Admin123` |

Treat these as environment-specific credentials:
- do not swap development and production passwords
- do not assume one environment token works in the other environment
- if login with a documented account fails, report that clearly and verify whether the credential changed

### Access Token Refresh Rules

When a workflow needs a fresh token, use this order:

1. Prefer the current environment's existing admin context:
   - `localStorage["mini_pos.token"]`
   - `localStorage["mini_pos.storeId"]`
   - `localStorage["mini_pos.orgId"]`
   - `localStorage["mini_pos.scope"]`
   - `localStorage["mini_pos.viewMode"]`
2. Validate the token with:
   - `GET /v1/auth/me`
3. If the token is missing, expired, or invalid, refresh it by logging in again:
   - `POST /v1/auth/login`
   - request body: `{"email":"<env email>","password":"<env password>"}`
4. After login returns a new token:
   - replace the bearer token used for API calls
   - re-read `mini_pos.storeId`, `mini_pos.orgId`, `mini_pos.scope`, and `mini_pos.viewMode` from the current environment context when available
   - re-run `GET /v1/auth/me`
   - re-confirm that the selected store still matches the user-confirmed target store

Never treat token refresh as complete until both are true:
- `GET /v1/auth/me` succeeds with the refreshed token
- the active store context is revalidated for the target environment

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

Then call the matching workflow-specific skill:
- `lite-product-batch-management`
- `lite-combos-creation`
- `lite-table-creation`

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

## Naming Rules For Future Workflows

Use this naming format for all new Lite backend workflow skills:

`lite-<domain>-<action>`

If the workflow is clearly batch-oriented, use:

`lite-<domain>-batch-<action>`

If the workflow is clearly review-only or inspection-only, use:

`lite-<domain>-inspection`
or
`lite-<domain>-audit`

Naming constraints:
- start with `lite-`
- describe the business object first
- describe the action second
- keep the name stable and literal
- avoid vague words such as `helper`, `tool`, `misc`, `ops`, `flow`, or `temp`
- do not mix multiple unrelated workflows into one name

## Folder Structure For Future Workflows

Each new workflow skill should use this structure:

```text
workflow-skill/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    └── guidance.md
```

Use `references/` only when the workflow has real extra detail that should not live in the main `SKILL.md`.

## Required Inheritance Pattern

Every new workflow skill must explicitly reference this file:
- [lite-operation-workflow](../lite-operation-workflow/SKILL.md)

Put this near the top of the new workflow skill:

```md
Shared operating rules live in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any submission
- verify token, store scope, and current admin context
- do not reuse IDs from another environment or another store
```

If the workflow extends an existing narrower skill, it may also reference that narrower skill.

If the workflow depends on environment login or token refresh, it should also inherit:
- the documented environment account mapping
- the token refresh order
- the requirement to revalidate `storeId` after refreshing the token

## Guidance Reference Rules

When a workflow has extra operational detail, create:
- `references/guidance.md`

Use that file for:
- page-by-page navigation notes
- request or payload examples
- naming dictionaries
- exception cases
- workflow-specific edge cases

Then reference it from `SKILL.md` with a direct instruction such as:

```md
For page mapping and edge cases, read [references/guidance.md](references/guidance.md) before editing.
```

Prefer one `guidance.md`.

Only split further when the workflow becomes large enough to justify:
- `references/api.md`
- `references/ui-map.md`
- `references/rules.md`
