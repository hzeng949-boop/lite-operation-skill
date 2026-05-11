---
name: lite-shorcode-creation
description: Bind a Lite store's current table URLs to PalmNFC short codes as a second-stage workflow after table creation. Use when the user wants to read the current store table list from Lite, generate the same number of PalmNFC short codes, rebuild a `Short Code,URL` CSV, and import the final bindings into `https://scode.palmnfc.eu/admin`.
---

# Lite Shorcode Creation

Use this skill for second-stage PalmNFC short-code binding after Lite table creation.

Shared operating rules live in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any write
- verify token, store scope, and current admin context
- do not reuse IDs, tokens, or exported rows from another environment or another store

This skill does not create tables. It only reads the current store tables from Lite and binds their existing table URLs to new PalmNFC short codes.

Prefer direct API handling.

## Hard Safety Gate

Before any PalmNFC write, explicitly confirm all three items with the user:
- target Lite environment
- target store
- exact change to apply
  - generate PalmNFC short codes for the current store tables
  - bind those short codes to the current store table URLs

Do not send any PalmNFC write request until those three items are confirmed in the current task.

If any one of them is missing or ambiguous, stop and ask.

## Required Input

Collect or infer these values before writing:
- Lite environment base URL
- target store ID or the currently selected store
- store display name to use as the PalmNFC `note`
- valid Lite bearer token and store-level scope
- PalmNFC Basic Auth credentials
- action type
  - `create`
  - `inspect-only`

Reject or pause on:
- missing Lite token
- missing selected store
- missing store-level scope
- missing PalmNFC credentials
- any request that assumes table URLs from an old file instead of the current Lite store data
- any request to generate or repair table URLs inside this workflow

## System Boundaries

Use two systems with different roles:
- Lite: read-only source of truth for the current store tables and table URLs
- PalmNFC: write target for short-code generation and redirect binding

This skill does not:
- create tables
- rename tables
- generate table URLs
- repair missing table URLs
- accept a manually edited legacy CSV as the default source

## Shared Interface Points

Use the shared auth and table endpoints defined in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

For this skill, the required interface points are:
- Lite `GET /v1/stores/{storeId}/tables`
- PalmNFC `POST /api/codes`
- PalmNFC `GET /api/export?recordId=...`
- PalmNFC `POST /api/import`

## Lite Read Rules

Read the current store tables at runtime. Do not rely on an old exported file.

Required Lite fields per table:
- table name
- table URL

If the table list response does not expose a usable table URL field, stop and report that the current Lite API payload is insufficient for this workflow.

Before any PalmNFC write:
- confirm the table list belongs to the user-confirmed store
- confirm every row needed for binding has a non-empty table URL
- count the total rows eligible for binding

## Ordering Rule

Use one-to-one sequential mapping between Lite table URLs and PalmNFC short codes.

Default ordering rule:
- prefer the stable order returned by the Lite table list endpoint
- if the endpoint order is not stable or not trustworthy, sort by table name using natural ordering before pairing

Once the order is chosen for a run:
- keep the Lite table rows in that order
- keep the PalmNFC exported short codes in CSV row order
- pair row `1` to row `1`, row `2` to row `2`, and so on

Do not reorder one side independently after pairing starts.

## PalmNFC Write Rules

Use the store display name as the `note` when generating short codes.

### Generate Codes

Send `POST /api/codes` with:
- `count` equal to the Lite table count selected for binding
- `note` equal to the confirmed store display name

Expected result:
- success response
- returned `recordId`

If the requested `count` does not exactly match the Lite eligible table count, stop.

### Export Codes

After generation:
- download the generated CSV from `GET /api/export?recordId=...`
- read the short code column from the exported file
- confirm the exported data row count matches the Lite eligible table count

If the exported CSV row count does not match, stop and do not upload anything.

### Rebuild Import CSV

Create a new CSV with this exact header:

```csv
Short Code,URL
```

For each paired row:
- `Short Code` comes from the PalmNFC exported CSV
- `URL` comes from the Lite runtime table URL list

Do not reuse the downloaded template as the default source of truth. The template only defines the required header shape.

### Import Bindings

Upload the rebuilt CSV to `POST /api/import` with:
- `file` set to the newly generated CSV
- `allowOverride=true`

Default override rule:
- use `allowOverride=true`
- this workflow assumes the latest generated mapping should replace any previous short-code binding for the same rows

## Preferred Workflow

1. Confirm environment, store, and exact second-stage change with the user.
2. Read the current Lite token, selected store, and store-level scope.
3. Read the current store table list from Lite.
4. Extract table names and table URLs.
5. Reject the run if any required table URL is missing.
6. Determine the final row order.
7. Count the eligible Lite rows.
8. Generate the same number of PalmNFC short codes using the store display name as `note`.
9. Read the returned `recordId`.
10. Export the generated PalmNFC CSV for that `recordId`.
11. Confirm the PalmNFC CSV row count matches the Lite eligible row count.
12. Rebuild a new `Short Code,URL` CSV by sequential pairing.
13. Upload the rebuilt CSV with `allowOverride=true`.
14. Read the import result and verify success and failure counts.

## Verification

A run is complete only when all of these are true:
- the Lite read used the confirmed environment and store
- the Lite table count is known
- every bound table row had a non-empty URL
- the PalmNFC generate call returned success
- a `recordId` was returned
- the PalmNFC export CSV row count matched the Lite eligible row count
- the rebuilt import CSV used the exact `Short Code,URL` header
- the PalmNFC import returned success
- the PalmNFC import failure count is `0`

Minimum checks:
- confirm the first paired row
- confirm one middle paired row when the batch has at least three rows
- confirm the last paired row
- report the Lite table count
- report the PalmNFC generated batch `recordId`
- report PalmNFC import success count and failure count

If any mismatch appears, stop and report it as unresolved. Do not describe the task as complete.

## Response Style

Report only:
- environment used
- store used
- whether the run used API or UI
- whether the action was `create` or `inspect-only`
- Lite table total count
- PalmNFC generated batch `recordId`
- PalmNFC import success count
- PalmNFC import failure count
- which verification points passed
- any unresolved exception that still needs user attention
