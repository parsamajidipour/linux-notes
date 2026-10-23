# Crontab Syntax and Time Matching

Cron syntax is compact enough to fit on one line, which is exactly why it is easy to underestimate. A crontab entry such as:

```cron
15 2 * * * /usr/local/sbin/backup
```

looks like little more than five numbers followed by a command. In reality, the five scheduling fields describe a set of calendar times, and the cron daemon repeatedly evaluates the current local time against that set. Small syntactic changes can therefore produce very different schedules. `*/15` does not mean "fifteen minutes after the previous run." `1-10/2` does not mean "every two units forever." `0 0 1 * 1` does not mean "the first Monday of every month" on traditional Vixie-style cron implementations. A percent sign in the command is not always an ordinary percent sign. A comment placed at the end of a job line may become part of the shell command instead of a comment.

The safest way to learn crontab syntax is to treat it as a matching language rather than as an English scheduling language. A crontab line defines conditions. Once per scheduling cycle, cron asks whether the current minute, hour, day, month, and weekday satisfy those conditions. If they do, the command becomes eligible to run.

That model is precise enough to explain nearly every scheduling surprise without relying on memorized phrases such as "every X minutes" or "every Monday."

---

## The shape of a user crontab entry

A normal user crontab job contains five time fields followed by the command:

```text
minute hour day-of-month month day-of-week command
```

For example:

```cron
30 4 * * * /home/alice/bin/report.sh
```

can be read mechanically:

```text
minute        = 30
hour          = 4
day-of-month  = any
month         = any
day-of-week   = any
command       = /home/alice/bin/report.sh
```

The schedule matches at 04:30 on every valid calendar day.

The conventional field ranges used by Vixie cron, Cronie, and compatible implementations are:

| Field | Typical numeric range | Meaning |
|---|---:|---|
| Minute | `0-59` | Minute within the hour |
| Hour | `0-23` | Hour of the day |
| Day of month | `1-31` | Calendar day within the month |
| Month | `1-12` | January through December |
| Day of week | `0-7` | Sunday through Saturday; both `0` and `7` commonly mean Sunday |

Month and weekday names are also accepted by many widely deployed cron implementations. The portable habit is to use the conventional three-letter English abbreviations when names are desired:

```cron
0 9 * JAN MON /usr/local/bin/example
```

Whether names are case-sensitive, whether longer names are accepted, and which nonstandard extensions exist depends on the implementation. A production schedule intended to move between Linux distributions should prefer syntax documented by the target cron implementation rather than assuming every extension is universal.

A system crontab such as `/etc/crontab` or a file under `/etc/cron.d/` usually contains one additional field:

```text
minute hour day-of-month month day-of-week user command
```

For example:

```cron
30 4 * * * backup /usr/local/sbin/backup
```

The `backup` token is not part of the command. It selects the account under which cron should execute the command. The same text would be incorrect in a user's personal crontab because a personal crontab already has an owner.

For syntax experiments in this chapter, assume a user crontab unless a system crontab is explicitly mentioned.

---

## A crontab schedule describes matching values, not elapsed intervals

Consider:

```cron
*/10 * * * * /usr/local/bin/poll
```

This is usually described as "run every ten minutes." Operationally that description is good enough, but the underlying behavior is more specific: the minute field matches the values divisible according to the step expression within the minute field's domain.

For a normal minute field, the matching values are effectively:

```text
0,10,20,30,40,50
```

The job therefore becomes eligible at times such as:

```text
08:00
08:10
08:20
08:30
08:40
08:50
09:00
```

Cron is not measuring ten minutes from the end of the previous command. It does not care whether the previous process ran for one second, nine minutes, or twenty minutes. The next matching wall-clock minute is still a match.

This distinction becomes visible with a long-running job:

```cron
*/5 * * * * /usr/local/bin/sync-data
```

Suppose one invocation starts at 12:00 and takes twelve minutes:

```text
12:00  process A starts
12:05  process B starts
12:10  process C starts
12:12  process A exits
12:15  process D starts
```

Nothing in the schedule means "wait five minutes after `sync-data` exits." Cron performs calendar matching, not interval scheduling relative to process completion.

This is one reason scheduling and concurrency must be treated as separate concerns. If overlapping execution is unsafe, the job itself needs locking or another concurrency-control mechanism. The scheduler expression alone does not provide it.

The same principle explains this entry:

```cron
0 */6 * * * /usr/local/bin/task
```

The hour field matches selected values in the 0-23 range. A typical interpretation produces:

```text
00:00
06:00
12:00
18:00
```

It does not mean "six hours after cron last managed to run this task." A reboot at 17:58 does not shift the next execution to 23:58. The next normal match remains 18:00.

Thinking in sets of matching clock values prevents a large class of incorrect assumptions.

---

## The wildcard selects the field's complete domain

The asterisk means that every valid value in a field is accepted.

The simplest possible schedule is:

```cron
* * * * * /usr/local/bin/task
```

The current minute can be any minute, the hour can be any hour, the day can be any day, the month can be any month, and the weekday can be any weekday. The entry therefore matches every minute.

A useful way to inspect a cron expression is to replace each wildcard mentally with the field's value set:

```text
minute:        0..59
hour:          0..23
day-of-month:  all valid days
month:         1..12
day-of-week:   all weekdays
```

Now compare:

```cron
0 * * * * /usr/local/bin/task
```

Only the minute field is restricted. It accepts exactly minute `0`, while every hour remains valid. The matches are:

```text
00:00
01:00
02:00
03:00
...
23:00
```

Compare again:

```cron
0 0 * * * /usr/local/bin/task
```

Now both minute and hour are fixed:

```text
00:00 every day
```

And:

```cron
0 0 1 * * /usr/local/bin/task
```

restricts the day of month as well:

```text
00:00 on the first calendar day of each month
```

The wildcard is simple, but it becomes important later when day-of-month and day-of-week interact. In traditional cron semantics, the difference between an unrestricted day field and a restricted day field affects how those two fields are combined.

---

## Lists select several discrete values

A comma-separated list allows one field to match several explicit values:

```cron
0 9,13,17 * * * /usr/local/bin/check
```

This job runs at:

```text
09:00
13:00
17:00
```

The list belongs to one field. It does not create separate cron entries; it simply expands the accepted values for that field.

Lists are useful when the schedule is irregular enough that a simple interval does not express it clearly. For example:

```cron
15 8,12,16,20 * * * /usr/local/bin/queue-report
```

is easier to audit than several duplicate job lines with identical commands.

Lists can often be combined with ranges and steps:

```cron
0 8-12,18-22 * * * /usr/local/bin/check
```

On compatible implementations this selects hours from 08 through 12 and from 18 through 22. Expanding the expression mentally gives:

```text
08:00 09:00 10:00 11:00 12:00
18:00 19:00 20:00 21:00 22:00
```

A good operational rule is to choose the representation that another administrator can verify quickly. Cron expressions are configuration, not a code-golf competition. A slightly longer list can be safer than a clever expression whose behavior is not obvious during an incident.

For example, both of the following may express the intended set on a compatible implementation:

```cron
0 0,6,12,18 * * * command
```

```cron
0 */6 * * * command
```

The step form is concise and communicates regularity. The list form makes the exact hours obvious. Neither is universally "better"; auditability is the important property.

---

## Ranges are inclusive

A hyphen defines a range of accepted values. In ordinary cron syntax, the endpoints are included.

Consider:

```cron
0 9-17 * * * /usr/local/bin/business-hour-task
```

The hour field accepts:

```text
9,10,11,12,13,14,15,16,17
```

so the task is eligible at the beginning of each hour from 09:00 through 17:00.

A weekday range is common:

```cron
0 9 * * 1-5 /usr/local/bin/weekday-task
```

Using the conventional weekday numbering where Monday is `1` and Friday is `5`, the schedule matches at 09:00 Monday through Friday.

Equivalent name-based syntax on implementations that support weekday names may be clearer:

```cron
0 9 * * MON-FRI /usr/local/bin/weekday-task
```

Ranges are not an arithmetic comparison evaluated against arbitrary numbers. They are part of the syntax of one field. That matters when people try to express wraparound ranges.

This looks attractive:

```cron
0 22-2 * * * command
```

with the intention of selecting 22:00, 23:00, 00:00, 01:00, and 02:00. Traditional cron syntax should not be assumed to interpret a descending range as wraparound. Express the set explicitly instead:

```cron
0 22,23,0,1,2 * * * command
```

or use two entries:

```cron
0 22-23 * * * command
0 0-2 * * * command
```

The second form can be easier to reason about because every range is monotonically increasing inside its field domain.

The same advice applies to weekday ranges crossing the end of the numbering cycle. If portability matters, avoid depending on undocumented wraparound behavior.

