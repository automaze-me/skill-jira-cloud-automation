# Smart values — core syntax and root objects

Source: `what-are-smart-values`, `jira-smart-values-issues`, `jira-smart-values-users`,
`jira-smart-values-projects`, `formatting-smart-values-in-jira-automation`,
`find-the-smart-value-for-a-field`, `pass-data-between-actions-in-a-flow`.

## Syntax

Dot notation inside double mustaches. Multi-word properties are camelCase.

```
{{issue}}                       top-level object
{{issue.summary}}               property
{{issue.assignee.displayName}}  sub-property
{{now.plusDays(5)}}             function — always has parentheses
```

Read them backwards to sanity-check: `{{page.parent.title}}` = "the title of the parent of the page".

### Chaining

Transformations compose left to right in a single expression:

```
{{issue.summary.toLowerCase().substring(0, 10).concat("!!")}}
```

### Default values

A missing field renders as **empty string**, never as an error. Supply a fallback with `|`:

```
{{invalid.reference|"Hello world"}}
{{issue.assignee.displayName|"Unassigned"}}
```

Use this liberally — most "my rule posted a half-empty comment" bugs are an unguarded missing value.

### Sections: iterating lists

Prefix with `#`, close with `{{/}}`. Inside the block you address properties of the current item directly, and `.` is shorthand for the item itself.

```
{{#issue.comments}}Author: {{author.displayName}}
{{/}}

{{#issue.labels}}{{.}}, {{/}}
```

### Math sections

`{{#=}}...{{/}}` switches to arithmetic mode. Required for bare operators:

```
{{#=}}{{issue.Invoice Amount}} + 100{{/}}
```

## Finding the right value for a field

Three routes, in order of reliability:

1. **The `{}` panel in the editor.** Only shows values valid for the current trigger and steps. Has a *Custom fields* tab (scope-specific) and an *Advanced formatting* tab (functions).
2. **Log action + manual trigger** — see SKILL.md. Definitive.
3. **The REST API** — `GET /rest/api/3/issue/{key}?expand=names` shows the actual field structure; whatever key path exists in that JSON is broadly what the smart value path looks like.

## Custom fields

Address by **name** or by **ID**:

```
{{issue.Team Leader}}              → Alana Grant
{{issue.customfield_10003}}
```

Rules for name access:
- case-insensitive
- spaces may be replaced with underscores (`{{issue.Team_Leader}}`)
- **must** use the ID when two custom fields share a name, or when a custom field collides with a system field name

Sub-properties available on any custom field: `.id`, `.name` (visible label), `.description`, and for option-type fields `.value`.

## Root objects

### `{{issue}}` / `{{triggerIssue}}`

