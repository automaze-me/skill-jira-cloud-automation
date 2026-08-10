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

Only route: **Send web request** against the REST content endpoint. Verified working end to end (2026-08-10).

```
GET https://api.atlassian.com/ex/jira/<cloudId>/rest/api/3/attachment/content/{{attachment.first.id}}?redirect=false
Authorization: Basic <base64 of email:apiToken>
Range: bytes=0-8191          (optional, see below)
```

Details that decide whether this works at all:

- **Use `{{attachment.first.id}}`, not `{{attachment.first.content}}`.** `content` hands you a *site* URL (`https://<site>.atlassian.net/rest/api/2/attachment/content/<id>`), which scoped tokens reject — see the auth section. `id` is **undocumented but real**; verified returning a numeric ID.
- **`?redirect=false`** — the default response is **303 See Other** pointing at a temporary media URL. With `redirect=false` you get **200 OK** with the content inline. Do not assume automation follows the 303.
- **Send web request supports JSON and plain-text response bodies only.** Text formats (`.eml`, `.csv`, `.json`, `.xml`, `.txt`) land in `{{webhookResponse.body}}`. Binary (PDF, images, Office files) cannot be usefully retrieved this way.
- **`Range` header works** for byte ranges. A 46 KB `.eml` returned in full with `length` matching the attachment `size` exactly, so the body cap is somewhere above 46 KB — but pulling 46 KB to read 2 KB of headers is waste. `bytes=0-8191` is plenty for a header block.

Permissions: *Browse projects* on the item's project; issue-level security and private-comment restrictions still apply. Scope `read:attachment:jira` (classic: `read:jira-work`).

### Authentication: the two traps

**1. Scoped tokens must use the gateway URL.** Service accounts can only create *scoped* API tokens, and those authenticate **only** against:

```
https://api.atlassian.com/ex/jira/<cloudId>/rest/api/3/…
```

Against the site URL they silently fail to authenticate. Find the cloud ID unauthenticated at `https://<site>.atlassian.net/_edge/tenant_info`.

**2. Failed auth disguises itself as a permission error.** The attachment content endpoint *can be accessed anonymously*. A rejected credential therefore does **not** return 401 — the request proceeds as anonymous and returns:

```json
{"errorMessages":["You do not have permission to view attachment with id: 12345"],"errors":{}}
```

Which reads like a project-permission problem and is not. **Always canary with `/rest/api/3/myself`**, which cannot be accessed anonymously and so returns an honest `Client must be authenticated to access this resource`:

```
curl -s --user 'svc@example.com:TOKEN' \
  'https://api.atlassian.com/ex/jira/<cloudId>/rest/api/3/myself'
```

A 200 there means token, URL form and scopes are all correct, and any later error is genuinely about permissions.

Third possibility, if the token is blocked outright: `{"message":"An admin blocked your API token access to this product.", "status":403}` — an org authentication policy. Only an org admin can lift it, typically by creating a policy that allows API token access and putting one service account in it. A **Forge app** sidesteps this entirely, since `asApp()` uses app-scoped OAuth rather than a user API token.

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

### Backslashes do not survive the smart-value parser

**The single most important constraint here.** Any regex construct requiring a backslash — `\n`, `\t`, `\R`, `\s`, `\w`, `\d`, `\p{…}` — silently returns empty inside `match()` and `replaceAll()`. Doubling them (`\\n`) does not help. Verified by testing `\R` (which needs no escaping in Java) alongside `\\n` and `\\t`; all returned empty.

Everything backslash-free works, including inline flags and lookarounds: `(?m)` `(?s)` `^` `$` `.` `*?` `+` `[^>]` `[A-Za-z]` `(?=…)` `(?!…)`.

Consequence: forget `substringBefore("\n\n")` and anything that needs to name a newline. Use `(?m)^` line anchoring instead.

### Extracting headers, including folded ones

RFC 5322 wraps long header values onto continuation lines that start with whitespace. A plain `(?m)^Subject: (.*)` stops at the first newline and silently truncates — a calendar-invite subject loses its time range, and you get a plausible-looking wrong answer.

This pattern handles folded and unfolded headers identically, with no backslashes:

```
{{webhookResponse.body.match("(?sm)^Subject: (.*?)(?=^[A-Za-z])")}}
```

- `(?s)` — `.` crosses newlines, so continuation lines are absorbed
- `(?m)^` in the lookahead — terminate at the next line starting with a **letter**, i.e. the next real header, since continuations start with whitespace
- non-greedy `(.*?)` — stop at the first such line, not the last

Generalised: `(?sm)^HeaderName: (.*?)(?=^[A-Za-z])`. Verified on `Delivered-To`, `From`, `Subject`, `Date`.

The captured value retains the embedded newline and the continuation's leading space. Writing it to a single-line Jira text field collapses that; if you need it clean in a comment, there is no backslash-free whitespace-normalising call, so accept it or offload.

> Don't reach for a shorter duplicate header as a workaround. `Thread-Topic` carries the same text as `Subject` but folds too, and is Exchange-only.

### Multiple matches

`match()` returns a **collection** when the pattern matches more than once, rendering as `[a, b]`. Forwarded mail routinely has two `Subject:` and several `Received:` headers. Append `.first` when you want one.

### Other formats

CSV — no way to name the newline, so split on a line-anchored match rather than `split("\n")`. JSON — skip string surgery entirely: `{{jsonStringToObject(webhookResponse.body).some.path}}`.

### Where this stops working

- multipart boundaries and nested parts — an embedded `message/rfc822` (a forwarded or reported mail) means the headers you extract are the **outer** envelope, not the message inside
- `quoted-printable` and `base64` bodies — **there is no decode function**
- charset conversion and `=?UTF-8?B?…?=` encoded words

Header extraction from the outer envelope: fine. Anything deeper: offload.

### Reading the audit log

The audit log **flattens newlines to spaces** when displaying a logged value. A body printed with `.left(1200)` looks like one long line even though the newlines are intact and `(?m)^` works on it. Don't debug the regex based on how the log renders — test with a known single-line header (`Delivered-To`) and see whether it returns just its value.

## Offloading

Send the attachment URL (plus credentials) to something that parses properly and posts results back to the work item: a **Forge app** (`requestJira` reaches the same endpoint — the API docs ship a Forge snippet), ScriptRunner, a small HTTP endpoint, or n8n/Make.

## Attachment actions

| Action | Behaviour |
|---|---|
| **Send email** | attach up to **10 files, 4 MB total**; the rule must be saved/enabled before attachments can be added |
| **Delete attachments** | cleans up by **regex match on filename** |
| **Clone / Create / Edit / Transition work item** | copy attachments by selecting the **Attachment** field and choosing the source work item |

Note the asymmetry: rules can *copy*, *delete* and *email* attachments, and *link* to them — but never read them.