---

## Steps filter values inside a field

The slash introduces a step. The most familiar form is:

```cron
*/5 * * * * command
```

For the minute field, that selects every fifth value from the field's wildcard domain. The resulting minute set is:

```text
0,5,10,15,20,25,30,35,40,45,50,55
```

A step can also be applied to a range:

```cron
0 8-18/2 * * * command
```

The hour field starts with the inclusive range:

```text
8,9,10,11,12,13,14,15,16,17,18
```

and selects every second value beginning at the range's start:

```text
8,10,12,14,16,18
```

The command therefore runs at:

```text
08:00
10:00
12:00
14:00
16:00
18:00
```

The starting point is important. Compare:

```cron
0 1-23/2 * * * command
```

which selects odd hours:

```text
01:00 03:00 05:00 ... 23:00
```

with:

```cron
0 0-23/2 * * * command
```

which selects even hours:

```text
00:00 02:00 04:00 ... 22:00
```

A step is therefore better understood as "select every Nth value from this syntactic range" than as "wait N time units."

That distinction becomes especially important with fields whose domains do not divide neatly by the step.

Consider:

```cron
0 */7 * * * command
```

The hour domain is 0 through 23. A typical expansion is:

```text
0,7,14,21
```

The gap from 21:00 to the next day's 00:00 is three hours, not seven. The expression does not implement a continuously repeating seven-hour interval across day boundaries. It selects every seventh value from the hour field each day.

If the operational requirement is truly "start, then run again seven hours after each previous run" regardless of wall-clock boundaries, classic cron is not the correct abstraction. A systemd timer using monotonic timing, a long-running service, or application-level scheduling may fit better.

The same trap appears in the minute field:

```cron
*/17 * * * * command
```

Within each hour, the selected minutes are typically:

```text
00,17,34,51
```

Then the field resets with the next hour:

```text
00:51
01:00
```

Only nine minutes separate those executions. Calling this "every 17 minutes" is therefore mathematically misleading. It is more precise to say "at minutes 0, 17, 34, and 51 of every hour."

For steps that divide the field size evenly, the simple English phrase happens to match the resulting intervals. For steps that do not, the field-set model is essential.

---

## Combining fields is an intersection, with one famous exception

For minute, hour, and month, the schedule becomes more specific as restrictions are added. All of those fields need to match the current time.

Take:

```cron
30 8-18/2 * 1,6,12 * command
```

The current time must satisfy all of these conditions:

```text
minute = 30
hour ∈ {8,10,12,14,16,18}
month ∈ {January, June, December}
```

Any normal day is acceptable because the two day fields are unrestricted.

At 14:30 on June 20, the minute, hour, and month match, so the job is eligible.

At 14:31 on June 20, it does not match because the minute field fails.

At 14:30 on July 20, it does not match because the month field fails.

This is ordinary conjunction: the selected values across fields are effectively intersected to describe valid calendar moments.

The special case is the relationship between **day of month** and **day of week** in traditional Vixie-derived cron semantics. When both are restricted, they are commonly treated as an OR rather than an AND.

That rule deserves its own treatment because it causes one of the most common cron configuration errors.

---

## Day of month and day of week do not behave like most people expect

Suppose an administrator wants a report at 09:00 on the first Monday of every month and writes:

```cron
0 9 1 * 1 /usr/local/bin/report
```

A natural English reading is:

```text
09:00
AND day of month is 1
AND weekday is Monday
```

On traditional Vixie-style cron implementations, that is not the usual behavior. If both day fields are restricted, the entry generally matches when **either** day condition matches.

The practical result is closer to:

```text
09:00 on every first day of the month
OR
09:00 on every Monday
```

That can turn an intended monthly job into a weekly job plus an additional monthly execution.

For example, imagine a month where the first day is Thursday. The expression:

```cron
0 9 1 * 1 command
```

may match:

```text
Thu  1 09:00   because day-of-month = 1
Mon  5 09:00   because weekday = Monday
Mon 12 09:00
Mon 19 09:00
Mon 26 09:00
```

The schedule is therefore not a representation of "first Monday."

This behavior exists for historical compatibility and is explicitly documented by common cron implementations. It is sufficiently unintuitive that any production use of both fields should be reviewed carefully.

A robust way to express "first Monday of the month" is to let cron select Mondays and let the command validate the day-of-month condition:

```cron
0 9 * * 1 [ "$(date +\%d)" -le 07 ] && /usr/local/bin/report
```

Here cron restricts execution attempts to Mondays. The shell condition then permits the command only if the calendar day is between 01 and 07. Every month has exactly one Monday in that range.

Notice the escaped percent sign in `date +\%d`. In a crontab command, an unescaped `%` has special meaning on many traditional implementations. That behavior is covered later in this chapter.

Another approach is to move the calendar logic into a script:

```bash
#!/bin/sh

day=$(date +%d)

if [ "$day" -le 7 ]; then
    exec /usr/local/bin/report-real
fi
```

and schedule only Mondays:

```cron
0 9 * * 1 /usr/local/bin/first-monday-report
```

This is often more maintainable than embedding complex date logic in crontab, especially once public holidays, business calendars, or timezone rules are involved.

The day-field rule also matters in less obvious expressions. Compare:

```cron
0 3 15 * * command
```

Only day-of-month is restricted; weekday is unrestricted. The job runs at 03:00 on the fifteenth of each month.

Now compare:

```cron
0 3 * * 5 command
```

Only weekday is restricted. The job runs at 03:00 every Friday.

Now:

```cron
0 3 15 * 5 command
```

Both are restricted. On traditional implementations, the job runs on the fifteenth **and** on Fridays rather than only when the fifteenth happens to be Friday.

This is exactly the kind of expression that should receive a comment above it explaining the intended semantics, because a reviewer accustomed to ordinary Boolean conjunction may read it incorrectly.

---

## A wildcard is not merely cosmetic in the two day fields

Documentation for Vixie-derived cron often describes the day-of-month/day-of-week rule in terms of fields being "restricted." A common wording is that if both fields are restricted, the command runs when either matches.

That raises a subtle question: what does "restricted" mean for an expression such as:

```cron
*/2
```

It contains an asterisk, but it does not select every value once the step is applied.

The exact interpretation of syntactic restrictions and extensions is implementation-specific enough that clever combinations of stepped day fields should not be relied upon without testing the actual daemon in use. This is a good example of the boundary between the portable cron language and implementation behavior.

If a schedule depends on subtle Boolean relations between calendar fields, there are three safer strategies:

- express only the coarse candidate times in cron and put the exact condition in a script;
- use a scheduler with richer calendar expressions, such as systemd timers on Linux;
- test the expression against the exact cron implementation and version deployed, then document the dependency.

A one-line cron expression is not automatically simpler than a five-line shell script. Simplicity should be measured by how confidently the schedule can be understood and verified.

---

## Weekday numbers and Sunday's two conventional values

Many cron implementations use:

```text
0 = Sunday
1 = Monday
2 = Tuesday
3 = Wednesday
4 = Thursday
5 = Friday
6 = Saturday
7 = Sunday
```

Allowing both `0` and `7` for Sunday is traditional and convenient, but it can create awkward-looking ranges. A range such as:

```cron
0 9 * * 1-5 command
```

is obvious: Monday through Friday.

Weekend scheduling is often clearer as a list:

```cron
0 9 * * 0,6 command
```

or with names where supported:

```cron
0 9 * * SAT,SUN command
```

instead of depending on how a particular parser treats ranges around the Sunday boundary.

The names are especially useful in repositories and infrastructure code because they reduce the need to remember numbering conventions. The tradeoff is portability to minimal cron implementations that may support a smaller grammar. On mainstream Linux systems running Cronie or Debian-style cron packages, three-letter names are commonly supported; on constrained systems such as embedded BusyBox deployments, the target documentation should be checked.

When numeric weekdays are used in security-sensitive or high-impact jobs, a nearby comment can prevent maintenance errors:

```cron
# Monday through Friday at 06:30
30 6 * * 1-5 /usr/local/sbin/open-business-day
```

The comment documents intent, while the expression remains the source of truth that the scheduler evaluates.

---

## Month names are useful, but numeric months are easier to generate

The month field follows the same general selection grammar:

```cron
0 0 1 1 * command
```

runs at midnight at the start of January 1.

A named version may be easier to read:

```cron
0 0 1 JAN * command
```

Multiple selected months can be written as a list:

```cron
0 3 1 JAN,APR,JUL,OCT * /usr/local/bin/quarterly-task
```

This is a clear representation of a job that runs at 03:00 on the first day of January, April, July, and October.

A range is also possible on compatible implementations:

```cron
0 7 * MAR-OCT * /usr/local/bin/seasonal-task
```

