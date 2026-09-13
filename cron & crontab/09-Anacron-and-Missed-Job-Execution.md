# Anacron and Missed Job Execution

Traditional cron assumes that the system is running when a scheduled time arrives.

That assumption is reasonable for servers that stay online continuously.

It is less reliable for:

```text
laptops
developer workstations
edge devices
office machines
branch systems
virtual machines that are routinely suspended
systems that boot only during business hours
```

Consider a daily cron entry:

```cron
30 2 * * * /opt/jobs/daily-report
```

If the machine is powered off at 02:30, traditional cron does not normally say:

```text
I missed that execution.
I will run it when the system comes back.
```

The time simply passes.

At the next boot, cron evaluates the current time.

02:30 yesterday is not the current time.

The job is therefore missed.

Anacron exists to solve a different scheduling problem:

> Run periodic jobs based on elapsed days, even if the machine was not running at the originally intended time.

This distinction is fundamental.

Cron is primarily a calendar/time matcher.

Anacron is primarily a missed-period executor.

They can work together, but they answer different scheduling questions.

---

## The problem anacron solves

Imagine a laptop that is used from:

```text
08:00 to 22:00
```

every day.

A traditional daily job:

```cron
0 3 * * * /opt/jobs/cleanup
```

never runs because the machine is usually powered off at 03:00.

Changing the schedule to:

```cron
0 12 * * * ...
```

may work for this particular user, but the machine could still be shut down at noon.

What the administrator actually wants is:

```text
run this approximately once per day
```

not:

```text
run only if the machine is alive at exactly 03:00
```

That is the scheduling model anacron provides.

Anacron tracks when jobs last ran.

When the machine starts, it can determine:

```text
this daily job has not run within its required period
```

and schedule it after a configured delay.

The schedule is therefore based on job age rather than exact wall-clock matching.

---

## Cron and anacron have different semantics

Cron expression:

```cron
0 3 * * *
```

means:

```text
when local calendar time matches 03:00 each day, start the command
```

Anacron period:

```text
1 day
```

means approximately:

```text
if this job has not successfully reached its scheduled anacron execution point within the current period, make it eligible
```

The two are not interchangeable.

Cron is appropriate when exact timing matters:

```text
open market report at 09:00
nightly snapshot at 02:00
monthly invoice at midnight on the first
```

Anacron is appropriate when frequency matters more than exact clock time:

```text
daily cleanup
periodic index update
weekly maintenance
log housekeeping
package metadata maintenance
```

If a job must run:

```text
exactly at 01:30
```

anacron is usually not the right primary abstraction.

If a job must run:

```text
at least approximately once per day
```

anacron is a better fit.

---

## `/etc/anacrontab`

On systems with anacron installed, configuration is commonly stored in:

```text
/etc/anacrontab
```

Inspect:

```bash
cat /etc/anacrontab
```

A typical file may look conceptually like:

```text
SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin

1       5       cron.daily      nice run-parts --report /etc/cron.daily
7       10      cron.weekly     nice run-parts --report /etc/cron.weekly
@monthly 15     cron.monthly    nice run-parts --report /etc/cron.monthly
```

Exact defaults differ by distribution and package version.

The important fields are:

```text
period
delay
job identifier
command
```

Conceptually:

```text
1       5       cron.daily      command
```

means:

```text
period: every 1 day
delay: wait 5 minutes after anacron starts processing eligible jobs
job id: cron.daily
command: command
```

This is not cron's five-field syntax.

Anacron has its own configuration grammar.

---

## The period field

A numeric period is usually expressed in days.

Example:

```text
1
```

means daily.

```text
7
```

means weekly.

```text
30
```

means every thirty days.

This is elapsed-period scheduling, not necessarily calendar-aware scheduling.

A thirty-day period is not identical to:

```text
once per calendar month
```

because months have different lengths.

Many implementations therefore support a special value such as:

```text
@monthly
```

for calendar-month semantics.

Always check:

```bash
man 5 anacrontab
```

for the installed implementation.

Do not assume a numeric `30` and `@monthly` mean the same thing.

---

## The delay field

Anacron does not necessarily start every overdue job immediately at boot.

A delay field allows jobs to start after a number of minutes.

Example:

```text
1 5 daily-cleanup /opt/jobs/cleanup
```

means roughly:

```text
once per day
if due, wait 5 minutes after anacron begins processing
then start cleanup
```

This protects boot performance.

Without delays, a machine could boot and immediately start:

```text
daily cleanup
weekly indexing
monthly archive
package maintenance
database reports
filesystem scanning
```

all at once.

Staggered delays reduce startup load.

---

## Delay does not define the daily clock time

This line:

```text
1 10 cleanup /opt/jobs/cleanup
```

does not mean:

```text
run every day at 00:10
```

It means:

```text
if the daily job is due when anacron evaluates it,
wait 10 minutes before starting
```

If anacron starts at:

```text
08:00
```

the job may start around:

```text
08:10
```

If anacron starts at:

```text
14:00
```

the job may start around:

```text
14:10
```

This is one of the biggest conceptual differences between cron and anacron.

---

## Job identifiers

Each anacron entry has a job identifier.

Example:

```text
cron.daily
```

or:

```text
weekly-security-scan
```

The identifier is used to track execution history.

It should be stable.

Changing:

```text
daily-report
```

to:

```text
daily-report-v2
```

may cause anacron to treat the job as a different task with independent timestamp state.

Job naming is therefore part of scheduler state, not merely a cosmetic label.

---

## Timestamp files

Anacron typically records execution timestamps under a spool directory.

Common locations include:

```text
/var/spool/anacron/
```

Inspect:

```bash
ls -l /var/spool/anacron
```

Possible files:

```text
cron.daily
cron.weekly
cron.monthly
```

Inspect one:

```bash
cat /var/spool/anacron/cron.daily
```

A timestamp might be represented as:

```text
20260913
```

meaning the job's anacron state was updated for that date.

Exact format and location are implementation-specific.

Use:

```bash
man anacron
man anacrontab
```

to confirm local behavior.

The conceptual purpose is simple:

```text
job id -> last execution date
```

That state allows anacron to decide whether a job is overdue.

---

## Why persistent scheduler state matters

Traditional cron can operate almost statelessly:

```text
current time matches?
yes -> run
no -> do nothing
```

Anacron needs historical state:

```text
When did this job last run?
How long is its period?
Is it due now?
```

That state persists across reboots.

This is why anacron can recover missed periodic work after downtime.

Without persisted timestamps, the scheduler would not know whether:

```text
daily cleanup ran yesterday
```

or:

```text
daily cleanup has not run for five days
```

after a reboot.

---

## Missed cron jobs are not automatically replayed by cron

A machine has:

