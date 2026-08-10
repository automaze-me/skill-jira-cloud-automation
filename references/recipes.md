# Recipes

Worked end-to-end patterns. Each states the rule shape, then the exact values to paste.

Before trusting any of these on a live site, run the smart values through a **Log action** on a manual trigger — instance-specific field names and the `webhookResponse` / `webResponse` naming difference both bite here.

---

## 1. JSM organizations by email domain

### 1a. Setting the work item's Organizations field — use the built-in option

**Try this before writing any web request.** The **Create work item**, **Edit work item** and **Transition work item** actions expose an Organizations field option called **"Use reporter's email domain"**:

- it matches against the domains on each organization's **Manage email domains** list — nothing else
- exact match on **one** organization → that organization is selected
- domain listed on **several** organizations → one of them is picked arbitrarily
- domain listed nowhere + **"Create organization if it doesn't exist"** → a new organization is created, named after the domain

That last behaviour is a footgun on a public portal: every unrecognised sender domain spawns an organization. Leave it off unless you mean it.

This sets the **Organizations field on the work item**. It does not add the *user* to the organization — for membership, use 1b.

### 1b. Adding a customer to an organization, skipping those already in one

There is no public REST API for an organization's **email domain** mappings (feature request CSM-117, unresolved), so mapping maintenance stays manual. Membership itself *is* scriptable.

**Shape:** Work item created → Send web request (check) → Advanced compare → Send web request (assign)

**Step 2 — check current membership.** GET, *Wait for response* enabled:

```
https://<site>.atlassian.net/rest/servicedeskapi/organization?accountId={{issue.reporter.accountId}}
```

Header: `Authorization: Basic <base64 of agentEmail:apiToken>`

> The token owner must hold a **Service Desk agent licence** — the `accountId` filter is agent-only, and a Jira admin without an agent seat gets an empty result rather than an error.

**Step 3 — Advanced compare condition:**

| Goal | First value | Condition | Second value |
|---|---|---|---|
| in *no* organization | `{{webhookResponse.body.size}}` | equals | `0` |
| not in *this* organization | `{{webhookResponse.body.values.id}}` | does not contain | `<orgId>` |

`{{…values.id}}` flattens the paged result to a comma-separated list of IDs, which is what makes *does not contain* work.

**Step 4 — assign.** POST to `/rest/servicedeskapi/organization/<orgId>/user`, same auth header:

```json
{ "accountIds": [{{issue.reporter.accountId.asJsonString}}] }
```

**Domain routing** — where 1a can't be used (it sets the field, not membership) and you need the domain yourself:

```
{{issue.reporter.emailAddress.substringAfter("@").toLowerCase()}}
```

Returns empty if the reporter's email visibility is "Only you and admins" — guard it with `{{…|"unknown"}}` or a *is not empty* condition, otherwise every such reporter routes to the same wrong bucket.

Pair with a **Create lookup table** for the domain → orgId mapping, which keeps the routing in one editable step:

```
{{orgLookup.get(issue.reporter.emailAddress.substringAfter("@").toLowerCase())}}
```

> Test whether the assign call is idempotent before building step 3 at all. Atlassian documents the sibling endpoint `POST /servicedesk/{id}/organization` as returning 204 for an existing association, but says nothing of the sort for the add-users endpoint — that asymmetry is worth one curl. If it is idempotent, delete the check and save a web request per run.

---

## 2. Weekly digest of open work

**Shape:** Scheduled → Lookup issues (JQL) → Send email / Slack

```
{{#if(lookupIssues.isEmpty)}}Nothing open. {{/}}
{{#if(not(lookupIssues.isEmpty))}}
{{lookupIssues.size}} open items, {{lookupIssues.Story Points.sum}} points:

{{#lookupIssues}}• {{key}} — {{summary}} ({{assignee.displayName|"Unassigned"}}, {{status}})
{{/}}
{{/}}
```

Remember Lookup issues stops at **100** results, silently.

---

## 3. Sum subtask estimates back onto the parent

**Shape:** Work item transitioned → Branch: parent → Edit work item

```
{{#=}}{{issue.subtasks.Story Points.sum}}{{/}}
```

Or without the math wrapper, since `sum` is a method: `{{issue.subtasks.Story Points.sum}}`.

Use `Story point estimate` instead of `Story Points` in team-managed projects.

---

## 4. Append to a multi-value field without destroying it

The *Choose fields to set* UI overwrites. To append, use **More options** JSON:

```json
{ "update": { "labels": [ { "add": "triaged" } ] } }
```

```json
{ "update": { "customfield_12346": [ { "add": { "id": "{{reporter.accountId}}" } } ] } }
```

Copy a whole list across from the parent (this one *does* overwrite, deliberately):

```json
{ "fields": { "labels": {{issue.parent.labels.asJsonStringArray}} } }
```

---

## 5. Extract structured data out of free text

Pull a PO number like `PO-4471` out of the description:

```
{{issue.description.match(".*(PO-[0-9]+).*")}}
```

Every email address in the latest comment (returns a collection — iterate it). Note the backslash-free character classes: `\w` does **not** work in smart values, see [text-and-lists.md](text-and-lists.md):

```
{{#issue.comments.last.body.match("([A-Za-z0-9._-]+@[A-Za-z0-9.-]+)")}}{{.}}{{^last}}, {{/}}{{/}}
```

Everything after a marker line:

```
{{issue.description.substringAfter("--- customer details ---").trim()}}
```

When interpolating user data into a pattern, wrap it in `quote()` so regex metacharacters can't break the match.

---