For hand-written configuration, names improve readability. For generated configuration, numbers can be simpler because they avoid questions about locale, accepted abbreviations, and parser extensions.

Cron's month names should not be confused with the system locale used by `date`. A scheduler parser recognizing `JAN` is applying its own grammar; it is not necessarily asking libc to parse a localized month name. Do not assume that changing the machine locale lets a crontab use arbitrary translated month names.

---

## Calendar validity still matters

The day-of-month field may syntactically allow values up to `31`, but not every month contains 31 days.

Consider:

```cron
0 2 31 * * /usr/local/bin/month-end
```

This does not mean "the final day of each month." It means "02:00 when the calendar day is 31." The job will not run in months that do not have a thirty-first day.

Similarly:

```cron
0 2 30 * * command
```

does not run in February in an ordinary year, and:

```cron
0 2 29 2 * command
```

runs only when February 29 exists.

Classic cron has no portable `L` operator meaning "last day of month" like some other scheduling syntaxes do. If the requirement is "last day of every month," a common pattern is to run on candidate days and test whether tomorrow belongs to a different month.

For example, GNU `date` can be used inside a script:

```bash
#!/bin/sh

current_month=$(date +%m)
tomorrow_month=$(date -d tomorrow +%m)

if [ "$current_month" != "$tomorrow_month" ]; then
    exec /usr/local/bin/month-end-report
fi
```

The script can be scheduled daily:

```cron
0 2 * * * /usr/local/bin/run-if-month-end
```

Moving the condition into a named script makes testing easier:

```bash
bash -x /usr/local/bin/run-if-month-end
```

and allows the date arithmetic to be replaced with application-specific logic if the definition of "month end" later becomes "last business day" rather than last calendar day.

Trying to force business-calendar semantics into the five classic cron fields usually makes the configuration harder to verify.

---

## The command begins after the scheduling fields

In a user crontab, everything after the first five scheduling fields belongs to the command text, subject to cron's own handling of percent signs and the implementation's parsing rules.

This is valid:

```cron
0 3 * * * /usr/bin/find /srv/cache -type f -mtime +7 -delete
```

The spaces after the fifth field are part of the separation before the command. The command itself may contain normal shell syntax:

```cron
0 3 * * * /usr/local/bin/backup >>/var/log/backup.log 2>&1
```

or:

```cron
0 3 * * * cd /srv/app && /usr/bin/php artisan schedule:run
```

or:

```cron
*/5 * * * * /usr/bin/curl -fsS https://127.0.0.1:8443/health || /usr/local/bin/alert
```

On traditional cron implementations, the command is generally passed to a shell such as `/bin/sh -c` unless the `SHELL` environment variable changes the shell. That means shell operators such as `&&`, `||`, `>`, `2>&1`, pipelines, quoting, and variable expansion are interpreted by the shell rather than by cron itself.

This distinction is important for debugging. Cron parses the schedule. The shell parses most of the command.

For example:

```cron
0 3 * * * echo *.log
```

Cron does not expand `*.log`. The shell invoked for the command performs pathname expansion in the job's working directory. If the working directory or matching files differ from the interactive session, the result differs too.

Similarly:

```cron
0 3 * * * echo "$HOME"
```

contains shell variable expansion. Cron supplies an environment, and the shell expands `$HOME` when executing the command.

When a cron line becomes rich enough to contain several shell operators, conditionals, nested quoting, or complex substitutions, moving the logic to a script is usually an improvement. A crontab is easier to review when it answers "when" and delegates complicated "how" logic to version-controlled executable files.

---

## Inline comments are a trap

A line beginning with optional whitespace followed by `#` is normally a comment:

```cron
# Database backup at 02:30
30 2 * * * /usr/local/sbin/db-backup
```

The dangerous habit is assuming that shell-style end-of-line comments are always safe in crontab entries:

```cron
30 2 * * * /usr/local/sbin/db-backup # nightly backup
```

Traditional `crontab(5)` documentation warns that comments are not allowed on the same line as cron commands because the text may be treated as part of the command. Whether the shell later interprets the `#` as a shell comment can depend on how the command text reaches the shell and on quoting. For environment-setting lines, trailing text can similarly become part of the value.

The maintainable form is to put comments on their own lines:

```cron
# Nightly database backup
30 2 * * * /usr/local/sbin/db-backup
```

This also produces cleaner diffs and avoids accidental changes when a command ends with a quoted string containing `#`.

A useful repository convention is to write comments that capture **why** a schedule exists rather than merely translating the expression:

```cron
# Run before the 04:00 warehouse import; typical runtime is 10-15 minutes.
30 3 * * * /usr/local/sbin/export-orders
```

That comment provides operational context unavailable from the expression itself.

---

## Blank lines are insignificant, but formatting still matters

Blank lines can be used to group related jobs:

```cron
# Backups
0 2 * * * /usr/local/sbin/db-backup
30 2 * * * /usr/local/sbin/files-backup

# Maintenance
0 4 * * 0 /usr/local/sbin/prune-cache
15 4 * * 0 /usr/local/sbin/prune-temp
```

Whitespace between schedule fields is normally flexible. One or more spaces or tabs can separate fields. Alignment is therefore possible:

```cron
0   2   *   *   *   /usr/local/sbin/db-backup
30  2   *   *   *   /usr/local/sbin/files-backup
```

However, spacing inside the grammar should not be invented. This is not equivalent:

```cron
0 2 * * 1, 3, 5 command
```

The spaces after commas can cause the parser to see extra fields rather than one weekday list. Keep a field expression contiguous:

```cron
0 2 * * 1,3,5 command
```

Cron syntax is token-oriented. Human-friendly alignment is safe between fields; arbitrary spaces inside a field expression are not.

---

## Percent signs have special command semantics

One of the least remembered crontab rules is the treatment of `%` in the command field by traditional Vixie-derived cron implementations.

An unescaped percent sign can be transformed into a newline, and text after the first unescaped percent can be supplied to the command's standard input. This behavior historically made it possible to construct mail bodies directly in crontab, but today it is more often encountered as a surprising failure mode.

Consider an apparently ordinary command:

```cron
0 0 * * * /bin/date +%F >> /tmp/date.log
```

A user might expect the shell to run:

```bash
/bin/date +%F >> /tmp/date.log
```

but under cron implementations with traditional percent handling, the `%` can be consumed by the crontab command parser before the shell sees the command as intended.

The safe form is:

```cron
0 0 * * * /bin/date +\%F >> /tmp/date.log
```

Likewise:

```cron
0 0 * * * /bin/date '+\%Y-\%m-\%d' >> /tmp/date.log
```

The backslash protects the percent sign from cron's special treatment.

This becomes especially important in shell conditions involving `date`:

```cron
0 9 * * 1 [ "$(date +\%d)" -le 07 ] && /usr/local/bin/first-monday-report
```

The escaping is for cron, not for `date`. After cron processes the command, the shell should ultimately invoke `date` with the expected format string.

A good experiment on a disposable system is to compare two jobs writing to separate files:

```cron
* * * * * /bin/date +%F >/tmp/cron-percent-unescaped 2>&1
* * * * * /bin/date +\%F >/tmp/cron-percent-escaped 2>&1
```

Then inspect the results and cron logs after a minute:

```bash
cat /tmp/cron-percent-unescaped
cat /tmp/cron-percent-escaped
journalctl -u cron --since '5 minutes ago'
```

Do this only as a learning exercise on the actual implementation you want to understand. Minimal cron implementations may differ, and systemd timers do not inherit this syntax merely because they also schedule commands.

The broader lesson is that a crontab command line is not simply copied byte-for-byte to `/bin/sh`. Cron performs some parsing of its own before the shell parses the resulting command.

---

## Environment assignments are part of crontab syntax

A crontab can contain environment assignments in addition to job entries:

```cron
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=ops@example.com

0 2 * * * /usr/local/sbin/backup
```

These lines do not contain five time fields. The parser recognizes them as variable assignments.

This distinction is why a crontab cannot be treated exactly like an arbitrary shell script. For example, this is a cron environment definition:

```cron
PATH=/opt/tools/bin:/usr/bin:/bin
```

but it is not the same as running an `export` command in an interactive shell startup file. Cron builds the job environment according to its configuration and implementation rules.

The environment deserves a separate deep dive because `PATH`, `HOME`, `LOGNAME`, `SHELL`, locale variables, mail settings, PAM, and distribution defaults have substantial operational and security consequences. From the syntax perspective, the important fact is that variable assignments and job lines are two distinct classes of crontab records.

Avoid trying to place a shell command on an environment-assignment line:

```cron
FOO=$(date)
```

