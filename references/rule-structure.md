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

Unverified: under *Work item created* a JQL condition may not yet find the new work item (search index lag); a work item fields or smart-value condition avoids the question.

## Branches

Branching re-points `{{issue}}` at a *related* set of items — subtasks, parent, epic, stories, linked issues, or a JQL result set. Inside a branch:

- `{{issue}}` = the branched item
- `{{triggerIssue}}` = the original trigger item
- the trigger's own smart values stay readable: `{{comment.body}}` under *Work item commented*, in a JQL branch (run, 2026-11)
- `{{triggerIssue.key}}` renders the trigger item's key in a JQL branch under *Work item commented*; `{{comment.url}}` and `{{comment.issue.key}}` render empty there (probe, run 2026-11)
- a variable created in the trunk is readable inside a JQL branch below it: *Create variable* `handoffKey` = `{{issue.key}}` before the branch, `{{handoffKey}}` inside it gave the trigger's key (rule without advanced steps; probe, run 2026-11)
- an If/else block can sit among the branch's steps (run, 2026-11)

Branches over JQL or related items run **in parallel and asynchronously**. Do not assume ordering between a branch and the steps after it, and do not have two branches write the same field.

## Advanced steps — Premium and Enterprise only

| Step | Smart values | Notes |
|---|---|---|
| **Branch at the same time** | — | runs several groups in parallel |
| **Branch with conditions** | yes | up to 3 branches + optional fallback; **only the first matching branch runs**; AND execution mode per branch, and the reference also lists an OR mode ("At least one condition matches") that was **absent from the editor on a live tenant on 2026-09-11** — do not design around it while a slot is free; **the fallback takes no conditions** — see below. Jira and JSM only |
| **Delay until** | no (outputs: yes) | 10 seconds – 90 days, or wait for an event (Work item updated, Alert updated, Incoming webhook). Must sit **before** any condition or action in its group |
| **Loop** | yes | repeats **max 3 times**; up to 32 conditions; optional **Run steps once before first loop**. Jira, JSM and Confluence |

Adding any advanced step wraps all steps in a **flow group**.

**Delay until outputs** — `reason` (condition met vs timed out), `resumedAt` (UTC timestamp), `payload` (the raw changelog at resume). Must be added as inputs to downstream groups to be usable.

**Loop variables** — `loop.count`, `loop.conditionMet` (string `"true"`/`"false"`). Updated only when the loop *completes*, so they are useless inside the loop itself. Loop conditions can read only the trigger context and declared loop inputs — not flow-group outputs, unless those are passed in as inputs.

**Nesting** — max 2 levels. Loops cannot nest inside loops. The docs list Branch-inside-Branch among the permitted combinations; **the editor refused it** when tried on a live tenant (2026-08-21). Treat branch-inside-branch as unavailable until your own editor proves otherwise, and see "How steps actually compose" below.

## How steps actually compose

Measured in the editor on a live tenant, 2026-08-21. Two of these contradict the docs, and all four change how a rule has to be laid out.