```cron
0 2 * * * /opt/jobs/report
```

It is powered off from:

```text
01:00 to 07:00
```

When the machine boots at 07:00:

```text
02:00 no longer matches
```

Traditional cron does not usually replay the missed 02:00 execution.

At the next day at 02:00, assuming the machine is online, the next scheduled execution occurs.

This means one entire daily run was skipped.

For some jobs that is harmless.

For others:

```text
daily backup
security signature update
billing export
maintenance cleanup
```

it may be unacceptable.

The scheduler must match the business requirement.

---

## Anacron does not replay every missed interval

Anacron is not generally a historical event replay system.

Suppose a daily anacron job has not run for five days.

When the system returns, anacron typically considers:

```text
job overdue
```

and runs it once.

It does not normally execute:

```text
Monday run
Tuesday run
Wednesday run
Thursday run
Friday run
```

five separate times.

This is a critical semantic distinction.

Anacron is suited to jobs where one current execution can satisfy the missed period.

Examples:

```text
cleanup current old files
refresh package metadata
rebuild current search index
update current cache
```

It is not automatically suitable for event-like jobs where each day represents distinct required work.

---

## Reconciliation jobs work well with anacron

Suppose the daily task is:

```bash
find /var/cache/app \
    -type f \
    -mtime +7 \
    -delete
```

If the machine is off for three days, one execution after boot can clean all files that are now older than seven days.

There is no need to replay three daily executions.

The job reconciles current state.

This is a natural fit for anacron.

---

## Event jobs may require durable state instead

Suppose a task means:

```text
generate one accounting record for each calendar day
```

If the machine is off for three days, running the command once may not create the missing daily records.

The application should identify missing dates:

```text
2026-09-10
2026-09-11
2026-09-12
```

and process each explicitly.

The scheduler should trigger a catch-up-capable application.

Do not expect anacron itself to become a business-event replay engine.

A robust job might query:

```text
last processed business date
current business date
```

then backfill missing periods.

---

## Anacron and `/etc/cron.daily`

On Debian-family systems, `/etc/crontab` may contain logic like:

```cron
25 6 * * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
```

This condition means:

```text
if anacron is not installed/executable,
cron directly runs /etc/cron.daily
```

If anacron exists, the direct cron execution is suppressed because anacron is expected to manage daily jobs.

This prevents duplicate execution.

The architecture becomes:

```text
cron/systemd starts anacron
    ->
anacron checks timestamp
    ->
anacron decides daily job is due
    ->
anacron runs run-parts /etc/cron.daily
```

The periodic directory itself still contains executable scripts.

Anacron only changes how due execution is determined.

---

## Determine who actually starts anacron

Do not assume.

Search:

```bash
grep -R -n "anacron" \
    /etc/crontab \
    /etc/cron.d \
    /etc/anacrontab \
    2>/dev/null
```

Inspect systemd units:

```bash
systemctl list-unit-files | grep -i anacron
```

Inspect timers:

```bash
systemctl list-timers --all | grep -i anacron
```

Inspect service status:

```bash
systemctl status anacron.service
```

or:

```bash
systemctl status anacron
```

depending on packaging.

Some distributions trigger anacron through cron.

Others use systemd.

Some do not install it by default.

The target machine is the source of truth.

---

## Running anacron manually

Check available options:

```bash
anacron --help
```

or:

```bash
man anacron
```

Common implementations provide modes useful for testing.

One typical pattern is:

```bash
sudo anacron -T
```

to test configuration syntax.

Another may use:

```bash
sudo anacron -n
```

to run due jobs immediately without normal delays.

Exact switches can differ, so verify before using them.

Do not memorize tool flags without checking the installed implementation.

---

## Never force all production jobs casually

Some anacron implementations provide a force option.

Forcing all jobs can immediately run:

```text
daily maintenance
weekly maintenance
monthly maintenance
large backups
package jobs
indexing
```

on a production machine.

That may create:

```text
high CPU
disk I/O saturation
database contention
unexpected network traffic
duplicate external actions
```

Use test modes and isolated labs first.

When manual execution is required, understand exactly which job will run.

---

## A safe anacron lab

Use a temporary configuration if the implementation supports specifying an alternate spool/config context, or perform the core logic manually in a non-production environment.

A simple conceptual lab:

Create a harmless script:

```bash
cat > /tmp/anacron-lab-job <<'EOF'
#!/bin/sh

echo "ran at $(date -Is)" \
    >> /tmp/anacron-lab.log
EOF

chmod +x /tmp/anacron-lab-job
```

A test anacrontab entry conceptually looks like:

```text
1 1 lab-job /tmp/anacron-lab-job
```

Run configuration validation using the local anacron's documented test mode.

Then inspect the timestamp state before and after.

The important observations are:

```text
job is considered due based on stored date
delay applies after scheduler start
timestamp is updated
next immediate evaluation does not rerun the job
```

Do not modify system `/etc/anacrontab` on a production host solely for experimentation.

---

## Timestamps are usually day-oriented

Traditional anacron's primary unit is days.

That means it is not intended for:

```text
every 10 minutes
every 2 hours
every 45 seconds
```

Cron or systemd timers are better for sub-day scheduling.

Anacron is designed around classes such as:

```text
daily
weekly
monthly
```

This makes it particularly useful for machines that are not online continuously.

---

## Anacron is not a daemon in the same sense as cron

Cron commonly runs continuously:

```text
daemon stays alive
checks schedule repeatedly
```

Anacron is often started periodically or at boot:

```text
start
check due jobs
wait configured delays
launch jobs
finish
```

The exact process lifecycle depends on implementation and integration.

This difference reflects the scheduler model.

Cron monitors wall-clock opportunities continuously.

Anacron can evaluate persistent job age whenever it is invoked.

---

## RANDOM_DELAY

Some anacron configurations support:

```text
RANDOM_DELAY
```

Example:

```text
RANDOM_DELAY=30
```

This introduces an additional random delay to job execution.

The purpose is usually load spreading.

Imagine 1,000 managed machines boot at:

```text
09:00
```

Without jitter, all may contact:

```text
package repository
central API
backup server
```

simultaneously.

Randomized delay spreads requests across a time window.

Exact semantics vary by implementation.

Check:

```bash
man 5 anacrontab
```

before relying on it.

---

## START_HOURS_RANGE

Some anacron implementations support:

```text
START_HOURS_RANGE
```

Example:

```text
START_HOURS_RANGE=7-22
```

This can restrict job starts to a daily window.

That is useful for laptops where maintenance should not run:

```text
in the middle of the night
```

or for branch systems where jobs should execute only during operational hours.

If a due job becomes eligible outside the allowed window, behavior depends on implementation.

Consult the local documentation.

