# JSON functions, advanced field editing, and REST calls

Source: `jira-smart-values-json-functions`, `advanced-field-editing-using-json`,
`issue-data-jira-format-payload-for-send-web-request-action`,
`issue-data-automation-format-payload-for-send-web-request`, `convert-usernames-to-account-ids`.

This is the file for the two things the UI editor is worst at: building JSON payloads and parsing responses.

## JSON functions

Examples assume: fix versions `Version 2.0` (id 10046) and `Version 3.0` (id 10047); single-select `Decision` = `Yes`; summary `Hello World`.

### Producing JSON

| Function | Applies to | Example | Produces |
|---|---|---|---|
| `asJsonString` | text values | `{{issue.summary.asJsonString}}` | `"Hello World"` |
| `asJsonStringArray` | lists of text/numbers | `{{issue.fixVersions.name.asJsonStringArray}}` | `["Version 2.0","Version 3.0"]` |
| `asJsonArray` | lists of **numbers only** | `{{issue.fixVersions.id.asJsonArray}}` | `[10046,10047]` |
| `asJsonObject("key")` | text values | `{{issue.summary.asJsonObject("title")}}` | `{ "title": "Hello World" }` |
| `asJsonObjectArray("prop")` | lists of objects | `{{issue.fixVersions.asJsonObjectArray("name")}}` | `[{ "name": "Version 2.0" },{ "name": "Version 3.0" }]` |

`asJsonString` **escapes special characters** — that's its real job, not just adding quotes. Any user-supplied text going into a JSON payload should pass through it.

> `asJsonArray` on a list of *text* renders incorrectly. Numbers only. If you need numbers out of text, transform first:
> `{{issue.fixVersions.name.right(3).asJsonArray}}` → `[2.0,3.0]`

`asJsonObjectArray` cannot rename the key. To rename, chain:

```
{{issue.fixVersions.name.asJsonObject("title").asJsonArray}}
→ [{ "title": "Version 2.0" },{ "title": "Version 3.0" }]
```

### Consuming JSON

| Function | Purpose |
|---|---|
| `jsonStringToObject(str)` | JSON string → object addressable by dot notation |
| `xmlToJson(str)` | XML string → JSON object |
| `list.flatten()` | collapse a nested collection into one dimension |

```
{{jsonStringToObject(webResponse.body)}}         → {value=Hello World!}
{{xmlToJson(webResponse.body).item}}             → Hello World
```

**`jsonStringToObject` limitation:** arrays nested directly inside arrays do not work.

```
✗ {"value": { {"key1":"val1"}, {"key3":"val3"} } }      unsupported
✓ {"value": [ {"key1":"val1"}, {"key3":"val3"} ] }      object-of-arrays is fine
```

## Advanced field editing

Available under **More options** on: Clone work item, Create work item, Create JSM request, Edit work item, Transition work item.

Only reach for it when a field cannot be set through *Choose fields to set* — typically third-party custom fields.

### `update` vs `fields`

```json
{
  "update": {
    "description": { "set": "a new description" },
    "labels": [{ "add": "test-label" }]
  },
  "fields": {
    "summary": "woohoo! a new summary"
  }
}
```

- `fields` is shorthand for `update` + `set` — it **overwrites**.
- `update` supports `set`, `add`, `remove` — use it to append to multi-value fields without destroying existing values.
- **The same field must never appear in both blocks.** That is an error, not a merge.
- At least one of `update` / `fields` must be present, or you get "valid JSON with missing keys".

### Discovering what a field accepts

```
GET /rest/api/3/issue/createmeta?projectKeys=<KEY>&expand=projects.issuetypes.fields
GET /rest/api/3/issue/{issueIdOrKey}/editmeta
```

The response gives each field's `schema`, its permitted `operations` (`set` / `add` / `remove`), and `allowedValues`. If a field is **absent** from that response, the rule will fail at runtime — the field is missing from the relevant create/edit **screen**, or the rule actor lacks permission. Fix the screen, not the JSON.

### Referencing fields

By ID or by name (case-insensitive, spaces may become underscores). Never both for the same field:

```json
{
  "fields": {
    "customfield_10003": "the value I want to set",
    "My Text Customfield": "same field as above — using both causes an error"
  }
}
```

