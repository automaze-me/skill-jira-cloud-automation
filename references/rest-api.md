# Rules through the REST API, and the component JSON

Jira Cloud has a public REST API that lists, reads, creates, updates and deletes automation rules. Rules can live as JSON in a repository and be written to a site by script, with the editor kept for review. Everything below was verified against live Cloud sites between August and October 2026 unless marked otherwise.

## The API

Base URL: `https://api.atlassian.com/automation/public/jira/<cloudId>/rest/v1`. Basic auth with an account email and a classic API token works. Spec: `https://dac-static.atlassian.com/cloud/automation/swagger.v3.json` (version 1.0.22 when checked).

| Call | What it does, and what bites |
|---|---|
| `POST /rule/summary` | Lists rules. Body `{"limit": 100, "scope": "ari:cloud:jira:<cloudId>:project/<projectId>"}`; answers `data[]` with `uuid`, `name`, `state`, `ruleScopeARIs`. |
| `GET /rule/<uuid>` | The whole rule under `rule`: `name`, `state`, `actor`, `authorAccountId`, `trigger`, `components`, `ruleScopeARIs`, `notifyOnError`, `writeAccessType`, `canOtherRuleTrigger`, `labels`, `collaborators`. |
| `POST /rule` | Creates a rule from `{"rule": {...}}` in the format GET returns. **Needs `authorAccountId`** (the caller's, from `/rest/api/3/myself`); without it the answer is a bare 400 "could not be parsed" with no hint. |
| `PUT /rule/<uuid>` | Replaces the rule. Send the live trigger back unchanged and it survives, including an incoming webhook's URL and secret. Set `state` here too. |
| `DELETE /rule/<uuid>` | Answers **415** unless the request carries `Content-Type: application/json`; send `{}` as the body. |
| `PUT /rule/<uuid>/state` | Switches a rule on or off without touching the rest: `{"value": "ENABLED"}` or `"DISABLED"`. |
| `POST /rule/manual/<ruleId>/invocation` | Runs a manual-trigger rule, which makes manual rules testable by script: `{"objects": ["<work item ARI>", ...], "userInputs": {"<input>": {"inputType": "TEXT", "value": "..."}}}`. Up to 50 objects, all from one space. Input types: `TEXT`, `PARAGRAPH`, `DROPDOWN`, `NUMBER`, `BOOLEAN`. |
| `POST /rule/manual/search` | Lists the manual rules available for an object. |

There is **no audit log endpoint**: why a run failed is visible only in the rule's audit log in the editor, or in the owner's error mail.

- **The JSON is the export format.** What `GET /rule` returns is what the editor's export produces. On create, leave out `uuid`, `created`, `updated` and the component ids.
- **Comparing a rule with its JSON:** drop `id`, `parentId`, `conditionParentId` and `connectionId` on both sides, which the API adds or changes. In jq: `walk(if type == "object" then del(.id, .parentId, .conditionParentId, .connectionId) else . end)`.
- **`GET` hands out the webhook secret.** An incoming-webhook trigger comes back with its `webhookToken` in clear text. Filter it out of anything you print, log or commit.
- **Keep secrets out of your tooling.** Create a webhook rule in the editor with only its trigger, so the editor generates URL and secret and a person stores them. Then `PUT` the steps onto it, echoing the trigger as read.
- **Create rules disabled** (`"state": "DISABLED"`) and let a person switch them on after reviewing them in the editor.
- **A rule across several spaces** takes several project ARIs in `ruleScopeARIs`. Jira then sets the trigger's `eventFilters` to every space in the scope, whatever you sent, and keeps its own order. So the trigger listens in all of them, and a comparison has to treat `eventFilters` as a set.
- **Error answers are thin.** A rule with only a trigger gets 400 "Automation rule must contain at least one valid condition or action"; one with only conditions, 400 "...at least one valid action". A malformed component usually gets a **500 with an empty body**. Find it by bisecting: post disabled throwaway rules with the first *n* components, delete each one created, and the first *n* that fails names the culprit.
- **Editor deep link** (seen on a service space): `https://<site>.atlassian.net/jira/servicedesk/projects/<KEY>/settings/automate#/rule/<uuid>`. The path segment is `automate`, not `automation`. An optional `/<componentId>` at the end selects a step.

## Finding a component's JSON: the probe rule

Atlassian documents the endpoints but not the shape of each component's `value`, and a wrong guess gets the same opaque "could not be parsed". When a component is missing from the catalogue below, have someone build a throwaway rule in the editor holding that component with dummy values, saved **disabled**. Read it with `GET /rule/<uuid>`, copy the shapes, delete the probe. That takes minutes; guessing takes hours.

## Component catalogue

`component` is `TRIGGER`, `CONDITION`, `CONDITION_BLOCK`, `BRANCH` or `ACTION`. A branch sent as `ACTION` gets the empty 500. Shapes are trimmed to the keys that matter; send `schemaVersion` as shown.

### Triggers

| `type` | `schemaVersion` | `value` |
|---|---|---|
| `jira.manual.trigger.issue` | 3 | `{"inputFromUsers": true, "inputPrompts": [{"inputType": "DROPDOWN", "displayName": "who", "required": true, "variableName": "varWho", "defaultValue": ["A", "B"]}, {"inputType": "TEXT", "displayName": "email", "required": true, "variableName": "varEmail", "defaultValue": ""}], "groups": [], "issueFilter": null, "jQLFilter": null}`. A dropdown's options sit in `defaultValue`. Read inputs as `{{userInputs.varWho}}`. The editor offers short text, paragraph, dropdown, number and checkbox; the spec's input types are `TEXT`, `PARAGRAPH`, `DROPDOWN`, `NUMBER` and `BOOLEAN`, and `TEXT` and `DROPDOWN` have been read back from a live rule. |
| `jira.incoming.webhook` | — | Keys `jql`, `searchOrProvide`, `webhookToken`. `searchOrProvide: "none"` is the editor's "No work items from the webhook"; `"provided"` expects work items in the POST body. Payload fields read as `{{webhookData.<key>}}`. |
| `jira.jql.scheduled` | 1 | `{"schedule": {"method": "BASIC", "rRule": "BEGIN:VEVENT\nDTSTART;TZID=America/Los_Angeles:20261027T060000\nRRULE:FREQ=DAILY;BYHOUR=6;BYMINUTE=0;BYSECOND=0;INTERVAL=1\nEND:VEVENT\n", "cronExpression": "", "rate": 0, "rateInterval": 86400}, "jql": "...", "executionMode": "jql", "onlyUpdatedIssues": false}`. The time zone lives in the `TZID` of the RRULE. |
| `jira.issue.field.changed` | 2 | `{"eventFilters": ["ari:cloud:jira:<cloudId>:project/<projectId>"], "changeType": "VALUE_ADDED", "fields": [{"type": "fieldName", "value": "<field name>"}], "actions": []}` |
| `jira.issue.event.trigger:commented` | 1 | `{"eventTypes": ["PRIMARY_ACTION"], "eventFilters": ["ari:cloud:jira:<cloudId>:project/<projectId>"]}` |

### Conditions and If/else

| `type` | `schemaVersion` | `value` |
|---|---|---|
| `jira.comparator.condition` | 1 | `{"first": "{{lookupIssues.size\|0}}", "second": "0", "operator": "EQUALS"}` (smart values condition) |
| `jira.jql.condition` | 1 | the JQL as a plain string |
| `jira.issue.condition` | 3 | `{"selectedField": {"type": "ID", "value": "status"}, "selectedFieldType": "status", "comparison": "NOT_ONE_OF", "compareValue": {"type": "NAME", "value": "[\"Open\",\"Closed\"]", "multiValue": true}}`; the list is a JSON-encoded string |
| `jira.user.condition` | 6 | `{"conditions": [{"field": "initiator", "check": "NOT_IN_ROLE", "criteria": [{"type": "NAME", "value": "Service Desk Team"}]}], "operator": "OR"}` |

**If/else** is one `CONDITION` of type `jira.condition.container.block` with `value: {}`. Its `children` are `CONDITION_BLOCK`s of type `jira.condition.if.block`, `value: {"conditionMatchType": "ALL"}`, each holding its conditions in `conditions` and its actions in `children`. The **else** is one more `jira.condition.if.block` whose `conditions` list is empty.

### Actions

| `type` | `schemaVersion` | `value` |
|---|---|---|
| `codebarrel.action.log` | 1 | the message as a plain string |
| `jira.create.variable` | 1 | `{"id": "_customsmartvalue_id_1793400000101", "name": {"type": "FREE", "value": "myVar"}, "type": "SMART", "query": {"type": "SMART", "value": "{{...}}"}, "lazy": false}`. The **`id` is required** and must be unique in the rule; without it the API answers an empty 500. The editor writes `_customsmartvalue_id_<epoch millis>`; any fixed number in that form works. |
| `jira.create.mapping-variable` | 1 | Create lookup table: `{"name": {"type": "FREE", "value": "table"}, "mappings": [{"key": "a", "value": "b"}]}`, read as `{{table.get("a")}}` |
| `jira.lookup.issues` | -1 | `{"id": "<any id>", "name": {"type": "FREE", "value": "lookupIssues"}, "type": "JQL", "query": {"type": "SMART", "value": "<JQL>"}, "lazy": false}` |
| `jira.issue.related` | 1 | Branch, `component: "BRANCH"`: `{"relatedType": "created", "jql": null, "linkTypes": [], "onlyUpdatedIssues": false, "similarityLimit": 40, "compareValue": 0}`; `relatedType` is also `jql`, linked work items and so on. The steps of the branch go in `children`. A branch can sit inside an If block's `children`, and a `created` branch **must** sit in the same If block as the create: work items created inside an If block are invisible outside it, and the run fails with "Unable to branch flow as it requires work items to have been previously created by this flow". |
| `jira.issue.create` | 12 | `{"operations": [{"field": {"type": "ID", "value": "project"}, "fieldType": "project", "type": "SET", "value": {"type": "ID", "value": "<projectId>"}}, ...], "advancedFields": "<JSON as a string>", "sendNotifications": true}`. Fields without an operation go into `advancedFields`, e.g. `{"fields": {"security": {"name": "<level>"}, "customfield_<n>": "<request type id>"}}` for the security level and the JSM request type, where `customfield_<n>` is the site's Request Type field. |
| `jira.issue.edit` | 12 | `{"operations": [{"field": {"type": "NAME", "value": "<field>"}, "fieldType": "<field type key>", "type": "SET", "value": "..."}], "advancedFields": null, "sendNotifications": true}` |
| `jira.issue.comment` | 2 | `{"comment": "...", "publicComment": false, "commentVisibility": null, "sendNotifications": true, "addCommentOnce": true}`; `publicComment: false` makes it internal on a service space |
| `jira.issue.transition` | 11 | `{"destinationStatus": {"type": "NAME", "value": "Started"}, "transitionMatch": "", "operations": [], "advancedFields": null, "sendNotifications": true}` |
| `jira.sd.add.customer` | 2 | Add service project customer: `{"email": "...", "displayName": "...", "organization": {"type": "SMART", "value": ""}}`. It creates the customer or returns the existing one; later steps read `{{customer.accountId}}`, which is how a rule makes an outside address the **reporter** of a request (a reporter cannot be set from an email string). The value names no service space, so check which space the customer lands in before relying on it in a multi-space rule. |
| `action.send-templated-email` | 1 | Send email: `{"templateContextData": {"emailName": "<internal name>", "subject": "...", "body": "<ADF document as a JSON string>", "bodyType": "ADF", "fromName": "...", "fromEmailAddress": "...", "replyToName": "...", "replyToEmailAddress": "..."}, "to": [{"type": "FREE", "value": "someone@example.com"}], "convertLineBreaks": false, "sendGroupedEmails": true, "legacyConfig": true}`. Recipients sit in `to` (and `cc`, `bcc`), outside `templateContextData`; a step saved without a recipient has no `to` key at all. A `PUT` that echoes the step keeps them. `fromEmailAddress` can be a service space's own email channel address. |