This variable adds clock-window semantics on top of anacron's period model.

---

## Window restrictions can delay overdue jobs further

Suppose:

```text
START_HOURS_RANGE=8-18
```

The machine boots at:

```text
19:00
```

The daily job is overdue.

Anacron may defer it because the allowed start window has already closed.

The job then waits until a later invocation during the allowed range.

This means:

```text
overdue
```

does not always mean:

```text
run immediately
```

Operational requirements should account for allowed execution windows.

---

## Boot storms

A server that returns after a long outage may have several overdue periodic jobs:

```text
daily
weekly
monthly
```

If every job runs immediately, the machine experiences a maintenance storm.

Potential impact:

```text
CPU spike
disk saturation
package manager contention
network burst
database load
slow application startup
```

Anacron delay values help stagger these jobs.

Application-level locking and resource controls may still be required.

---

## Daily, weekly, and monthly jobs can all become due at once

Imagine a laptop was off for 40 days.

On boot:

```text
daily job overdue
weekly job overdue
monthly job overdue
```

If configuration is:

```text
1        5   daily    ...
7        10  weekly   ...
@monthly 20  monthly  ...
```

execution might be spread roughly across:

```text
+5 minutes
+10 minutes
+20 minutes
```

depending on implementation and random delay.

This staggering is operationally meaningful.

Do not choose every delay as:

```text
0
```

without considering resource contention.

---

## Delay values are not concurrency control

Suppose daily starts after 5 minutes.

Weekly starts after 10 minutes.

If daily takes one hour, weekly still starts while daily is running unless another coordination mechanism prevents it.

Anacron delays reduce expected simultaneous startup.

They do not guarantee non-overlap.

Use:

```text
flock
application locks
resource locks
```

when jobs must not overlap.

---

## Shared maintenance lock

If:

```text
daily cleanup
weekly reindex
monthly archive
```

all modify the same dataset, they can share:

```text
/run/company-maintenance.lock
```

Example daily script:

```bash
exec 9>/run/company-maintenance.lock

if ! flock -n 9; then
    logger -t company-daily \
        "event=skip reason=maintenance_lock"
    exit 0
fi
```

Weekly and monthly jobs use the same lock.

This encodes the real conflict.

Anacron decides when each is due.

The lock decides whether shared state can be modified safely.

---

## Skipping an anacron job can create a subtle issue

Suppose anacron decides the daily job is due.

The command starts.

The script immediately exits because it cannot obtain a lock:

```text
skip
```

Depending on how anacron updates timestamp state, the scheduler may consider the job "run" even though the application skipped its work.

This is important.

Scheduler-level execution and business success are different.

If missing work matters, the application must record:

```text
last successful completion
```

separately from anacron's timestamp.

Do not treat the anacron spool timestamp as proof that the job accomplished its purpose.

---

## Scheduler timestamp versus application success timestamp

Anacron state answers approximately:

```text
When did the scheduler launch/record this job?
```

Application state should answer:

```text
When did the intended operation last complete successfully?
```

Keep them separate.

Example:

```text
/var/spool/anacron/cron.daily
```

is scheduler metadata.

This:

```text
/var/lib/company-backup/last-success
```

is application metadata.

Monitoring should usually care more about the second.

---

## A due job can fail

Anacron can correctly launch:

```bash
/opt/jobs/backup
```

The command may fail because:

```text
backup disk not mounted
database unavailable
network offline
credentials expired
filesystem full
application exception
```

What happens next depends on scheduler implementation and timestamp-update semantics.

Do not assume automatic immediate retry.

Critical jobs should implement:

```text
bounded retry
failure logging
last-success monitoring
```

or use a scheduler with explicit retry policies.

---

## Test failure semantics on the installed implementation

Create an isolated lab job that exits:

```bash
exit 42
```

Observe:

```text
whether timestamp state changes
whether anacron retries on next invocation
what logs are produced
what mail/output is generated
```

This is better than relying on generic memory.

Scheduler behavior around failed child processes is an important implementation detail.

---

## Anacron output

Like cron, anacron jobs can produce:

```text
stdout
stderr
exit status
```

Anacron may route output through local mail or inherited logging behavior depending on integration.

A production job should still define its own observability.

For example:

```bash
logger -t daily-maintenance \
    "event=start"
```

and:

```bash
logger -t daily-maintenance \
    "event=finish rc=0"
```

The scheduler should not be the only place where business execution state exists.

---

## Anacron logs

Search system journal:

```bash
journalctl | grep -i anacron
```

If a systemd service exists:

```bash
journalctl -u anacron.service
```

Traditional syslog may contain entries in:

```text
/var/log/syslog
/var/log/messages
```

depending on system configuration.

Look for messages such as:

```text
job started
job terminated
normal exit
execution delayed
```

Exact log format varies.

---

## Boot-time execution can fail because dependencies are not ready

Anacron may run shortly after boot.

The system may technically be online while dependencies are not ready.

Examples:

```text
network route not established
VPN not connected
NFS mount not available
database still recovering
DNS resolver not ready
cloud metadata unavailable
encrypted volume not mounted
```

A job that works during normal operation may fail only after reboot.

This is a classic timing problem.

---

## Check dependency readiness explicitly

Suppose a daily job requires:

```text
/mnt/archive
```

Do not assume boot completion means the mount exists.

Check:

```bash
mountpoint -q /mnt/archive || {
    echo "archive not mounted" >&2
    exit 1
}
```

For network:

```bash
getent hosts backup.internal >/dev/null || exit 1
```

For PostgreSQL:

```bash
pg_isready -q || exit 1
```

For a service:

```bash
systemctl is-active --quiet myservice || exit 1
```

Anacron's missed-job recovery makes boot dependency checks especially important.

---

## Bounded retry after boot

A job can wait for a transient dependency:

```bash
#!/bin/sh
set -u

attempt=1
max=10

while [ "$attempt" -le "$max" ]; do
    if /usr/bin/mountpoint -q /mnt/archive; then
        exec /opt/jobs/archive
    fi

    echo "archive not ready attempt=$attempt" >&2
    sleep 30
    attempt=$((attempt + 1))
done

echo "archive never became ready" >&2
exit 1
```

This gives the mount up to:

```text
5 minutes
```

to appear.

Do not retry forever.

A permanent dependency failure must become visible.

---

## Systemd dependency ordering may be better for boot-sensitive jobs

If a job must run:

```text
after network-online.target
after a specific mount
after PostgreSQL
```

a systemd service/timer can express dependencies more directly.

For example conceptually:

```text
After=network-online.target
RequiresMountsFor=/mnt/archive
```

Anacron itself is simpler.

When boot ordering becomes complex, use a scheduler/service manager that models dependencies.

---