## 6. Parse a JSON response from a third-party API

**Shape:** any trigger → Send web request (*Wait for response*) → Create variable → use it

If the body arrives as an object, address it directly:

```
{{webhookResponse.body.data.customerTier}}
```

If it arrives as a *string*, convert first:

```
{{jsonStringToObject(webhookResponse.body).data.customerTier}}
```

Then gate on it:

```
{{#if(equals(jsonStringToObject(webhookResponse.body).data.customerTier, "enterprise"))}}
Escalating — enterprise customer.
{{/}}
```

Guard against a failed call before reading the body:

```
{{#if(equals(webhookResponse.status, 200))}}…{{/}}
```

---

## 7. Due-date and SLA arithmetic

Three business days from now, as a date field value:

```
{{now.plusBusinessDays(3).jiraDate}}
```

Next Monday 09:00 in the assignee's day, written as a date-time field:

```
{{now.withNextDayOfWeek("MON").withHour(9).withMinute(0).jiraDateTime}}
```

Age in business days, for a comment:

```
Open for {{now.diff(issue.created).businessDays.abs}} business days.
```

Overdue check:

```
{{#if(now.isAfter(issue.duedate))}}⚠️ Overdue by {{now.diff(issue.duedate).days.abs}} days{{/}}
```

Last business day of the month, for month-end rules:

```
{{now.lastBusinessDayOfMonth.jiraDate}}
```

> Business days are Mon–Fri 9–18 with no holiday calendar. For real holiday handling, keep a JQL-queryable calendar or accept the approximation.

---

## 8. Copy a value only when the target is empty

Avoids the "field copies from itself while empty, nothing happens" trap:

```
{{#if(issue.customfield_10050.isEmpty)}}…run the edit…{{/}}
```

Better, as a condition rather than inline: **`{{smart values}}` condition** with first value `{{issue.customfield_10050}}`, condition *is empty*.

---

## 9. Build a JQL string dynamically

Inject dates with `jqlDate` / `jqlDateTime`, never the default format:

```
project = {{issue.project.key}} AND created >= "{{now.minusDays(7).jqlDate}}" AND status != Done
```

Quote anything that can contain spaces, and `urlEncode` it if the JQL is going into a URL rather than a JQL field.

---

## 10. Read headers out of an attached `.eml`

**Shape:** Work item updated → Issue attachments condition → Send web request (fetch) → Create variable / Comment

Full detail in [attachments.md](attachments.md); this is the assembled version.

Built and verified end to end on a real 46 KB `.eml` (2026-08-10).

**Gate on the file type** — Issue attachments condition on `mimeType` = `message/rfc822`. More reliable than a `.eml` filename suffix; Jira sets it correctly.

**Fetch it** — Send web request, *Wait for response* ticked:

| Field | Value |
|---|---|
| URL | `https://api.atlassian.com/ex/jira/<cloudId>/rest/api/3/attachment/content/{{attachment.first.id}}?redirect=false` |
| Method | `GET` |
| Body | empty |
| Headers | `Authorization: Basic <base64 of email:apiToken>` and `Range: bytes=0-8191` |

Three things that are not optional:

- **`{{attachment.first.id}}`, not `{{attachment.first.content}}`** — `content` is a *site* URL, which scoped tokens reject. Get `<cloudId>` from `https://<site>.atlassian.net/_edge/tenant_info`.
- **`?redirect=false`** — otherwise a 303 to a temporary media URL.
- **`Range`** — the whole file returns fine (no truncation at 46 KB), but you only need the header block.

Tick **Hidden** on the Authorization value once it works. It's irreversible and lost on export/duplicate, so do it last.

**Extract** — one pattern, folded or not:

```
--- extract ---
delivered: {{webhookResponse.body.match("(?sm)^Delivered-To: (.*?)(?=^[A-Za-z])")}}
from:      {{webhookResponse.body.match("(?sm)^From: (.*?)(?=^[A-Za-z])")}}
subject:   {{webhookResponse.body.match("(?sm)^Subject: (.*?)(?=^[A-Za-z])")}}
date:      {{webhookResponse.body.match("(?sm)^Date: (.*?)(?=^[A-Za-z])")}}
--- end ---
```

`(?s)` absorbs continuation lines; the `(?m)^[A-Za-z]` lookahead stops at the next real header. **Do not** use `(?m)^Subject: (.*)` — it truncates folded values and gives a plausible wrong answer (a calendar-invite subject silently loses its time range).

**Debugging notes from the build:**

- The audit log flattens newlines to spaces on display. The body is fine; don't rewrite the regex because the log looks like one line.
- Test auth with `/rest/api/3/myself`, never the attachment endpoint — that one allows anonymous access and disguises auth failure as a permission error.
- `match()` returns `[a, b]` when a header appears twice (common in forwarded mail). Append `.first`.

**Ceiling:** these are the *outer* envelope's headers. A forwarded or reported mail sits inside as an embedded `message/rfc822`, and reaching its headers means walking MIME boundaries — plus `base64` / `quoted-printable` decoding that smart values cannot do. Offload to Forge or ScriptRunner at that point.

---

## 11. Make a rule idempotent

Rules re-fire more often than expected — an edit by another rule, a bulk change, a re-import. Cheap guards:

- Label as a sentinel: condition *labels does not contain* `auto-processed`, then add it in the same run
- Compare before writing with the `{{smart values}}` condition
- Check `{{issue.comments}}` for a marker string before commenting again
- Watch the **loop detection limit (10)** — a rule that edits an item it also triggers on will be stopped and marked LOOP