`{{issue}}` = the work item in the current context (inside a branch, that's the *branched* item).
`{{triggerIssue}}` = the item that originally triggered the rule. Same property set. Use it inside branches when you need to refer back.

Notable sub-objects: `.comments`, `.components`, `.fixVersions`, `.versions`, `.watchers`, `.parent`, `.epic`, `.security`, `.timetracking`, `.properties`, `.url`, `.affectedServices`, `.Request Type` (JSM), `.latest`.

### `{{lookupIssues}}`

Result list from the **Lookup issues** action (JQL, **max 100 items**).

```
{{#lookupIssues}}* {{key}} — {{summary}}
{{/}}

There are {{lookupIssues.Story Points.sum}} story points remaining.
```

Documented per-item properties: `key`, `url`, `summary`, `assignee.displayName`, `reporter.displayName`, `status`, `issueType.name`, `resolution`, `fixVersion`, `created`, `updated`, `Story Points` (company-managed), `Story point estimate` (team-managed). Any other field or custom field is also reachable.

> If a rule uses more than one Lookup issues action, the UI names every result `lookupIssues` and later ones overwrite earlier ones. Capture each into a **Create variable** immediately after its lookup (remembering the string caveat), or restructure into separate rules.

### `{{createdIssue}}` / `{{createdIssues}}`

The item(s) created earlier in this rule. Full `{{issue}}` property set.

### `{{changelog}}`

Available on any trigger that edits an item. Keyed by field:

```
{{#changelog.status}}{{fromString}}{{/}}    previous status, as displayed
{{#changelog.status}}{{from}}{{/}}          previous status ID — use this in an Edit action
{{#changelog.summary}}{{toString}}{{/}}     new summary
{{#changelog.issuetype}}{{fromString}}{{/}} previous issue type
```

`fromString`/`toString` = display values. `from`/`to` = raw JSON values (IDs). Feed IDs, not display strings, back into edits.

### `{{fieldChange}}`

The changed field, anywhere smart values are supported.

| Property | Returns |
|---|---|
| `fieldChange.fromString` / `.toString` | previous / new value as displayed |
| `fieldChange.from` / `.to` | previous / new raw JSON value |
| `addedFieldChange.field` / `.fieldId` / `.fieldType` | field changed by an addition |
| `addedFieldChange.values` / `.valueIds` | values added |
| `deletedFieldChange.field` / `.fieldId` / `.fieldType` | field changed by a deletion |
| `deletedFieldChange.values` / `.valueIds` | values deleted |

**Only the first changed field is present.** For several changed values iterate the changelog instead.

### `{{comment}}`

**Issue commented trigger only.** Silently empty under every other trigger, including transitions that happen to add a comment. Under that trigger it stays readable inside a branch, unlike a variable (run, 2026-11).

`comment.body`, `.author`, `.created`, `.internal` (false = customer-visible in JSM; a condition `{{comment.internal}}` equals `false` under *Work item commented* passed a public agent comment and stopped an internal one, run 2026-11), `.visibility.type`, `.visibility.value`, `.properties`, `.properties."sd.public.comment".internal`.

### `{{webhookData}}`

Body and payload of the **Incoming webhook** trigger. Access with dot notation: `{{webhookData.issue.key}}`. If it arrives as a JSON *string* rather than an object, run it through `jsonStringToObject()` — see json-and-rest.md.

### `{{webResponse}}`

Response from **Send web request** with *Wait for response* enabled.

```
{{webResponse.status}}                    e.g. 200
{{webResponse.headers.Content-Type}}
{{webResponse.body}}
{{webResponse.body.name}}
{{webResponses.last.body}}                when the action ran over a list
{{#webResponses}}{{someValue.childValue}}{{/}}
```

> **Naming: `{{webhookResponse}}` no longer populates.** Atlassian's docs carried both spellings for a long time, and logging `{{webhookResponse.status}}` beside `{{webResponse.status}}` returned `200` for both on 2026-08-10. On 2026-08-21, on a live tenant, the old name rendered empty while `{{webResponse}}` worked. Use `{{webResponse}}` and nothing else.
>
> The retirement is dangerous precisely because it is silent. A missing value renders as empty rather than raising, so `{{webhookResponse.status}}` **equals** `200` simply evaluates false, and a rule that branches on that check runs its failure path on every execution while reporting no error of its own. Rules authored before the change do not break loudly; they start lying.
>
> The list form above is written `{{webResponses}}` by extension from the singular. That extension has not been logged since the rename — confirm it with a **Log action** before relying on it in a rule that iterates.

### `{{flow}}`

`{{flow.name}}` — name of the running rule. `{{flow.actor}}` — the rule actor user object.

### Users

Every user-valued smart value exposes: `accountId`, `active`, `displayName`, `emailAddress`, `timeZone`, `avatarUrls` (`{{assignee.avatarUrls."48x48"}}`).

Named user roots: `{{assignee}}`, `{{reporter}}`, `{{creator}}` (immutable), `{{initiator}}` (whoever triggered the rule), `{{comment.author}}`.

> `emailAddress` returns **empty** when the user's profile visibility for email is "Only you and admins". This is the single most common cause of a blank email in a rule that works fine for the author.

### Other roots

`{{baseUrl}}`, `{{attachment}}`, `{{worklog}}`, `{{approval}}` (JSM), `{{eventType}}`, `{{comment.id}}`, `{{issue.InsightField}}` (Assets), `{{issue.TempoAccountField}}`.

## Create variable

Defines a reusable smart value for later steps in the same rule. Can contain other smart values and math.

**It always returns a string.** A list stored in a variable is rendered flat — you cannot iterate it afterwards with `{{#myVar}}`. Where you need a real list downstream, re-derive it rather than storing it.

## Passing entities between actions

Some actions emit a typed entity (e.g. Start Jira sprint → a sprint) that a later action consumes (Add work item to sprint). Reference the **root** smart value of the producing action, not a child field — the platform extracts the identifier itself.

Two inline errors you will hit:
- *"Enter a valid smart value…"* → you passed plain text or drilled into a child field. Pass the root.
- *"This smart value doesn't contain the specified entity."* → right syntax, wrong entity type. Pick a value from an action that produces the type this field wants.

Validate with **More actions → Check for errors** before saving.