## Laptop suspend versus power off

A machine may not shut down.

It may suspend for hours.

Traditional cron behavior around suspended intervals depends on implementation and the relationship between wall clock and daemon wake-up.

Do not assume every missed timestamp during suspend will be replayed.

If missed-period execution matters, anacron or a persistent timer model is more explicit.

Test the actual target environment.

---

## Hibernate

Hibernate preserves memory to disk but the machine is not actively executing tasks while hibernated.

When resumed, wall-clock time has advanced.

Again, exact cron behavior should not be used as a substitute for explicit catch-up semantics.

If the requirement is:

```text
run once each day the machine is used
```

anacron fits the intention better.

---

## Virtual machine snapshots and scheduler state

Suppose a VM snapshot contains:

```text
anacron timestamp = 2026-09-01
```

The VM is restored on:

```text
2026-09-20
```

Anacron sees the persisted old timestamp and may consider jobs overdue.

This is often correct.

But restore operations can create unexpected catch-up load.

After restoring an old snapshot, administrators should consider:

```text
scheduler state
database state
backup state
certificate state
package maintenance state
```

A snapshot restores more than application files.

It restores scheduler metadata too.

---

## Cloned machines can duplicate anacron work

Suppose a VM with anacron state is cloned into:

```text
node-a
node-b
```

Both machines may eventually run the same due job.

If the job sends:

```text
daily customer email
```

or modifies a shared database, duplication occurs.

Anacron is local-machine scheduling.

It does not provide cluster uniqueness.

Use application-level distributed coordination when clones or replicas exist.

---

## Anacron and cloud autoscaling

Ephemeral cloud instances are often unsuitable places for machine-local periodic business jobs.

A new instance may start with:

```text
empty or image-baked anacron state
```

and independently decide a task is due.

With autoscaling:

```text
5 instances
```

you may get five executions.

Use:

```text
cloud scheduler
queue
Kubernetes CronJob
designated scheduler service
database coordination
```

for fleet-wide tasks.

Anacron is strongest on one-machine periodic maintenance.

---

## Anacron is excellent for workstation maintenance

Examples:

```text
refresh local locate database
clean caches
rotate local artifacts
update local documentation index
perform local backup
check local integrity
```

The machine may not be online at fixed times.

One execution after it becomes available is often exactly what is wanted.

This was a primary historical use case for anacron.

---

## Anacron can run as root, but jobs do not all need root

System anacron often executes from a privileged context.

That does not mean every command should perform all work as root.

A wrapper can delegate:

```bash
/usr/sbin/runuser \
    -u report \
    -- \
    /opt/report/bin/daily
```

or schedule through a mechanism appropriate for the service account.

Exact architecture depends on distribution and security policy.

The same least-privilege rules apply as with cron.

---

## Do not run user-controlled code from system anacron

Bad:

```text
1 5 user-maintenance /home/alice/maintenance.sh
```

inside a root-controlled anacrontab if the command executes as root.

If Alice controls the script, Alice controls root code.

Use:

```text
root-owned wrapper
or
execution under Alice's UID
```

depending on the intended authority.

Anacron changes timing semantics, not permission semantics.

---

## Daily scripts remain subject to run-parts rules

If anacron executes:

```bash
run-parts /etc/cron.daily
```

then filename and executable rules still apply.

A file:

```text
/etc/cron.daily/backup.sh
```

may be skipped by strict `run-parts` filename rules.

Anacron can be working correctly while the intended script is never selected.

Test:

```bash
run-parts --test /etc/cron.daily
```

This isolates:

```text
anacron due decision
```

from:

```text
run-parts file selection
```

Two different layers can fail independently.

---

## Debug by layers

For a daily job that did not run:

Check anacron availability:

```bash
command -v anacron
```

Check configuration:

```bash
cat /etc/anacrontab
```

Check timestamp:

```bash
ls -l /var/spool/anacron
cat /var/spool/anacron/cron.daily
```

Check invocation source:

```bash
grep -R -n "anacron" \
    /etc/crontab \
    /etc/cron.d \
    2>/dev/null
```

Check service/timer:

```bash
systemctl list-timers --all | grep -i anacron
```

Check logs:

```bash
journalctl | grep -i anacron
```

Check run-parts:

```bash
run-parts --test /etc/cron.daily
```

Check script directly:

```bash
sudo /etc/cron.daily/myjob
```

This layered approach prevents blaming anacron for a run-parts or application failure.

---

## Timestamp manipulation is a useful lab tool, but risky in production

Anacron determines eligibility from timestamp state.

In a lab, changing or removing a timestamp can make a job appear overdue.

On production systems, manually editing:

```text
/var/spool/anacron/*
```

can cause unexpected job execution.

Before touching state:

```bash
sudo cp -a \
    /var/spool/anacron \
    /var/spool/anacron.backup
```

but even a backup copy inside the same directory may interact with tools if names are interpreted unexpectedly.

The safest practice is to test on a disposable VM or isolated config/spool directory if supported.

---

## Do not change the system clock to test missed jobs

Changing:

```bash
date
```

or:

```bash
timedatectl set-time ...
```

on a production system can break:

```text
TLS
database timestamps
authentication
Kerberos
distributed leases
monitoring
log order
certificate validity
cluster behavior
```

Use scheduler state or a test environment instead.

Time travel is not a safe cron/anacron test technique on live infrastructure.

---

## Monthly semantics

A monthly maintenance job may mean:

```text
once each calendar month
```

Anacron's:

```text
@monthly
```

may model that more accurately than:

```text
30
```

because:

```text
February
March
April
```

do not all contain 30 days.

But if the business requirement is:

```text
run on the first business day
```

neither simple anacron nor a naive monthly cron expression fully captures the requirement.

The application or scheduler must encode business-calendar logic.

---

## Weekly semantics

A period:

```text
7
```

means approximately every seven days.

That is not necessarily the same as:

```text
every Sunday
```

Traditional cron:

```cron
0 3 * * 0 ...
```

is calendar-weekday scheduling.

Anacron:

```text
7 days
```

is period-based catch-up scheduling.

Choose based on whether:

```text
weekday identity
```

or:

```text
frequency
```

matters.

---

## Daily semantics and timezone

Anacron's day calculations depend on local system date behavior and implementation.

Timezone changes can affect:

```text
which date is considered current
```

especially around travel or server timezone reconfiguration.

For local workstation maintenance, this is usually acceptable.

For financial or global business workflows, use explicit application time semantics.

Scheduler "daily" is not a complete business-time definition.

---

## DST and anacron

Traditional cron has complicated behavior around daylight-saving transitions because specific local times may:

```text
not exist
or
occur twice
```

Anacron's day-based model is less tied to a specific clock minute.

