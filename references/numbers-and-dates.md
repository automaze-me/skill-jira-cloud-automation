# Numbers, math expressions, dates and times

Source: `jira-smart-values-math-expressions`, `jira-smart-values-date-and-time`,
`examples-of-using-math-expression-smart-values`, `examples-of-using-smart-values-with-dates`.

## Two ways to do arithmetic

**Method form** — works inline anywhere:

```
{{issue.Story Points.plus(issue.subtasks.size)}}
{{issue.timetracking.timeSpentSeconds.divide(3600).round}}
```

**Expression form** — required for bare operators, wrapped in `{{#=}}…{{/}}`:

```
{{#=}}{{issue.Invoice Amount}} * 1.2{{/}}
{{#=}}{{issue.Invoice Amount}} + 100{{/}}
```

`{{issue.Amount * 1.2}}` without the wrapper produces nothing. This is the single most common numeric mistake.

## Numeric methods

| Method | Meaning |
|---|---|
| `plus(n)` / `minus(n)` | add / subtract |
| `multiply(n)` / `divide(n)` | multiply / divide |
| `abs` | absolute value |
| `round` | nearest whole number |
| `floor` / `ceil` | down / up to integer |

```
Projected growth is off by {{issue.Projected growth.minus(issue.Actual growth).abs}}
We spent {{issue.timetracking.timeSpentSeconds.divide(3600).divide(issue.Request participants.size)}} hours per person.
{{now.plusBusinessDays(issue.Story Points.multiply(3))}}
```

## Numeric comparisons

Return true/false; pair with `{{#if}}`.

| Method | Meaning |
|---|---|
| `gt(n)` | greater than |
| `gte(n)` | greater than or equal |
| `eq(n)` | equal |
| `lt(n)` | less than |
| `lte(n)` | less than or equal |

```
{{#if(issue.Votes.gt(100))}} Popular issue {{/}}
{{#if(issue.Request Participants.size().eq(1))}} Only one person involved {{/}}
{{#if(issue.Multi Select custom field.size().lt(2))}} Only one option selected {{/}}
```

## Number formatting

| Method | Effect |
|---|---|
| `format` | US locale grouping — `123412345` → `123,412,345` |
| `format("###")` | Java `DecimalFormat` pattern |
| `formatWithLocale("fr_FR")` | `123 412 345` |
| `asPercentage` | `0.123` → `12%` |
| `asPercentage("fr_FR")` | localized |
| `asCurrency` | `0.123` → `$0.12` |
| `asCurrency("fr_FR")` | `0,12€` |

## Inside `{{#=}}` — operators, functions, constants

Operators: `+` `-` `*` `/` `%` (modulo) `^` (power).

Boolean operators evaluate to `1` or `0`: `=` `==` `!=` `<>` `<` `<=` `>` `>=` `&&` `||`. Any non-zero value is true.

Functions (case-insensitive): `NOT(e)`, `IF(cond, ifTrue, ifFalse)`, `RANDOM()`, `MIN(e1,e2,…)`, `MAX(e1,e2,…)`, `ABS(e)`, `ROUND(e, precision)`, `FLOOR(e)`, `CEILING(e)`, `LOG(e)`, `LOG10(e)`, `SQRT(e)`, `SIN/COS/TAN(e)`, `ASIN/ACOS/ATAN(e)`, `SINH/COSH/TANH(e)`, `RAD(e)`, `DEG(e)`.

Constants: `e`, `PI`, `TRUE`, `FALSE`, `NULL`.

`IF()` inside a math expression is the closest thing to a numeric ternary:

```
{{#=}}IF({{issue.Story Points}} > 8, 2, 1){{/}}
```

## Dates

### `{{now}}`

Current date/time in **UTC+00:00**: `2020-08-18T02:40:37.0+0000`.

### Formatting — append to any date value

| Format | Output for Thu, 1 Nov 1979 06:23:12 UTC |
|---|---|
| *(none)* | `1979-11-01T11:23:12.0+0000` |
| `jiraDate` | `1979-11-01` |
| `jiraDateTime` | `1979-11-01T06:23:12.0-0500` |
| `jqlDate` | `1979-11-01` |
| `jqlDateTime` | `1979-11-01 06:23` |
| `shortDate` / `shortTime` / `shortDateTime` | `11/1/79` / `6:23 AM` / `11/1/79 6:23 AM` |
| `mediumDate` / `mediumTime` / `mediumDateTime` | `Nov 1, 1979` / `6:23:12 AM` / `Nov 1, 1979 6:23:12 AM` |
| `longDate` / `longTime` / `longDateTime` | `November 1, 1979` / `6:23:12 AM EST` / `November 1, 1979, 6:23:12 AM UTC` |
| `fullDate` / `fullTime` / `fullDateTime` | `Thursday, November 1, 1979` / … |
| `format("dd/MM/yyyy")` | `01/11/1979` — Java pattern syntax |

