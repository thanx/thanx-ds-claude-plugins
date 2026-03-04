---
description: Validate whether a segment's JSON rule matches its human-readable description by reading the targeting rule source code in thanx-nucleus.
---

# Segment Validator

Validate a segment definition by comparing a human-readable description against its JSON rule. Uses the thanx-nucleus targeting rule source code to confirm whether the rule accurately implements the described behavior.

## Usage

```bash
/ds:segment-validator <description_and_rule>
```

**Required:**

- `description_and_rule` - A message containing the segment description and the JSON rule, typically in the format: "Does this [description] fit [rule JSON]?" or any variation that includes both pieces

---

You are executing the `/ds:segment-validator` command.

## Step 1: Parse Input

Arguments: $ARGUMENTS

Extract two components from the input:

1. **Segment description** - The human-readable description of what the segment is supposed to target (e.g., "members who visited in the last 30 days and spent more than $50")
2. **JSON rule** - The targeting rule JSON object (e.g., `{"type": "compound", "operator": "&", "rules": [...]}`)

If the JSON is malformed, report the syntax error and stop.

If either component is missing, ask the user to provide both:

> To validate a segment, I need both the **description** (what the segment should target) and the **JSON rule** (the targeting rule definition). Please provide both.

## Step 2: Identify Rule Types Used

Walk the JSON rule tree and list every rule type referenced:

- If the root rule is `compound`, recursively inspect each sub-rule in the `rules` array. If the rule tree exceeds 10 levels of nesting or contains circular references, report this as a structural issue in the INVALID RULE output
- Record each unique rule type encountered (e.g., `segment`, `purchase`, `membership`, `purchase_count`, `purchase_most_recent`, `reward`, `feedback`, `user_tag`, `point`, `compound`, `dynamic`, `order`, `exclusion`, `special_occasion`, `feedback_most_recent`, `snowflake_query`, `empty`, `purchase_amount`). This list is not exhaustive — if the rule JSON contains a type not listed here, still attempt to read its source from thanx-nucleus before reporting it as unknown
- Note the compound operators used (`&` for AND, `|` for OR, `-` for EXCEPT)

## Step 3: Read Rule Source Code from Keystone

For each unique rule type identified in Step 2, use Keystone MCP tools to read the implementation:

1. **Read the rule model**: Use `read_file` to read `thanx-nucleus:app/models/target/rule/{type}.rb` for each rule type
2. **Read the README**: Use `read_file` to read `thanx-nucleus:app/models/target/README.md` for the full targeting DSL documentation
3. **Check field definitions**: For each field referenced in the rule (e.g., `purchased_at`, `segment`, `state`, `source`), search for its validation and allowed values in `thanx-nucleus:app/classes/target/rule/validator.rb` and `thanx-nucleus:app/classes/target/rule/field_validators.rb`
4. **Check valid segment values**: If the rule uses `type: "segment"`, read `thanx-nucleus:app/models/segment.rb` to confirm valid segment labels (member, vip, daily, weekly, etc.)
5. **Handle missing rule files**: If a rule type's `.rb` file is not found at the expected path, use `search_code` or `find_files` to search the repository for the rule class definition before reporting it as unknown.
6. **Cross-reference Notion documentation**: Use the following Notion pages as additional context for understanding segment patterns and validating descriptions:
   - [Segment Cookbook – Common Segment Recipes](https://www.notion.so/626b427d63ad44fc97dbee847d304ca4) — common segment patterns, reward state definitions, and time delay guidance
   - [Comprehensive Segmentation Attributes Guide](https://www.notion.so/153a84ed402480eab6cfe4ba6e65ca18) — full catalog of available segment attributes (membership, purchases, frequency, feedback, rewards, points, tiers, custom tags) with compound examples
   - If these pages are inaccessible, proceed with validation using only the nucleus source code. The Notion documentation provides additional context but is not required for a valid assessment.

This is mandatory - do NOT rely on assumptions about what rule fields mean. Always verify against the source code.

## Step 4: Translate Rule to Plain English

Using the source code from Step 3, translate the JSON rule into a precise, plain-English description:

1. Start from the outermost rule and work inward
2. For compound rules, clearly state the logical operator:
   - `&` - "users who match ALL of the following"
   - `|` - "users who match ANY of the following"
   - `-` - "users who match the first condition EXCEPT those who match the second"
3. For each leaf rule, describe exactly what it targets based on the source code:
   - Include the field name, operator, and value
   - Translate operators to readable form based on what the source code reveals. Common examples: `eq` = "equals", `gte` = "on or after", `lte` = "on or before", `gt` = "greater than", `lt` = "less than", `in` = "is one of", `not_in` = "is not one of" — but always check the rule model for the full set of supported operators per field
   - For date fields, distinguish between absolute dates (ISO 8601 format like `"2024-01-01T00:00:00Z"`) and relative dates (offset expressions like `"7 days ago"` if supported by the field)
   - For segment rules, name the specific segment labels
4. Present this as a structured breakdown:

```text
Rule Translation:
  [Compound: AND]
    |- Segment equals "member"
    +- Purchase date on or after "2024-01-01"
```

## Step 5: Compare and Validate

Compare the plain-English translation (Step 4) against the original description (Step 1):

Check for these categories of mismatches:

1. **Missing conditions** - The description mentions criteria not present in the rule
2. **Extra conditions** - The rule contains criteria not mentioned in the description
3. **Wrong operators** - The description says "greater than" but the rule uses `lte`, or similar
4. **Wrong values** - The description says "30 days" but the rule uses a different timeframe
5. **Wrong logic** - The description implies AND but the rule uses OR, or vice versa
6. **Wrong segment labels** - The description says "VIP" but the rule targets "daily", etc.
7. **Invalid rule structure** - Fields that don't exist for a given rule type, or invalid values

## Step 6: Report Results

Present the validation result:

**If the rule matches the description:**

```text
Segment Validation: MATCH

Description: "{original description}"

Rule Translation:
  {structured breakdown from Step 4}

Verdict: The JSON rule accurately implements the described segment.
No discrepancies found.
```

**If the rule does NOT match the description:**

```text
Segment Validation: MISMATCH

Description: "{original description}"

Rule Translation:
  {structured breakdown from Step 4}

Discrepancies:
  1. {category}: {explanation}
  2. {category}: {explanation}

Suggested Fix:
  {If possible, show what the corrected JSON rule should look like}
```

**If the rule is structurally invalid:**

```text
Segment Validation: INVALID RULE

Description: "{original description}"

Issues:
  1. {structural problem}
  2. {invalid field or value}

The rule cannot be validated because it contains structural errors.
```

## Rules

1. **Always use Keystone MCP tools** to read rule source code. Do not guess what fields mean or what values are valid.
2. **Be precise about operators.** The difference between `gte` and `gt`, or `eq` and `in`, matters for validation accuracy.
3. **Handle nested compounds.** Rules can be deeply nested - walk the entire tree.
4. **Check date formats.** Validate date values against the field's supported format from source code — ISO 8601 for absolute dates, or relative expressions where explicitly supported by the field. Flag malformed or unsupported formats.
5. **Do not modify any data.** This command is read-only. Never create, update, or delete segments.
6. **When uncertain about a field's meaning**, say so and cite the source file where you looked. Do not present uncertain interpretations as fact.
7. **If Keystone is unavailable**, report that validation could not be completed without Keystone access. Do not attempt to validate rules based on assumptions alone.
