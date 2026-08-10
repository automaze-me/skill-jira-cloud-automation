---
name: jira-cloud-automation
description: Use when building, reviewing, or debugging Jira Cloud automation rules (now called "flows") in the UI editor — writing smart values, manipulating text/lists/numbers/dates, building JSON for advanced field editing or Send web request, wiring Lookup issues / Create variable / branches, or when a rule silently produces an empty value, a wrong type, or stale issue data.
---

# Jira Cloud automation — smart values & data manipulation

## Overview

Look-up skill for authoring Jira Cloud automation in the **UI rule editor**. Its job is to hand over exact, copy-pasteable smart values instead of plausible-looking ones.

Core principle: **look the syntax up, don't recall it.** Smart-value function names are drawn from three different Java libraries (`String`, `StringUtils`, Joda-style date helpers) and are not guessable — `substringBeforeLast` exists, `substringBeforeFirst` does not; `asJsonStringArray` and `asJsonArray` mean different things and silently misrender if swapped.

Docs base: `https://support.atlassian.com/cloud-automation/docs/<slug>/`

## Which reference file

| Question | File |
|---|---|
| Syntax rules, root objects (`{{issue}}`, `{{lookupIssues}}`, `{{webhookResponse}}`, `{{changelog}}`…), iteration, defaults, variables, how to test a value | [references/smart-values-core.md](references/smart-values-core.md) |
| Text functions (substring, replaceAll, match, split, pad, encode) and list functions (join, sum, distinct, first/last, size) | [references/text-and-lists.md](references/text-and-lists.md) |
| Math expressions `{{#=}}`, numeric comparisons, formatting/currency, all date & time functions | [references/numbers-and-dates.md](references/numbers-and-dates.md) |
| JSON functions, **advanced field editing** (`update` vs `fields`, per-field-type JSON), Send web request payloads, parsing responses | [references/json-and-rest.md](references/json-and-rest.md) |
| Triggers/conditions/actions/branches, advanced steps (Loop, Delay until), service limits, permissions, debugging | [references/rule-structure.md](references/rule-structure.md) |
| Worked end-to-end recipes assembled from the above | [references/recipes.md](references/recipes.md) |

## Terminology: the 2026 rename

Atlassian renamed the vocabulary in the UI and docs, but **not in the smart values or the URLs**. Both sets are live, so read and write accordingly:

| Docs/UI now say | Smart values / REST still say |
|---|---|
| flow | rule (`{{flow.name}}` is the exception — it replaced `{{rule.name}}`) |
| step | component / action |
| work item | issue (`{{issue}}`, `{{lookupIssues}}`, `{{triggerIssue}}`) |
| Jira space | project (`{{project}}`) |

Never "correct" `{{issue.summary}}` to `{{workItem.summary}}` — that value does not exist.

## Facts that are commonly guessed wrong

- **`{{issue}}` goes stale mid-run.** It holds the values from when the rule triggered and is *not* refreshed by your own edits. Insert the **Re-fetch work item data** action before any step that must see a value an earlier step just wrote.
- **Math operators need a wrapper.** `{{issue.Amount * 1.2}}` does nothing. Use `{{#=}}{{issue.Amount}} * 1.2{{/}}`. Only the method forms (`.plus()`, `.divide()`, `.abs`) work bare.
- **`Create variable` always returns a string** — even when you feed it a list. `{{issue.watchers.distinct}}` stored in a variable comes back as one rendered string, not an iterable list.
- **`asJsonArray` is numbers-only.** Text lists silently render wrong; use `asJsonStringArray` for text. See [json-and-rest.md](references/json-and-rest.md).
- **Never put the same field in both `update` and `fields`** in advanced field editing — `fields` is just shorthand for `update` + `set`, and using both errors.
- **User fields need `{"id": "<accountId>"}`**, never `name` — GDPR change. `{{initiator.accountId.asJsonObject("id")}}` builds it correctly and escapes it.
- **`{{webhookResponse}}` vs `{{webResponse}}`**: Atlassian's own docs use both spellings on different pages. The issues reference documents `{{webhookResponse.status/.headers/.body}}`; the JSON-functions and text-fields pages use `jsonStringToObject(webResponse.body)`. Confirm which one your instance emits with a **Log action** before building on it.
- **`setTimeZone` ≠ `convertToTimeZone`.** `setTimeZone` relabels the same wall-clock time with a new zone; `convertToTimeZone` actually shifts it. Picking the wrong one is an off-by-hours bug that survives review.
- **`{{fieldChange}}` holds only the *first* changed field.** For multi-value changes iterate the changelog: `{{#changelog.fixVersion}}{{toString}}{{/}}`.
- **`{{comment}}` only exists under the Issue commented trigger** — not when a comment happens to be added by another trigger.
- **Lookup issues caps at 100 work items**, silently. Anything past 100 is dropped with no audit-log warning.
- **A missing value renders as empty, not as an error.** Guard with the default pipe: `{{some.value|"fallback"}}`.
- **`htmlEncode` should be used in function form** — `{{htmlEncode(issue.summary)}}`. The inline form `{{webhookData.htmlEncode}}` will return a *property named* `htmlEncode` if the payload happens to have one.
- **Advanced steps are Premium/Enterprise only** (Branch with conditions, Delay until, Loop). Loop repeats a maximum of 3 times and its conditions can only see trigger context + declared loop inputs.

## Testing any smart value

Fastest loop, worth using before writing anything complex:

1. Trigger: **Manual trigger from work item**
2. Action: **Log action** containing the candidate smart value
3. Run it from a real work item → read the result in the audit log

This is the only way to validate a smart value — the editor cannot validate them before execution, and "valid JSON but smart value can't be validated" is an expected warning, not an error.