That can make it more resilient for:

```text
once-per-day maintenance
```

when exact wall-clock time is unimportant.

However, if START_HOURS_RANGE or other clock-window features are used, timezone transitions can still affect execution timing.

---

## Missed execution and system uptime

If a traditional daily cron job is scheduled at:

```text
03:00
```

and the server reboots at:

```text
03:02
```

the job may be missed.

If that job is critical, relying on:

```text
server is usually up
```

is weak.

Servers reboot for:

```text
kernel updates
power events
hypervisor maintenance
cloud migration
administrator action
hardware faults
```

Critical daily work should have explicit catch-up semantics.

---

## Anacron is not necessarily required on always-on servers

For a continuously running server:

```text
traditional cron
```

may be perfectly appropriate.

Adding anacron can complicate:

```text
execution timing
boot load
monitoring
scheduler topology
```

Use it because the problem requires catch-up behavior, not because it sounds more reliable in general.

---

## Systemd persistent timers

Modern systemd timers can provide missed-run behavior through:

```text
Persistent=true
```

Conceptually:

```text
timer should have run while system was down
system boots
systemd notices missed activation
service runs after boot
```

This overlaps with anacron's use case.

Systemd timers also offer:

```text
dependency ordering
random delay
accuracy windows
resource controls
service isolation
journal integration
```

On modern distributions, a systemd timer may be a cleaner choice for new system jobs.

Anacron remains important because:

```text
existing distributions use it
legacy systems depend on it
cron.daily integration often relies on it
portable Unix administration still encounters it
```

---

## Anacron versus `Persistent=true`

Conceptually:

```text
anacron:
    persistent timestamp per periodic job
    day-oriented period model
    delay after scheduler invocation

systemd timer:
    timer unit state
    calendar/monotonic expressions
    optional persistent catch-up
    service-unit integration
```

Both can recover missed work.

Systemd provides more native service lifecycle control.

Anacron is simpler and closely integrated with traditional cron directory conventions.

---

## Do not run the same job under anacron and a persistent timer

A migration may leave:

```text
/etc/cron.daily/company-job
```

active through anacron and also create:

```text
company-job.timer
```

Now the job can run twice.

Search:

```bash
grep -R -n "company-job" \
    /etc/cron* \
    /etc/anacrontab \
    2>/dev/null
```

and:

```bash
systemctl list-timers --all | grep -i company-job
```

Whenever changing scheduling systems, disable the old source explicitly.

---

## Migrate with overlap awareness

A safe migration process:

```text
identify old scheduler
identify execution user
identify missed-run semantics
identify lock behavior
identify logging
identify timezone
identify last-success monitoring
deploy new scheduler disabled
validate configuration
disable old scheduler
enable new scheduler
observe first execution
```

Do not simply:

```text
create timer
leave cron entry
```

and assume one will dominate.

---

## Backfill logic belongs in the application when exact periods matter

Suppose the application must produce one report per calendar day.

Scheduler trigger:

```text
run whenever machine is available
```

Application:

```text
read last completed date
calculate every missing date
generate each report
commit progress
```

Now the system survives:

```text
downtime
missed cron
anacron delay
manual rerun
scheduler migration
```

This architecture is stronger than tying data correctness to one scheduler implementation.

---

## Example: daily local backup on a laptop

Requirement:

```text
perform one backup each day the laptop is used
if laptop was off overnight, backup after next boot
exact time not important
avoid duplicate concurrent backups
```

Anacron is a good fit.

A root-controlled job:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

LOCK=/run/laptop-backup.lock
DEST=/mnt/backup

exec 9>"$LOCK"

if ! flock -n 9; then
    logger -t laptop-backup \
        "event=skip reason=already_running"
    exit 0
fi

if ! mountpoint -q "$DEST"; then
    logger -p user.err -t laptop-backup \
        "event=fail reason=destination_not_mounted"
    exit 1
fi

logger -t laptop-backup \
    "event=start"

/usr/bin/rsync \
    -a \
    --delete \
    /home/ \
    "$DEST/home/"

date -Is \
    > /var/lib/laptop-backup/last-success.tmp

mv \
    /var/lib/laptop-backup/last-success.tmp \
    /var/lib/laptop-backup/last-success

logger -t laptop-backup \
    "event=finish rc=0"
```

Anacron entry conceptually:

```text
1 15 laptop-backup /usr/local/sbin/laptop-backup
```

The 15-minute delay allows boot to settle.

The script itself verifies the destination mount.

Locking prevents overlap.

The `last-success` file provides business-level monitoring.

---

## Example: weekly search index rebuild

Requirement:

```text
approximately once per week
exact weekday does not matter
one execution after long downtime is sufficient
```

Anacron entry:

```text
7 30 search-reindex /opt/search/bin/reindex
```

This is an excellent anacron workload because the current index can be rebuilt once.

If the machine was off for two weeks, replaying two historical index builds has no value.

---

## Example: monthly billing is a poor naive anacron job

Requirement:

```text
issue customer invoices exactly once for each accounting month
```

A naive entry:

```text
@monthly 10 billing /opt/billing/run
```

does not itself guarantee:

```text
one invoice set per business month
no duplicates
correct month after long downtime
```

The application should accept or calculate a billing period and enforce uniqueness.

For example:

```text
billing_period = 2026-08
unique constraint on billing_period/customer
```

Anacron can trigger catch-up.

The billing system must own accounting correctness.

---

## Example: package maintenance

Periodic package tasks often fit anacron because:

```text
exact minute is unimportant
one current maintenance run is enough
missed execution should happen after machine returns
```

This is why desktop-oriented Linux distributions historically use anacron for daily/weekly maintenance.

But modern package tools may use systemd timers instead.

Inspect the actual package architecture before adding custom jobs.

---

## Example: log rotation

Traditional log rotation may be triggered by:

```text
cron.daily
anacron
systemd timer
```

The rotation tool itself uses state to determine which logs are due.

This creates two layers of periodic state:

```text
scheduler decides when to invoke logrotate
logrotate decides which files need rotation
```

That is a good example of robust architecture.

The scheduler does not need perfect exact timing because the application maintains its own rotation state.

---

## Application state makes scheduling less fragile

A mature periodic application often tracks:

```text
last successful run
last processed item
last processed period
state version
checkpoint
```

Then the scheduler only needs to provide opportunities to make progress.

This makes the system tolerant of:

```text
missed runs
late runs
duplicate runs
manual runs
scheduler changes
```

Anacron embodies this philosophy at the scheduler level for coarse periodic jobs.

---

## Catch-up does not mean urgency

An overdue daily cleanup may be fine to execute:

```text
30 minutes after boot
```

An overdue security revocation update may be more urgent.

Different missed jobs deserve different delay policies.

Do not assign delay values mechanically.

Consider:

```text
business urgency
boot resource load
dependency readiness
network availability
user experience
```

---

## Desktop user experience

A laptop user boots at 09:00 for a meeting.

If anacron immediately starts:

```text
filesystem scan
package update
backup compression
index rebuild
```

the machine may become slow exactly when the user needs it.

Delay and nice/ionice settings can reduce impact.

Example:

```bash
/usr/bin/nice -n 10 \
    /usr/bin/ionice -c2 -n7 \
    /opt/jobs/reindex