Cron configuration is not a general shell parser. Do not assume command substitution occurs when the crontab is loaded. If dynamic state is required, compute it inside the scheduled command or script.

---

## Special schedule strings are shorthand, not richer calendars

Many cron implementations support symbolic schedule forms beginning with `@`. Common examples include:

```cron
@reboot   /usr/local/bin/on-boot
@hourly   /usr/local/bin/hourly-task
@daily    /usr/local/bin/daily-task
@weekly   /usr/local/bin/weekly-task
@monthly  /usr/local/bin/monthly-task
@yearly   /usr/local/bin/yearly-task
```

`@annually` is commonly provided as an alias for `@yearly`, and `@midnight` as an alias for `@daily` on Vixie-style implementations.

The exact equivalences commonly documented are conceptually similar to:

```text
@hourly   → 0 * * * *
@daily    → 0 0 * * *
@weekly   → 0 0 * * 0
@monthly  → 0 0 1 * *
@yearly   → 0 0 1 1 *
```

The important exception is `@reboot`. It is not a calendar expression. It means the job should be started once when the cron daemon considers the system or daemon to have started, according to implementation behavior.

Do not interpret `@reboot` as a dependency-aware boot ordering mechanism. If a job requires the network, a mounted filesystem, a database, or another service to be ready, systemd service dependencies are generally a stronger model than sleeping for an arbitrary number of seconds in a cron command.

For the periodic aliases, symbolic forms can improve readability when midnight or the top of the hour is genuinely desired. A numeric expression is preferable when the actual time should be chosen deliberately to avoid load spikes.

For example, a fleet of machines all using:

```cron
@daily /usr/local/bin/update-index
```

may create a synchronized midnight workload. Choosing staggered explicit times can reduce contention:

```cron
17 1 * * * /usr/local/bin/update-index
```

Cronie and some other implementations provide additional mechanisms such as randomized delay, but those are implementation extensions rather than a property of classic five-field syntax.

---

## Cron evaluates wall-clock calendar time

Classic cron is fundamentally a wall-clock scheduler. That means its interpretation depends on the machine's civil time: timezone configuration, clock changes, daylight-saving transitions, and sometimes daemon-specific timezone features.

Suppose a job is configured as:

```cron
30 2 * * * /usr/local/bin/task
```

In a timezone with daylight-saving transitions, a local time such as 02:30 may fail to exist on the spring transition day, or a local time may occur twice during the autumn transition. Different cron implementations have historically handled these situations in different ways, and some include logic intended to avoid duplicate or missed executions for jobs affected by small clock changes.

The correct engineering approach is not to memorize one universal DST rule. It is to identify the deployed cron implementation and read its `crontab(5)` and daemon documentation.

On a system using Cronie, for example:

```bash
man 5 crontab
man 8 crond
```

On Debian-family systems:

```bash
man 5 crontab
man 8 cron
```

The package can also be identified:

```bash
dpkg -S "$(command -v cron)" 2>/dev/null
rpm -qf "$(command -v crond)" 2>/dev/null
```

When a job has financial, billing, backup, or compliance significance, timezone and daylight-saving behavior belong in the design review. "Runs daily at 02:30" is not a complete operational specification if the machine operates in a timezone where 02:30 is not guaranteed to occur exactly once every civil day.

For infrastructure where UTC is acceptable, scheduling the host or service in UTC can remove many daylight-saving ambiguities. Where local business time is required, the ambiguity is real and should be handled intentionally.

---

## CRON_TZ is useful but not universally portable

Cronie and some related implementations support a `CRON_TZ` variable that changes the timezone used to interpret following schedule entries.

A configuration may look like:

```cron
CRON_TZ=UTC
0 3 * * * /usr/local/bin/global-report
```

The intent is that the schedule fields are evaluated against UTC even if the host's local timezone is different.

This can be useful on servers hosting jobs for several regions, but it introduces an important distinction: the scheduler's interpretation timezone and the command's process environment are separate concerns. The timestamp printed by the command can still depend on the environment passed to that command and the system's timezone configuration.

If a script must produce UTC timestamps, make that requirement explicit too:

```bash
TZ=UTC date --iso-8601=seconds
```

or configure the application's timezone deliberately.

Because `CRON_TZ` is not a safe assumption across every cron implementation, configuration intended for heterogeneous systems should either avoid it or enforce the required cron package as part of the platform specification.

A useful repository comment is:

```cron
# Requires Cronie CRON_TZ support. Schedule is intentionally UTC.
CRON_TZ=UTC
0 3 * * * /usr/local/bin/global-report
```

That turns an otherwise invisible portability dependency into explicit documentation.

---

## "Every N days" is not what a stepped day-of-month means

An expression such as:

```cron
0 0 */2 * * command
```

is frequently described as "every two days." More precisely, it selects a stepped set of **day-of-month values** inside each month. The sequence resets when the month changes.

A typical expansion is based on days:

```text
1,3,5,7,...
```

within each month. Now consider the boundary between January and February:

```text
Jan 31
Feb 1
```

Both may be selected, producing executions on consecutive calendar days. The expression does not represent a continuous 48-hour interval across month boundaries.

The same problem exists for:

```cron
0 0 */10 * * command
```

The field might select days 1, 11, 21, and 31 where valid, then restart with day 1 of the next month. The intervals vary.

If the real requirement is "run no sooner than 48 hours after the previous execution," use an interval-oriented mechanism rather than encoding it as a day-of-month step. A systemd monotonic timer can model elapsed time directly, and applications can persist the last-run timestamp when business logic needs stronger guarantees.

Cron steps are operations on field values, not elapsed-time arithmetic.

---

## "Every N months" has the same boundary property

Consider:

```cron
0 0 1 */2 * command
```

This typically selects stepped month numbers beginning with January:

```text
January, March, May, July, September, November
```

That is a perfectly reasonable "every other odd-numbered month" schedule. It should not be interpreted as "two months after whenever the last successful run occurred."

If the job fails in March, cron does not shift the next intended run to May plus some recovery interval. The May schedule remains May because the expression describes calendar positions.

Similarly:

```cron
0 0 1 2-12/3 * command
```

starts its step at February, producing a different selected set than `*/3`.

Always expand step expressions into concrete field values during review. That five-second exercise catches many schedule mistakes.

---

## Schedule examples are easier to verify when expanded

Consider:

```cron
*/20 8-12 * * 1-5 /usr/local/bin/check
```

Rather than reading it as one opaque line, expand each field:

```text
minute        = {0,20,40}
hour          = {8,9,10,11,12}
day-of-month  = unrestricted
month         = unrestricted
day-of-week   = Monday through Friday
```

Then generate representative matches:

```text
Monday 08:00
Monday 08:20
Monday 08:40
Monday 09:00
...
Monday 12:40
Tuesday 08:00
...
Friday 12:40
```

Now test a boundary:

```text
Friday 12:40  → match
Friday 13:00  → no match
Saturday 08:00 → no match
```

This manual expansion is far more reliable than relying on an English phrase such as "every twenty minutes during business mornings."

For a more complex example:

```cron
5,35 6-18/3 1,15 * * /usr/local/bin/task
```

Expand it:

```text
minute        = {5,35}
hour          = {6,9,12,15,18}
day-of-month  = {1,15}
month         = all
weekday       = all
```

The task runs twice in each selected hour, five selected hours per selected date, on the first and fifteenth of each month.

That is twenty invocations per month in months where both dates exist, independent of how long each invocation takes.

This style of expansion should become a habit for code review.

---

## Construct schedules from requirements instead of guessing expressions

Suppose the requirement is:

> Run a cleanup at 01:15 every Sunday.

Translate one constraint at a time:

```text
minute = 15
hour = 1
day-of-month = any
month = any
day-of-week = Sunday
```

Then encode it:

```cron
15 1 * * 0 /usr/local/sbin/cleanup
```

or, where names are supported:

```cron
15 1 * * SUN /usr/local/sbin/cleanup
```

Now a different requirement:

> Run at 06:10 and 18:10 every weekday.

Translate:

```text
minute = 10
hour = {6,18}
weekday = Monday-Friday
```

Then:

```cron
10 6,18 * * 1-5 /usr/local/bin/task
```

Another:

> Run every fifteen minutes from 09:00 through 16:59 on weekdays.

The minute set is:

```text
0,15,30,45
```

and the hour set is:

```text
9,10,11,12,13,14,15,16
```

so:

```cron
*/15 9-16 * * 1-5 /usr/local/bin/task
```

Notice that this does **not** include 17:00. If the requirement says "through 17:00 inclusive," one clean representation is to add a second entry:

```cron
*/15 9-16 * * 1-5 /usr/local/bin/task
0    17   * * 1-5 /usr/local/bin/task
```

Trying to compress every boundary into a single expression is not always worth the reduction in readability.