Use the ID when names collide with each other or with a system field.

### Smart values inside the JSON

Always specify the sub-property on user fields — `{{issue.reporter}}` alone is not usable:

```json
{ "fields": { "summary": "Reporter: {{issue.reporter.displayName}}" } }
```

Let the JSON functions build the structure rather than hand-writing quotes:

```json
{ "fields": { "assignee": { "id": "{{initiator.accountId}}" } } }
{ "fields": { "assignee": {{initiator.accountId.asJsonObject("id")}} } }
{ "fields": { "assignee": { "id": {{initiator.accountId.asJsonString}} } } }
```

All three work; the second is preferred — correct shape *and* correct escaping in one step.

Arrays:

```json
{ "fields": { "labels": {{issue.parent.labels.asJsonStringArray}} } }
{ "fields": { "Multi User Customfield": {{issue.parent.Multi User Customfield.accountId.asJsonObjectArray("id")}} } }
```

> **GDPR:** user fields take `{"id": "<accountId>"}`. `name` / `key` no longer resolve. See `convert-usernames-to-account-ids` for bulk migration of old rules.

### Field syntax by type

```jsonc
// System fields
"summary":  "A summary is one line of text"
"description": "many lines\n separated by\n line feeds"
"duedate":  "2015-11-18"                                  // YYYY-MM-DD
"labels":   ["label1", "label2"]
"components": [{ "name": "Active Directory" }, { "name": "Network Switch" }]
"fixVersions": [{ "name": "2.0" }]
"versions": [{ "name": "Version 1.0" }]                   // Affects Version/s

// Custom fields
"customfield_11444": 664                                   // Number
"customfield_11441": "2015-11-18"                          // Date picker
"customfield_11442": "2015-11-18T14:39:00.000+1100"        // Date time picker (ISO 8601)
"customfield_11450": "a single line of text"               // Text
"customfield_11452": "http://www.atlassian.com"            // URL
"customfield_11443": ["rest_label1", "rest_label2"]        // Labels
"customfield_11445": { "value": "option2" }                // Radio  (or { "id": 10112 })
"customfield_11449": { "value": "option3" }                // Single select
"customfield_11448": [{ "value": "option1" }, { "value": "option2" }]  // Multi select
"customfield_11440": [{ "value": "option1" }, { "value": "option2" }]  // Checkboxes
"customfield_11447": { "value": "parent_option1", "child": { "value": "p1_child1" } }  // Cascading
"customfield_11453": { "id": "2s1863211f0z284c45269423" }  // Single user picker
"customfield_11458": [{ "id": "..." }, { "id": "..." }]    // Multi user picker
"customfield_10700": ["10300", "10400"]                    // Elements Connect
```

Every option-type field accepts `{"id": …}` instead of `{"value": …}` — IDs are stable, names are not.

### Common operations

Append a label without clobbering existing ones:

```json
{ "update": { "labels": [ { "add": "my-new-label" } ] } }
```

Append to a multi-select from another field:

```json
{ "update": { "customfield_12345": [ { "add": { "value": "{{issue.customfield_12224}}" } } ] } }
```

Append a user to a multi-user picker:

```json
{ "update": { "customfield_12346": [ { "add": { "id": "{{reporter.accountId}}" } } ] } }
```

Set security level:

```json
{ "update": { "security": [ { "set": { "name": "Public" } } ] } }
```

Set a single-select during an edit (note the array wrapper under `update`):

```json
{ "update": { "Single Select": [ { "set": { "value": "green" } } ] } }
```

Link work items — look up link type names at `<site>/secure/admin/ViewLinkTypes!default.jspa`:

```json
{
  "update": {
    "issuelinks": [
      { "add": { "type": { "name": "Relates" }, "outwardIssue": { "key": "{{issue.key}}" } } }
    ]
  }
}
```

JSM request participants:

```json
{ "update": { "Request participants": [ { "add": { "id": "{{issue.comments.last.author.accountId}}" } } ] } }
```

Log work, optionally with a calculated duration:

```json
{ "update": { "worklog": [ { "add": { "timeSpent": "6m" } } ] } }
{ "update": { "worklog": [ { "add": { "timeSpent": "{{now.diff(issue.created).businessDays}}d" } } ] } }
```