```

when supported.

This is a performance policy, not a correctness mechanism.

---

## Battery-aware scheduling

Anacron itself may not know whether a laptop is:

```text
on battery
charging
low power
```

A job can check.

On systems exposing power information:

```bash
upower -e
```

or:

```text
/sys/class/power_supply/
```

can provide state.

A heavy backup might choose:

```text
if on battery -> exit or delay
```

But beware of the scheduler timestamp issue.

If the job exits as "skipped" and anacron records it as run, it may not retry that day.

A better architecture may use:

```text
systemd timer with conditions
desktop scheduler
application retry state
```

if power-aware catch-up is important.

---

## Network-aware scheduling

A laptop backup may require corporate VPN.

At boot:

```text
network exists
VPN does not
```

Anacron starts the job.

It fails.

If the scheduler does not retry soon, backup remains missed.

The application can:

```text
detect VPN absence
record pending state
retry later
```

or a different event-driven mechanism can run when VPN becomes available.

Anacron is time-oriented, not event-oriented.

---

## Event-driven scheduling can be better than periodic catch-up

Requirement:

```text
run backup whenever external drive is connected
```

This is not primarily a cron/anacron problem.

Better triggers may include:

```text
udev
systemd device units
desktop event service
mount hooks
```

Similarly:

```text
run sync when network comes online
```

may be better modeled through network/service events.

Use anacron when the requirement is periodic.

Use event-driven mechanisms when the requirement is event-based.

---

## Monitoring missed periodic work

A scheduler can say:

```text
job was invoked
```

Monitoring should ask:

```text
when did the job last succeed?
```

For a daily job:

```bash
stat -c %Y /var/lib/job/last-success
```

Current epoch:

```bash
date +%s
```

Calculate age:

```bash
last=$(stat -c %Y /var/lib/job/last-success)
now=$(date +%s)
age=$((now - last))
```

Alert if:

```text
age > 36 hours
```

rather than expecting an exact daily timestamp.

This works naturally with anacron's flexible execution timing.

---

## Success freshness window

If a job is expected once per day, a monitoring threshold might be:

```text
26 hours
36 hours
48 hours
```

depending on allowed delays.

Do not alert at:

```text
24h + 1 second
```

if the scheduler intentionally allows random delay and boot-dependent execution.

Monitoring thresholds should match scheduler semantics.

---

## Distinguish due, late, and failed

A useful state model:

```text
not due
due
late
running
succeeded
failed
```

Anacron internally deals mostly with:

```text
due based on period
```

Application monitoring can define:

```text
late if no success for 36h
```

This provides operational clarity.

---

## Reboot immediately after anacron starts a job

Suppose anacron launches a daily job.

The machine loses power one minute later.

Depending on when timestamp state is recorded, the scheduler may consider the job executed even though it never completed.

This is another reason application-level success state matters.

Test implementation behavior if the distinction is critical.

---

## Durable success markers

Write only after complete success:

```bash
tmp=$(mktemp /var/lib/job/.success.XXXXXX)

date -Is > "$tmp"

mv "$tmp" /var/lib/job/last-success
```

Monitoring reads:

```text
last-success
```

If the process crashes before the final rename, the previous success timestamp remains.

That accurately reflects reality.

---

## Do not write last-success at process start

Bad:

```bash
date -Is > /var/lib/job/last-success
do_work
```

A crash leaves false success.

Use:

```text
last-start
```

separately if you want to observe starts.

Then:

```text
last-success
```

only after completion.

---

## A practical status directory

Example:

```text
/var/lib/company-job/
├── last-start
├── last-success
└── last-failure
```

Start:

```bash
date -Is > "$STATE/last-start.tmp"
mv "$STATE/last-start.tmp" "$STATE/last-start"
```

Success:

```bash
date -Is > "$STATE/last-success.tmp"
mv "$STATE/last-success.tmp" "$STATE/last-success"
```

Failure:

```bash
printf 'date=%s\nrc=%s\n' \
    "$(date -Is)" \
    "$rc" \
    > "$STATE/last-failure.tmp"

mv \
    "$STATE/last-failure.tmp" \
    "$STATE/last-failure"
