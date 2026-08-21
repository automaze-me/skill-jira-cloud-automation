# skill-jira-cloud-automation

A Claude Code / Agent Skill for building **Jira Cloud automation rules** in the UI editor — with the smart-value reference and data-manipulation syntax at hand instead of half-remembered.

The UI rule editor is weakest exactly where automation gets useful: composing JSON for advanced field editing, calling REST APIs and parsing the response, and chaining text/list/date functions. This skill exists to hand over exact, copy-pasteable smart values for that work.

## What's in it

| File | Covers |
|---|---|
| `SKILL.md` | Routing, the 2026 rules→flows rename, and the facts that are commonly guessed wrong |
| `references/smart-values-core.md` | Syntax, root objects, iteration, defaults, variables, testing loop |
| `references/text-and-lists.md` | Text functions, list functions, conditional logic |
| `references/numbers-and-dates.md` | Math expressions, numeric formatting, the full date/time surface |
| `references/json-and-rest.md` | JSON functions, advanced field editing per field type, Send web request |
| `references/attachments.md` | Attachment metadata, reading file contents, `.eml` handling and its limits |
| `references/rovo.md` | AI steps: Use Rovo vs Use Rovo agent, response smart values, JSON parsing, credits |
| `references/rule-structure.md` | Triggers, conditions, branches, advanced steps, service limits, debugging |
| `references/recipes.md` | Worked end-to-end patterns |

## Install

Standalone:

```bash
git clone https://github.com/automaze-me/skill-jira-cloud-automation.git \
  .claude/skills/jira-cloud-automation
```

On Joern's machine it is packaged as the `jira-automation` skill pack — `"install the jira-automation pack here"`.

Skills are discovered at session start, so start a new Claude Code session after installing.

## Sources

Distilled from Atlassian's Cloud automation documentation (`support.atlassian.com/cloud-automation/docs/…`), across ~45 pages covering smart values, advanced field editing, rule components, and service limits. Each reference file names the specific doc slugs it came from.

Written against the state of the docs in August 2026, at which point Atlassian had renamed rules → flows, components → steps, issues → work items and projects → Jira spaces in the UI while leaving the smart values (`{{issue}}`, `{{lookupIssues}}`) on the old vocabulary.

## Caveats

- Smart values can only be validated at runtime. Verify anything load-bearing with a **Log action** on a manual trigger before shipping.
- `{{attachment.content}}` is a download URL, not file contents — the smart-values doc wording implies otherwise.
- The web-request response smart value is `{{webResponse}}`. Atlassian's docs long spelled it `{{webhookResponse}}` too; that alias stopped populating in August 2026 and renders empty rather than erroring.
- Field names differ between company-managed and team-managed projects (`Story Points` vs `Story point estimate`).

## Licence

MIT
