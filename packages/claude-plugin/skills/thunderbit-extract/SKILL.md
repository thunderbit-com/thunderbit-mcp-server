---
name: thunderbit-extract
description: >
  Extract structured data from web pages using AI. Automatically suggests
  extraction fields if no schema is provided, then lets the user edit fields
  via natural language before extracting.
  Use for scraping product listings, contact info, tables, or any structured content.
---

# Thunderbit Extract

Extract structured data from a web page. If the user doesn't provide a schema,
automatically discover fields first, then let the user edit via natural language.

## Workflow

1. Parse the user's input:
   - `url` (required): the web page URL
   - `schema` (optional): JSON Schema defining what to extract
   - `prompt` (optional): natural language guidance (e.g. "Extract product info")
   - `renderMode`: "none" (default), "basic", or "full"
   - `timeout`: timeout in ms, 5000-120000 (default: 60000)
   - `waitFor`: wait time after page load, 0-10000ms (default: 0)

2. **If no schema is provided**:
   a. Call `thunderbit_suggest_fields` with the URL and prompt to discover extractable fields.
   b. Present the suggested fields in an **editable numbered table** using this format:

      ```
      AI suggested N extractable fields:

      | #  | Field Name      | Type   | Instruction                          | Status |
      |----|-----------------|--------|--------------------------------------|--------|
      | 1  | block_height    | NUMBER | Height of the Bitcoin block          | ✅     |
      | 2  | miner_address   | TEXT   | Address of the block producer        | ✅     |
      | 3  | total_txs       | NUMBER | Total number of transactions in block| ✅     |
      ...

      You can adjust fields in natural language, for example:
      - "remove fields #2 and #4"
      - "rename 'miner_address' to 'pool_address'"
      - "add a new 'pool_name' TEXT field, instruction: name of the mining pool"
      - "keep only 1, 3, 5"
      - "keep all, extract now"

      What would you like to adjust?
      ```

   c. **Wait for user response** — interpret the user's natural language instructions:
      - **remove / delete / drop**: Remove specified fields by number or name
      - **add / new / append**: Add a new field with name, type, and instruction
      - **rename**: Rename a field
      - **change type**: Change field type (TEXT/NUMBER/URL/EMAIL/DATE)
      - **change instruction**: Update extraction instruction
      - **keep only**: Keep only the specified fields, remove the rest
      - **keep all / extract now / confirm / OK / proceed**: Accept current fields and proceed
      - User may give **multiple instructions at once** (e.g. "remove 2 and 4, add a pool_name field")

   d. After each edit, **show the updated table** with the same numbered format so the user can verify. Repeat steps c-d until the user confirms.

   e. Convert the confirmed fields into a JSON Schema for extraction:
      ```json
      {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "field_name": {
              "type": "text|number|url|email|date",
              "instruction": "How to extract this field"
            }
          }
        }
      }
      ```

3. **If schema is provided** (or confirmed from step 2):
   a. Call `thunderbit_extract` with the URL and schema.
   b. Return the extracted data in a clear, formatted table.

## Field Editing Rules

- Field types must be one of: `TEXT`, `NUMBER`, `URL`, `EMAIL`, `DATE`
- When the user says "remove URL fields" or similar, remove all fields with type URL
- When adding a field, infer reasonable defaults:
  - If no type specified, default to `TEXT`
  - If no instruction specified, generate one based on the field name and page context
- Preserve the original field order; append new fields at the end
- Always show the # column for easy reference

## Schema Format

The schema for `thunderbit_extract` should follow this structure:
```json
{
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "field_name": {
        "type": "text|number|url|email|date",
        "instruction": "How to extract this field"
      }
    }
  }
}
```

## Error Handling

- **401**: "API Key invalid. Check your THUNDERBIT_API_KEY environment variable."
- **402**: "Insufficient credits. Top up at https://app.thunderbit.com/console/billing"
- **429**: "Rate limit exceeded. Please try again shortly."
- **Timeout**: "Page load timed out. Try increasing timeout or switching renderMode to 'full'."

## Cost

- Suggest fields: free (0 credits)
- Extract: 20 credits per call