```

This is independent of anacron's own spool state.

---

## Manual execution and anacron state

Running:

```bash
/opt/jobs/daily
```

manually may not update anacron's timestamp.

Therefore anacron can run it again later because, from the scheduler's perspective, the job is still due.

If manual runs should satisfy the periodic requirement, use an application-level idempotency model or deliberately update scheduler state using supported mechanisms.

Do not edit timestamps casually.

---

## Manual run should usually be safe to repeat

This is another argument for idempotency.

If an operator runs the daily maintenance manually and anacron later runs it again, the system should remain correct.

Design:

```text
safe repeated cleanup
atomic report replacement
unique database keys
checkpointed processing
```

rather than relying on perfect scheduler coordination.

---

## Multiple anacron invocations

What if anacron itself is started twice?

Implementations typically provide mechanisms to avoid unsafe duplicate handling, but exact behavior should be verified.

Do not assume:

```text
starting two anacron processes can never matter
```

Use the distribution-provided service/timer integration rather than custom parallel launch scripts.

---

## Do not wrap anacron itself in naive cron entries without understanding defaults

An administrator may create:

```cron
* * * * * root /usr/sbin/anacron
```

thinking:

```text
more frequent checks are safer
```

This can be unnecessary and may interact poorly with the distribution's own scheduler integration.

First inspect:

```text
existing cron entries
systemd units
timers
package defaults
```

Avoid duplicate scheduler orchestration.

---

## Distribution behavior differs

Debian-family, Ubuntu, Fedora/RHEL-family, Arch-derived systems, and minimal containers can have very different combinations of:

```text
cron
cronie
anacron
systemd timers
run-parts
package maintenance
```

A command available on one system may not exist on another.

Never write documentation such as:

```text
Linux always stores anacron timestamps here
```

without qualification.

Prefer:

```text
commonly
typically
on this implementation
verify with man page
```

Scientific documentation distinguishes model from implementation detail.

---

## Identify the installed package

Debian-family:

```bash
dpkg -S "$(command -v anacron)"
```

or:

```bash
dpkg -l | grep -i anacron
```

RPM-family:

```bash
rpm -qf "$(command -v anacron)"
```

Package metadata can tell you which implementation and version the system uses.

Then:

```bash
man anacron
```

becomes authoritative for local behavior.

---

## Containers usually do not need anacron internally

Containers are often:

```text
ephemeral
single-purpose
orchestrated
```

Running:

```text
cron + anacron + app
```

inside one container increases complexity.

Use:

```text
host scheduler
Kubernetes CronJob
cloud scheduler
application queue
```

where possible.

Anacron makes the most sense for a machine whose uptime itself is intermittent.

An ephemeral container is a different lifecycle model.

---

## WSL and developer environments

Developer Linux environments such as WSL may not run continuously and may not start background services the same way as a normal booted Linux host.

Anacron can conceptually fit missed periodic maintenance, but integration depends heavily on:

```text
service manager availability
distribution configuration
session lifecycle
host shutdown behavior
```

Test actual process startup rather than assuming normal server boot semantics.

---

## Embedded systems

An edge device may:

```text
boot intermittently
have unreliable connectivity
use flash storage
have limited CPU
```

Anacron-style periodic catch-up can be useful.

But write frequency to persistent spool state matters on flash-based systems.

For very constrained devices, a custom scheduler or systemd timer may be more appropriate.

Architecture should reflect hardware constraints.

---

## Security implications of delayed execution

A job intended for a quiet night period may execute after morning boot.

Example:

```text
security scan
backup
filesystem cleanup
```

Now it runs while users are active.

This changes:

```text
resource contention
data consistency
user-visible effects
```

Missed-run recovery is useful, but delayed context must be safe.

---

## Maintenance tasks should tolerate different execution times

An anacron-suitable job should not assume:

```text
it is exactly 03:00
```

Bad:

```bash
YESTERDAY=$(date -d yesterday +%F)
process "$YESTERDAY"
```

If the machine was off for three days and the job runs after boot, this processes only yesterday.

Maybe that is wrong.

Stronger:

```text
query last completed date
process every missing date
```

This makes catch-up semantics explicit.

---

## Date-derived filenames can create missed data

Suppose backup:

```bash
FILE="/backup/$(date +%F).tar.gz"
```

If an overdue daily job runs at boot on:

```text
2026-09-13
```

it creates:

```text
2026-09-13.tar.gz
```

even if the missed intended run was for:

```text
2026-09-12
```

Whether this is correct depends on what the date represents.

If it means:

```text
backup creation date
```

fine.

If it means:

```text
business period represented by backup
```

the application needs explicit period logic.

---

## Anacron does not create historical time context

When a missed job runs now, the process sees:

```text
current date
current time
current environment
current files
```

It is not magically executed "as if" it were yesterday at 03:00.

This is crucial.

Catch-up means:

```text
perform work now because work was missed
```

not:

```text
recreate historical machine state
```

Applications that depend on historical periods must model those periods explicitly.

---

## Example: missed log summary

A daily log summary script does:

```bash
journalctl --since yesterday
```

If it misses three days and anacron runs once, it summarizes only a limited current window.

A stronger application tracks:

```text
last summarized timestamp
```

then queries:

```text
from last checkpoint
to current cutoff
```

Anacron provides an opportunity to run.

The application provides correct historical coverage.

---

## Example: security signature update

Job:

```bash
update-signatures
```

If missed for five days, one current update is normally enough because the newest signature set supersedes older ones.

This is ideal anacron behavior.

No replay of each missed day is necessary.

---

## Example: rotating a cache

Job:

```bash
rebuild-current-cache
```

If missed three times, one rebuild now is enough.

Again, ideal reconciliation workload.

---

## Example: daily export archive

If the requirement is:

```text
one archive representing every calendar day
```

anacron alone is insufficient.

Use:

```text
last exported date
current date
loop over missing dates
```

The scheduler can trigger the catch-up process.

---

## Anacron and lock policy

For reconciliation jobs:

```text
non-blocking lock
skip duplicate invocation
```

is often appropriate.

For jobs that must eventually run once after outage:

```text
if lock conflict -> scheduler timestamp may advance
```

could accidentally lose the work opportunity.

A better pattern is for the application to maintain its own pending state.

Or use a blocking lock with bounded waiting if execution must not be discarded.

Choose policy based on how anacron records attempted execution.

---

## Scheduler state is not transaction state

Do not use:

```text
anacron timestamp
```

as a substitute for:

```text
database transaction completion
```

The timestamp is scheduler bookkeeping.

Business state belongs in the application or data store.

This separation prevents many false-success assumptions.

---

## A missed-job-safe application pattern

Pseudo-workflow:

```text
read last_successful_period

determine all required periods up to current cutoff

for each missing period:
    acquire application-level idempotency key
    process period
    validate output
    mark period complete transactionally

update last-success
```

Now the scheduler can be:

```text
cron
anacron
systemd timer
manual execution
cloud scheduler
```

without changing business correctness.

This is the strongest long-term architecture.

---

## Anacron troubleshooting checklist

When a job does not execute:

Check binary:

```bash
command -v anacron
```

Check package/service:

```bash
systemctl status anacron 2>/dev/null
```

Check config:

```bash
cat /etc/anacrontab
```

Validate syntax using supported test mode.

Check state:

```bash
ls -l /var/spool/anacron
```

Check logs:

```bash
journalctl | grep -i anacron
```

Check periodic directory discovery:

```bash
run-parts --test /etc/cron.daily
```

Run target manually:

```bash
sudo /etc/cron.daily/job
```

Check dependency readiness.

Check application last-success state.

Check duplicate scheduler definitions.

This workflow follows the full chain.

---

## A boot-time diagnostic wrapper

Example:

```bash
#!/usr/bin/env bash
set -u

LOG=/var/log/anacron-debug.log

{
    echo "=== $(date -Is) ==="
    echo "uptime=$(cut -d. -f1 /proc/uptime)"
    echo "user=$(id -un)"
    echo "uid=$(id -u)"
    echo "pwd=$PWD"
    echo "path=$PATH"
    echo
    echo "--- mounts ---"
    mount
    echo
    echo "--- network ---"
    ip addr
    echo
} >> "$LOG" 2>&1
```

Use only temporarily because:

```text
mount output
network addresses
environment
```

may contain sensitive operational information.

The purpose is to identify what system state looks like when an overdue job starts shortly after boot.

---

## Measure boot age

Anacron jobs may start too early for dependencies.

Check uptime:

```bash
cut -d. -f1 /proc/uptime
```

or:

```bash
uptime -s
```

A log line:

```text
uptime_seconds=312
```

shows the job began about five minutes after boot.

This can correlate dependency failures with startup timing.

---

## Network-online is not always application-ready

Even after:

```text
network-online.target
```

the following may not be ready:

```text
VPN
DNS search domains
remote API
database
NFS
cloud identity
```

Application-specific readiness checks remain valuable.

Anacron's delay helps but cannot know every dependency.

---

## A practical missed-job architecture for backups

A robust daily backup design can use:

```text
anacron for opportunity
application state for success
flock for local non-overlap
mountpoint check for destination
timestamped immutable backup files
retention logic
monitoring for last-success age
```

Example state:

```text
/var/lib/company-backup/last-success
```

Monitoring:

```bash
last=$(stat -c %Y /var/lib/company-backup/last-success)
now=$(date +%s)

