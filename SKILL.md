---
name: jira-cloud-automation
description: Use when building, reviewing, or debugging Jira Cloud automation rules (now called "flows") in the UI editor or through the automation REST API — writing smart values, manipulating text/lists/numbers/dates, building JSON for advanced field editing or Send web request, wiring Lookup issues / Create variable / branches, or when a rule silently produces an empty value, a wrong type, or stale issue data.
---

# Jira Cloud automation — smart values & data manipulation

## Overview

Look-up skill for authoring Jira Cloud automation in the **UI rule editor**, or as rule JSON written through the REST API. Its job is to hand over exact, copy-pasteable smart values and component shapes instead of plausible-looking ones.

Core principle: **look the syntax up, don't recall it.** Smart-value function names are drawn from three different Java libraries (`String`, `StringUtils`, Joda-style date helpers) and are not guessable — `substringBeforeLast` exists, `substringBeforeFirst` does not; `asJsonStringArray` and `asJsonArray` mean different things and silently misrender if swapped.

Docs base: `https://support.atlassian.com/cloud-automation/docs/<slug>/`

## Which reference file

| Question | File |
|---|---|
| Syntax rules, root objects (`{{issue}}`, `{{lookupIssues}}`, `{{webResponse}}`, `{{changelog}}`…), iteration, defaults, variables, how to test a value | [references/smart-values-core.md](references/smart-values-core.md) |
| Text functions (substring, replaceAll, match, split, pad, encode) and list functions (join, sum, distinct, first/last, size) | [references/text-and-lists.md](references/text-and-lists.md) |
| Math expressions `{{#=}}`, numeric comparisons, formatting/currency, all date & time functions | [references/numbers-and-dates.md](references/numbers-and-dates.md) |
| JSON functions, **advanced field editing** (`update` vs `fields`, per-field-type JSON), Send web request payloads, parsing responses | [references/json-and-rest.md](references/json-and-rest.md) |
| Attachment metadata, reading file contents (`.eml`, CSV…), attachment triggers/conditions/actions | [references/attachments.md](references/attachments.md) |
| AI steps: **Use Rovo** vs **Use Rovo agent**, `{{rovoResponse}}` / `{{agentResponse}}`, parsing a JSON answer, credit consumption | [references/rovo.md](references/rovo.md) |
| Triggers/conditions/actions/branches, advanced steps (Loop, Delay until), service limits, permissions, debugging | [references/rule-structure.md](references/rule-structure.md) |
| Worked end-to-end recipes assembled from the above | [references/recipes.md](references/recipes.md) |
| Creating, updating and diffing rules through the **REST API**, the JSON of each trigger, condition, If/else and action, keeping webhook secrets out of scripts | [references/rest-api.md](references/rest-api.md) |

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
- **Branches do not nest inside branches, and `If/else` caps at 2 layers.** One decision point per rule; split at a field change and chain rules when you need more. Branch conditions evaluate when the branch step is reached, so **gather in the trunk and fork once** — a value computed inside a branch cannot be branched on. A bare condition inside a branch stops only the rest of that branch and cannot alert, which makes bare conditions your silent stops and branches your alerts. See [rule-structure.md](references/rule-structure.md).
- **The fallback branch takes no conditions** — the UI says "Runs when no conditions are met." It is the one unconditional path out of a `Branch with conditions`, so every case that must *not* reach it has to be caught by one of the three conditional branches, including the cases that should end silently. Give those a branch with a Log action and nothing else, and combine them with OR execution mode to stay inside the three-branch budget.
- **Variables die at the group boundary; work item fields do not.** [AUTO-154](https://jira.atlassian.com/browse/AUTO-154) (155 votes, filed as a Suggestion, still open). For anything that has to cross a group, branch or rule, write it to a field, **Re-fetch work item data**, then read `{{issue.…}}`. An Assets object field can be set straight from an AQL query, replacing Lookup objects — and it is multi-value, so branch on `.size`.
- **`{{webResponse}}` dies at that boundary too, and it takes your error handling with it.** A request in the trunk populates it; a branch below reads `EMPTY`. Since `EMPTY does not equal 200` is **true**, a branch guarding on HTTP status fires on every run and reports a failure that never happened, with nothing in the audit log to say a smart value went missing. **A branch condition may only read `{{issue.…}}`** — write the response to a field first. One text field carries several facts: `{{webResponse.status}}:{{webResponse.body.size}}` yields `200:0`, `200:1`, `401:`, all reachable with `contains` and `equals`.
- **A failed condition stops its flow group, not the rule.** Documented in one sentence on *Use automation steps in a flow*, and flatly contradicted by *Add conditions to an automation flow*. The group-scoped behaviour is what happens once advanced steps are in play.
- **`{{issue.reporter.emailAddress}}` renders empty for JSM customer accounts**, under any actor — tested with both the *Automation for Jira* default and a full org admin. Customer email visibility is a per-account profile setting you cannot change for customers on domains you do not own. Fetch it instead from `GET /rest/servicedeskapi/servicedesk/<projectKey>/customer?query=<accountId>` with `X-ExperimentalApi: opt-in` and the `manage:servicedesk-customer` scope; `emailAddress` is in that endpoint's documented response.
- **Send web request has a checkbox deciding whether a non-2xx response fails the rule or continues.** Set it to continue when a branch needs to read `{{webResponse.status}}` and alert on it; set it to fail when the audit log and the owner's error notification are the intended destination.
- **The web-request response is `{{webResponse}}`. `{{webhookResponse}}` is dead.** The old name was an accepted alias until recently and now renders empty. That failure is quiet: `{{webhookResponse.status}}` returns nothing rather than erroring, so an `equals 200` check just fails and the rule takes its error branch as though the request had failed. Any rule written before this needs every reference renamed. Both spellings were logged returning `200` on 2026-08-10; on 2026-08-21 only `{{webResponse}}` populated, so the alias was retired between those dates.
- **Backslashes die in the smart-value parser.** `\n`, `\t`, `\R`, `\s`, `\w`, `\d` all return empty inside `match()` / `replaceAll()`, and doubling doesn't help. Use `[0-9]`, `[A-Za-z0-9_]`, and `(?m)^` anchoring instead of naming a newline. See [text-and-lists.md](references/text-and-lists.md).
- **Calling your own site's REST API needs the gateway URL if the token is scoped.** Service accounts can only make scoped tokens, and those authenticate solely against `https://api.atlassian.com/ex/jira/<cloudId>/…`. Worse, endpoints that allow anonymous access return a *permission* error rather than 401 when auth fails — canary with `/rest/api/3/myself`. See [attachments.md](references/attachments.md).
- **`setTimeZone` ≠ `convertToTimeZone`.** `setTimeZone` relabels the same wall-clock time with a new zone; `convertToTimeZone` actually shifts it. Picking the wrong one is an off-by-hours bug that survives review.
- **`{{fieldChange}}` holds only the *first* changed field.** For multi-value changes iterate the changelog: `{{#changelog.fixVersion}}{{toString}}{{/}}`.
- **`{{comment}}` only exists under the Issue commented trigger** — not when a comment happens to be added by another trigger.
- **`{{attachment.content}}` is a download URL, not file contents** — despite the docs saying "returns the content of the attachment, with binary content rendered as a String". Automation cannot read a file; you must fetch it with Send web request. See [attachments.md](references/attachments.md).
- **Setting Organizations has a built-in domain option.** Create / Edit / Transition work item offer **"Use reporter's email domain"** on the Organizations field — no web request needed. It reads each organization's *Manage email domains* list.
- **Lookup issues caps at 100 work items**, silently. Anything past 100 is dropped with no audit-log warning.
- **A missing value renders as empty, not as an error.** Guard with the default pipe: `{{some.value|"fallback"}}`.
- **`htmlEncode` should be used in function form** — `{{htmlEncode(issue.summary)}}`. The inline form `{{webhookData.htmlEncode}}` will return a *property named* `htmlEncode` if the payload happens to have one.
- **Advanced steps are Premium/Enterprise only** (Branch with conditions, Delay until, Loop). Loop repeats a maximum of 3 times and its conditions can only see trigger context + declared loop inputs.
- **There are two AI steps and they populate different smart values.** *Use Rovo* → `{{rovoResponse}}`; *Use Rovo agent* → `{{agentResponse}}`. Crossing them renders empty. Text functions applied directly to either return blank — store the answer with **Create variable** first. See [rovo.md](references/rovo.md).

## Testing any smart value

Fastest loop, worth using before writing anything complex:

1. Trigger: **Manual trigger from work item**
2. Action: **Log action** containing the candidate smart value
3. Run it from a real work item → read the result in the audit log

This is the only way to validate a smart value — the editor cannot validate them before execution, and "valid JSON but smart value can't be validated" is an expected warning, not an error.