```
{{now.jiraDate}}
{{issue.created.format("dd/MM/yyyy")}}
{{issue.updated.shortTime}}
```

Use `jqlDate` / `jqlDateTime` when the value is being injected into a JQL string.

### Arithmetic

`plus[Unit](n)` and `minus[Unit](n)` where Unit ∈ `Millis, Seconds, Minutes, Hours, Days, Weeks, Months, Years, BusinessDays`.

```
{{now.plusHours(1).shortTime}}
{{now.minusWeeks(7)}}
{{sprint.startDate.plusDays(14)}}
```

**Business days are Monday–Friday, 9am–6pm.** No holiday calendar.

For *date-and-time* custom fields, only `Millis`, `Seconds`, `Minutes`, `Hours` are supported.

### Differences

```
{{[date1].diff([date2]).[unit]}}
```

Units: `millis, seconds, minutes, hours, days, weeks, months, years, businessDays`.
Modifiers: `.abs` (positive), `.prettyPrint` (`2 days 3 hours`).

Negative when `date2` is earlier than `date1`:

```
{{now.diff(issue.created).weeks}}            → -4
This issue was created {{now.diff(issue.created).weeks.abs}} weeks ago.
```

### Comparisons

| Expression | Returns |
|---|---|
| `{{d1.isAfter(d2)}}` | true / false |
| `{{d1.isBefore(d2)}}` | true / false |
| `{{d1.isEquals(d2)}}` | true / false |
| `{{d1.compareTo(d2)}}` | `<0` if d1 earlier, `>0` if d2 earlier, `0` if equal |

### Time zones — the trap

```
{{now.setTimeZone("Australia/Sydney")}}
// 03:17 UTC → 03:17+1000   — relabels, same wall clock

{{now.convertToTimeZone("Australia/Sydney").shortTime}}
// 03:17 UTC → 1:17 PM      — actually converts
```

Use canonical IANA IDs (`Europe/Berlin`, not `CET`).

### Setting components

`with[Attribute](value)`: `withYear`, `withDayOfMonth`, `withDayOfYear`, `withHour`, `withMinute`, `withMonthOfYear`, `withNextDayOfWeek("MON")`…

```
{{now.withYear(2005)}}
{{now.withNextDayOfWeek("MON").withHour(14).withMinute(0)}}   → next Monday 2pm
```

Readable attributes: `millis, second, minute, hour, dayOfMonth, monthOfYear, year, dayOfYear, dayOfWeek, dayOfWeekName, dayOfWeekShortName, weekOfYear, weekOfYearIso, zoneId, toDays, isLeapYear, lengthOfMonth, lengthOfYear, timeZoneShort, timeZoneFull, timeZoneNarrow, offsetShort, offsetFull, offsetNarrow`.

### Calendar helpers

| Expression | Returns |
|---|---|
| `{{date.toBusinessDay}}` | next business day (or the same day if already one) |
| `{{date.toBusinessDayBackwards}}` | previous business day |
| `{{date.startOfMonth}}` / `{{date.endOfMonth}}` | month boundaries |
| `{{date.firstBusinessDayOfMonth}}` / `{{date.lastBusinessDayOfMonth}}` | |
| `{{date.firstOfTheMonth(n)}}` | first Mon(1)…Sun(7) of the month |
| `{{date.lastOfTheMonth(n)}}` | last Mon(1)…Sun(7) of the month |
| `{{date.ofTheMonth(n, day)}}` | nth given weekday — `ofTheMonth(2, 1)` = 2nd Monday |
| `{{date.weekOfYear}}` / `{{date.weekOfYearIso}}` | traditional / ISO week number |
| `{{now.toStartOfDay}}`, `{{now.toCurrentTime}}`, `{{now.toDateTimeAtStartOfDay(tz)}}`, `{{now.toDateTimeAtCurrentTime(tz)}}` | day-boundary conversions |

Day numbering for `firstOfTheMonth` / `lastOfTheMonth` / `ofTheMonth`: 1 = Monday … 7 = Sunday.
`withNextDayOfWeek` uses names instead: `MON, TUE, WED, THU, FRI, SAT, SUN`.
