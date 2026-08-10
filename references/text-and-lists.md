# Text and list functions

Source: `jira-smart-values-text-fields`, `jira-smart-values-lists`,
`jira-smart-values-conditional-logic`, `examples-of-using-smart-values-with-text-strings`,
`examples-of-using-smart-values-with-lists`, `convert-wiki-markup-to-html-or-plain-text-in-jira-automation`.

All text examples below assume `issue.summary` = `Hello World!`.

## Text functions

### Extracting

| Function | Example | Result |
|---|---|---|
| `left(int)` | `{{issue.summary.left(5)}}` | `Hello` |
| `right(int)` | `{{issue.summary.right(6)}}` | `World!` |
| `substring(start)` | `{{issue.summary.substring(7)}}` | `orld!` |
| `substring(start, end)` | `{{issue.summary.substring(1,3)}}` | `el` |
| `substringAfter(sep)` | `{{issue.summary.substringAfter("W")}}` | `orld!` |
| `substringAfterLast(sep)` | `{{issue.summary.substringAfterLast("o")}}` | `rld!` |
| `substringBefore(sep)` | `{{issue.summary.substringBefore("W")}}` | `Hello ` |
| `substringBeforeLast(sep)` | `{{issue.summary.substringBeforeLast("o")}}` | `Hello W` |
| `substringBetween(open, close)` | `{{issue.summary.substringBetween("e","d")}}` | `llo Worl` |
| `charAt(int)` | `{{issue.summary.charAt(0)}}` | `H` |
| `split(sep)` | `{{issue.summary.split(" ").first}}` | `Hello` |

`split()` returns a **list** — chain `.first`, `.last`, `.get(n)`, or iterate it.

### Transforming

| Function | Example | Result |
|---|---|---|
| `toLowerCase()` / `toUpperCase()` | `{{issue.summary.toLowerCase()}}` | `hello world!` |
| `capitalize()` | `{{issue.summary.capitalize()}}` | `Hello world!` |
| `trim()` | strips leading/trailing whitespace | |
| `concat(str)` | `{{issue.summary.concat(" Bye")}}` | `Hello World! Bye` |
| `replace(target, repl)` | `{{issue.summary.replace("Hello","Goodbye")}}` | `Goodbye World!` |
| `replaceAll(regex, repl)` | `{{issue.summary.replaceAll("(lo)","xx$1yy")}}` | `Helxxloyy World!` |
| `remove(str)` | `{{issue.summary.remove("l")}}` | `Heo Word!` |
| `reverse()` | | `!dlroW olleH` |
| `abbreviate(int)` | `{{issue.summary.abbreviate(8)}}` | `Hello Wo...` |
| `leftPad(len, pad)` | `{{issue.summary.leftPad(14,"-")}}` | `--Hello World!` |
| `rightPad(len, pad)` | | `Hello World!--` |

`replaceAll` uses Java regex; `$1` references a capture group in the replacement.

### Inspecting — these return true/false and belong in conditional logic

| Function | Notes |
|---|---|
| `length()` | character count |
| `indexOf(str)` / `lastIndexOf(str)` | position, or -1 |
| `startsWith(str)` / `endsWith(str)` | |
| `isEmpty()` / `isNotEmpty()` | |
| `isAlpha()` / `isAlphanumeric()` / `isNumeric()` | |
| `equals(str)` / `equalsIgnoreCase(str)` | |
| `asNumber` | parses to number, or `null` if it can't |

### Regex matching

`match()` runs a Java regex and returns the **first capture group**. If the pattern matches multiple times, it returns a *collection* you can iterate.

```
{{issue.summary.match(".*(lo).*")}}   → lo
{{issue.summary.match(".*(o).*")}}    → [o, o]
```

Typical uses: pull a version number out of a description, or harvest every email address from a comment.

> **Backslashes do not survive the smart-value parser.** `\n`, `\t`, `\R`, `\s`, `\w`, `\d`, `\p{…}` all return empty inside `match()` and `replaceAll()`, and doubling them (`\\w`) does not help. Verified 2026-08-10.
>
> Write patterns with backslash-free constructs — character classes and inline flags cover most of it:
>
> | Instead of | Use |
> |---|---|
> | `\d` | `[0-9]` |
> | `\w` | `[A-Za-z0-9_]` |
> | `\s` | `[ ]`, or a line-anchored `(?m)^` |
> | `\n` as a terminator | `(?m)$`, or a `(?=^…)` lookahead |
>
> Inline flags and lookarounds work fine: `(?m)` `(?s)` `(?i)` `(?=…)` `(?!…)`.
>
> The line-anchored lookahead is the general escape from needing to name a newline — it captures across wrapped lines and stops at the next line starting with a letter:
>
> ```
> {{someText.match("(?sm)^Key: (.*?)(?=^[A-Za-z])")}}
> ```

