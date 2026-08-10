# Attachments

Source: `jira-smart-values-issues` (`{{attachment}}`), `jira-automation-conditions`
(Issue attachments), `jira-automation-actions` (Send email, Delete attachments,
Clone/Create/Edit/Transition work item), Atlassian KB
`how-to-send-customers-a-notification-with-the-most-recent-attachment-using-automation`,
and the Jira Cloud platform REST API `api-group-issue-attachments`.

**The headline: automation gives you attachment *metadata*, never file contents.** Reading a file means fetching it yourself over HTTP.

## `{{attachment}}` — metadata

Returns a **list** when the item has more than one attachment, so the list functions apply (`.first`, `.last`, `.size`, `{{#attachment}}…{{/}}`).

| Smart value | Returns |
|---|---|
| `{{attachment.filename}}` | file name |
| `{{attachment.mimeType}}` | file format |
| `{{attachment.created}}` | date/time added |
| `{{attachment.first.size}}` | size in bytes — **must** address a specific attachment |
| `{{attachment.content}}` | **a download URL** — see below |
| `{{attachment.author}}` | user who attached it |
| `{{attachment.author.accountId}}` / `.emailAddress` / `.displayName` / `.active` / `.timeZone` | author details |

### `{{attachment.content}}` is a URL, not the file

The smart-values reference describes it as "the content of the attachment, with binary content rendered as a String", which reads like file bytes and is not. It is the download URL, matching the REST attachment schema where `content` is a link.

Atlassian's own KB uses it as an `href`:

```html
<a href="{{webhookData.attachment.content}}">{{webhookData.attachment.filename}}</a>
```

That KB also notes the link **only works for licensed Jira users** — portal-only JSM customers cannot download from it.

`{{attachment.size}}` without an index is documented on the *condition* page but the smart-values page requires an indexed form (`{{attachment.first.size}}`). Use the indexed form.

## Triggering on attachments

**There is no "attachment added" trigger.** Options:

- **Work item updated** — fires on attachment changes, but on much else too; gate it
- **Field value changed** scoped to the Attachment field
- **Incoming webhook** — what the Atlassian KB uses, giving `{{webhookData.attachment.filename}}` / `.content`

## Filtering on attachments

The **Issue attachments** condition (smart values: *no*) checks whether the description or comments carry attachments, and can compare on `filename`, `mimeType`, `size`, `created`, `author`, `author.accountId`, `author.emailAddress`, `author.displayName`, `author.active`, `author.timeZone`.

Combine with the JQL condition for filename-pattern work. Extension filtering belongs here — you cannot inspect content to decide.

## Reading file contents

Only route: **Send web request** against the REST content endpoint.

```
GET https://<site>.atlassian.net/rest/api/3/attachment/content/<id>?redirect=false
Authorization: Basic <base64 of email:apiToken>
```

Two details decide whether this works at all:

- **`?redirect=false`** — the default response is **303 See Other** pointing at a temporary media URL. With `redirect=false` you get **200 OK** with the content inline. Do not assume automation follows the 303.
- **Send web request supports JSON and plain-text response bodies only.** Text formats (`.eml`, `.csv`, `.json`, `.xml`, `.txt`) can land in `{{webhookResponse.body}}`. Binary (PDF, images, Office files) cannot be usefully retrieved this way.

The endpoint honours a **`Range` header** (HTTP byte ranges). For a large file, request only the first few KB — the standard trick for reading mail headers without pulling the whole message.

Permissions: *Browse projects* on the item's project; issue-level security and private-comment restrictions still apply. Scope `read:attachment:jira` (classic: `read:jira-work`).

Related endpoints:

| Endpoint | Purpose |
|---|---|
| `GET /rest/api/3/attachment/{id}` | attachment metadata |
| `GET /rest/api/3/attachment/content/{id}` | contents (303 by default) |
| `GET /rest/api/3/attachment/thumbnail/{id}` | thumbnail |
| `GET /rest/api/3/attachment/{id}/expand/human` | list an **archive's** contents |
| `GET /rest/api/3/attachment/{id}/expand/raw` | archive contents, raw form |
| `GET /rest/api/3/attachment/meta` | whether attachments are enabled + max size |

`expand/human` / `expand/raw` only enumerate entries inside archive attachments — they do not extract files.

> Unverified: the practical body-size ceiling on a Send web request response. Test with a representative file before designing around it.

## Parsing what you fetched

For a **simple, single-part, 7-bit `.eml`**, header extraction is workable:

```
{{webhookResponse.body.substringBefore("\n\n")}}                     all headers
{{webhookResponse.body.substringBetween("Subject: ", "\n").trim()}}
{{webhookResponse.body.match("From: .*<([^>]+)>")}}
{{webhookResponse.body.match("Date: (.*)")}}
```

For CSV:

```
{{webhookResponse.body.split("\n").get(1).split(",").get(2)}}
```

For JSON, skip string surgery entirely — `{{jsonStringToObject(webhookResponse.body).some.path}}`.

### Where this stops working

Real MIME defeats smart values:

- multipart boundaries and nested parts
- `quoted-printable` and `base64` bodies — **there is no decode function**
- RFC 5322 folded headers (continuation lines)
- charset conversion and `=?UTF-8?B?…?=` encoded words

Do not attempt full `.eml` parsing in a rule. Extract headers, or offload.

## Offloading

Send the attachment URL (plus credentials) to something that parses properly and posts results back to the work item: a **Forge app** (`requestJira` reaches the same endpoint — the API docs ship a Forge snippet), ScriptRunner, a small HTTP endpoint, or n8n/Make.

## Attachment actions

| Action | Behaviour |
|---|---|
| **Send email** | attach up to **10 files, 4 MB total**; the rule must be saved/enabled before attachments can be added |
| **Delete attachments** | cleans up by **regex match on filename** |
| **Clone / Create / Edit / Transition work item** | copy attachments by selecting the **Attachment** field and choosing the source work item |

Note the asymmetry: rules can *copy*, *delete* and *email* attachments, and *link* to them — but never read them.
