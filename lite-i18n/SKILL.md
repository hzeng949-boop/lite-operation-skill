---
name: lite-i18n
description: Split and write category, product, and modifier-group translations in the Lite / Palmnet admin for a selected store. Use when the user wants to add, clean up, or normalize multilingual fields in `https://lite-es1.palmnet.co/admin/i18n/categories`, `https://lite-es1.palmnet.co/admin/i18n/products`, `https://lite-es1.palmnet.co/admin/i18n/modifier-groups`, or related Lite admin translation screens. Prefer store-level API updates, and use the workflow reference for the exact read, split, write, and verification sequence.
---

# Lite I18n

Use this skill for store-level i18n work in Lite.

Shared operating rules live in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

Always apply the shared rules first, especially:
- confirm environment, store, and exact change before any write
- verify token, store scope, and current admin context
- do not reuse IDs from another environment or another store

This skill only adds translation-specific rules.

## Hard Safety Gate

Before any write, explicitly confirm all three items with the user:
- target environment
- target store
- exact i18n scope
  - categories
  - products
  - modifier groups
  - or any combination of them

Do not send `PATCH`, `POST`, `PUT`, or `DELETE`, and do not submit any admin form, until those three items are confirmed in the current task.

If the current admin context is on a different store than the user requested, stop and ask whether to switch or continue.

## Required Input

Collect or infer these values before writing anything:
- environment base URL
- target admin context and active store
- i18n object scope
- target locales, usually `zh` and `es`
- source pattern
  - mixed Chinese and Spanish in one field
  - Chinese-only field missing Spanish
  - Spanish-only field missing Chinese
  - explicit translation table from the user
- update mode
  - split-only
  - fill-known-missing
  - inspect-only

Reject ambiguous data instead of guessing:
- unclear target store
- unclear locale target
- text that cannot be safely split
- strings that require invented translation without user approval

## Shared Interface Points

Use the shared auth and store endpoints in [lite-operation general skill](/Users/haoxian/Documents/Cursor/Lite-operation/SKILL.md).

For this skill, the main read and write path is:
- categories through store category endpoints
- products through store product endpoints
- modifier groups through store modifier-group endpoints

Read the exact workflow in [references/workflow.md](references/workflow.md) before writing.

## Operating Rules

Prefer API updates over UI editing.

Base name rule:
- preserve the visible display style unless the user explicitly asks to rewrite it
- when the original display name is a combined Chinese-Spanish label, keep `_base` as `中文·西语`
- never overwrite `_base` with pure Spanish unless the user explicitly asks for that output

Split rule:
- if one field contains both Chinese and Spanish and the separation is clear, split it
- write Chinese into `zh`
- write Spanish into `es`
- keep `_base` in the store's original display style
- default combined format is `中文·西语` when both languages are present

Do not invent translations by default:
- if the field is pure Spanish and has no Chinese source, leave `zh` empty unless the user explicitly asks to generate or supply Chinese
- if the field is pure Chinese and has no Spanish source, leave `es` empty unless the user explicitly asks to generate or supply Spanish

Preserve unrelated fields:
- categories: keep color and layout mode unchanged
- products: keep tax, category links, variants, and other untouched fields unchanged
- modifier groups: keep selection rules unchanged
- modifier items: keep price delta, active state, sort order, and default state unchanged

## Preferred Workflow

1. Confirm environment, store, and exact i18n scope.
2. Read the current store and confirm `defaultLocale` plus `supportedLocales`.
3. Read the target objects from store APIs.
4. Determine the current base-name display style from existing data or user instruction.
5. Separate records into:
   - safe to split automatically
   - already correct
   - missing translation and not safe to infer
6. Write only the safe records.
7. Read back the updated records.
8. Verify no mixed Chinese-Spanish strings remain in the fields that were supposed to be split.
9. Report updated counts and list any remaining items that still need manual translation.

Use the UI only when the API path is blocked or the user explicitly asks for UI entry.

## Verification

An i18n batch is complete only when all of these are true:
- the requested object scope was processed
- updated entries show the expected `zh` and `es`
- `_base` matches the preserved display-name rule
- no API write errors remain
- no mixed-language strings remain in the fields that were supposed to be split

Minimum checks:
- re-read categories, products, and modifier groups in scope
- confirm updated counts by object type
- confirm at least one sample from the first part of the batch
- confirm at least one sample from the last part of the batch
- list any entries still missing translation because no safe source existed

## Response Style

Report only:
- whether the run used API or UI
- environment used
- store used
- which object types were processed
- how many entries were updated by object type
- which verification points passed
- which items still need manual translation
