---
name: lite-table-creation
description: Create table batches in the Lite / Palmnet admin for a selected store. Use when the user wants to add many tables quickly in `https://lite-es1.palmnet.co/admin/tables` or the same Lite admin table screen, especially when the request is given as area prefixes plus numeric ranges such as `SALON S1-S40`, `BARRA B1-B10`, `TERRAZA T1-T10`, or similar grouped table creation tasks.
---

# Lite Table Creation

Use this skill for store-level table work in Lite.

Shared operating rules live in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any submission
- verify token, store scope, and current admin context
- do not submit table creation in the wrong store

This skill only adds table-specific rules.

Default path is the admin tables page, not direct API creation, unless the user explicitly asks for API-oriented handling.

## Hard Safety Gate

Before any write, explicitly confirm all three items with the user:
- target environment
- target store
- exact change to apply

Do not submit any admin form and do not send any write request until those three items are confirmed in the current task.

If any one of them is missing or ambiguous, stop and ask.

## Required Input

Collect or infer these values before creating anything:
- environment base URL
- target admin page, normally `https://lite-es1.palmnet.co/admin/tables`
- the active store or org context already selected in the admin
- action type
  - `create`
  - `update`
  - `delete`
  - `inspect-only`
- one or more table groups, each defined as:
  - `area`
  - `prefix`
  - `start`
  - `end`
- whether the numbering keeps zero padding or not

Default assumption for numbering:
- keep the exact style the user writes
- `S1-S40` means `S1, S2, ... S40`
- `T01-T10` means `T01, T02, ... T10`

Reject ambiguous instructions instead of guessing:
- mixed padding styles
- unclear area names
- duplicate ranges
- requests that collide with existing naming rules the user explicitly gave earlier in the same task

## Expand The Request

Convert each group into an explicit table list before touching the page.

Example:
- `SALON, S1-S40` -> `S1` to `S40`, all in area `SALON`
- `BARRA, B1-B10` -> `B1` to `B10`, all in area `BARRA`
- `TERRAZA, T1-T10` -> `T1` to `T10`, all in area `TERRAZA`
- `JARDIN, J1-J8` -> `J1` to `J8`, all in area `JARDIN`

Before creation:
- count the full total
- keep a written expansion list in working memory
- preserve the user's ordering by group unless the user asked for a different order

## Shared Interface Points

Use the shared table endpoints defined in [lite-operation-workflow](../lite-operation-workflow/SKILL.md).

The normal creation workflow should still follow the admin screen unless the user asks otherwise.

## Admin Workflow

Use the in-app browser on the admin page.

1. Confirm environment, store, and exact change with the user.
2. Confirm the page is `桌台` and the current URL is the admin tables page.
3. Confirm there is a valid store context.
4. If the page shows no store selected, stop and tell the user that table management is store-level.
5. Use the `创建` entry on the page.
6. For each expanded table:
   - set `名称` to the exact table name
   - set `区域` to the exact area name
   - submit creation
   - wait until the table appears back in the list
7. Continue until the full batch is complete.

Prefer the UI path unless the user explicitly asks for an API-oriented operation.

## Validation

A task is complete only when all of these are true:
- the total count matches the expanded list
- every requested group exists with the correct area label
- the last item of each range exists
- no name formatting drift was introduced

Minimum checks after creation:
- confirm the table counter on the page
- search or inspect at least the last item of every created range
- confirm area names for the sampled items

Good verification examples:
- `S40`
- `B10`
- `T10`
- `J8`

## Response Style

Report only:
- environment used
- store used
- whether the run used API or UI
- whether the action was create, update, delete, or inspect-only
- what was created
- total count
- which verification points passed
- any real mismatch that still needs user attention