---

## The midnight boundary deserves explicit testing

Human descriptions involving "night" often cross a calendar boundary, while a cron expression treats each calendar time literally.

Suppose the requirement is:

> Run every hour from 22:00 until 02:00.

A clear representation is:

```cron
0 22,23,0,1,2 * * * /usr/local/bin/task
```

or two jobs:

```cron
0 22-23 * * * /usr/local/bin/task
0 0-2   * * * /usr/local/bin/task
```

Now add a weekday requirement:

> Run during the night shift from Monday night through Friday night.

This is no longer a trivial hour-range problem because times after midnight belong to the next calendar day. Is 01:00 Saturday considered part of Friday's night shift? Humans often say yes; the cron weekday field sees Saturday.

The correct expression depends on the business definition. One maintainable design is to schedule the candidate hours every day and let a small script decide whether the current timestamp belongs to an active shift according to the organization's rules.

This is a recurring theme in reliable scheduling: cron is excellent at selecting simple calendar coordinates. Once the requirement refers to semantic concepts such as "night shift," "business day," "last working day," or "after the previous successful run," those concepts may belong in application logic rather than in a compressed calendar expression.

---

## Multiple cron lines are sometimes the cleanest syntax

Suppose a service should be checked every five minutes during the day but only every thirty minutes overnight.

Trying to encode everything in one expression is unnecessary. Two lines communicate the policy well:

```cron
*/5  7-22 * * * /usr/local/bin/check-service
0,30 0-6 * * * /usr/local/bin/check-service
23,53 23 * * * /usr/local/bin/check-service
```

The exact boundaries should be chosen based on the requirement, but the structural point remains: a crontab is allowed to contain multiple entries invoking the same command.

Another example is a job that runs at 08:00 on weekdays and 10:00 on weekends:

```cron
0 8  * * 1-5 /usr/local/bin/daily-report
0 10 * * 0,6 /usr/local/bin/daily-report
```

This is immediately understandable during review. A more compressed representation would not necessarily be an improvement.

Multiple lines also permit different environment variables, wrappers, or logging targets when the operational behavior differs by schedule.

---

## Validate syntax before trusting the schedule

The most dangerous cron error is not a parser rejection. Parser errors are relatively friendly because the job does not install or logs an obvious failure. The more dangerous error is a syntactically valid expression that means something different from the operator's intent.

Start with the implementation's own parser. When editing via:

```bash
crontab -e
```

many implementations validate the file before installing it and report syntax errors.

You can also prepare a file and install it:

```bash
crontab ./my-crontab
```

but remember that this replaces the current user's installed crontab rather than appending to it. On a real machine, preserve the existing table first:

```bash
crontab -l >crontab.backup
```

A safer learning environment is a disposable VM or container configured with the cron daemon being studied.

Syntax validation alone cannot prove semantic correctness. For that, construct known timestamps and ask whether they should match.

For example, given:

```cron
15 4 * * 1-5 command
```

write down test cases:

```text
Monday    04:15 → yes
Monday    04:14 → no
Monday    05:15 → no
Saturday  04:15 → no
Friday    04:15 → yes
```

For a monthly schedule:

```cron
0 3 1 * * command
```

check boundaries:

```text
May 1 03:00 → yes
May 1 03:01 → no
May 2 03:00 → no
June 1 03:00 → yes
```

For any expression using both day-of-month and day-of-week, explicitly test dates where only one of the two conditions is true. That catches the OR rule immediately.

---

## Use a temporary logging job to observe actual matching

When learning or debugging an unfamiliar cron implementation, a simple observation job is often better than guessing.

Create a script:

```bash
sudo install -m 0755 /dev/stdin /usr/local/bin/cron-probe <<'SCRIPT'
#!/bin/sh
printf '%s pid=%s ppid=%s uid=%s\n' \
    "$(date --iso-8601=seconds)" \
    "$$" \
    "$PPID" \
    "$(id -u)" >> /tmp/cron-probe.log
SCRIPT
```

Then add a temporary crontab entry:

```cron
*/2 * * * * /usr/local/bin/cron-probe
```

After several minutes:

```bash
cat /tmp/cron-probe.log
```

You should see timestamps corresponding to the selected minute values.

Change the schedule to a list:

```cron
1,4,7,10 * * * * /usr/local/bin/cron-probe
```

and observe again.

A probe turns syntax into empirical behavior. It also confirms that the daemon is running, that the crontab was installed for the expected user, and that the command path is valid.

Remove temporary jobs after the experiment. Test crontabs have a habit of becoming accidental production configuration when cleanup is postponed.

---

## Test command parsing separately from schedule parsing

When a cron job fails, determine whether the schedule failed to match or the command failed after being launched.

Suppose the entry is:

```cron
*/5 * * * * /opt/app/bin/export --format csv >>/var/log/app/export.log 2>&1
```

First verify that cron is attempting to run it using daemon logs:

```bash
journalctl -u cron --since '30 minutes ago'
```

or, depending on the distribution:

```bash
grep CRON /var/log/syslog
```

If executions appear at the expected times, the scheduling expression is probably not the problem.

Then reproduce the execution environment as closely as possible. At minimum, test the command through the same shell style:

```bash
/bin/sh -c '/opt/app/bin/export --format csv >>/var/log/app/export.log 2>&1'
```

If the cron job runs under another user:

```bash
sudo -u appuser /bin/sh -c '/opt/app/bin/export --format csv >>/var/log/app/export.log 2>&1'
```

Environment, working directory, permissions, and noninteractive behavior still need to be considered, but the separation is useful:

```text
Did cron select this minute?
        ↓
Did cron launch the command?
        ↓
Did the shell parse the command as intended?
        ↓
Did the program itself succeed?
```

Treating all four questions as "cron did not work" makes troubleshooting much slower.

---

## Shell syntax can make a valid schedule look broken

This line has a perfectly valid schedule:

```cron
0 2 * * * /usr/local/bin/job > /var/log/job.log 2>&1
```

If `/var/log/job.log` is not writable by the job user, the shell can fail while setting up redirection before the program even starts.

Likewise:

```cron
0 2 * * * cd /srv/app && ./job
```

will not run `./job` if `cd /srv/app` fails.

And:

```cron
0 2 * * * /usr/local/bin/job | /usr/bin/logger -t myjob
```

creates a shell pipeline whose exit behavior may not mean what the administrator assumes. `/bin/sh` commonly reports the status of the final pipeline command, not necessarily the first failed component.

None of these are time-field syntax problems. They illustrate why cron configuration has two parsers in the path: cron parses the crontab record, then a shell commonly parses the command text.

When the command becomes nontrivial, this is easier to debug:

```cron
0 2 * * * /usr/local/sbin/nightly-maintenance
```

with logic in a script:

```bash
#!/bin/sh
set -eu

cd /srv/app
/usr/local/bin/job
/usr/bin/logger -t nightly-maintenance 'job completed'
```

The schedule stays declarative and the shell logic becomes testable outside cron.

---

## Common schedule mistakes and their actual behavior

The following examples are worth memorizing not as recipes, but as demonstrations of the matching model.

### `* * * * *`

```cron
* * * * * command
```

Matches every minute. This can create up to 1,440 launch attempts per day.

If the command takes longer than one minute and has no locking, multiple copies can accumulate rapidly.

### `*/60 * * * *`

This should not be used as a clever substitute for hourly scheduling. The minute field's domain is 0-59, and step semantics or parser validation around a step equal to or larger than the domain are implementation-specific enough that the intent is clearer as:

```cron
0 * * * * command
```

### `0 24 * * *`

The normal hour field is 0-23. Midnight is hour `0`, not `24`:

```cron
0 0 * * * command
```

### `0 0 0 * *`

Day-of-month normally starts at 1, so `0` is invalid for that field. The first day of every month is:

```cron
0 0 1 * * command
```

### `0 0 31 * *`

Runs only in months with a thirty-first day. It does not mean last day of month.

### `0 0 */2 * *`

Selects stepped day-of-month values. It is not a stable 48-hour interval across month boundaries.

### `*/17 * * * *`

Selects minutes such as 0, 17, 34, and 51 each hour. The interval across the hour boundary is not seventeen minutes.

### `0 9 1 * 1`

On traditional cron semantics, means the first day of the month **or** Monday when both day fields are restricted, not "the first Monday."

### `0 0 * * 7`

Commonly means Sunday, but `0` or `SUN` may communicate the convention more clearly depending on the team.

### `0 0 1 */3 *`

Selects every third month from the step's starting position in the month field. It is calendar selection, not a timer measured from the previous successful execution.

Understanding why each example behaves this way is more valuable than memorizing a list of "cron gotchas."

---

## Practical recipes, derived rather than memorized