Log work and reset the estimate together:

```json
{
  "update": { "worklog": [ { "add": { "timeSpent": "6m" } } ] },
  "fields": { "timetracking": { "originalEstimate": "10", "remainingEstimate": "5" } }
}
```

### Why edits silently do nothing

In order of likelihood:

1. **Stale `{{issue}}`** — an earlier step changed the value but `{{issue}}` still holds trigger-time data. Insert **Re-fetch work item data**.
2. **Field copying from itself while empty** — produces no change. Guard with a default or use a different source field.
3. **Rule actor lacks permission** to view or edit the field in that project.
4. **Field not on the create/edit screen** — confirm via `createmeta` / `editmeta`.
5. The smart value resolved to empty and you wrote an empty string over real data.

The editor warning *"Valid JSON but smart value can't be validated"* is **expected** — smart values can only resolve at runtime. Read the audit log after the first execution.

## Send web request

- Smart values are supported in URL, headers, and body.
- Enable **Wait for response** to get `{{webResponse}}` (see smart-values-core.md for the `webResponse` naming caveat).
- A **checkbox decides whether a non-2xx response fails the rule or lets execution continue.** Continue when a later branch reads `{{webResponse.status}}` and alerts on it; fail when the audit log and the rule owner's error notification are where the failure should land. Note the branch has to sit *after* the request but the status has to exist *before* the branch step, so a request whose failure you want to alert on belongs in the trunk.
- Calls back to your own Jira site are **not implicitly authenticated** — supply `Authorization: Basic <base64 of email:apiToken>` yourself.
- Global admins can restrict which domains this action may reach.
- Outbound traffic comes from Atlassian IP ranges; the target must allow them.

Payload shape is configurable — `issue-data-jira-format-payload-for-send-web-request-action` documents the Jira-format body, `issue-data-automation-format-payload-for-send-web-request` the automation format.

### Reading a response

```
{{webResponse.status}}
{{webResponse.body.values.id}}
{{webResponse.body.size}}
{{#webResponses}}{{someValue.childValue}}{{/}}
```

Paged Jira REST responses (`{size, start, limit, values[]}`) work well with `.size` for existence checks and `{{…values.id}}` to flatten a list of IDs for a "does not contain" comparison.

### Guarding a call

The **Advanced compare condition** is where response-based branching happens: first value `{{webResponse.body.size}}`, condition `equals`, second value `0`. Flattened list membership works the same way with *contains* / *does not contain*.

### Getting a JSM customer's email address

`{{issue.reporter.emailAddress}}` renders **empty** for JSM customer accounts, whoever the actor is — verified with both the *Automation for Jira* default actor and a full org admin. Email visibility is a per-account profile setting, and for customers on domains your org has not claimed, nobody on your side can change it. No actor, permission or scope fixes the smart value.

Fetch it with a web request instead:

```
GET https://api.atlassian.com/ex/jira/<cloudId>/rest/servicedeskapi/servicedesk/<projectKey>/customer?query={{issue.reporter.accountId.urlEncode}}
Accept: application/json
X-ExperimentalApi: opt-in
```

```
{{webResponse.body.values.first.emailAddress}}
```

- Scope: `manage:servicedesk-customer`. The endpoint is flagged **Experimental**, hence the header.
- `emailAddress` is in its **documented** response schema, unlike the other routes below.
- The service desk is addressed by **project key**; a numeric project ID returns "service space does not exist".
- Note the paged envelope — `.values.first`, not `.first`.
- Matching an `accountId` through `query` is undocumented (the docs describe `query` as matching name and email). The same behaviour holds on `/rest/api/3/user/search?query=<accountId>`, which suggests a shared search backend rather than a fluke.

Three sibling routes return the user **without** `emailAddress`, all verified on the same token and account: `/rest/api/3/user?accountId=`, `/rest/api/3/user/bulk?accountId=`, and `/rest/api/3/user/search?accountId=`. Only the `query=` form carries it. `/rest/api/3/user/email` is the documented endpoint for this and needs a scope most tokens lack; it is also limited to managed accounts, so it will not help for external customers.