`quote()` escapes a value so it can be used as a *literal* inside another regex — `\QHello(.*)World!\E`. Use it whenever you interpolate user data into a `match()` or `replaceAll()` pattern.

### Encoding

Four encoders, each with a function form and an inline form. **Prefer the function form.**

```
{{htmlEncode(issue.summary)}}          HTML — R&D → R&amp;D
{{#jsonEncode}}{{issue.summary}}{{/}}  JSON — "Hi" → \"Hi\"
{{#urlEncode}}{{issue.summary}}{{/}}   URL  — Hello & World → Hello+%26+World
{{#xmlEncode}}{{issue.description}}{{/}} XML — & → &amp;
```

Inline equivalents exist (`{{issue.summary.jsonEncode}}`) but are unsafe on payload objects: if the object has a property literally named `htmlEncode`, `{{webhookData.htmlEncode}}` returns *that property* instead of encoding anything.

### Wiki markup

Text fields containing wiki markup can be rendered — see `convert-wiki-markup-to-html-or-plain-text-in-jira-automation` for the HTML/plain-text conversion action.

## List functions

| Function | Example |
|---|---|
| `size` | `{{issue.comments.size}}` |
| `isEmpty` | `{{issue.issuelinks.isEmpty}}` |
| `first` / `last` | `{{issue.comments.first.body}}` |
| `get(index)` | `{{lookupIssues.get(0).summary}}` — 0-based |
| `getFromEnd(index)` | `{{issue.comments.getFromEnd(1).body}}` — 0 = last |
| `join(separator)` | `{{issue.fixVersions.name.join(" - ")}}` |
| `distinct` | `{{lookupIssues.parent.distinct}}` |
| `sum` / `average` | `{{issue.subtasks.Story Points.sum}}` |
| `min` / `max` | works on numbers **and** dates: `{{issue.subtasks.Due date.max}}` |
| `flatten()` | `{{webhookData.yourNestedList.flatten()}}` → `[1,2,3,4,5]` |

Accessing a property across a list maps it: `{{issue.fixVersions.name}}` yields all names.

### Section modifiers

Inside a `{{#list}}…{{/}}` block:

| Modifier | Meaning |
|---|---|
| `{{#first}}…{{/}}` | only on the first element |
| `{{^first}}…{{/}}` | on every element **except** the first |
| `{{#last}}…{{/}}` | only on the last element |
| `{{^last}}…{{/}}` | on every element except the last |
| `{{index}}` | current 0-based index |
| `{{.}}` | the current item itself |

If the element itself has a property called `first`/`last`/`index`, use `_first`, `_last`, `_index`.

Comma-separated list with no trailing comma — the canonical use of `{{^last}}`:

```
{{#issue.comments}}{{body}}{{^last}}, {{/}}{{/}}
```

Numbered list:

```
{{#issue.comments}}{{index}}. {{body}}{{^last}}
{{/}}{{/}}
```

Prefix every label and comma-separate:

```
{{#issue.labels}}option-{{.}}{{^last}},{{/}}{{/}}
```

## Conditional logic

These live inside text-producing actions (Add comment, Send email, Send Slack message, Log action).

### `if`

```
{{if(smartValue)}}                              → true or null
{{if(smartValue, "Hello")}}                     → Hello or null
{{if(smartValue, "if true", "if false")}}       → either branch
{{#if(condition)}}block text{{/}}               → block form
```

### Operators

| Function | Returns |
|---|---|
| `{{equals(a, b)}}` | true / false — takes strings or smart values |
| `{{exists(value)}}` | true / null |
| `{{not(value)}}` | true / false |
| `{{and(a, b)}}` | true / false |
| `{{or(a, b)}}` | true / false |

### Worked examples

Comment only when the item has links:

```
{{#if(not(issue.issuelinks.isEmpty))}}
This bug has {{issue.issuelinks.size}} related tickets
{{/}}
```

Flag self-assignment:

```
{{#if(equals(issue.assignee, issue.reporter))}}
Assign this issue to someone else so they can review.
{{/}}
```

Guard on an optional custom field:

```
{{#if(exists(issue.designer))}}
The designer for this issue is {{issue.designer.displayName}}
{{/}}
```

Numeric range with `and`:

```
{{#if(and(issue.Votes.gt(100),issue.Votes.lt(150)))}}
Moderately Popular issue
{{/}}
```

Either-or with `or`:

```
{{#if(or(issue.Votes.gt(100),issue.comments.gt(20)))}}
Heavily requested issue
{{/}}
```

> There is no `else`. Use the three-argument `{{if(cond, "a", "b")}}` form, or two complementary `{{#if}}` blocks with `not()`.
