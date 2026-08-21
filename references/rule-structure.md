# Rule structure, limits, and debugging

Source: `jira-automation-triggers`, `jira-automation-conditions`, `jira-automation-actions`,
`jira-automation-branches`, `what-is-rule-branching`, `advanced-automation-components`,
`automation-service-limits`, `permissions-required-for-jira-cloud-automation-rules`,
`what-is-a-rule-actor`, `run-jira-rules-as-another-user`, `debug-an-automation-rule`,
`limitations-in-team-managed-projects-for-automation-rules`,
`best-practices-for-optimizing-automation-rules`, `import-and-export-jira-automation-rules`.

A rule = one trigger → any number of conditions, branches and actions.

## Triggers

**General:** Field value changed · Work item created / updated / assigned / deleted / moved / transitioned / linked / link deleted · Work item commented · Work item comment edited · Work logged · Manual trigger from work item · Multiple work item events · Scheduled · Incoming webhook · Space (project) created · Form attached / submitted / opened for edits

**Software:** Sprint created, started or completed · Version created, updated, released

**DevOps:** Branch created · Commit created · Pull request created / declined / merged · Build successful / failed / status changed · Deployment successful / failed / status changed · Compass component created · Compass scorecard status change

**Jira Service Management:** Alert created / updated / status changed / note added · Manual trigger from alert · SLA threshold breached · Approval required · Approval completed · Object trigger (Assets) · Attribute value changed · Emoji reaction to Slack message

**Security / Guard:** Vulnerability found · Content scanning alert for Jira

**Design:** Design linked to work item

Trigger choice determines which smart values exist. `{{comment}}` only under Work item commented; `{{changelog}}` only under editing triggers; `{{webhookData}}` only under Incoming webhook.

> Scheduled triggers can run with or without a JQL search. Without JQL there is no `{{issue}}` context at all — everything must come from lookups or the webhook payload.

## Conditions

Work item (issue) fields condition · Alert fields condition · **`{{smart values}}` condition** · JQL condition · AQL condition (Assets) · User condition · Related issues condition · Work item attachments · Affected services · Forms attached · Work item has design linked · **If/else block**

The **`{{smart values}}` condition** (a.k.a. Advanced compare) is the workhorse for anything the field conditions can't express: put a smart value on the left, choose equals / contains / does not contain / greater than / matches regex, put a literal or another smart value on the right.

**If/else block** is the only way to get real branching logic on the standard plans — everything below it in the block runs only when its condition matches.

## Branches

Branching re-points `{{issue}}` at a *related* set of items — subtasks, parent, epic, stories, linked issues, or a JQL result set. Inside a branch:

- `{{issue}}` = the branched item
- `{{triggerIssue}}` = the original trigger item

Branches over JQL or related items run **in parallel and asynchronously**. Do not assume ordering between a branch and the steps after it, and do not have two branches write the same field.

## Advanced steps — Premium and Enterprise only

| Step | Smart values | Notes |
|---|---|---|
| **Branch at the same time** | — | runs several groups in parallel |
| **Branch with conditions** | yes | up to 3 branches + optional fallback; **only the first matching branch runs**; AND/OR execution mode per branch. Jira and JSM only |
| **Delay until** | no (outputs: yes) | 10 seconds – 90 days, or wait for an event (Work item updated, Alert updated, Incoming webhook). Must sit **before** any condition or action in its group |
| **Loop** | yes | repeats **max 3 times**; up to 32 conditions; optional **Run steps once before first loop**. Jira, JSM and Confluence |

Adding any advanced step wraps all steps in a **flow group**.

**Delay until outputs** — `reason` (condition met vs timed out), `resumedAt` (UTC timestamp), `payload` (the raw changelog at resume). Must be added as inputs to downstream groups to be usable.

**Loop variables** — `loop.count`, `loop.conditionMet` (string `"true"`/`"false"`). Updated only when the loop *completes*, so they are useless inside the loop itself. Loop conditions can read only the trigger context and declared loop inputs — not flow-group outputs, unless those are passed in as inputs.

**Nesting** — max 2 levels. Loops cannot nest inside loops. Permitted: Branch-at-same-time inside Loop, Loop inside Branch-at-same-time, Branch inside Branch (each optionally with Delay until).

## Actions worth knowing

| Action | Why it matters |
|---|---|
| **Log action** | prints to the audit log — the only real smart-value debugger |
| **Create variable** | defines a reusable value; **always returns a string** |
| **Lookup issues** | JQL search, **max 100 results**, exposed as `{{lookupIssues}}` |
| **Re-fetch work item data** | refreshes `{{issue}}`; required after any step whose result a later step must read |
| **Send web request** | outbound HTTP; enable *Wait for response* for `{{webResponse}}` |
| **Re-fetch alert data** | same as re-fetch, for `{{alert}}`; runs in the background |

## Service limits

| Limit | Value |
|---|---|
| Steps per rule | **65** |
| Steps per advanced rule | 500 |
| Steps per rule group | 65 |
| Rule labels per site | 600 |
| New subtasks per action | 50 |
| Work items searched (scheduled JQL) | **999** |
| Concurrent runs of one scheduled rule | **1** |
| Associated items per execution | **5,000** — rule is *disabled* on breach |
| Items queued globally | 50,000 |
| Daily processing time | 60 min per 12 hrs |
| Loop detection (rule re-triggering itself) | 10 |
| Send email (Free / Trial only) | 100 per 24 h |

Note the compounding rule on associated items: a scheduled trigger fetching 100 items followed by a branch fetching 10 subtasks each = 1,000 against the 5,000 budget.

Also note **Lookup issues (100)** ≠ **scheduled JQL search (999)**. Different ceilings, both silent.

## Permissions and the rule actor

Rules run as the **rule actor** — by default *Automation for Jira*, changeable per rule. Everything the rule does is subject to that user's permissions: browse, edit, transition, field-level visibility, and issue security.

Symptoms of an actor permission problem: rule reports success, nothing changed; fields come back empty that are populated in the UI; `createmeta`/`editmeta` omits a field you can see.

## Debugging checklist

1. **Audit log first** — every execution, its outcome, and per-action detail. Filter by rule.
2. **Log action** with the suspect smart value, on a manual trigger.
3. **Stale data?** Insert Re-fetch work item data.
4. **Empty value?** Add a `|"fallback"` default and re-run to distinguish "empty" from "wrong path".
5. **Permission?** Temporarily switch the rule actor to a site admin to confirm, then fix properly.
6. **Team-managed project?** Several components behave differently or are unavailable — see `limitations-in-team-managed-projects-for-automation-rules`. Story point field names differ: `Story Points` (company-managed) vs `Story point estimate` (team-managed).

## Import / export

Export produces a JSON file (individual rule, or all rules at once). Import is UI-only — there is **no public REST API to create rules**.

Constraints that bite:
- The JSON must come from the **same version** of Jira automation or the import fails
- Max **5 MB**
- Imported rules arrive **disabled** and must be enabled manually
- Data Center and Cloud formats differ — cross-import commonly throws `MismatchedInputException`

Practical consequence: to script or clone rules, **build once in the UI, export, then edit that JSON**. Hand-authoring rule JSON from scratch against an undocumented, drifting schema is not worth it.
