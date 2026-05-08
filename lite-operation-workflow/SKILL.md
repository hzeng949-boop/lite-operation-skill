---
name: lite-operation-workflow
description: Workflow guidance for extending Lite / Palmnet admin operations. Use when creating a new Lite backend workflow, naming a new workflow skill, or deciding how a workflow should inherit and reference shared guidance from `lite-operation`.
---

# Lite Operation Workflow

Use this skill when a later agent needs to add a new Lite backend workflow.

This skill does not define one business object. It defines how new workflow skills should be named, scoped, and linked back to shared guidance.

## Purpose

When a new Lite admin workflow appears, do not write it as an isolated skill.

Create it as:
- one workflow-specific skill
- one clear reference to shared guidance in `lite-operation`
- only the workflow-specific rules that are not already covered by the shared guidance

## Naming Rules

Use this naming format for all new Lite backend workflow skills:

`lite-<domain>-<action>`

Good examples:
- `lite-order-refund`
- `lite-discount-management`
- `lite-customer-merge`
- `lite-kitchen-ticket-reprint`
- `lite-store-hours-update`

If the workflow is clearly batch-oriented, use:

`lite-<domain>-batch-<action>`

Examples:
- `lite-order-batch-cancel`
- `lite-coupon-batch-import`

If the workflow is clearly review-only or inspection-only, use:

`lite-<domain>-inspection`
or
`lite-<domain>-audit`

Examples:
- `lite-payment-audit`
- `lite-menu-inspection`

## Naming Constraints

Every new workflow name should follow these rules:
- start with `lite-`
- describe the business object first
- describe the action second
- keep the name stable and literal
- avoid vague words such as `helper`, `tool`, `misc`, `ops`, `flow`, or `temp`
- do not mix multiple unrelated workflows into one name

Prefer:
- `lite-table-merge`

Avoid:
- `lite-table-stuff`

Prefer:
- `lite-membership-points-adjustment`

Avoid:
- `lite-membership-all-in-one`

## Folder Structure

Each new workflow skill should use this structure:

```text
workflow-skill/
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    └── guidance.md
```

Use `references/` only when the workflow has real extra detail that should not live in the main `SKILL.md`, such as:
- page maps
- field dictionaries
- API payload examples
- business rules that are long or likely to change

## Required Inheritance Pattern

Every new workflow skill must explicitly reference:
- [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md)

Put this near the top of the new workflow skill:

```md
Shared operating rules live in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any submission
- verify token, store scope, and current admin context
- do not reuse IDs from another environment or another store
```

If the workflow is table-specific, combo-specific, product-specific, or otherwise extends an existing narrower skill, it may also reference that narrower skill.

Use this rule:
- shared global rules -> always reference `lite-operation`
- object-specific rules -> reference the closest existing sub-skill only if it actually reduces duplication

## What Belongs In The New Workflow Skill

Keep only workflow-specific content in the new skill:
- what object or page the workflow controls
- default execution path
  - API
  - admin UI
- required workflow-specific parameters
- workflow-specific validation rules
- destructive-action confirmation rules
- workflow-specific verification points

Do not duplicate long shared sections for:
- environment mapping
- auth pattern
- token keys
- store scope rules
- generic confirmation rules
- generic response style

Those belong in `lite-operation`.

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

Do not create multiple scattered guidance files unless there is a real need to split by domain.

Prefer:
- one `guidance.md`

Only split further when the workflow becomes large enough to justify:
- `references/api.md`
- `references/ui-map.md`
- `references/rules.md`

## Decision Rule For New Workflows

Before creating a new workflow skill, decide which case applies:

1. If the work is already fully covered by `lite-operation` plus an existing sub-skill:
   do not create a new workflow skill.
2. If the work introduces a distinct backend process with its own fields, validations, or verification:
   create a new workflow skill.
3. If the work is only a one-off task with no reusable pattern:
   do not create a new skill yet.

## Minimum SKILL.md Template

Use this template for future Lite workflow skills:

```md
---
name: lite-<domain>-<action>
description: <clear trigger description>
---

# <Readable Title>

Use this skill for store-level <workflow> work in Lite.

Shared operating rules live in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any submission
- verify token, store scope, and current admin context
- do not reuse IDs from another environment or another store

This skill only adds <workflow>-specific rules.

## Hard Safety Gate

Before any write, explicitly confirm all three items with the user:
- target environment
- target store
- exact change to apply

## Required Input

- environment base URL
- target store
- action type
- workflow-specific fields

## Shared Interface Points

Use the shared endpoints defined in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

## Workflow-Specific Rules

- only list unique rules here

## Verification

- only list workflow-specific verification here

## Response Style

- report only the outcome, verification, and real mismatches
```

## Completion Standard

A new Lite workflow skill is complete only when:
- its name follows the naming rules
- it references `lite-operation`
- it keeps shared rules out of duplicated sections
- it has clear workflow-specific parameters
- it has clear verification rules
- it tells later agents where extra guidance should live
