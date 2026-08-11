# Rovo AI steps

Source: `automation-smart-values-rovo-agents`, Atlassian KB
`rovo-automation-rule-fails-to-update-fields-from-json-response`,
`rovo/docs/rovo-usage-limits`, the community announcement
*Introducing the Use Rovo action in Automations*, and issues AUTO-1873 /
AUTO-1897 / ROVO-101 / ROVO-145.

> **Status: documentation-derived, not bench-verified.** Unlike
> [attachments.md](attachments.md), nothing here has been built and run end to
> end. Treat the smart-value names as reliable (they come from the official
> reference) and the traps as reported-by-others until confirmed. Test with a
> Log action before designing around any of it.

**The headline: there are two different AI steps, they populate two different smart values, and the JSON-parsing idiom is different for each.** Picking the wrong pairing returns an empty string with no error.

## The two actions

| | **Use Rovo** | **Use Rovo agent** |
|---|---|---|
| What it is | Self-contained AI step | Invokes a configured Rovo agent |
| Input | A prompt box: your instructions plus whatever data you pass in via smart values | The agent's own configuration and knowledge sources |
| Output | `{{rovoResponse}}` | `{{agentResponse}}` (+ variants below) |
| Can reach | Nothing. No internet, no projects, no spaces, no pages, no actions | Knowledge sources, plugins, connected third-party data |
| Menu location | **Rovo AI** category, top of the action list | same |

Both need a following step to do anything with the result — neither writes to the work item on its own.

### Choosing between them

**Use Rovo** for a pure transformation in the middle of a flow: summarise, translate, extract, classify, rewrite. You already hold the data, you just want a model applied to it. Its inability to reach anything is a security property, not a shortcoming — the step sees exactly the smart values you interpolate and nothing else, which is what makes it defensible for content of unknown provenance.

**Use Rovo agent** when the step genuinely needs to go and find something, or when a curated agent persona already exists and you want the flow to reuse it.

Reaching for the agent action to give a transformation step "more context" is usually the wrong trade: you inherit the agent's retrieval surface and its failure modes for no gain.

## `{{agentResponse}}` variants (Use Rovo agent)

| Smart value | Returns |
|---|---|
| `{{agentResponse}}` | rich text, markdown formatting, the default |
| `{{agentResponse.markdown}}` | same as default, explicit |
| `{{agentResponse.adf}}` | Atlassian Document Format as JSON. Use where ADF is accepted |
| `{{agentResponse.wiki}}` | legacy Jira wiki markup — **being phased out** in favour of ADF |
| `{{agentResponse.asObject}}` | raw JSON, for extracting values or feeding an API call |
| `{{agentResponse.asList}}` | raw JSON as a list, for extracting inside a branch step |
| `{{agentResponse.asString}}` | plain text, for string manipulation |

`{{rovoResponse}}` has **no documented variants**. Do not assume `.asObject` exists on it.

## Parsing a JSON answer

The idiom splits by action:

```
Use Rovo agent   {{agentResponse.asObject.category}}
Use Rovo         {{jsonStringToObject(rovoResponse).category}}
```

See [json-and-rest.md](json-and-rest.md) for `jsonStringToObject` generally.

**Instruct the model to emit bare JSON.** A response wrapped in triple-backtick fences is no longer parseable, and this is common enough that Atlassian has a dedicated KB page for rules failing to update fields from a JSON response. Put it in the prompt explicitly:

```
Return only the raw JSON object. No prose, no explanation,
no markdown code fences.
```

Keep the schema flat and the field names boring. Nested objects and arrays parse, but every level you add is another place a malformed answer silently yields empty.

## Traps

### 1. Text functions on the response return blank

`{{agentResponse.length()}}` and every other text function return nothing — not an error, an empty string. Reported in AUTO-1897, closed as a duplicate of **AUTO-1873** (open, medium priority).

**Workaround:** store the response with **Create variable** first, then operate on the variable.

```
Create variable   name: verdict   value: {{rovoResponse}}
… then use {{verdict.match(...)}}, {{verdict.length()}} etc.
```

Do this unconditionally. It costs one step and removes a class of silent failure. Note the [smart-values-core.md](smart-values-core.md) caveat that Create variable always stringifies — fine here, since you want text.

### 2. The wrong smart value for the action

`{{agentResponse}}` after a Use Rovo step, or `{{rovoResponse}}` after a Use Rovo agent step, renders empty. Same symptom as every other missing value, so it does not announce itself. Check which action is actually in the flow before debugging the prompt.

### 3. Generative output is not validated output

ROVO-101 (fixed, Oct 2025) had agents emitting confidently wrong Confluence URLs — correct information, hallucinated link. Fixed, but the lesson generalises: anything the step returns that will be **acted on** rather than displayed deserves a condition between the AI step and the action. Branch on an enum you defined, not on free text. Never let a generated URL, account ID, or issue key reach an action unchecked.

### 4. Error handling inside the flow is thin

**ROVO-145**, improving Rovo error handling within automation, is open. Decide what the flow does when the step fails or returns something unparseable, and build that path explicitly. A default of "fall through to the human queue" beats a rule that silently proceeds on an empty verdict.

## Credits and plan limits

Rovo is metered in **credits from a shared organisation pool**, not per-call currency.

| | |
|---|---|
| Chat and agent request | 10 credits |
| Deep Research | 100 credits |
| Rovo Search | free |

| Plan | Credits / seat / month |
|---|---:|
| Standard | 25 |
| Premium | 70 |
| Enterprise | 150 |

Credits pool across the org, so the practical ceiling is seats × allowance. There are no overage charges at time of writing; credits and variable rates began surfacing in the Atlassian Administration dashboard in August 2026.

**The consequence for flow design:** a rule that calls an AI step on every matching work item is spending from the same pool that funds interactive Rovo use across the company. A high-volume trigger can quietly drain it. Estimate `runs/month × 10` against `seats × allowance` before shipping, and gate the step behind conditions so it only fires when its answer will actually be used.

> Unverified: whether the **Use Rovo** action consumes credits at the same 10-credit rate as an agent request. The published rate names chat and agent requests specifically. Confirm against the admin dashboard before sizing a high-volume rule.

## Where this stops

- **No streaming and no partial results** — the flow waits for the whole answer.
- **No conversation state** between runs. Each invocation is independent; pass anything it needs to remember through the prompt.
- **Prompt size is bounded by smart-value rendering**, so a large interpolated body can be truncated upstream of the model without a warning. Log the rendered prompt once with a Log action when the input is user-supplied text of unpredictable length.
- **Determinism is not on offer.** Two runs on identical input can differ. Anything downstream that requires stability should branch on a constrained enum and treat free text as advisory.

For work needing decoding, external lookups, or a model you can pin and version, offload to Forge, ScriptRunner, or an HTTP endpoint and post the result back — the pattern in [attachments.md](attachments.md#offloading).
