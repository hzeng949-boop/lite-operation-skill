# Lite I18n Workflow

Use this reference when the task is to clean up or complete multilingual fields in Lite.

## Confirm First

Before any write, confirm:
- environment
- target store
- object scope
  - categories
  - products
  - modifier groups

If the user only says “current page” or “right-side store”, confirm the resolved store name before writing.

## Read Path

1. Log in and get a valid bearer token.
2. Read the current store:
   - `GET /v1/stores/{storeId}`
3. Read the target objects:
   - categories: `GET /v1/stores/{storeId}/categories`
   - products: `GET /v1/stores/{storeId}/products`
   - modifier groups: `GET /v1/stores/{storeId}/modifier-groups`

Check:
- `defaultLocale`
- `supportedLocales`
- current object count
- whether the target fields already contain `zh` or `es`

## Safe Split Rule

Use automatic splitting only when the same field clearly contains Chinese followed by Spanish.

Conservative split rule:
- detect a string that contains both Chinese and Latin characters
- split at the first Latin character
- trim separators around both sides
  - spaces
  - `·`
  - `-`
  - `:`
  - `/`
- left side becomes `zh`
- right side becomes `es`

Examples:

| Source | zh | es |
|---|---|---|
| `辣度 Niveles de Picante` | `辣度` | `Niveles de Picante` |
| `主食 · Arroz y Fideos` | `主食` | `Arroz y Fideos` |
| `不辣 No picante` | `不辣` | `No picante` |

Do not auto-split when:
- Chinese and Spanish are interleaved in a way that makes the boundary unclear
- the value is Spanish-only and Chinese would need to be invented
- the value is Chinese-only and Spanish would need to be invented

## Write Rules

### Categories

Read current values first, then write only `name` while preserving other fields implicitly.

Verified endpoint:
- `PATCH /v1/stores/{storeId}/categories/{categoryId}`

Expected result shape:

```json
{
  "name": {
    "_base": "Arroz y Fideos",
    "es": "Arroz y Fideos",
    "zh": "主食"
  }
}
```

### Products

For product updates, preserve unrelated fields.

Verified endpoint:
- `PATCH /v1/stores/{storeId}/products/{productId}`

Preserve at least:
- `brief`
- `description`
- `taxRateId`
- `categoryLinks`
- `variantsEnabled`
- `variants`

Typical patch shape:

```json
{
  "name": {
    "_base": "Fideos de arroz estilo Yunnan",
    "es": "Fideos de arroz estilo Yunnan",
    "zh": "小锅米线"
  },
  "brief": { "_base": "", "es": "" },
  "description": { "_base": "", "es": "" },
  "taxRateId": "CURRENT_STORE_TAX_ID",
  "categoryLinks": [{ "categoryId": "CURRENT_STORE_CATEGORY_ID", "sortOrder": 4 }],
  "variantsEnabled": false,
  "variants": [
    {
      "id": "VARIANT_ID",
      "name": { "_base": "", "es": "" },
      "priceCents": 1095
    }
  ]
}
```

### Modifier Groups

Verified endpoints:
- `PATCH /v1/stores/{storeId}/modifier-groups/{groupId}`
- `PATCH /v1/stores/{storeId}/modifier-groups/{groupId}/items/{itemId}`

Preserve:
- `minSelect`
- `maxSelect`
- `allowRepeat`
- `showItemImages`
- item `priceDeltaCents`
- item `isActive`
- item `sortOrder`
- item `isDefault`

## Base Name Rule

Do not rewrite `_base` by default.

Preserve the store's visible naming style:
- if the store already uses combined labels, keep `_base` as `中文·西语`
- if the user explicitly asks for another display format, follow that instruction

For translation cleanup, the default action is:
- keep `_base` as the display name
- write Chinese to `zh`
- write Spanish to `es`

## Verification Checklist

After updates:

1. Re-read all updated objects.
2. Count updated records by object type.
3. Check that target fields no longer contain mixed Chinese and Spanish where splitting was expected.
4. Check that `zh` contains Chinese and `es` contains Spanish for the updated entries.
5. Check that `_base` matches the preserved display-name rule.
6. Produce a list of remaining entries that still have no safe translation source.

## Report Format

Return:
- object types processed
- counts updated by object type
- confirmation that mixed strings were cleared
- examples of before and after
- remaining untranslated items that need manual translation