| Fact | Consequence |
|---|---|
| **Branches do not nest inside branches** | One decision point per rule. More decisions means more rules, chained on something durable. |
| **Branch conditions evaluate when the branch step is reached** | Anything you fork on must already exist. A value computed *inside* a branch cannot be branched on in the same rule. |
| **A bare condition inside a branch stops the rest of that branch only** | It has no else, so it can only stop. Never alert. |
| **The fallback branch takes no conditions** | The UI states it outright: *"Runs when no conditions are met."* Every guard has to be spent as one of the three conditional branches. |
| **`If/else` caps at 2 nested layers** | Shipped Nov 2024 ([AUTO-67](https://jira.atlassian.com/browse/AUTO-67), 372 votes, Fixed). Deeper guard chains are not buildable. |

Two rules follow, and they are the whole trick to laying out a non-trivial flow:

- **Gather in the trunk, fork once.** Every web request, edit and re-fetch goes above the branch step.
- **Branches alert, bare conditions stop.** Spend a branch only on an outcome that needs an action; use a bare condition for anything meant to end quietly.

### The fallback is unguarded, so budget the three branches for stops too

The fallback is the only unconditional path out of a `Branch with conditions`, which means it runs for *everything* the three conditions did not catch. A case that should end silently cannot be filtered out down there — it has to be caught above, as a branch whose condition matches it and whose body does nothing but a Log action.

Three conditional branches, and both the alerts and the silent stops come out of the same three. If the editor offers the OR execution mode (documented, but missing on one tenant on 2026-09-11), combine stops under it to save slots; otherwise give each stop its own branch, which also makes the audit log name the guard that fired:

```
1  <alert condition A>              → Slack
2  <alert condition B>              → Slack
3  <stop condition C> OR <stop D>   → Log "no action: expected case"
   fallback                         → the write
```

Put the stop conditions in the fallback instead and the write happens for cases it was never meant to touch. Which is silent, and reads in the audit log as a completed run.

When one rule runs out of branches, split at a **field change** and let the next rule trigger on it — the second rule needs **"Allow other rules to trigger this rule"** enabled or the chain silently does nothing.

## Condition failure is group-scoped

**"Add criteria to determine if the flow continues running just for the steps in a flow group. Only applies when using advanced steps."** — Atlassian, *Use automation steps in a flow*.

The *Add conditions to an automation flow* page says the opposite in three words: "the flow stops running." The first is what happens once advanced steps are in play; the second describes a rule with no flow groups in it. Nothing on either page cross-references the other, which is why this reads as a bug the first time it bites.

## Variables are scoped to their group — and so is `{{webResponse}}`

A variable created inside a group, branch or `If/else` block is gone outside it — [AUTO-154](https://jira.atlassian.com/browse/AUTO-154), 155 votes, still open, filed as a **Suggestion** rather than a Bug. Values the trigger provides are not affected: `{{comment}}` reads fine inside a branch (see [Branches](#branches)). Nor is the way in: a trunk variable reaches a branch below it (see [Branches](#branches)); what dies is a variable created *inside* a block, outside that block. The sanctioned mechanism for crossing a group boundary is the group's own **inputs and outputs**.

**`{{webResponse}}` obeys the same rule, in the direction that hurts.** A Send web request in the trunk populates it; a branch below cannot read it. Measured on a live tenant with two Log actions, one step apart:

```
last step before the branch    memb status: 200      memb size: 1
first step inside a branch     memb status: EMPTY    memb size: EMPTY
```

This is worth more attention than the variable case, because the branch *condition* reads it too. `{{webResponse.status}}` **does not equal** `200` evaluates `EMPTY != 200` → **true**, so a branch meant to catch HTTP failures fires on every single run, reporting a failure that never happened. Nothing in the audit log mentions an unresolved smart value; the branch simply looks like it matched.

The corollary is a hard rule: **a branch condition may only read `{{issue.…}}`.** Anything a web request produced has to be written to a field in the trunk first.

**The general pattern: put the state on the work item.** Write it to a field, **Re-fetch work item data**, then read `{{issue.…}}` anywhere — including in later rules. Fields are global scope, survive the rule ending, and leave an inspectable record of what the rule decided months later.

Pack several response facts into one field rather than minting a field each. A status and a result count as `{{webResponse.status}}:{{webResponse.body.size}}` gives `200:0`, `200:1`, `401:` — every branch you need, off one text field, using `contains` and `equals`:

```
Edit work item → <check field> → {{webResponse.status}}:{{webResponse.body.size}}
Re-fetch work item data
branch 1:  {{issue."<check field>"}}  does not contain  200:     ← the call failed
branch 2:  {{issue."<check field>"}}  does not equal    200:0    ← it succeeded, result non-empty
```

Before building that, ask whether the status needs to reach a branch at all. Set the request's non-2xx checkbox to **fail the rule** and a failure goes to the audit log and the rule owner's error notification on its own — then the field carries only the result, and the status branch disappears with it. Worth doing whenever the failure is an admin's problem (a bad token, a revoked scope) rather than something the flow can route per work item. Check the rule's **Notify on error** setting is on, or that failure reaches nobody.

An **Assets object field can be set from an AQL query** in Edit work item, which replaces the Lookup objects action outright:

```
Edit work item → <assets field> → objectTypeId = 46 AND "email-domains" LIKE ",{{issue.customfield_10312}},"
Re-fetch work item data
{{issue.customfield_10246."jsm-organization-id"}}     reads an attribute off the match
{{issue.customfield_10246.size}}                      the field is MULTI-VALUE — check this
```

That field holds every matching object, so an AQL matching two objects assigns neither cleanly. Branch on `.size`, not on whether it is empty.

**Quote any field name containing a space**: `{{issue."Initial Reporter Email Domain"}}`. Unquoted it renders empty, like every other parser failure here.

**But never inside an AQL or JQL string literal.** `LIKE ",{{issue."Some Field"}},"` puts a `"` inside a `"…"`, and AQL accepts **double quotes only** — single quotes are a syntax error, verified against a live workspace, so there is no alternative quote to escape with. Use the `customfield_NNNNN` form there instead; it needs no quotes at all. The two forms are otherwise interchangeable, which is why this only bites in the one place where the field reference is wrapped in a string.

## Actions worth knowing

| Action | Why it matters |
|---|---|
| **Log action** | prints to the audit log — the only real smart-value debugger |
| **Create variable** | defines a reusable value; **always returns a string** |
| **Lookup issues** | JQL search, **max 100 results**, exposed as `{{lookupIssues}}` |
| **Re-fetch work item data** | refreshes `{{issue}}`; required after any step whose result a later step must read |
| **Send web request** | outbound HTTP; enable *Wait for response* for `{{webResponse}}` |
| **Re-fetch alert data** | same as re-fetch, for `{{alert}}`; runs in the background |
| **Transition work item** | needs a transition from the current status to the target and does not route through other statuses, so a move over two transitions is two steps (assumed from the workflow; the two-step version passed its run, 2026-11). Rules act after the event, so none can block a transition; use a workflow validator |

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

### Billing: steps from 2026-12-03

Automation is metered in **steps** (docs, checked 2026-10): every trigger, condition, action and branch that runs counts. JSM Standard includes 3,000 steps per agent per month, pooled across the organisation. Past that, extra usage costs $0.50 per 1,000 steps if it is switched on; otherwise rules pause. Single-project rules count too. The model before counted executions (1,700 a month on Jira Standard) and left single-project rules out ([service limits](https://support.atlassian.com/cloud-automation/docs/automation-service-limits/)), so older designs that relied on that need rechecking.

Cheap rules: conditions ahead of actions, filters on the trigger, schedules no more often than needed, branches only where the flow forks.

## Permissions and the rule actor

Rules run as the **rule actor** — by default *Automation for Jira*, changeable per rule. Everything the rule does is subject to that user's permissions: browse, edit, transition, field-level visibility, and issue security.

Symptoms of an actor permission problem: rule reports success, nothing changed; fields come back empty that are populated in the UI; `createmeta`/`editmeta` omits a field you can see.

- **The actor and issue security.** *Automation for Jira* gets its project access through the `atlassian-addons-project-access` project role. A security level that grants neither that role nor the actor hides its work items from the rule, and setting a level needs *Set Issue Security* for the role ([KB](https://support.atlassian.com/automation/kb/automation-or-set-issue-security-level-via-automation-for-jira/)). Granting *Set Issue Security* only to admins and that role lets rules set levels that agents cannot (run, 2026-11). On JSM, a level hides a request from its own reporter unless it grants Reporter ([KB](https://support.atlassian.com/jira/kb/a-service-management-request-is-not-visible-in-the-customer-portal-due-to-a-security-level-set-in-the-request/)), and it cannot grant request participants at all ([KB](https://support.atlassian.com/jira/kb/set-up-issue-security-for-request-participants-using-automation/)).
- **A rule scoped to more than one space needs Administer Jira** to create or edit (docs, checked 2026-10).

## Debugging checklist

1. **Audit log first** — every execution, its outcome, and per-action detail. Filter by rule.
2. **Log action** with the suspect smart value, on a manual trigger. A scheduled rule runs at once with **Run rule** in the editor.
3. **Stale data?** Insert Re-fetch work item data.
4. **Empty value?** Add a `|"fallback"` default and re-run to distinguish "empty" from "wrong path".
5. **Permission?** Temporarily switch the rule actor to a site admin to confirm, then fix properly.
6. **Team-managed project?** Several components behave differently or are unavailable — see `limitations-in-team-managed-projects-for-automation-rules`. Story point field names differ: `Story Points` (company-managed) vs `Story point estimate` (team-managed).

## Import / export

Export produces a JSON file (individual rule, or all rules at once). Import in the UI takes that file. Separately, a **public REST API** now lists, creates, updates and deletes rules in the same JSON format; see [rest-api.md](rest-api.md).

Constraints that bite:
- The JSON must come from the **same version** of Jira automation or the import fails
- Max **5 MB**
- Imported rules arrive **disabled** and must be enabled manually
- Data Center and Cloud formats differ — cross-import commonly throws `MismatchedInputException`

Practical consequence: to script rules, take each component's shape from a rule read back from the site (an export, `GET /rule/<uuid>`, or a throwaway probe rule built in the editor), never from memory. The endpoints are documented; the component values are not. [rest-api.md](rest-api.md) has a catalogue of verified shapes.