A useful reference chapter should contain examples, but each recipe should still be read through the field model.

Run every minute:

```cron
* * * * * /usr/local/bin/task
```

Run at minute 10 of every hour:

```cron
10 * * * * /usr/local/bin/task
```

Run every five minutes:

```cron
*/5 * * * * /usr/local/bin/task
```

Run at 02:30 every day:

```cron
30 2 * * * /usr/local/bin/task
```

Run at 09:00 and 17:00 every day:

```cron
0 9,17 * * * /usr/local/bin/task
```

Run at 09:00 every Monday:

```cron
0 9 * * 1 /usr/local/bin/task
```

Run at 09:00 Monday through Friday:

```cron
0 9 * * 1-5 /usr/local/bin/task
```

Run every fifteen minutes during hours 08 through 17 on weekdays:

```cron
*/15 8-17 * * 1-5 /usr/local/bin/task
```

Run at midnight on the first day of every month:

```cron
0 0 1 * * /usr/local/bin/task
```

Run quarterly at 04:20 on January 1, April 1, July 1, and October 1:

```cron
20 4 1 1,4,7,10 * /usr/local/bin/task
```

Run every Sunday at 03:00:

```cron
0 3 * * 0 /usr/local/bin/task
```

Run at 00:00 on January 1:

```cron
0 0 1 1 * /usr/local/bin/task
```

Run at the start of every six-hour block:

```cron
0 */6 * * * /usr/local/bin/task
```

Run at minutes 5 and 35 during even-numbered hours:

```cron
5,35 0-23/2 * * * /usr/local/bin/task
```

Each expression can be verified by expanding the selected sets before deployment.

---

## A real backup schedule should include more than a clever expression

Assume a database backup should run every night at 02:15. The minimal schedule is:

```cron
15 2 * * * /usr/local/sbin/database-backup
```

A production-quality crontab might instead make intent and output handling visible:

```cron
# Nightly database backup. Retention and locking are handled by the script.
15 2 * * * /usr/local/sbin/database-backup >>/var/log/database-backup.log 2>&1
```

The script can contain the complexity:

```bash
#!/bin/sh
set -eu

backup_dir=/srv/backups/database
stamp=$(date +%Y%m%d-%H%M%S)

mkdir -p "$backup_dir"
exec /usr/bin/pg_dump -Fc appdb >"$backup_dir/appdb-$stamp.dump"
```

Notice that `%` is no longer a crontab parsing concern because the `date` command lives inside the script rather than directly in the crontab command field.

This separation improves several properties at once:

- the calendar expression is easy to inspect;
- the shell program can be run manually for testing;
- quoting is simpler;
- percent signs no longer need cron-specific escaping;
- version control can track the executable logic independently;
- locking, retention, alerting, and error handling can evolve without rewriting the schedule.

The shortest crontab line is often the most maintainable crontab line.

---

## A monitoring schedule demonstrates the difference between frequency and concurrency

Suppose a health check should run once per minute:

```cron
* * * * * /usr/local/bin/health-check
```

If `health-check` usually takes two seconds, the schedule seems harmless. Then a remote dependency hangs and the command takes five minutes. Cron continues matching every minute, so several copies may coexist.

Observe them with:

```bash
pgrep -af health-check
```

or:

```bash
ps -eo pid,ppid,lstart,cmd | grep '[h]ealth-check'
```

The schedule is still working exactly as configured. The failure is in the operational assumption that execution time would always be shorter than the scheduling period.

A locking wrapper can prevent overlap:

```cron
* * * * * /usr/bin/flock -n /run/health-check.lock /usr/local/bin/health-check
```

Now time matching remains "every minute," while locking decides whether a new execution is allowed to proceed.

This is an important systems-design principle: **scheduling policy and concurrency policy are independent**. Cron expressions answer the first question only.

---

## A first-Monday schedule demonstrates when shell logic is justified

The requirement is:

> Run at 07:30 on the first Monday of each month.

Do not write:

```cron
30 7 1 * 1 command
```

because traditional day-field semantics do not mean "first AND Monday."

Instead, select Mondays:

```cron
30 7 * * 1 /usr/local/bin/run-first-monday
```

and implement the second condition in a script:

```bash
#!/bin/sh
set -eu

day=$(date +%d)

if [ "$day" -le 7 ]; then
    exec /usr/local/bin/monthly-operation
fi
```

Test the logic independently:

```bash
shellcheck /usr/local/bin/run-first-monday
sh -x /usr/local/bin/run-first-monday
```

For stronger testing, make the script accept an optional date input in test mode so boundary cases can be checked without changing the system clock.

The resulting design is longer than one crontab line but far easier to prove correct.

---

## Avoid embedding secrets in cron commands

Crontab syntax allows arbitrary command arguments, but that does not make the crontab an appropriate secret store.

This is a poor design:

```cron
0 2 * * * /usr/local/bin/upload --token 'super-secret-value' /srv/export
```

Depending on the system and command behavior, the secret may be exposed through crontab access, backups, configuration management, shell process listings, audit logs, monitoring tools, or error reports.

Prefer a purpose-built credentials mechanism with appropriate permissions, such as a root-owned configuration file, service account credentials, a secrets manager, or an application-specific credential store.

The crontab should ideally contain schedule and executable path, not sensitive data:

```cron
0 2 * * * /usr/local/sbin/export-upload
```

Syntax permits much more than good operational practice should allow.

---

## Repository-friendly crontabs should be deterministic to review

When cron configuration is stored in Git, readability matters because the file becomes part of a code-review process.

A clean example:

```cron
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Refresh the customer cache before the business day begins.
10 5 * * 1-5 /usr/local/sbin/refresh-customer-cache

# Generate the morning operations report after cache refresh normally completes.
40 5 * * 1-5 /usr/local/sbin/generate-operations-report

# Full database backup during the nightly low-traffic window.
20 2 * * * /usr/local/sbin/database-backup
```

A reviewer can inspect:

```text
05:10 cache refresh
05:40 report
02:20 backup
```

But note that scheduling jobs thirty minutes apart does not create a true dependency. If the cache refresh takes forty minutes, the report may start while it is still running. If the second task truly depends on the first, encode that dependency in a script, workflow system, or service manager rather than relying only on expected runtimes.

For example:

```cron
10 5 * * 1-5 /usr/local/sbin/morning-pipeline
```

where `morning-pipeline` runs the tasks sequentially and stops on failure.

Cron expresses time. It is not a general workflow engine.

---

## Cron expression generators are aids, not authorities

Web-based cron expression generators are convenient for simple schedules, but they should not replace understanding of the target implementation.

There are several reasons:

- some tools generate Quartz syntax rather than Unix cron syntax;
- Quartz commonly uses a seconds field and supports operators absent from classic cron;
- cloud schedulers may use five or six fields with their own weekday rules;
- GitHub Actions cron uses UTC and has platform-specific execution guarantees;
- Kubernetes CronJob uses cron-like scheduling but adds controller behavior, concurrency policies, missed-run handling, and optional timezone support depending on version;
- systemd `OnCalendar=` is a different calendar language entirely.

An expression copied from another scheduler may be syntactically invalid or, worse, syntactically valid with different semantics.

Always identify the language first:

```text
Unix crontab?
Cronie?
BusyBox crond?
Quartz?
AWS EventBridge?
Kubernetes CronJob?
GitHub Actions?
systemd OnCalendar?
```

"Cron syntax" is often used casually to describe several related but nonidentical languages.

For this repository, the unqualified term `crontab` refers to the conventional Linux user/system crontab grammar unless an implementation extension is explicitly named.

---

## A small parser exercise makes the model concrete

You do not need to reimplement cron to understand field matching. A tiny shell experiment can make the selection model tangible.

For a schedule that should run at minutes 0, 15, 30, and 45, inspect the current minute:

```bash
minute=$(date +%M)
```

Normalize it as decimal if your shell arithmetic has implementation-specific leading-zero behavior, then test the selected set. In shell, a `case` statement avoids unnecessary arithmetic:

```bash
case $(date +%M) in
    00|15|30|45)
        echo match
        ;;
    *)
        echo no-match
        ;;
esac
```

For weekdays:

```bash
case $(date +%u) in
    1|2|3|4|5)
        echo weekday
        ;;
    6|7)
        echo weekend
        ;;
esac
```

GNU `date +%u` uses 1 through 7 with Monday as 1 and Sunday as 7; cron's weekday numbering conventions are related but not identical in every representation, so this is an educational model rather than a replacement for the daemon.

The useful idea is that each field can be thought of as membership in a selected set:

```text
current minute ∈ selected minutes?
current hour ∈ selected hours?
current month ∈ selected months?
```