if [ $((now-last)) -gt 129600 ]; then
    echo "backup stale" >&2
    exit 1
fi
```

`129600` seconds is 36 hours.

This tolerates anacron delay while detecting real missed work.

---

## Do not couple retention to exact run count

A backup cleanup such as:

```bash
ls -1t /backup/*.tar.gz | tail -n +8 | xargs rm
```

assumes one backup per expected period.

If downtime creates fewer backups, retention semantics become confusing.

A time-based cleanup may better match:

```bash
find /backup \
    -type f \
    -name '*.tar.gz' \
    -mtime +30 \
    -delete
```

The right policy depends on whether retention is:

```text
last N copies
last N days
monthly snapshots
```

Catch-up scheduling makes these distinctions visible.

---

## A practical workstation cleanup

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

exec 9>/run/user-cache-clean.lock

if ! flock -n 9; then
    exit 0
fi

/usr/bin/find \
    /var/tmp/company-cache \
    -type f \
    -mtime +14 \
    -delete

/usr/bin/logger -t cache-clean \
    "event=finish rc=0"
```

Anacron:

```text
1 20 cache-clean /usr/local/sbin/cache-clean
```

If machine is off for three days, one cleanup after boot is enough.

This is a clean, simple anacron workload.

---

## A practical weekly integrity scan

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

exec 9>/run/integrity-scan.lock

if ! flock -n 9; then
    logger -t integrity-scan \
        "event=skip reason=already_running"
    exit 0
fi

/usr/bin/nice -n 10 \
    /usr/local/bin/integrity-scan \
    --config /etc/integrity-scan.conf
```

Anacron:

```text
7 45 integrity-scan /usr/local/sbin/run-integrity-scan
```

The delay avoids running heavy scanning immediately after boot.

Exact weekday is unimportant.

This is a strong fit.

---

## A practical monthly archive

Requirement:

```text
once per calendar month
```

Anacron conceptually:

```text
@monthly 60 archive /usr/local/sbin/monthly-archive
```

Application should still create a unique month key:

```text
2026-09
```

and refuse duplicate publication if already completed.

Now manual reruns and scheduler duplication remain safe.

---

## When to use cron instead of anacron

Use traditional cron when:

```text
machine is reliably online
exact time matters
weekday matters
multiple executions per day are needed
sub-day cadence is required
```

Examples:

```cron
0 9 * * 1 /opt/jobs/monday-report
*/10 * * * * /opt/jobs/health-check
0 0 1 * * /opt/jobs/month-start
```

Anacron's day-oriented model is not intended for these cases.

---

## When to use anacron

Use anacron when:

```text
machine may be offline at scheduled times
job needs daily/weekly/monthly frequency
exact execution time is flexible
one catch-up run is sufficient
```

Examples:

```text
cache cleanup
local backup
index refresh
local security scan
package housekeeping
```

---

## When to use systemd timers

Systemd timers are attractive when:

```text
Linux system already uses systemd
missed-run persistence is needed
service dependencies matter
resource limits matter
sandboxing matters
logging through journal is desired
randomized delay is useful
```

A timer can model both calendar and monotonic schedules.

For new Linux-only system automation, this is often worth considering.

---

## When to use an application scheduler

Use an application scheduler or queue when:

```text
business periods matter
retries matter
distributed locking matters
per-task history matters
concurrency limits matter
every event must be preserved
jobs run across multiple hosts
```

Cron/anacron can trigger the application scheduler.

They should not become the business workflow database.

---

## A scientific way to think about missed execution

Define the required guarantee.

Possible guarantees:

```text
best effort at exact time
at least one attempt per day
one successful completion per day
every calendar period processed
no concurrent execution
eventually completed after downtime
exactly-once external effect
```

These are different guarantees.

Cron provides roughly:

```text
attempt when time expression matches and daemon/system is available
```

Anacron provides roughly:

```text
attempt overdue coarse-period work after system becomes available
```

Neither alone provides:

```text
business exactly-once
```

That must come from application state and data semantics.

---

## Failure mode analysis

Take a daily job and ask:

```text
What if machine is off?
What if machine boots late?
What if job starts and crashes?
What if machine reboots mid-job?
What if dependency is unavailable after boot?
What if job runs twice?
What if manual execution occurs?
What if scheduler timestamp updates but application fails?
What if multiple machines have same job?
What if the machine clock changes?
```

A reliable design has an answer for each.

This is much stronger than simply choosing:

```text
cron or anacron
```

based on habit.

---

## Anacron makes uptime assumptions explicit

The real benefit of understanding anacron is not memorizing `/etc/anacrontab`.

It forces the administrator to ask:

```text
Does this job depend on the machine being online at one exact moment?
```

If yes, the system has an uptime dependency.

Sometimes that is acceptable.

Sometimes the scheduler should compensate.

Sometimes the application should catch up.

Sometimes the architecture should move to a central scheduler.

The correct choice begins with recognizing the assumption.

---

## Final perspective

Traditional cron and anacron solve different classes of time.

Cron is based on matching the current clock and calendar.

Anacron is based on whether coarse periodic work is overdue.

Cron asks:

```text
Is it 02:30 now?
```

Anacron asks:

```text
Has this daily job been serviced recently enough?
```

That is why anacron is valuable on machines that are not continuously available.

It provides missed-period recovery for jobs where one current execution is enough.

But anacron is not a historical replay engine.

It does not reconstruct every missed cron tick.

It does not guarantee business success.

It does not solve concurrency.

It does not solve distributed uniqueness.

It does not replace application checkpoints.

It simply adds persistent periodic state to scheduling.

A reliable missed-job design therefore combines the right layers:

```text
anacron or persistent timer
    for catch-up opportunity

application success state
    for proof of completion

flock or another lock
    for concurrency

idempotency
    for safe retries

validation
    for business correctness

monitoring
    for stale success detection
```

Once those responsibilities are separated, anacron becomes easy to reason about.

The scheduler decides that work is overdue.

The application decides how to perform that work safely.

That separation is what makes periodic automation resilient to downtime.
