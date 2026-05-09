---
name: lite-table-qrcode-export
description: Export a store's Lite table list into a matching sheet of table name and QR code URL. Use when the user wants the table QR code mapping for a selected store in `https://lite-es1.palmnet.co/admin/tables` or another Lite environment, especially when the final output should be a two-column table or CSV with `桌台名称` and URL.
---

# Lite Table QR Code Export

Use this skill for store-level table QR code export work in Lite.

Shared operating rules live in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment and target store before any read
- verify token, store scope, and current admin context
- do not reuse token, store ID, or table ID from another environment or another store

This skill only adds QR code export specific rules.

Prefer API reads over manual page copying.

## Scope

This skill is for read-only export work.

Typical outputs:
- markdown table
- CSV text
- spreadsheet-ready two-column table

Default output columns:
- `桌台名称`
- `URL`

Only include extra columns such as `区域`, `桌台ID`, or `门店ID` if the user explicitly asks for them.

## Required Input

Collect or confirm these values before exporting:
- environment base URL
- target store
  - store name, store ID, or active selected store
- action type
  - `inspect-only`
- desired output format
  - markdown table
  - CSV
  - spreadsheet-ready plain text
- QR code URL rule, if the user supplied one

If the user does not provide a QR code URL rule, read the current confirmed rule from the task context before generating links.

Reject ambiguous input instead of guessing:
- unclear target store
- more than one candidate store
- conflicting QR code URL formats in the same task
- table data returned from a different store than the one the user confirmed

## Shared Interface Points

Use the shared auth and table endpoints defined in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Main read path:
- `POST /v1/auth/login`
- `GET /v1/auth/me`
- `GET /v1/stores/{storeId}/tables`

Preferred auth order:
1. current environment token from admin context
2. validate with `GET /v1/auth/me`
3. if missing or invalid, refresh with `POST /v1/auth/login`

## Export Rules

Read the current store tables from:
- `GET /v1/stores/{storeId}/tables`

Expected table fields:
- `id`
- `storeId`
- `name`
- `area`

Use these mapping rules:
- `桌台名称` = `name`
- `URL` = confirmed QR code format with current store ID and current table ID

Confirmed QR code format for the current verified workflow:
- `https://app-es1.palmnet.co/t/{storeId}/{tableId}`

Only use another format if the user explicitly corrects it in the current task.

Sorting rules:
- keep API order by default
- if the user asks for sort by name, sort by the visible table name
- do not silently reorder mixed naming styles unless the user asks

Output rules:
- if the user asks for a matching sheet, return only `桌台名称` and `URL`
- wrap URLs in backticks for markdown output
- do not shorten URLs
- preserve exact table names

## Preferred Workflow

1. Confirm environment and target store.
2. Resolve or confirm the target `storeId`.
3. Validate the current token with `GET /v1/auth/me`.
4. Refresh token through login only if needed.
5. Read `GET /v1/stores/{storeId}/tables`.
6. Check that every returned row belongs to the confirmed `storeId`.
7. Build the export rows from `name` and `id`.
8. Format the final output in the user-requested table format.
9. Count the exported rows and verify the first and last rows.

Use the UI only when the API read path is blocked.

## Verification

An export is complete only when all of these are true:
- the target environment matches the user-confirmed environment
- the target store matches the user-confirmed store
- the table response comes from the confirmed `storeId`
- every exported URL uses the same confirmed format
- the exported row count matches the API row count

Minimum checks:
- confirm total table count
- confirm the first exported row
- confirm the last exported row
- confirm at least one generated URL contains the expected store ID

## Response Style

Report only:
- whether the run used API or UI
- environment used
- store used
- total exported tables
- output format used
- which verification points passed
- any real mismatch that still needs attention