That is much closer to the scheduler's conceptual job than natural-language phrases are.

---

## Review schedules at boundaries, not only at typical times

Most scheduling bugs hide at boundaries:

```text
59 → 00 minute transition
23 → 00 hour transition
Sunday → Monday
month end → next month
December → January
DST transition
leap day
```

Suppose the expression is:

```cron
*/20 * * * * command
```

Typical times look fine:

```text
10:00
10:20
10:40
```

Check the boundary:

```text
10:40 → 11:00 = 20 minutes
```

This particular step divides sixty evenly, so the interval remains twenty minutes.

Now compare:

```cron
*/25 * * * * command
```

Selected minutes are typically:

```text
00,25,50
```

The boundary is:

```text
10:50 → 11:00 = 10 minutes
```

The phrase "every twenty-five minutes" would conceal the actual behavior.

For day-of-month steps, check month boundaries. For month steps, check year boundaries. For weekday expressions, check Sunday conventions. For schedules around 02:00 in DST-observing zones, check clock transitions.

A schedule is not adequately tested until its boundaries are tested.

---

## Review the number of expected invocations

A simple sanity check is to calculate roughly how often a job should run.

For:

```cron
*/5 * * * * command
```

there are twelve selected minutes per hour:

```text
60 / 5 = 12
```

and 24 hours per day:

```text
12 × 24 = 288 executions/day
```

For:

```cron
*/10 8-17 * * 1-5 command
```

there are six executions per selected hour and ten selected hours:

```text
6 × 10 = 60 executions per weekday
```

approximately:

```text
300 executions in a five-day work week
```

This arithmetic is useful for capacity planning. If each execution performs a database query taking 500 ms, the schedule might be fine. If each execution scans a 500 GB filesystem, the same frequency may be absurd even though the expression is syntactically correct.

It also catches accidental wildcards. An operator intending "once per hour" may write:

```cron
* * * * * command
```

and a simple execution-count review immediately reveals 1,440 daily launches rather than 24.

---

## Time matching does not imply execution guarantees

When the current time matches an entry, cron attempts to start the command. That should not be confused with a transactional guarantee that the job ran exactly once and completed successfully.

Several independent failures are possible:

```text
schedule matches
    ↓
cron launches shell
    ↓
shell starts program
    ↓
program performs work
    ↓
program commits result
```

The machine may crash between any two stages. A command may be killed. The filesystem may be full. DNS may fail. The database may reject a transaction. A duplicated clock interval may create another launch depending on implementation behavior. A long-running previous copy may overlap.

For jobs where "exactly once" has strong business meaning, the application must provide idempotency, locking, transactions, durable job state, or a scheduling/workflow system with the appropriate guarantees. No arrangement of five cron fields can create those properties by itself.

The schedule specifies **eligibility by calendar time**, not successful business execution.

---

## Build idempotent scheduled commands when possible

Because scheduled work can be retried manually, overlap accidentally, or be triggered after a partial failure, idempotent behavior is valuable.

Suppose a daily job creates a row saying that a report was generated:

```text
report_date = 2026-09-13
```

A naive script may insert a duplicate row every time it runs. A safer application design can use `report_date` as a unique key and update or no-op when the day's record already exists.

Then the cron entry remains simple:

```cron
10 4 * * * /usr/local/bin/generate-daily-report
```

but operational recovery becomes safer:

```bash
/usr/local/bin/generate-daily-report
```

can be run manually after a failure without blindly duplicating work.

Again, the calendar expression and the execution semantics solve different problems. Good scheduled systems design accounts for both.

---

## A disciplined method for writing a cron expression

When creating a new schedule, begin with the requirement in concrete calendar language. Avoid vague phrases where possible.

Instead of:

```text
run regularly in the morning
```

write:

```text
run at 06:15, 06:30, 06:45, and 07:00 every Monday-Friday in the server's local timezone
```

Then map the times to fields.

If the times form a simple set, encode them directly. If they cross awkward boundaries, use more than one entry. If the requirement depends on business semantics, move that condition to a script or a more expressive scheduler.

After writing the expression:

- expand lists, ranges, and steps into representative concrete values;
- test one expected match;
- test one non-match for every restricted field;
- test boundary transitions;
- inspect day-of-month/day-of-week behavior if both are restricted;
- identify the timezone used for matching;
- estimate expected invocation count;
- verify that overlap is safe or controlled;
- confirm that command parsing, permissions, environment, and output handling are separately correct.

This is not bureaucratic overhead. For a backup, billing process, certificate renewal, security scan, or database maintenance job, a scheduling error can remain invisible for weeks before anyone notices.

---

## Reading an unfamiliar crontab during an incident

Imagine finding this on a production server:

```cron
*/7 1-23/3 1,15 * 1-5 /opt/ops/run-maintenance --fast >>/var/log/maintenance.log 2>&1
```

Do not immediately translate it into an English sentence. Decompose it.

The minute field:

```text
*/7
```

selects values like:

```text
0,7,14,21,28,35,42,49,56
```

The hour field:

```text
1-23/3
```

selects:

```text
1,4,7,10,13,16,19,22
```

The day-of-month field:

```text
1,15
```

is restricted.

The month field is unrestricted.

The weekday field:

```text
1-5
```

is also restricted.

That last combination should immediately trigger a review of day-of-month/day-of-week OR semantics on the installed cron implementation. The job may run on every weekday **or** on the first and fifteenth, producing far more invocations than someone expecting an AND relationship intended.

Then estimate scale. Nine minute values multiplied by eight hour values gives up to 72 attempts per matching day. If weekdays are matches, this may be hundreds of executions per week.

Only after understanding the schedule should you investigate whether `/opt/ops/run-maintenance` is safe to execute that often and whether concurrent runs are controlled.

This decomposition method is especially useful in security reviews, where a root-owned crontab may execute writable scripts frequently enough to provide an easy privilege-escalation path.

---

## Security review begins with knowing exactly when a privileged command runs

Cron syntax is not itself a security mechanism, but schedule interpretation affects attack windows and forensic analysis.

Suppose root has:

```cron
*/5 * * * * /opt/company/bin/cleanup
```

and `/opt/company/bin/cleanup` is writable by a non-root user. The schedule tells an attacker that any modification may be executed as root at one of the selected minutes.

An investigator who understands step semantics can predict the next candidate execution without waiting randomly:

```text
minute ∈ {0,5,10,15,...,55}
```

Similarly, an unexpected privileged process appearing every seven minutes may correspond to:

```cron
*/7 * * * * command
```

but remember the hour-boundary behavior: the minute pattern is fixed within each hour rather than a perfect continuous seven-minute interval.

During auditing, enumerate root's scheduling sources:

```bash
sudo crontab -l
sudo cat /etc/crontab
sudo find /etc/cron.d -maxdepth 1 -type f -ls
```

and then inspect ownership and permissions of every executed path and its parent directories. The scheduling expression tells you when; filesystem and execution-context analysis tells you whether it is safe.

The security details are developed in the dedicated cron security chapter, but correct syntax interpretation is the first prerequisite.

---

## The five fields are a compact language, not a complete scheduler

Classic crontab syntax is powerful because a small grammar covers a large class of recurring calendar schedules:

```text
wildcard
single value
list
range
step
```

Those primitives combine well for:

```text
hourly maintenance
daily backups
weekday reports
weekly cleanup
monthly rotations
seasonal tasks
fixed periodic polling aligned to wall-clock values
```

They are less suitable for requirements such as:

```text
exactly 90 minutes after the previous successful run
third business day of the month
last weekday before a public holiday
first Monday unless it is a holiday, otherwise Tuesday
start 20 minutes after service X becomes healthy
retry after failure with exponential backoff
run only after job A completes successfully
run once for every queued object
```

Trying to encode these requirements entirely in cron usually produces brittle shell one-liners or incorrect schedules.

The engineering skill is not knowing how to force every problem into five fields. It is knowing where those five fields are the right abstraction and where another mechanism should take over.

---

## A compact field-reading checklist

When you encounter a cron expression, read it in this order:

```text
minute
hour
day of month
month
day of week
```

For each field, determine its concrete selected set.

Given:

```cron
10,40 6-18/4 1-7 1,7 1 command
```

expand it:

```text
minute = {10,40}
hour = {6,10,14,18}
day-of-month = {1,2,3,4,5,6,7}
month = {January, July}
day-of-week = {Monday}
```

Then stop before declaring the final meaning. Both day fields are restricted, so apply the implementation's day-field rule. On a traditional Vixie-style cron, the job can match during the first seven days of January/July **or** on Mondays in those months, provided the minute and hour fields also match.

That final step is exactly where casual readers make mistakes.

---

## Laboratory: prove list, range, and step behavior

A disposable Linux VM is enough for a useful cron syntax lab.

Create a probe script:

```bash
cat > /tmp/cron-probe.sh <<'EOF_SCRIPT'
#!/bin/sh
printf '%s\n' "$(date --iso-8601=seconds)" >> /tmp/cron-times.log
EOF_SCRIPT
chmod 755 /tmp/cron-probe.sh
```

Install a temporary job:

```bash
crontab -e
```

Add:

```cron
*/3 * * * * /tmp/cron-probe.sh
```

Wait long enough to observe several executions:

```bash
cat /tmp/cron-times.log
```

You should see timestamps aligned to minute values divisible according to the expression, not spaced three minutes after the previous process's completion.

Replace the schedule with:

```cron
1,2,8,13 * * * * /tmp/cron-probe.sh
```

and observe that only those minute values are candidates.

Then test a range:

```cron
0 9-11 * * * /tmp/cron-probe.sh
```

If waiting for hours is impractical, temporarily choose a nearby minute range instead and apply the same syntax idea to the minute field.

The goal is not merely to see cron run. It is to develop the habit of predicting exact matches first and comparing observation with prediction afterward.

Clean up when finished:

```bash
rm -f /tmp/cron-probe.sh /tmp/cron-times.log
```

and remove the test entry from the crontab.

---

## Laboratory: observe the day-of-month/day-of-week problem safely

Waiting for calendar dates is inconvenient, so the best way to study this rule is usually through documentation, a test harness, or a cron-expression evaluator known to implement the same semantics as the daemon being studied.

First identify the daemon:

```bash
ps -ef | grep '[c]ron'
```

Then inspect its manual:

```bash
man 5 crontab
```

Search for the day fields:

```text
/day of month
```

or:

```text
/day of week
```

On Vixie-derived documentation, look for language explaining that when both fields are restricted, one or the other may match.

Now choose a hypothetical month where the first is not Monday and evaluate:

```cron
0 9 1 * 1 command
```

Write the dates matching day-of-month and the dates matching Monday. The union of those sets is the expected run-date set under OR semantics.

This paper exercise is more educational than blindly pasting the expression into an online tool because it forces the Boolean rule to become explicit.

---

## Laboratory: prove percent-sign handling

On a disposable system using a Vixie-derived cron implementation, add two temporary entries:

```cron
* * * * * /bin/date +%F >/tmp/percent-a 2>&1
* * * * * /bin/date +\%F >/tmp/percent-b 2>&1
```

After cron has attempted both jobs, inspect:

```bash
printf '%s\n' '--- unescaped ---'
cat /tmp/percent-a 2>/dev/null || true

printf '%s\n' '--- escaped ---'
cat /tmp/percent-b 2>/dev/null || true
```

Also inspect the cron service logs:

```bash
journalctl -u cron --since '5 minutes ago'
```

The purpose is to observe that command-field parsing has cron-specific rules before the shell receives the text.

Remove the entries afterward. If the behavior differs from the description, record the cron implementation and version. That difference is itself useful knowledge: cron syntax is not one universal binary specification across every Unix-like system.

---

## Laboratory: compare a step with a true interval requirement

Suppose someone proposes:

```cron
*/17 * * * * /usr/local/bin/task
```

Write the candidate timestamps for two hours:

```text
10:00
10:17
10:34
10:51
11:00
11:17
11:34
11:51
```

Now calculate the gaps:

```text
17
17
17
9
17
17
17
```

The expression clearly does not represent a continuous seventeen-minute interval.

Repeat with:

```cron
*/15 * * * * command
```

Candidate times:

```text
10:00
10:15
10:30
10:45
11:00
```

All gaps are fifteen minutes because fifteen divides the sixty-minute field domain exactly.

This tiny exercise explains why some `*/N` descriptions are safe shorthand in conversation while others are not.

---

## Choosing between numeric and symbolic syntax

These two entries communicate the same intended weekday schedule on a compatible implementation:

```cron
0 8 * * 1-5 command
```

```cron
0 8 * * MON-FRI command
```

The numeric form has broad familiarity and is easy for configuration generators. The symbolic form is often easier for humans to review.

A reasonable team standard is:

```text
use numeric minute/hour/day-of-month fields;
use names for weekdays and months when it improves readability;
use only extensions guaranteed by the deployed cron implementation.
```

For example:

```cron
30 6 * JAN,APR,JUL,OCT MON-FRI /usr/local/bin/task
```

is highly readable to a human, but a very minimal cron implementation should be checked before depending on it.

Consistency matters more than personal preference. A repository mixing three different conventions makes code review slower.

---

## When a cron line deserves a comment

A self-explanatory entry needs little annotation:

```cron
0 2 * * * /usr/local/sbin/backup
```

A non-obvious entry should explain intent:

```cron
# Candidate Mondays; script exits unless today is the first Monday of the month.
30 7 * * MON /usr/local/bin/run-first-monday
```

An implementation-specific feature should identify the dependency:

```cron
# Requires Cronie CRON_TZ support.
CRON_TZ=UTC
0 3 * * * /usr/local/bin/global-report
```

A schedule chosen to coordinate with another system should record that context:

```cron
# Vendor settlement file normally arrives by 02:10; allow 20 minutes of margin.
30 2 * * * /usr/local/bin/import-settlement
```

Comments should not merely restate syntax:

```cron
# Run at 2:30
30 2 * * * command
```

The expression already says that. Good comments preserve reasoning that the expression cannot encode.

---

## Prefer explicit timezone documentation for important jobs

Even when `CRON_TZ` is not used, a repository should state which timezone a schedule assumes.

A comment can be enough:

```cron
# Host timezone must be Asia/Muscat. Business cutoff is 23:30 local time.
30 23 * * * /usr/local/bin/close-day
```

Or the deployment can enforce a known host timezone:

```bash
timedatectl
```

and configuration management can verify it.

Without this discipline, migrating a crontab from one server to another can silently move executions by hours even though the file is unchanged.

For globally distributed systems, UTC often reduces ambiguity:

```text
schedule in UTC
convert business timestamps explicitly inside the application
```

but local-time scheduling remains appropriate when the event is tied to human business hours. The important property is that the assumption is intentional and documented.

---

## Cron syntax should be reviewed together with the command's expected runtime

A schedule cannot be judged in isolation from task duration.

Suppose:

```cron
*/10 * * * * /usr/local/bin/rebuild-index
```

The expression generates six launch opportunities per hour. If the task normally takes thirty seconds, that may be fine. If it normally takes twelve minutes, overlap is the default behavior unless prevented elsewhere.

A code review should therefore ask:

```text
How often does this expression match?
How long does one run normally take?
What is the worst observed runtime?
Can two copies run safely?
If not, where is locking enforced?
```

The answers may lead to:

```cron
*/10 * * * * /usr/bin/flock -n /run/rebuild-index.lock /usr/local/bin/rebuild-index
```

or to a different scheduling architecture entirely.

The expression itself has not changed its meaning. The surrounding operational model determines whether that meaning is safe.

---

## The syntax is easy; proving intent is the real skill

The grammar of a traditional cron time specification can be learned quickly:

```text
*        every value
,        list
-        inclusive range
/        step within a selected domain
```

The difficult part is converting a human requirement into those primitives without silently changing its meaning.

Several principles make that conversion reliable:

**Treat fields as sets.** Expand an expression into concrete selected values instead of relying only on phrases such as "every N minutes."

**Remember the calendar boundaries.** Steps reset within their field domains. A minute step resets at the next hour, a day-of-month step at the next month, and a month step at the next year.

**Treat day-of-month and day-of-week as special.** On traditional cron implementations, restricting both usually introduces OR semantics.

**Keep business-calendar logic out of clever one-liners.** First Monday, last business day, holiday exceptions, and dependency chains are often clearer in scripts or richer schedulers.

**Separate schedule parsing from command execution.** A matching expression can launch a command that immediately fails because of permissions, environment, redirection, quoting, or filesystem state.

**Do not forget cron-specific command parsing.** The percent sign is a classic example where the crontab parser changes text before the shell sees it.

**Test boundaries.** Most mistakes appear at minute, hour, day, month, year, or timezone transitions rather than at ordinary timestamps.

**Estimate execution count.** A valid expression can still create an operationally unreasonable number of processes.

**Document assumptions.** Timezone, implementation-specific extensions, dependencies, and reasons for unusual times belong near the configuration.

Once these habits are in place, crontab syntax stops being a collection of magical strings. It becomes a compact calendar-matching language whose behavior can be expanded, tested, reviewed, and predicted.

That predictability is the foundation required for the next layers of cron administration: understanding where user and system crontabs live, how execution identities differ, what environment a job receives, how output and failures are recorded, and how scheduled privileged commands can become security boundaries.
